---
title: C++ memory order
date: 2021-01-23 16:19:11
tags:
- C++
- Arch
- Atomic
categories:
- Atomic
---

从程序员使用的角度介绍 6 种 memory order

<!-- more -->


# 一、概述

在 C++ 中，对 `std::atomic<T>`  的访问可以建立线程间同步，对内存的“读-修改-写”是一个原子操作，而且按照 `std::memory_order` 指定的 memory order 对非原子内存访问定序  

C++ 标准提供了 6 种 memory order

```cpp
typedef enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
} memory_order;
```

第一次看到这些概念肯定会云里雾里，因为在现代计算机中，为了提高执行速度，CPU 中增加了 cache，store buffer，有了乱序执行，出现了编译器优化(reorder)，多 CPU 以及多线程等等，就使得问题变得非常复杂。

考虑下面简单的代码

```cpp
extern int avalue;
extern int bvalue;

void setValues(int a, int b) {
    avalue = a;  // 1
    bvalue = b;  // 2
}

void getValues(int& a, int& b) {
    a = avalue; // 3
    b = bvalue; // 4
}
```

因为 1 和 2 没有依赖关系，有的编译器可能把 2 优化到 1 的前面，而且 CPU 的乱序执行也有可能先执行完 2，后执行完 1；还有如果 `setValues` 和 `getValues` 分别在两个线程，能够获取到的 a 和 b 值也无法确定

首先记住一个前提，memory order 并**不是限制多线程的执行顺序**，而且**规定一个线程内的访问共享内存指令如何执行**，其中的共享内存更确切应该叫做共享变量，比如全局变量等等

从程序员使用的角度来分析以下的四种情况：

<br>



# 二、release/acquire

```cpp
#include <atomic>

extern std::atomic<int> avalue;
extern int bvalue;

void setValues(int a, int b) {
    bvalue = b; // 1
    avalue.store(a, std::memory_order_release); // 2
}

void getValues(int& a, int& b) {
    a = avalue.load(std::memory_order_acquire); // 3
    b = bvalue; // 4
}
```

`memory_order_release`：确保当前线程中的读或写不能被重排到此**存储**后，也就是 2 之前的内存读写操作不能重排到 1 的后面

`memory_order_acquire`：确保当前线程中的读或写不能被重排到此**加载**前，其他 release 同一原子变量的线程的所有写入，能为当前线程所见，也就是 3 一定在 4 之前

综上两点，如果语句 3 在 2 之后执行的话，也就能确保 4 一定在 1 之后，从而达到某些同步的目的

<br>



# 三、release/consume

```cpp
#include <atomic>

extern std::atomic<int*> guard;
extern int value;

void setValue(int val) {
    value = val; // 1
    guard.store(&value, std::memory_order_release); // 2
}

void getValue(int& val) {
    int* pv = guard.load(std::memory_order_consume); // 3
    if (pv) {
        val = *pv;  // 4
    }
}
```

`memory_order_consume`: 确保当前线程中依赖于当前加载的该值的读或写不能被重排到此加载前，也就是 4 需要依赖 3 中的 `pv`，4 也就不能被重排到 3 之前，也就保证了执行顺序。

如果 4 处的操作为 `val = value` 的话，3 和 4 就不存在依赖关系，也就不能保证一定的执行顺序，但是 `consume` 没有被主流的编译器实现

<br>



# 四、relaxed

仅仅实现原子性，没有线程同步的保证

可以想象 A、B 二人同时在同一个白板上写 word，假设每次内存读修改写类比为写一个 word，如果没有原子操作限制，A、B 可能会把单词写串，使用原子操作，A 和 B 写入一个一个的单词，但是写入单词顺序没有保证，其他的 memory order 对顺序进行了一些不同粒度的规定  

<br>



# 五、seq_cst

顺序一致性（sequential consistency），如果 load 就是 acquire 语义，如果 store 就是 release 语义，如果是读取+写入就是 acquire-release 语义，也就是对于所有 `acq_rel` 语义加上所有 `seq_cst` 的指令有严格的顺序一致性：

- 在每个线程内部，每个处理器的执行顺序和代码中的顺序（program order）一样
- 所有的处理器都看到了相同的执行顺序

比如像 IM 群聊，每个成员就是一个处理器，如果满足下列条件就是顺序一致性：

- 每个人发出去的消息的顺序和他自己看到发出消息的顺序是一致的
- 群聊中所有人看到消息的顺序都一致

当然，顺序一致性同步性最强，当然会造成很大的开销

<br>



# 六、总结

如果使用单一的计数值，可以使用 load/store + `std::memory_order_relaxed` 

需要同步某区块可以选择使用 load-acquire + store-release

如果同步的需求比较复杂，干脆使用 `std::mutex` 就可以，没必要没事找事  

<br>



# 参考资料

[Memory Models for C/C++ Programmers](https://arxiv.org/pdf/1803.04432.pdf)

[C++ standard memory order](https://zh.cppreference.com/w/cpp/atomic/memory_order)

[codedump cxx11 memory model](https://www.codedump.info/post/20191214-cxx11-memory-model-1/)

[zhihu: 如何理解 C++ 11 的六种 memory order](https://www.zhihu.com/question/24301047)

[consume memory order](https://preshing.com/20140709/the-purpose-of-memory_order_consume-in-cpp11/)

