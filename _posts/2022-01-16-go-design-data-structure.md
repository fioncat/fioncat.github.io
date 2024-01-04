---
layout:       post
title:        "Go设计与实现笔记 一 数据结构"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Go
---

## 数组

数组在内存上是一段连续空间组成的结构。

数组类型同时由元素类型和长度组成，元素类型相同但是长度不同的数组不是同一个类型。

数组在初始化的时候，依据以下原则（不考虑逃逸分析）：

- 元素少于等于4个时，直接初始化在栈上面
- 元素多于4个时，将数组中的元素放到静态区，并在运行时取出

数组的访问，一些简单的越界错误可以在编译时发现，如果是通过变量访问数组，需要在运行时检查是否越界，如果越界会进行panic。

## 切片

切片是对数组中部分连续片段的引用，可以在运行时修改它的长度和范围，在长度不足时会自动进行扩容。一个切片包含三个元素：

- `Data uintptr`指向原始数组的指针
- `Len int`当前切片的长度
- `Cap int`当前切片的容量，即`Data`数组的大小

初始化切片：

- `slice = arr[0:3]`使用数组的下标进行初始化，这是最底层的一种方式，会直接创建一个引用原始数组的切片，这不会创建新的数组，因此对切片的改变会影响到原始数组。
- `slice = []int{1, 2, 3}`使用字面量初始化。这会先使用字面量创建一个数组，然后使用`slice = arr[:]`来创建一个对该数组的引用。
- `slice = make([]int, len, cap)`使用关键字。在编译时会判断切片是否逃逸，或其容量是否很大，如果是的话，会调用`runtime.makeslice`函数，在运行时在堆中初始化切片；如果其没有逃逸并且容量比较小，会将`make`转换为上面的方式进行初始化。

`runtime.makeslice`会计算切片占用的内存空间，并调用`runtime.mallocgc`在堆上申请一段连续的空间。

访问切片时，在编译时就会替换为对切片底层数组的访问。`len(slice)`和`cap(slice)`会在运行时获取切片的长度和容量，但是这里在编译时可能触发优化，直接替换为长度和容量值。

在对切片进行`append`操作时，编译器会根据返回值是否覆盖原变量而生成两种代码：

- `slice = append(slice, ...)`：直接在原切片上面操作，不用担心会发生切片拷贝。
- `slice2 = append(slice1, ...)`：会调用`runtime.makeslice`创建一个新的切片，并将原来的元素和追加的元素一起拷贝过去并返回。

不论是否覆盖原切片，操作都是类似的，如果追加后len小于等于cap，则操作类似于数组赋值，但是会增加切片的len的值；如果追加后len大于cap，会调用`runtime.growslice`进行扩容。

`runtime.growslice`会创建一个新的切片，它的容量是这么确定的：

- 期望容量大于当前两倍，直接使用
- 长度小于1024，将容量翻倍
- 大于1024，每次增加25%容量，直到大于期望容量

上面确定了切片的大致容量，最终容量还需要根据切片中的元素大小进行对齐。如果占用元素的大小为1，2，8的整数，会进行向上取整对齐。

例如，一个`[]int64`切片，扩容后的容量为5，计算得到需要分配的内存为`8*5 = 40Bytes`，但是因为`int64`占用`8`字节，需要调用`runtime.roudupsize`将内存对齐到`48`，最终分配的容量为`6`。

在调用`copy`进行切片复制时，最终会调用`runtime.memmove`进行整块内存的复制，这比依次复制元素的性能要高。因此，如果涉及到切片复制，请使用`copy`而不要自己一个一个元素复制。

## 哈希表

map是一种表示键值映射关系的数据结构。

### 原理

哈希函数的选择比较重要，它用于将键映射到值索引上。一个好的哈希函数输出应该尽可能均匀。如果两个键被映射到了同一个索引上面，就表示发生了哈希冲突。

我们有多种方法来解决哈希冲突：

- 开放寻址法：哈希表的底层数据结构是数组，如果发生冲突，就将键值对写入下一个索引不为空的位置。在读取时，计算哈希后需要比较key是否相等，不想等需要向后继续比较key。装载因子是数组中元素数量与大小的比值，如果达到100%，则对键值对的插入时间复杂度会退化为O(n)。
- 拉链法：底层是数组加上链表，因为链表是动态申请的，所以会比开放寻址节省空间，这是大多数语言的实现方法。底层数组的每个元素存放一个桶，当发生哈希冲突时，会在桶的后面进行元素的追加，这里装载因子是元素数量跟桶数量的比值，如果过大，也会影响哈希表的性能。这是Go采用的方法。

