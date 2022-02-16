---
title: Golang map
date: 2019-03-19 21:25:45
tags:
- Golang
- data structure
categories:
- Golang
mathjax: true
---

Go map 删除元素及源码分析

<!-- more -->

# 一、引子

最近写 Go 时遇到一个这样的问题

```go
m := make(map[int]string) 
// ... 
for k := range m {
    if k == 42 {
        delete(m, k)
    }
}
```

这样的代码是否安全呢？



冒出这样的疑问是因为原先 C++ 时的经验：

```cpp
std::map<int, std::string> m;
// ...

for (auto iter = m.begin(); iter != m.end(); ++iter) {
    if (iter->first == 42) {
        m.erase(iter);
    }
}
```

由于 `m.erase(iter)` 将时被删除元素的 iterator 被非法化，见 [std::map::erase](https://zh.cppreference.com/w/cpp/container/map/erase)

接下来进行 `++iter` 将产生 undefined behavior。

正确的做法应该是：

```cpp
std::map<int, std::string> m;
for (auto iter = m.begin(); iter != m.end(); ) {
    if (iter->first == 42) {
        iter = m.erase(iter);
    } else {
        ++iter;
    }
}
```

---

# 二、分析

于是乎去查了一下，先说结论，在 go 中是安全的。简单来说就是迭代器不会像 C++ 那样失效。

接下来剖析下 `map` 的实现原理：底层数据结构为哈希表。结构体定义在[这儿](https://github.com/golang/go/blob/release-branch.go1.11/src/runtime/map.go#L108)。

```go
type hmap struct {
    count     int // # live cells == size of map.
    flags     uint8
    B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    noverflow uint16 // approximate number of overflow buckets
    hash0     uint32 // hash seed
    
    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // previous bucket array of half the size
    nevacuate  uintptr       

    extra *mapextra // optional fields
}
```

`buckets` 是一个 $2^B$ 大小的 `bucket` 数组，`oldbuckets` 在扩容时才会使用，考虑到直接全部扩容可能造成短暂阻塞的情况，所以采用增量扩容策略，当元素增多到一定数量，原 `buckets` 变为原来的两倍，之后逐步将 `oldbuckets` 重新计算到新的 `buckets` 中。当扩容发生时对 `map` 进行操作时也有对应的策略，最后全部完成时释放 `oldbuckets`，扩容时机由 `loadFactor` 控制，1.11 版本是 6.5。

[`bucket` 结构体](https://github.com/golang/go/blob/release-branch.go1.11/src/runtime/map.go#L142)主要有 `tophash [bucketCnt]uint8， keys, values, overflow `

每个 `bucket` 最多有 `bucketCnt == 8` 个 pair，更多元素在 `overflow(bucket 链表指针)` 串联的桶。

`tophash` 存放对应元素 hash 值的最高 8 位。`tophash` 只是为了加速 key 比较的。

`v, found := m[key]`

```go
// https://github.com/golang/go/blob/release-branch.go1.11/src/runtime/map.go#L470
// ..., xxx, yyy 为省略部分，接下来的代码类似
for ; b != nil; b = b.overflow(t) {
    for i := uintptr(0); i < bucketCnt; i++ {
        if b.tophash[i] != top {
            continue
        }
        k := xxx 
        if alg.equal(key, k) {
            v := yyy
            return v, true
        }
    }	
}
return unsafe.Pointer(&zeroVal[0]), false
```



上述代码先去比较 `tophash` 是否相等，再比较 key 是否真正相等，在 `alg.equal` 复杂度较高的情况下，能够对性能有所提升。

`m[key] = val`

```go
again:
bucket := hash & bucketMask(h.B) // 根据 hash 值计算在哪个桶
if h.growing() {
    growWork(t, h, bucket)
}
top := tophash(hash)
var inserti *uint8
var insertk unsafe.Pointer
var val unsafe.Pointer

for {
    for i := uintptr(0); i < bucketCnt; i++ {
        if b.tophash[i] != top {
            if b.tophash[i] == empty && inserti == nil {
                // 找到空位准备插入
                inserti = &b.tophash[i]
                insertk = xxx
                val = yyy
            }
            continue
        }
        k := xxx
        if !alg.equal(key, k) {
            continue
        }
        // already have a mapping for key. Update it.
        if t.needkeyupdate {
            typedmemmove(t.key, k, key)
        }
        val = yyy
        goto done
    }
    ovf := b.overflow(t)
    if ovf == nil {
        break
    }
    b = ovf
}

// Did not find mapping for key. Allocate new cell & add entry.

// If we hit the max load factor or we have too many overflow buckets,
// and we're not already in the middle of growing, start growing.
if !h.growing() && (overLoadFactor(h.count+1, h.B)
    || tooManyOverflowBuckets(h.noverflow, h.B)) {
    hashGrow(t, h)
    goto again // Growing the table invalidates everything, so try again
}

if inserti == nil {
    // all current buckets are full, allocate a new one.
    newb := h.newoverflow(t, b)
    inserti = &newb.tophash[0]
    insertk = add(unsafe.Pointer(newb), dataOffset)
    val = add(insertk, bucketCnt*uintptr(t.keysize))
}

// store new key/value at insert position
// ...
typedmemmove(t.key, insertk, key)
*inserti = top
h.count++

done:
if h.flags&hashWriting == 0 {
    throw("concurrent map writes")
}
h.flags &^= hashWriting
if t.indirectvalue {
    val = *((*unsafe.Pointer)(val))
}
return val
```



`delete(m, key)` 

```go
// in mapdelete: L660
for ; b != nil; b = b.overflow(t) {
    for i := uintptr(0); i < bucketCnt; i++ {
        if b.tophash[i] != top {
            continue
        }
        
        // ...
        // clear key and value ...
        // ...
        
        b.tophash[i] = empty
        h.count--
        break search
    }
}
```

可以看到 `b.tophash[i] = empty`，清理了 key 和 value 之后，更新对应的 `tophash[i]` 为 `empty`。

`empty` 是 0，问题又来了，那求出 hash 值的高 8 位正好是 `empty` 怎么办呢？

看 `tophash` 的实现和一些常量定义就一目了然了：

```go
const (empty       = 0 // cell is empty
    evacuatedEmpty = 1 // cell is empty, bucket is evacuated.
    evacuatedX     = 2 // key/value is valid.  
                       // Entry has been evacuated to first half of larger table.
    evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
    minTopHash     = 4 // minimum tophash for a normal filled cell.
)

func tophash(hash uintptr) uint8 {
    top := uint8(hash>> (sys.PtrSize*8 - 8))
    if top < minTopHash {
        top += minTopHash
    }
    return top
}
```

接下来看迭代器的结构体和迭代器的 `next`。

```go
// 删除了部分过长的注释
type hiter struct {
    key         unsafe.Pointer 
    value       unsafe.Pointer 
    t           *maptype
    h           *hmap
    buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
    bptr        *bmap          // current bucket
    overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
    oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
    startBucket uintptr        // bucket iteration started at
    offset      uint8          
    wrapped     bool           
    B           uint8
    i           uint8
    bucket      uintptr
    checkBucket uintptr
}
```

```go
// https://github.com/golang/go/blob/release-branch.go1.11/src/runtime/map.go#L783
// 省略大量涉及 evacuate 的代码
next:
if b == nil {
    if bucket == it.startBucket && it.wrapped {
        // end of iteration
        it.key = nil
        it.value = nil
        return
    }
    bucket++
    if bucket == bucketShift(it.B) {
        bucket = 0
        it.wrapped = true
    }
    i = 0
}
for ; i < bucketCnt; i++ {
    offi := (i + it.offset) & (bucketCnt - 1)
    if b.tophash[offi] == empty || b.tophash[offi] == evacuatedEmpty {
        continue
    }
    
    // ...
    it.key = xxx
    it.value = yyy

    it.bucket = bucket
    if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
        it.bptr = b
    }
    it.i = i + 1
    it.checkBucket = checkBucket
    return
}
b = b.overflow(t)
i = 0
goto next
```

可以看到在 `delete` 之后把 `tophash` 置为 `empty`，接下来调用 `next`，碰到 `empty` 的就直接跳过去，所以不存在迭代器失效的问题，也就解释了为什么不会产生问题的原因，也捎带分析了一下 golang 中 map 的实现。

顺带提一句，这儿就是为什么 range map 两次顺序不一样的原因，`it.offset` 是个随机值。