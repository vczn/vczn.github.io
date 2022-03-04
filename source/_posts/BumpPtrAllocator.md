---
title: BumpPtrAllocator
date: 2022-02-18 20:18:13
tags: 
 - LLVM
 - Clang
 - Allocator
categories:
 - Clang
---

`llvm::BumpPtrAllocator` 源码分析

<!-- more -->



# 一、概述

最近开始捣鼓一个玩具 C 语言编译器项目，打算不借助编译器相关的库，写一个基本完整的编译器。

为了对各种类型、声明、语句、表达式等大量的较小对象快速分配内存，打算写一个内存池分配器来解决。毕竟不能闭门造车，这里参考 Clang 的解决方案，使用了 bump-pointer allocator 来分配这些对象的内存。

之前没有听说过 bump-pointer allocator，在网上搜索了一番，找到 [CS107 Lecture14](https://web.stanford.edu/class/archive/cs/cs107/cs107.1222/lectures/14/Lecture14.pdf) 这个 PPT，对 bump-pointer allocator 有了一个大概的认知。bump-pointer allocator 仅在 allocate 时分配下一个可用内存地址，在 deallocate 时不执行任何操作。在 bump-pointer allocator 生命期结束时，回收所有分配出去的内存。

接下来对 `llvm::BumpPtrAllocator` 进行源码分析。



# 二、源码分析

下文代码主要参考 llvm 12.0.0 版本的代码。在 `include/llvm/Support/Allocator.h` 中，可以找到 `llvm::BumpPtrAllocator` 的定义：

```cpp
typedef BumpPtrAllocatorImpl<> BumpPtrAllocator;
```

接下来跳转到 `BumpPtrAllocatorImpl` 中，首先列出一些接口方法以及主要成员变量：

```cpp
template <typename AllocatorT = MallocAllocator, size_t SlabSize = 4096,
          size_t SizeThreshold = SlabSize, size_t GrowthDelay = 128>
class BumpPtrAllocatorImpl
    : public AllocatorBase<BumpPtrAllocatorImpl<AllocatorT, SlabSize,
                                                SizeThreshold, GrowthDelay>>,
      private AllocatorT {
public:
  BumpPtrAllocatprImpl() = default;
  ~BumpPtrAllocatprImpl();
  
  void *Allocate(size_t Size, Align Aligment);
  void Deallocate(const void *Ptr, size_t Size, size_t /*Alignment*/);
private:
//                                 Slabs
//  +-----------+-----------+-----------+------------------------------------
//  |   slab0   |   slab1   |   slab2   |                  ...    
//  +-----------+-----------+-----------+------------------------------------
//                          ^           ^
//                          |           |
//                         CurPtr      End
//
//                              CustomSizedSlabs:
// +-----------------------------+-----------+-------------------------------
// |         CSSlab0             |  CSSlab1  |               ...
// +-----------------------------+-----------+-------------------------------

  char *CurPtr = nullptr; // 指向下一个空闲字节的指针
  char *End = nullptr;    // 指向当前 Slab 的 End 指针
  // 普通的 Slabs
  SmallVector<void *, 4> Slabs;
  // 自定义大小的 slabs 为了满足过大的(SizeThreshold)内存分配请求
  SmallVector<std::pair<void *, size_t>, 0> CustomSizedSlabs;
  size_t BytesAllocated = 0;    // 已经分配了多少字节的内存

  // 运行在 sanitizer 下在内存分配之间额外放置多少字节，虽然下面的代码剖析不会涉及它
  // sanitizer 全称是 AddressSanitizer(aka. ASan)，是内存错误的检测器，可以检测
  // 内存越界、释放后使用、使用无效栈地址、多次释放、内存泄漏等内存问题
  // 具体见 https://clang.llvm.org/docs/AddressSanitizer.html
  size_t RedZoneSize = 1;
}
```



先从简单的开始看，因为在 `BumpPtrAllocator` 生命周期内永远不会释放内存，所以 `Deallocate` 什么都不做：

```cpp
// 因为排版原因和上面的代码拆分开写，实际在模板类中实现
void Deallocate(const void *Ptr, size_t Size, size_t /*Alignment*/) {
  // Do nothing
}
```



接下来看 `Allocate` 方法：

```cpp
// 同样因排版原因，拆分到这里
void *Allocate(size_t Size, Align Aligment) {
  BytesAllocated += Size;  // 跟踪已经分配了多少字节

  // 计算内存对齐需要的字节数
  size_t Adjustment = offsetToAlignedAddr(CurPtr, Aligment);
  size_t SizeToAllocate = Size;

  // 如果当前的 Slab 有足够内存，直接划分地址空间并返回
  if (Adjustment + SizeToAllocate <= size_t(End - CurPtr)) {
    char *AlignedPtr = CurPtr + Adjustment;
    CurPtr = AlignedPtr + SizeToAllocate;
    return AlignedPtr;
  }

  // 当前 slab 没有足够的字节数，需要使用新的 Slab
  // 如果请求的内存字节数过大，不适合存放到普通 Slabs 中，所以使用自定义大小的 Slabs
  size_t PaddedSize = SizeToAllocate + Alignment.value() - 1;
  if (PaddedSize > SizeThreshold) {
    void *NewSlab = AllocateT::Allocate(PaddedSize, alignof(std::max_align_t));
    CustomSizedSlabs.push_back(std::make_pair(NewSlab, PaddedSize));

    uintptr_t AlignedAddr = alignAddr(NewSlab, Alignment);
    char *AlignedPtr = (char*)AlignedAddr;
    return AlignedPtr;
  }

  // 否则，启用一个新的 Slab 并划分内存空间并返回
  StartNewSlab();
  uintptr_t AlignedAddr = alignAddr(NewSlab, Alignment);
  char *AlignedPtr = (char*)AlignedAddr;
  CurPtr = AlignedPtr + SizeToAllocate;
  return AlignedPtr;
}

// 启用新的 Slab
// 分配一块新的 Slab，注册到 Slabs 中，重置 CurPtr 和 End 指针即可
void StartNewSlab() {
  size_t AllocatedSlabSize = computeSlabSize(Slabs.size());
  void *NewSlab =
      AllocatorT::Allocate(AllocatedSlabSize, alignof(std::max_align_t));
  Slabs.push_back(NewSlab);
  CurPtr = (char *)(NewSlab);
  End = ((char *)NewSlab) + AllocatedSlabSize;
}
```



最后看一下析构函数：

```cpp
// 被分配的内存的生命周期和 BumpPtrAllocator 的声明周期一致
// 析构函数释放注册到 Slabs 和 CustomSizedSlabs 中的内存
~BumpPtrAllocatprImpl() {
  DeallocateSlabs(Slabs.begin(), Slabs.end());
  DeallocateCustomSizedSlabs();
}
```



实现起来非常简单，虽然浪费了一些内存，但是速度非常快。对于我们的场景使用 `BumpPtrAllocator` 非常合适。
