#                        Golang中map增删改查操作的实现原理

> 本文内容主要引用自[深入解析Golang的map设计](https://zhuanlan.zhihu.com/p/273666774)

上一篇文章主要介绍了map的基本原理与创建map是如何实现的，这一篇文章让我们学习map增删改查操作的实现原理

### 查找key

对于map的元素查找，其源码实现如下：

```go
//src/runtime/hashmap_fast.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  // 如果开启了竞态检测 -race
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        pc := funcPC(mapaccess1)
        racereadpc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
  // 如果开启了memory sanitizer -msan
    if msanenabled && h != nil {
        msanread(key, t.key.size)
    }
  // 如果map为空或者元素个数为0，返回零值
    if h == nil || h.count == 0 {
        if t.hashMightPanic() {
            t.hasher(key, 0) // see issue 23734
        }
        return unsafe.Pointer(&zeroVal[0])
    }
  // 注意，这里是按位与操作
  // 当h.flags对应的值为hashWriting（代表有其他goroutine正在往map中写key）时，那么位计算的结果不为0，因此抛出以下错误。
  // 这也表明，go的map是非并发安全的
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
  // 不同类型的key，会使用不同的hash算法，可详见src/runtime/alg.go中typehash函数中的逻辑
    hash := t.hasher(key, uintptr(h.hash0))
    m := bucketMask(h.B)
  // 按位与操作，找到对应的bucket
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  // 如果oldbuckets不为空，那么证明map发生了扩容
  // 如果有扩容发生，老的buckets中的数据可能还未搬迁至新的buckets里
  // 所以需要先在老的buckets中找
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            m >>= 1
        }
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    // 如果在oldbuckets中tophash[0]的值，为evacuatedX、evacuatedY，evacuatedEmpty其中之一
    // 则evacuated()返回为true，代表搬迁完成。
    // 因此，只有当搬迁未完成时，才会从此oldbucket中遍历
        if !evacuated(oldb) {
            b = oldb
        }
    }
  // 取出当前key值的tophash值
    top := tophash(hash)
  // 以下是查找的核心逻辑
  // 双重循环遍历：外层循环是从桶到溢出桶遍历；内层是桶中的cell遍历
  // 跳出循环的条件有三种：第一种是已经找到key值；第二种是当前桶再无溢出桶；
  // 第三种是当前桶中有cell位的tophash值是emptyRest，这个值在前面解释过，它代表此时的桶后面的cell还未利用，所以无需再继续遍历。
bucketloop:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
      // 判断tophash值是否相等
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
      }
      // 因为在bucket中key是用连续的存储空间存储的，因此可以通过bucket地址+数据偏移量（bmap结构体的大小）+ keysize的大小，得到k的地址
      // 同理，value的地址也是相似的计算方法，只是再要加上8个keysize的内存地址
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
      // 判断key是否相等
            if t.key.equal(key, k) {
                e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                if t.indirectelem() {
                    e = *((*unsafe.Pointer)(e))
                }
                return e
            }
        }
    }
  // 所有的bucket都未找到，则返回零值
    return unsafe.Pointer(&zeroVal[0])
}
```

以下是mapaccess1的查找过程图解

![img](https://pic1.zhimg.com/80/v2-8547e5fdbc7f51e6d5aec5d51ed658b0_1440w.webp)

map的元素查找，对应go代码有两种形式

```go
// 形式一
    v := m[k]
    // 形式二
    v, ok := m[k]
```

形式一的代码实现，就是上述的mapaccess1方法。此外，在源码中还有个mapaccess2方法，它的函数签名如下。

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {}
```

与mapaccess1相比，mapaccess2多了一个bool类型的返回值，它代表的是是否在map中找到了对应的key。因为和mapaccess1基本一致，所以详细代码就不再贴出。

同时，源码中还有mapaccessK方法，它的函数签名如下。

```go
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {}
```

与mapaccess1相比，mapaccessK同时返回了key和value，其代码逻辑也一致。

### 赋值key

对于写入key的逻辑，其源码实现如下

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  // 如果h是空指针，赋值会引起panic
  // 例如以下语句
  // var m map[string]int
    // m["k"] = 1
    if h == nil {
        panic(plainError("assignment to entry in nil map"))
    }
  // 如果开启了竞态检测 -race
    if raceenabled {
        callerpc := getcallerpc()
        pc := funcPC(mapassign)
        racewritepc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
  // 如果开启了memory sanitizer -msan
    if msanenabled {
        msanread(key, t.key.size)
    }
  // 有其他goroutine正在往map中写key，会抛出以下错误
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
  // 通过key和哈希种子，算出对应哈希值
    hash := t.hasher(key, uintptr(h.hash0))

  // 将flags的值与hashWriting做按位或运算
  // 因为在当前goroutine可能还未完成key的写入，再次调用t.hasher会发生panic。
    h.flags ^= hashWriting

    if h.buckets == nil {
        h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
}

again:
  // bucketMask返回值是2的B次方减1
  // 因此，通过hash值与bucketMask返回值做按位与操作，返回的在buckets数组中的第几号桶
    bucket := hash & bucketMask(h.B)
  // 如果map正在搬迁（即h.oldbuckets != nil）中,则先进行搬迁工作。
    if h.growing() {
        growWork(t, h, bucket)
    }
  // 计算出上面求出的第几号bucket的内存位置
  // post = start + bucketNumber * bucketsize
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    top := tophash(hash)

    var inserti *uint8
    var insertk unsafe.Pointer
    var elem unsafe.Pointer
bucketloop:
    for {
    // 遍历桶中的8个cell
        for i := uintptr(0); i < bucketCnt; i++ {
      // 这里分两种情况，第一种情况是cell位的tophash值和当前tophash值不相等
      // 在 b.tophash[i] != top 的情况下
      // 理论上有可能会是一个空槽位
      // 一般情况下 map 的槽位分布是这样的，e 表示 empty:
      // [h0][h1][h2][h3][h4][e][e][e]
      // 但在执行过 delete 操作时，可能会变成这样:
      // [h0][h1][e][e][h5][e][e][e]
      // 所以如果再插入的话，会尽量往前面的位置插
      // [h0][h1][e][e][h5][e][e][e]
      //          ^
      //          ^
      //       这个位置
      // 所以在循环的时候还要顺便把前面的空位置先记下来
      // 因为有可能在后面会找到相等的key，也可能找不到相等的key
            if b.tophash[i] != top {
        // 如果cell位为空，那么就可以在对应位置进行插入
                if isEmpty(b.tophash[i]) && inserti == nil {
                    inserti = &b.tophash[i]
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
                    elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                }
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
      // 第二种情况是cell位的tophash值和当前的tophash值相等
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
      // 注意，即使当前cell位的tophash值相等，不一定它对应的key也是相等的，所以还要做一个key值判断
            if !t.key.equal(key, k) {
                continue
            }
            // 如果已经有该key了，就更新它
            if t.needkeyupdate() {
                typedmemmove(t.key, k, key)
            }
      // 这里获取到了要插入key对应的value的内存地址
      // pos = start + dataOffset + 8*keysize + i*elemsize
            elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
      // 如果顺利到这，就直接跳到done的结束逻辑中去
            goto done
        }
    // 如果桶中的8个cell遍历完，还未找到对应的空cell或覆盖cell，那么就进入它的溢出桶中去遍历
        ovf := b.overflow(t)
    // 如果连溢出桶中都没有找到合适的cell，跳出循环。
        if ovf == nil {
            break
        }
        b = ovf
    }

    // 在已有的桶和溢出桶中都未找到合适的cell供key写入，那么有可能会触发以下两种情况
  // 情况一：
  // 判断当前map的装载因子是否达到设定的6.5阈值，或者当前map的溢出桶数量是否过多。如果存在这两种情况之一，则进行扩容操作。
  // hashGrow()实际并未完成扩容，对哈希表数据的搬迁（复制）操作是通过growWork()来完成的。
  // 重新跳入again逻辑，在进行完growWork()操作后，再次遍历新的桶。
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again // Growing the table invalidates everything, so try again
    }

  // 情况二：
// 在不满足情况一的条件下，会为当前桶再新建溢出桶，并将tophash，key插入到新建溢出桶的对应内存的0号位置
    if inserti == nil {
        // all current buckets are full, allocate a new one.
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        elem = add(insertk, bucketCnt*uintptr(t.keysize))
    }

  // 在插入位置存入新的key和value
    if t.indirectkey() {
        kmem := newobject(t.key)
        *(*unsafe.Pointer)(insertk) = kmem
        insertk = kmem
    }
    if t.indirectelem() {
        vmem := newobject(t.elem)
        *(*unsafe.Pointer)(elem) = vmem
    }
    typedmemmove(t.key, insertk, key)
    *inserti = top
  // map中的key数量+1
    h.count++

done:
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    h.flags &^= hashWriting
    if t.indirectelem() {
        elem = *((*unsafe.Pointer)(elem))
    }
    return elem
}
```

通过对mapassign的代码分析之后，发现该函数并没有将插入key对应的value写入对应的内存，而是返回了value应该插入的内存地址。为了弄清楚value写入内存的操作是发生在什么时候，分析如下map.go代码。

```go
package main

func main() {
    m := make(map[int]int)
    for i := 0; i < 100; i++ {
        m[i] = 666
    }
}
```

m[i] = 666对应的汇编代码

```bash
$ go tool compile -S map.go
...
        0x0098 00152 (map.go:6) LEAQ    type.map[int]int(SB), CX
        0x009f 00159 (map.go:6) MOVQ    CX, (SP)
        0x00a3 00163 (map.go:6) LEAQ    ""..autotmp_2+184(SP), DX
        0x00ab 00171 (map.go:6) MOVQ    DX, 8(SP)
        0x00b0 00176 (map.go:6) MOVQ    AX, 16(SP)
        0x00b5 00181 (map.go:6) CALL    runtime.mapassign_fast64(SB) // 调用函数runtime.mapassign_fast64，该函数实质就是mapassign（上文示例源代码是该mapassign系列的通用逻辑）
        0x00ba 00186 (map.go:6) MOVQ    24(SP), AX 24(SP), AX // 返回值，即 value 应该存放的内存地址
        0x00bf 00191 (map.go:6) MOVQ    $666, (AX) // 把 666 放入该地址中
...
```

赋值的最后一步实际上是编译器额外生成的汇编指令来完成的，可见靠 runtime 有些工作是没有做完的。所以，在go中，编译器和 runtime 配合，才能完成一些复杂的工作。同时说明，在平时学习go的源代码实现时，必要时还需要看一些汇编代码。

### 删除key

根据 key 类型的不同，删除操作会被优化成更具体的函数：

| key 类型 | 删除                                              |
| -------- | ------------------------------------------------- |
| uint32   | mapdelete_fast32(t *maptype, h *hmap, key uint32) |
| uint64   | mapdelete_fast64(t *maptype, h *hmap, key uint64) |
| string   | mapdelete_faststr(t *maptype, h *hmap, ky string) |

当然，我们只关心 `mapdelete` 函数。它首先会检查 h.flags 标志，如果发现写标位是 1，直接 panic，因为这表明有其他协程同时在进行写操作。

计算 key 的哈希，找到落入的 bucket。检查此 map 如果正在扩容的过程中，直接触发一次搬迁操作。

删除操作同样是两层循环，核心还是找到 key 的具体位置。寻找过程都是类似的，在 bucket 中挨个 cell 寻找。

找到对应位置后，对 key 或者 value 进行“清零”操作：

src/runtime/map.go的mapdelete方法相关逻辑。

```golang

// 对 key 清零
if t.indirectkey {
	*(*unsafe.Pointer)(k) = nil
} else {
	typedmemclr(t.key, k)
}

// 对 value 清零
if t.indirectvalue {
	*(*unsafe.Pointer)(v) = nil
} else {
	typedmemclr(t.elem, v)
}
```

最后，将 count 值减 1，将对应位置的 tophash 值置成 `Empty`。

### 遍历map

**结论：迭代 map 的结果是无序的**

```go
m := make(map[int]int)
    for i := 0; i < 10; i++ {
        m[i] = i
    }
    for k, v := range m {
        fmt.Println(k, v)
    }
```

运行以上代码，我们会发现每次输出顺序都是不同的。

map遍历的过程，是按序遍历bucket，同时按需遍历bucket中和其overflow bucket中的cell。但是map在扩容后，会发生key的搬迁，这造成原来落在一个bucket中的key，搬迁后，有可能会落到其他bucket中了，从这个角度看，遍历map的结果就不可能是按照原来的顺序了（详见下文的map扩容内容）。

但其实，go为了保证遍历map的结果是无序的，做了以下事情：map在遍历时，并不是从固定的0号bucket开始遍历的，每次遍历，都会从一个**随机值序号的bucket**，再从其中**随机的cell**开始遍历。然后再按照桶序遍历下去，直到回到起始桶结束。

![img](https://pic3.zhimg.com/80/v2-f418d848668c62eac039507d4f460ec2_1440w.webp)

上图的例子，是遍历一个处于未扩容状态的map。如果map正处于扩容状态时，需要先判断当前遍历bucket是否已经完成搬迁，如果数据还在老的bucket，那么就去老bucket中拿数据。

注意：在下文中会讲解到增量扩容和等量扩容。当发生了增量扩容时，一个老的bucket数据可能会分裂到两个不同的bucket中去，那么此时，如果需要从老的bucket中遍历数据，例如1号，则不能将老1号bucket中的数据全部取出，仅仅只能取出老 1 号 bucket 中那些在裂变之后，分配到新 1 号 bucket 中的那些 key（这个内容，请读者看完下文map扩容的讲解之后再回头理解）。

详细内容可自行查看源码src/runtime/map.go的`mapiterinit()`和`mapiternext()`方法逻辑。

这里注释一下`mapiterinit()`中随机保证的关键代码

```go
// 生成随机数
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
   r += uintptr(fastrand()) << 31
}
// 决定了从哪个随机的bucket开始
it.startBucket = r & bucketMask(h.B)
// 决定了每个bucket中随机的cell的位置
it.offset = uint8(r >> h.B & (bucketCnt - 1))
```

### map扩容

装载因子是决定哈希表是否进行扩容的关键指标。在go的map扩容中，除了**装载因子**会决定是否需要扩容，**溢出桶的数量**也是扩容的另一关键指标。

为了保证访问效率，当map将要添加、修改或删除key时，都会检查是否需要扩容，扩容实际上是以空间换时间的手段。在之前源码mapassign中，其实已经注释map扩容条件，主要是两点:

1. 判断已经达到装载因子的临界点，即元素个数 >= 桶（bucket）总数 * 6.5，这时候说明大部分的桶可能都快满了（即平均每个桶存储的键值对达到6.5个），如果插入新元素，有大概率需要挂在溢出桶（overflow bucket）上。

```go
func overLoadFactor(count int, B uint8) bool {
    return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

1. 判断溢出桶是否太多，当桶总数 < 2 ^ 15 时，如果溢出桶总数 >= 桶总数，则认为溢出桶过多。当桶总数 >= 2 ^ 15 时，直接与 2 ^ 15 比较，当溢出桶总数 >= 2 ^ 15 时，即认为溢出桶太多了。

```go
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
    if B > 15 {
        B = 15
    }
    return noverflow >= uint16(1)<<(B&15)
}
```

对于第2点，其实算是对第 1 点的补充。因为在装载因子比较小的情况下，有可能 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是桶数量多（真实分配的桶数量多，包括大量的溢出桶）。

在某些场景下，比如不断的增删，这样会造成overflow的bucket数量增多，但负载因子又不高，未达不到第 1 点的临界值，就不能触发扩容来缓解这种情况。这样会造成桶的使用率不高，值存储得比较稀疏，查找插入效率会变得非常低，因此有了第 2 点判断指标。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

![img](https://pic3.zhimg.com/80/v2-33636572ffb353268b2162940514a1ce_1440w.webp)

如上图所示，由于对map的不断增删，以0号bucket为例，该桶链中就造成了大量的稀疏桶。

两种情况官方采用了不同的解决方案

- 针对 1，将 B + 1，新建一个buckets数组，新的buckets大小是原来的2倍，然后旧buckets数据搬迁到新的buckets。该方法我们称之为**增量扩容**。
- 针对 2，并不扩大容量，buckets数量维持不变，重新做一遍类似增量扩容的搬迁动作，把松散的键值对重新排列一次，以使bucket的使用率更高，进而保证更快的存取。该方法我们称之为**等量扩容**。

对于 2 的解决方案，其实存在一个极端的情况：如果插入 map 的 key 哈希都一样，那么它们就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 `O(n)`。但 Go 的每一个 map 都会在初始化阶段的 makemap时定一个随机的哈希种子，所以要构造这种冲突是没那么容易的。

在源码中，和扩容相关的主要是`hashGrow()`函数与`growWork()`函数。`hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在`mapassign()` 和 `mapdelete()` 函数中。也就是插入（包括修改）、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。它们会先检查 oldbuckets 是否搬迁完毕（检查 oldbuckets 是否为 nil），再决定是否进行搬迁工作。

`hashGrow()`函数

```go
func hashGrow(t *maptype, h *hmap) {
  // 如果达到条件 1，那么将B值加1，相当于是原来的2倍
  // 否则对应条件 2，进行等量扩容，所以 B 不变
    bigger := uint8(1)
    if !overLoadFactor(h.count+1, h.B) {
        bigger = 0
        h.flags |= sameSizeGrow
    }
  // 记录老的buckets
    oldbuckets := h.buckets
  // 申请新的buckets空间
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
  // 注意&^ 运算符，这块代码的逻辑是转移标志位
    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
    // 提交grow (atomic wrt gc)
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
  // 搬迁进度为0
    h.nevacuate = 0
  // overflow buckets 数为0
    h.noverflow = 0

  // 如果发现hmap是通过extra字段 来存储 overflow buckets时
    if h.extra != nil && h.extra.overflow != nil {
        if h.extra.oldoverflow != nil {
            throw("oldoverflow is not nil")
        }
        h.extra.oldoverflow = h.extra.overflow
        h.extra.overflow = nil
    }
    if nextOverflow != nil {
        if h.extra == nil {
            h.extra = new(mapextra)
        }
        h.extra.nextOverflow = nextOverflow
    }
}
```

`growWork()`函数

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
  // 为了确认搬迁的 bucket 是我们正在使用的 bucket
  // 即如果当前key映射到老的bucket1，那么就搬迁该bucket1。
    evacuate(t, h, bucket&h.oldbucketmask())

    // 如果还未完成扩容工作，则再搬迁一个bucket。
    if h.growing() {
        evacuate(t, h, h.nevacuate)
    }
}
```

从`growWork()`函数可以知道，搬迁的核心逻辑是`evacuate()`函数。这里读者可以思考一个问题：为什么每次至多搬迁2个bucket？这其实是一种性能考量，如果map存储了数以亿计的key-value，一次性搬迁将会造成比较大的延时，因此才采用逐步搬迁策略。

在讲解该逻辑之前，需要读者先理解以下两个知识点。

- 知识点1：bucket序号的变化

前面讲到，增量扩容（条件1）和等量扩容（条件2）都需要进行bucket的搬迁工作。对于等量扩容而言，由于buckets的数量不变，因此可以按照序号来搬迁。例如老的的0号bucket，仍然搬至新的0号bucket中。

![img](https://pic4.zhimg.com/80/v2-9d60c8b3096e2dab7a87c64bf349097b_1440w.webp)

但是，对于增量扩容而言，就会有所不同。例如原来的B=5，那么增量扩容时，B就会变成6。那么决定key值落入哪个bucket的低位哈希值就会发生变化（从取5位变为取6位），取新的低位hash值得过程称为rehash。

![img](https://pic4.zhimg.com/80/v2-15b4f455a970e9d3406cd47765bcca8b_1440w.webp)

因此，在增量扩容中，某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于低 hash 值第倒数第B+1位是 0 还是 1。

![img](https://pic2.zhimg.com/80/v2-c50f4e8f7e2769313b4f2d680fa91925_1440w.webp)

如上图所示，当原始的B = 3时，旧buckets数组长度为8，在编号为2的bucket中，其2号cell和5号cell，它们的低3位哈希值相同（不相同的话，也就不会落在同一个桶中了），但是它们的低4位分别是0010、1010。当发生了增量扩容，2号就会被搬迁到新buckets数组的2号bucket中去，5号被搬迁到新buckets数组的10号bucket中去，它们的桶号差距是2的3次方。

- 知识点2：确定搬迁区间

在源码中，有bucket x 和bucket y的概念，其实就是增量扩容到原来的 2 倍，桶的数量是原来的 2 倍，前一半桶被称为bucket x，后一半桶被称为bucket y。一个 bucket 中的 key 可能会分裂到两个桶中去，分别位于bucket x的桶，或bucket y中的桶。所以在搬迁一个 cell 之前，需要知道这个 cell 中的 key 是落到哪个区间（而对于同一个桶而言，搬迁到bucket x和bucket y桶序号的差别是老的buckets大小，即2^old_B）。

这里留一个问题：为什么确定key落在哪个区间很重要？

![img](https://pic1.zhimg.com/80/v2-197a831e09d60dd04566163be0458b5c_1440w.webp)

确定了要搬迁到的目标 bucket 后，搬迁操作就比较好进行了。将源 key/value 值 copy 到目的地相应的位置。设置 key 在原始 buckets 的 tophash 为 `evacuatedX` 或是 `evacuatedY`，表示已经搬迁到了新 map 的bucket x或是bucket y，新 map 的 tophash 则正常取 key 哈希值的高 8 位。

下面正式解读搬迁核心代码`evacuate()`函数。

`evacuate()`函数

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  // 首先定位老的bucket的地址
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
  // newbit代表扩容之前老的bucket个数
    newbit := h.noldbuckets()
  // 判断该bucket是否已经被搬迁
    if !evacuated(b) {
    // 官方TODO，后续版本也许会实现
        // TODO: reuse overflow buckets instead of using new ones, if there
        // is no iterator using the old buckets.  (If !oldIterator.)

    // xy 包含了高低区间的搬迁目的地内存信息
    // x.b 是对应的搬迁目的桶
    // x.k 是指向对应目的桶中存储当前key的内存地址
    // x.e 是指向对应目的桶中存储当前value的内存地址
        var xy [2]evacDst
        x := &xy[0]
        x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        x.k = add(unsafe.Pointer(x.b), dataOffset)
        x.e = add(x.k, bucketCnt*uintptr(t.keysize))

    // 只有当增量扩容时才计算bucket y的相关信息（和后续计算useY相呼应）
        if !h.sameSizeGrow() {
            y := &xy[1]
            y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
            y.k = add(unsafe.Pointer(y.b), dataOffset)
            y.e = add(y.k, bucketCnt*uintptr(t.keysize))
        }

    // evacuate 函数每次只完成一个 bucket 的搬迁工作，因此要遍历完此 bucket 的所有的 cell，将有值的 cell copy 到新的地方。
    // bucket 还会链接 overflow bucket，它们同样需要搬迁。
    // 因此同样会有 2 层循环，外层遍历 bucket 和 overflow bucket，内层遍历 bucket 的所有 cell。

    // 遍历当前桶bucket和其之后的溢出桶overflow bucket
    // 注意：初始的b是待搬迁的老bucket
        for ; b != nil; b = b.overflow(t) {
            k := add(unsafe.Pointer(b), dataOffset)
            e := add(k, bucketCnt*uintptr(t.keysize))
      // 遍历桶中的cell，i，k，e分别用于对应tophash，key和value
            for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
                top := b.tophash[i]
        // 如果当前cell的tophash值是emptyOne或者emptyRest，则代表此cell没有key。并将其标记为evacuatedEmpty，表示它“已经被搬迁”。
                if isEmpty(top) {
                    b.tophash[i] = evacuatedEmpty
                    continue
                }
        // 正常不会出现这种情况
        // 未被搬迁的 cell 只可能是emptyOne、emptyRest或是正常的 top hash（大于等于 minTopHash）
                if top < minTopHash {
                    throw("bad map state")
                }
                k2 := k
        // 如果 key 是指针，则解引用
                if t.indirectkey() {
                    k2 = *((*unsafe.Pointer)(k2))
                }
                var useY uint8
        // 如果是增量扩容
                if !h.sameSizeGrow() {
          // 计算哈希值，判断当前key和vale是要被搬迁到bucket x还是bucket y
                    hash := t.hasher(k2, uintptr(h.hash0))
                    if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
            // 有一个特殊情况：有一种 key，每次对它计算 hash，得到的结果都不一样。
            // 这个 key 就是 math.NaN() 的结果，它的含义是 not a number，类型是 float64。
            // 当它作为 map 的 key时，会遇到一个问题：再次计算它的哈希值和它当初插入 map 时的计算出来的哈希值不一样！
            // 这个 key 是永远不会被 Get 操作获取的！当使用 m[math.NaN()] 语句的时候，是查不出来结果的。
            // 这个 key 只有在遍历整个 map 的时候，才能被找到。
            // 并且，可以向一个 map 插入多个数量的 math.NaN() 作为 key，它们并不会被互相覆盖。
            // 当搬迁碰到 math.NaN() 的 key 时，只通过 tophash 的最低位决定分配到 X part 还是 Y part（如果扩容后是原来 buckets 数量的 2 倍）。如果 tophash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。
                        useY = top & 1
                        top = tophash(hash)
          // 对于正常key，进入以下else逻辑  
                    } else {
                        if hash&newbit != 0 {
                            useY = 1
                        }
                    }
                }

                if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
                    throw("bad evacuatedN")
                }

        // evacuatedX + 1 == evacuatedY
                b.tophash[i] = evacuatedX + useY
        // useY要么为0，要么为1。这里就是选取在bucket x的起始内存位置，或者选择在bucket y的起始内存位置（只有增量同步才会有这个选择可能）。
                dst := &xy[useY]

        // 如果目的地的桶已经装满了（8个cell），那么需要新建一个溢出桶，继续搬迁到溢出桶上去。
                if dst.i == bucketCnt {
                    dst.b = h.newoverflow(t, dst.b)
                    dst.i = 0
                    dst.k = add(unsafe.Pointer(dst.b), dataOffset)
                    dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
                }
                dst.b.tophash[dst.i&(bucketCnt-1)] = top
        // 如果待搬迁的key是指针，则复制指针过去
                if t.indirectkey() {
                    *(*unsafe.Pointer)(dst.k) = k2 // copy pointer
        // 如果待搬迁的key是值，则复制值过去  
                } else {
                    typedmemmove(t.key, dst.k, k) // copy elem
                }
        // value和key同理
                if t.indirectelem() {
                    *(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
                } else {
                    typedmemmove(t.elem, dst.e, e)
                }
        // 将当前搬迁目的桶的记录key/value的索引值（也可以理解为cell的索引值）加一
                dst.i++
        // 由于桶的内存布局中在最后还有overflow的指针，多以这里不用担心更新有可能会超出key和value数组的指针地址。
                dst.k = add(dst.k, uintptr(t.keysize))
                dst.e = add(dst.e, uintptr(t.elemsize))
            }
        }
    // 如果没有协程在使用老的桶，就对老的桶进行清理，用于帮助gc
        if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
            b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
      // 只清除bucket 的 key,value 部分，保留 top hash 部分，指示搬迁状态
            ptr := add(b, dataOffset)
            n := uintptr(t.bucketsize) - dataOffset
            memclrHasPointers(ptr, n)
        }
    }

  // 用于更新搬迁进度
    if oldbucket == h.nevacuate {
        advanceEvacuationMark(h, t, newbit)
    }
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
  // 搬迁桶的进度加一
    h.nevacuate++
  // 实验表明，1024至少会比newbit高出一个数量级（newbit代表扩容之前老的bucket个数）。所以，用当前进度加上1024用于确保O(1)行为。
    stop := h.nevacuate + 1024
    if stop > newbit {
        stop = newbit
    }
  // 计算已经搬迁完的桶数
    for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
        h.nevacuate++
    }
  // 如果h.nevacuate == newbit，则代表所有的桶都已经搬迁完毕
    if h.nevacuate == newbit {
    // 搬迁完毕，所以指向老的buckets的指针置为nil
        h.oldbuckets = nil
    // 在讲解hmap的结构中，有过说明。如果key和value均不包含指针，则都可以inline。
    // 那么保存它们的buckets数组其实是挂在hmap.extra中的。所以，这种情况下，其实我们是搬迁的extra的buckets数组。
    // 因此，在这种情况下，需要在搬迁完毕后，将hmap.extra.oldoverflow指针置为nil。
        if h.extra != nil {
            h.extra.oldoverflow = nil
        }
    // 最后，清除正在扩容的标志位，扩容完毕。
        h.flags &^= sameSizeGrow
    }
}
```

代码比较长，但是文中注释已经比较清晰了，如果对map的扩容还不清楚，可以参见以下图解。

![img](https://pic2.zhimg.com/80/v2-3df0b0907c1a309a58c7d40d0fc41a59_1440w.webp)

针对上图的map，其B为3，所以原始buckets数组为8。当map元素数变多，加载因子超过6.5，所以引起了增量扩容。

以3号bucket为例，可以看到，由于B值加1，所以在新选取桶时，需要取低4位哈希值，这样就会造成cell会被搬迁到新buckets数组中不同的桶（3号或11号桶）中去。注意，在一个桶中，搬迁cell的工作是有序的：它们是依序填进对应新桶的cell中去的。

当然，实际情况中3号桶很可能还有溢出桶，在这里为了简化绘图，假设3号桶没有溢出桶，如果有溢出桶，则相应地添加到新的3号桶和11号桶中即可，如果对应的3号和11号桶均装满，则给新的桶添加溢出桶来装载。

![img](https://pic2.zhimg.com/80/v2-47aafdcac1db08845444e14f55adb3a5_1440w.webp)

对于上图的map，其B也为3。假设整个map中的overflow过多，触发了等量扩容。注意，等量扩容时，新的buckets数组大小和旧buckets数组是一样的。

以6号桶为例，它有一个bucket和3个overflow buckets，但是我们能够发现桶里的数据非常稀疏，等量扩容的目的就是为了把松散的键值对重新排列一次，以使bucket的使用率更高，进而保证更快的存取。搬迁完毕后，新的6号桶中只有一个基础bucket，暂时并不需要溢出桶。这样，和原6号桶相比，数据变得紧密，使后续的数据存取变快。