### 数据结构

Go用多个数据结构来实现map，其中最核心的结构是`runtime.hmap`，其中比较重要的几个字段：

```go
type hmap struct {
    count int // 当前哈希表的元素数量

    // 当前哈希表的buckets数量，但是因为桶数量都是2的倍数，所以
    // 这里会储存对数即len(buckets) == 2^B
    B uint8

    // 哈希表的种子，为哈希函数的结果引入随机性
    hash0 uint32

    // 指向当前的桶列表
    buckets unsafe.Pointer

    // 哈希表在扩容时用于保存之前的buckets的字段，它的大小是
    // buckets的一半
    oldbuckets unsafe.Pointer

    extra *mapextra
}

type mapextra struct {
    // 溢出桶列表
    overflow *[]*bmap

    // 保存之前的溢出桶数据，作用跟oldbuckets类似
    oldoverflow *[]*bmap

    // 指向下一个溢出桶
    nextOverflow *bmap
}
```

桶的数据结构是`runtime.bmap`，每一个`runtime.bmap`可以储存8个键值对，当哈希表中储存的数据过多，单个桶装不下时就会使用`extra.nextOverflow`中的桶储存溢出数据。溢出桶是保留的C语言实现的设计，它能够降低map的扩容频率，正常桶和溢出桶在内存上是连续的。

桶的结构体`runtime.bmap`：

```go
type bmap struct {
    // 储存了键的哈希高8位，Go在比较哈希时只会比较高8位，以减少访问
    // 键值对的次数以提高性能
    tophash [bucketCnt]uint8

    // 以下字段需要在编译时确定，因为Go没有范型（1.18之前）
    // 所以键值对占据的空间大小只能在编译时进行推导
    topbit   [8]uint8
    keys     [8]keytype
    values   [8]keytype
    pad      uintptr
    overflow uintptr
}
```

`tophash`储存了键的哈希的高8位，通过比较哈希的高8位可以减少访问键值对的次数。

### 初始化

我们可以通过字面量来初始化map，当字面量的元素小于等于25时，编译器会将字面量初始化代码转换为：

```go
hash := map[string]int{"a": 1, "b": 2, "c": 3}
// 会被编译器转换为：
hash := make(map[string]int, 3)
hash["a"] = 1
hash["b"] = 2
hash["c"] = 3
```

如果元素个数超过25，编译器会创建两个数组分别储存键和值，随后通过for循环进行初始化。但是最终都会使用`make`来初始化map。

有一种特殊情况，当map的容量小于8，并且被分配到栈上时，会使用如下方式快速初始化哈希表：

```go
var h *hmap
var hv hmap
var bv bmap
h := &hv
b := &bv
h.buckets = b
h.hash0 = fashtrand0()
```

在其它情况下，都会使用`make`初始化哈希表，其最终会调用`runtime.makemap`。

这个函数会做如下的事情：

- 计算哈希表占用的内存是否溢出或超出能分配的最大值
- 调用`runtime.fastrand`获取一个随机的哈希种子
- 根据传入的hint计算至少需要多少桶
- 使用`runtime.makeBucketArray`创建用于保存桶的数组
- `runtime.makeBuketArray`会判断，如果桶数量小于24，则忽略溢出桶的创建；否则，会额外创建`2^(B-4)`个溢出桶

在正常情况下，正常桶和溢出桶在内存中储存是连续的，只是被不同的字段引用，正常桶会被事先创建，溢出桶只会在桶数量过多的时候被创建。设计溢出桶的目的就是让哈希表容忍一定的溢出以减少扩容的次数。

### 访问

两种对map的访问在编译时都会替换为不同的运行时函数：

- `v := hash[key]`会被替换为`v := *mapaccess1(maptype, hash, &key)`
- `v, ok := hash[key]`会被替换为`v, ok := mapaccess2(maptype, has, &key)`

`runtime.mapaccess1`的处理过程如下：

