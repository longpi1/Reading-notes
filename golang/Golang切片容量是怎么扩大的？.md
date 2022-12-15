#                                  Golang 切片容量是怎么扩大的？

## 前言

在Go中，数组就是一片连续的内存， slice 实际上是一个结构体，包含三个字段：长度、容量、底层数组。

```golang
// runtime/slice.go
type slice struct {
	array unsafe.Pointer // 元素指针
	len   int // 长度 
	cap   int // 容量
}
```

切片的数据结构如下：

![](https://golang.design/go-questions/slice/assets/0.png)

切片的数据结构中，包含一个指向数组的指针 `array` ，当前长度 `len` ，以及最大容量 `cap` 。在使用 `make([]int, len)` 创建切片时，实际上还有第三个可选参数 `cap` ，也即 `make([]int, len, cap)` 。在不声明 `cap` 的情况下，默认 `cap=len` 。当切片长度没有超过容量时，对切片新增数据，不会改变 `array` 指针的值。

当对切片进行 `append` 操作，导致长度超出容量时，就会创建新的数组，这会导致和原有切片的分离。

需要注意的是，底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。为了避免因为切片是否发生扩容的问题导致bug，最好的处理办法还是在必要时使用 `copy` 来复制数据，保证得到一个新的切片，以避免后续操作带来预料之外的副作用。

下面看看用 `a := []int{}` 这种方式来创建切片会是什么情况。

```
a := []int{}
for i := 0; i < 16; i++ {
    a = append(a, i)
    fmt.Print(cap(a), " ")
}
// 1 2 4 4 8 8 8 8 16 16 16 16 16 16 16 16
```

可以看到，空切片的容量为0，但后面向切片中添加元素时，并不是每次切片的容量都发生了变化。这是因为，如果增大容量，也即需要创建新数组，这时还需要将原数组中的所有元素复制到新数组中，开销很大，所以GoLang设计了一套扩容机制，以减少需要创建新数组的次数。但这导致无法很直接地判断 `append` 时是否创建了新数组。

## Golang的扩容机制

### 原理

使用 append 可以向 slice 追加元素，实际上是往底层数组添加元素。但是底层数组的长度是固定的，如果索引 `len-1` 所指向的元素已经是底层数组的最后一个元素，就没法再添加了。

这时，slice 会迁移到新的内存位置，新底层数组的长度也会增加，这样就可以放置新增的元素。同时，为了应对未来可能再次发生的 append 操作，新的底层数组的长度，也就是新 `slice` 的容量是留了一定的 `buffer` 的。否则，每次添加元素的时候，都会发生迁移，成本太高。

新 slice 预留的 `buffer` 大小是有一定规律的。在golang1.18版本更新之前网上大多数的文章都是这样描述slice的扩容策略的：

> 当原 slice 容量小于 `1024` 的时候，新 slice 容量变成原来的 `2` 倍；原 slice 容量超过 `1024`，新 slice 容量变成原来的`1.25`倍。

在1.18版本更新之后，slice的扩容策略变为了：

> 当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍；原slice容量超过256，新slice容量newcap = oldcap+(oldcap+3*256)/4

### 源码分析

**golang版本1.9**

```golang
// go 1.9 src/runtime/slice.go:82
func growslice(et *_type, old slice, cap int) slice {
    // ……
    newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			for newcap < cap {
				newcap += newcap / 4
			}
		}
	}
	// ……
	
	capmem = roundupsize(uintptr(newcap) * ptrSize)
	newcap = int(capmem / ptrSize)
}
```

**golang版本1.18**

```golang
// go 1.18 src/runtime/slice.go:178
func growslice(et *_type, old slice, cap int) slice {
    // ……
    newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		const threshold = 256
		if old.cap < threshold {
			newcap = doublecap
		} else {
			for 0 < newcap && newcap < cap {
                // Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
	// ……
    
	capmem = roundupsize(uintptr(newcap) * ptrSize)
	newcap = int(capmem / ptrSize)
}
```

## 参考链接

1. 切片的容量是怎么增长的https://golang.design/go-questions/slice/grow/
2. GoLang中的切片扩容机制，https://studygolang.com/articles/21396