---
title: Golang 中 Slice 的坑
date: 2018-10-16 21:38:02
tags:
- Golang
- data structure
categories:
- Golang
---

深入分析使用 Go Slice 时遇到的一个坑

<!-- more -->

最近在学 Golang，发现 `Slice` 的一个坑。引发坑的原因是多个 `Slice` 可以共享底层的数据。

Golang 的 `Slice` 类似于 C++ 的 `std::vector<T>`，功能上是动态数组。

参考代码 golang 1.11

---



# 一、底层原理

`slice` 的结构体：

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```



看一下动态数组的底层是如何扩容的。首先判断是否有剩余空间容纳新元素，没有的话就重新分配一个适合的空间(一般为原空间的 2 或 1.5 倍)，然后把原先的元素移动到新的空间中。之后再把新元素移动到新的空间中。
这种增长策略可以减少开销，摊还分析得到 `append` 的时间复杂度为 O(1)。

我们以 `{ len: 0, cap: 0}`，增长因子为 2 举例。下图为依次添加 `1,2,3,4,5` 之后动态数组的变化。实线框为已有元素，虚线框为剩余空间。

{% asset_img pit1_1.png slice %}



当然 go 中的实现有多种增长策略，情况略微复杂。下面是 go 1.11源码中的增长策略。

```go
// growslice in src/runtime/slice.go
// growslice handles slice growth during append.
// ...
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
    newcap = cap
} else {
    if old.len < 1024 {
        newcap = doublecap
    } else {
        // Check 0 < newcap to detect overflow
        // and prevent an infinite loop.
        for 0 < newcap && newcap < cap {
            newcap += newcap / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
            newcap = cap
        }
    }
}
```



`Slice` 切片操作 `s[i:j]` 会创建新的 `Slice`，而且多个 `Slice` 底层共享同一内存空间，不是 C++ 那种直接把元素 copy 或 move 进去。比如：

```go
s1 := []int{1,2,3,4,5}
s2 := s1[:]
s2[2] = 42
fmt.println(s1)      // 1, 2, 42, 4, 5
```

这样就会引发一些较坑的问题。下面以实际例子说明。


# 二、实际例子

```go
func main() {
    s := []int{0}
    s = append(s, 1)
    s = append(s, 2)     // s == [0 1 2]
    x := append(s, 3)    // 
    y := append(s, 4)
    fmt.println(s, x, y) // [0 1 2] [0 1 2 4] [0 1 2 4]
}
```

可以看到最后 `x` 和 `y` 都为 `[0 1 2 4]`，这样的结果就不符合本来的期待了。分析一下为什么出现这样的现象。

由于增长策略的问题，在 `append(s, 2)` 之后，底层数组为 `[0 1 2]`，`s.len == 3, s.cap == 4`。
当 `x := append(s, 3)` 时，`s` 有足够的容量放置元素 3。由于 `x` 和 `s` 的底层数据指针相同，此时 `s.len` 和 `s.cap` 没有发生改变，`x.len == 4, x.cap == 4`，底层数组变为 `[0 1 2 3]`。
当 `y := append(s, 4)` 时，仍然检测到 `s` 有足够的容量放置元素 4，`y.len == 4, y.cap == 4`，此时 `s, x, y` 三者引用相同的底层数组，底层数组变为 `[0 1 2 4]`，也就解释了为何出现了上面的现象。



再来看一个把 `Slice` 作为函数参数的问题。`Slice` 结构体为值传递。

```go
func main() {
    nums := []int{1, 2, 3}
    modifySlice(nums)
    fmt.Println(nums)     // [1 2 3]
    modifyElement(nums)
    fmt.Println(nums)     // [0 2 3]
}

func modifySlice(nums1 []int) {
    nums1 = append(nums1, 4) 
}

func modifyElement(nums2 []int) {
    nums2[0] = 0
}
```

实质上 `nums, nums1, nums2` 的底层数组指针值相同，不过 `Slice` 按值传递。

`modifySlice` 就相当于修改指针副本的值。就类似于

```cpp
void modifyPointer(int* p) {
    p = nullptr;
}
```

`modifyElement` 修改了底层指针所指向的元素。类似于

```cpp
void modifyElement(int* p) {
    p[0] = 0;
}
```



# 三、总结

使用 `Slice` 应该尽量避免实际例子1 中的操作，尽量使用 `s = append(s, x)` 这样的形式。