```go
// 注意：这里省略了哈希扩容的相关代码，我们将在后面介绍
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 获取哈希函数
    alg := t.key.alg
    // 根据哈希种子计算出key的哈希值
    hash := alg.hash(key, uintptr(h.hash0))

    // 根据hash获取该键值对所在的桶，这里需要根据hash掩码获取
    m := bucketMask(h.B)
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

    // 获取哈希的高8位，以加快搜寻键值对的速度
    top := tophash(hash)

bucketloop:
    // 依次遍历正常桶和溢出桶的数据，第一轮编译正常桶，当没有找到数据
    // 下一轮循环会开始处理溢出桶
    for ; b != nil; b = b.overflow(t) {
        // 遍历桶中的所有元素
        for i := uintptr(0); i < bucketCnt; i++ {
            // 比较hash的高8位是否相等，如果不相等，继续遍历下一个元素
            if b.tophash[i] != top {
                // 没有数据了，退出
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            // 到这里高8位相等了，说明很有可能就是它了
            // 我们通过指针偏移来获取哈希表中储存的键
            // 以进行进一步确定
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // 哈希表中的键跟目标键进行对比，如果二者相等
            // 获取值并返回
            if alg.equal(key, k) {
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                return v
            }
        }
    }
    // 没有找到，返回空指针
    return unsafe.Pointer(&zeroVal[0])
}
```

另外一个访问函数`runtime.mapaccess2`跟上面十分类似，只是在上面的基础上多返回了一个表示是否存在的布尔值，这里不再展示。

上面的代码省略了哈希扩容的处理，哈希扩容并不是原子的，在扩容过程中需要保证元素的访问，在后面我们会介绍相关内容。

### 写入

形如`hash[k] = v`的哈希表赋值操作会被编译器转换为`runtime.mapassign`函数调用：

```go
// 同样，忽略了扩容的相关代码
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 计算哈希值
    alg := t.key.alg
    hash := alg.hash(key, uintptr(h.hash0))

    // 标识目前哈希表正在写入
    h.flags ^= hashWriting

again:
    // 获取键值对所在的桶
    bucket := hash & bucketMask(h.B)
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    // 哈希值的高8位
    top := tophash(hash)

    var inserti *uint8 // 目标元素在桶中的索引
    // 键值对的地址
    var insertk unsafe.Pointer
    var val unsafe.Pointer
bucketloop:
    // 这个循环会依次遍历正常桶和溢出桶
    for {
        // 遍历桶中的元素
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                if isEmpty(b.tophash[i]) && inserti == nil {
                    // 遇到了一个空闲的键值对，说明目标键在桶中不存在
                    // 并且桶还没有溢出。因此我们可以直接将数据写入这
                    // 个空闲的键值对中。
                    // 这里不直接写数据，而是将该空闲键值对的指针带出去
                    inserti = &b.tophash[i]
                    insertk = add(unsafe.Pointer(b),
                        dataOffset+i*uintptr(t.keysize))
                    val = add(unsafe.Pointer(b),
                        dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                }
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
            if !alg.equal(key, k) {
                continue
            }
            // 到这里我们在桶中找到了目标键，就表示是更新操作
            // 直接将值的带出去以进行赋值
            val = add(unsafe.Pointer(b),
                dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
            goto done
        }
        // 继续到溢出桶寻找
        ovf := b.overflow()
        if ovf == nil {
            break
        }
        b = ovf
    }

    if inserti == nil {
        // 这说明不管是正常桶还是溢出桶都已经满了，需要创建新的溢出桶
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        val = add(insertk, bucketCnt*uintptr(t.keysize))
    }

    // 将键数据写入对应的内存空间中
    typedmemmove(t.key, insertk, key)
    *inserti = top
    h.count++

done:
    return val
}
```

注意，`runtime.mapassign`最后并没有把值写入对应的内存，它只是返回了值的地址，真正的写入操作会在编译时插入代码通过汇编指令完成。

### 扩容

随着哈希表中元素逐渐增加，哈希表的性能会恶化，这时就需要更多的桶和更大的内存来保证哈希表的读写性能。

上面的哈希表写入操作省略了扩容操作，在写入时会判断是否需要进行扩容：

```go
func mapassign(...) {
    ...
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again
    }
    ...
}
```

有以下两个条件会触发哈希扩容：

- 装载因子超过了6.5
- 使用了太多溢出桶

因为哈希表的扩容不是原子操作，它是在写入和读取中分布完成的，因此`runtime.mapaassign`会判断当前是否已经处于扩容中了，以避免二次扩容造成混乱。

如果扩容是因为使用太多溢出桶造成的，会触发`sameSizeGrow`，这是一种特殊的扩容。因为如果我们不断插入数据并删除，则新插入的数据会全部放到溢出桶中，造成溢出桶的不断积累而导致缓慢的内存泄漏。`sameSizeGrow`会创建新的桶来保存溢出桶的数据，并让垃圾收集器清理老的溢出桶并释放内存。它复用了扩容的代码，但是本质是对内存的整理，而不会增加桶的数量。

扩容的入口是`runtime.hashGrow`：

```go
func hashGrow(t *maptype, h *hmap) {
    // 默认情况下扩容一倍
    bigger := uint8(1)

    if !overLoadFactor(h.count+1, h.B) {
        // 因为使用过多溢出桶导致的sameSizeGrow，它不改变桶的总数量
        // 因此将bigger设为0
        bigger = 0
        h.flags |= sameSizeGrow
    }
    oldbuckets := h.buckets
    // 创建一组新的桶和溢出桶，使用新的容量
    newbuckets, nextOverflow := makeBucketArray(t, h.B+nigger, nil)

    // 将新的桶和溢出桶保存到hmap上
    // 注意这里保留了oldbucket，用于稍后的数据迁移
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0

    h.extra.oldoverflow = h.extra.overflow
    h.extra.overflow = nil
    h.extra.nextOverflow = nextOverflow
}
```

上面我们准备好了新的桶和溢出桶，但是它们是空的，还没有将旧的数据分流过去，这个过程在`runtime.evacuate`完成：

```go
// 扩容过程中的数据迁移，将指定旧桶oldbucket迁移到新桶中
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    // 取出旧桶数据
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
    // 新的桶掩码，用于根据hash值计算键值对应该分流到哪个桶中
    // 它应当比旧的桶掩码多了一位，例如，4个桶的哈希表扩容到了8个桶
    // 旧的掩码为0b11，新的掩码为0b111，则旧的3号桶的数据会根据
    // 新掩码分配到新的3号和新的7号桶
    newbit := h.noldbuckets()
    if !evacuated(b) {
        // 我们会将旧桶的数据分流到两个新桶中，因此这里我们使用两个
        // evacDst来分别保存两个新桶的上下文数据
        var xy [2]evacDst
        x := &xy[0]
        x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        x.k = add(unsafe.Pointer(x.b), dataOffset)
        x.v = add(x.k, bucketCnt*uintptr(t.keysize))

        y := &xy[1]
        y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
        y.k = add(unsafe.Pointer(y.b), dataOffset)
        y.v = add(y.k, bucketCnt*uintptr(t.keysize))

        // 遍历所有的旧桶数据
        for ; b != nil; b = b.overflow(t) {
            // 当前桶的初始偏移量
            k := add(unsafe.Pointer(b), dataOffset)
            v := add(k, bucketCnt*uintptr(t.keysize))
            // 遍历所有键值对
            for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
                top := b.tophash[i]
                k2 := k
                var useY uint8
                // 我们需要确认这个键值对应该被分流到哪个桶中
                // 仅靠hash无法确定，需要进行掩码操作确认
                hash := t.key.alg.hash(k2, uintptr(h.hash0))
                if hash&newbit != 0 {
                    useY = 1
                }
                b.tophash[i] = evacuatedX + useY
                // 选择分流桶
                dst := &xy[useY]

                if dst.i == bucketCnt {
                    // 满了，需要创建溢出桶
                    dst.b = h.newoverflow(t, dst.b)
                    dst.i = 0
                    dst.k = add(unsafe.Pointer(dst.b), dataOffset)
                    dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
                }

                // 将数据复制到新的桶中
                dst.b.tophash[dst.i&(bucketCnt-1)] = top
                typedmemmove(t.key, dst.k, k)
                typedmemmove(t.elem, dst.k, v)
                dst.i++
                dst.k = add(dst.k, uintptr(t.keysize))
                dst.v = add(dst.v, uintptr(t.valuesize))
            }
        }
    }
    ...
}
```

在分流的最后，会调用`runtime.advanceEvacuationMark`增加哈希表的`nevacuate`计数器，并在所有旧桶都被分流之后清空哈希表的`oldbuckets`和`oldoverflow`。

在访问哈希表元素的时候，我们省略了一段有关扩容的逻辑：

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...
    alg := t.key.alg
    hash := alg.hash(key, uintptr(h.hash0))
    m := bucketMask(h.B)
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

    // 这段是我们之前省略的逻辑
    // 如果有旧桶，说明我们目前处于扩容阶段，可能需要从旧桶读取数据
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            // 不是sameSizeGrow时，掩码需要多出一位
            m >>= 1
        }
        // 取出当前键值对的旧桶，如果它还没有被迁移，则要从旧桶读取数据
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        if !evacuated(oldb) {
            b = oldb
        }
    }
    ...
}
```

在赋值时，也有一段有关扩容的逻辑：

```go
func mapassign(t *maptype, key unsafe.Pointer) unsafe.Pointer {
    ...
again:
    bucket := hash & bucketMask(h.B)
    if h.growing() {
        // 如果哈希表处于扩容状态，每次写入时都会增量触发数据分流
        growWork(t, h, bucket)
    }
    ...
}
```

`runtime.growWork`会对还没有分流的桶调用`runtime.evacuate`进行分流。这种在写操作进行增量分流的设计，是为了防止性能的瞬间巨大抖动，将数据迁移操作分配给多个写操作来完成。

总结：

- 哈希表装载因子过大或溢出桶过多会触发扩容。
- 溢出桶过多触发的扩容是等量扩容，不会改变桶的总数，用于重新编排哈希表以减少溢出桶的个数。
- 扩容不是原子的，会首先创建新的桶，大小是原来的一半，然后将哈希表标记为扩容中状态。
- 如果读取扩容状态中的哈希表，会判断桶是否分流完毕，如果没有，会从旧的桶读取数据。
- 如果对扩容状态的哈希表赋值或删除，会触发哈希表的增量数据分流。新的数据会直接写到新桶中。
- 所有旧桶的数据分流结束后，删除旧桶和扩容状态。

### 删除

Go专门提供了一个关键词`delete`用于对哈希表进行删除操作，在编译时，它会被转换为下面几种函数调用：

- `runtime.mapdelete`
- `runtime.mapdelete_faststr`
- `runtime.mapdelete_fast32`
- `runtime.mapdelete_fast64`

这些函数的差距不大，逻辑基本都是：

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
    // 这里的逻辑跟mapassign类似，省略
    ...
    if h.growing() {
        // 如果处于扩容中，进行分流操作
        growWork(t, h, bucket)
    }
    ...
search:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] == emptyRest {
                break search
            }
            continue
        }
        k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
        k2 := k
        if !alg.equal(key, k2) {
            continue
        }
        // 删除元素，将键值对的指针设置为nil，GC会自动清除它们
        *(*unsafe.Pointer)(k) = nil
        v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
        *(*unsafe.Pointer)(v) = nil
        b.tophash[i] = emptyOne
        ...
    }
}
```

## 字符串

字符串本质是只读的字符数组，底层的数据结构是：

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
```

跟切片相比也只是少了个cap，因此字符串也可以认为是只读的切片。对字符串的任何修改只能通过复制来完成。

当使用`+`对字符串进行拼接时，最终会转换为`runtime.concatstrings`函数调用，它会调用copy将输入的多个字符串复制到目标字符串的内存空间上面，新的字符串是新的内存空间，与原来的字符串没有关联，如果字符串很大，这会带来很大的性能损失。

将`[]byte`转换为字符串会调用`runtime.slicebytetostring`函数：

```go
func slicebytetostring(buf *tmpBuf, b []byte) (str string) {
    l := len(b)
    if l == 0 {
        // 特殊情况：长度为0，返回空字符串
        return ""
    }
    if l == 1 {
        // 特殊情况：长度为1，可以直接赋值
        stringStructOf(&str).str = unsafe.Pointer(&staticbytes[b[0]])
        stringStructOf(&str).len = 1
        return
    }
    var p unsafe.Pointer
    if buf != nil && len(b) <= len(buf) {
        // 缓冲区够用，使用缓冲区
        p = unsafe.Pointer(buf)
    } else {
        // 缓冲区不够，分配新的空间
        p = mallocgc(uintptr(len(b)), nil, false)
    }
    stringStructOf(&str).str = p
    stringStructOf(&str).len = len(b)
    // 将字节数组写入新的字符串对象中
    memmove(p, (*(*slice)(unsafe.Pointer(&b))).array, uintptr(len(b)))
    return
}
```

将`[]byte`转换为字符串：

```go
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
    var b []byte
    if buf != nil && len(s) <= len(buf) {
        // 传入了缓冲区，并且够用。使用缓冲区
        *buf = tmpBuf{}
        b = buf[:len(s)]
    } else {
        // 创建新的字节切片
        b = rawbyteslice(len(s))
    }
    // 将字符串内容拷贝到字节切片中
    copy(b, s)
    return b
}
```

可见，转换一定会涉及到内存拷贝。如果长度很大，我们一定要考虑转换导致的开销。
