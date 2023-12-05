#                                  Golang接口的底层实现

## 前言

接口是高级语言中的一个规约，是一组方法签名的集合。在Golang中， 接口是非侵入式的，具体类型实现 interface 不需要在语法上显式的声明，只需要具体类型的方法集合是 interface 方法集合的超集，就表示该类实现了这一 interface。编译器在编译时会进行 interface 校验。interface 和具体类型不同，它不能实现具体逻辑，也不能定义字段。



## 底层数据结构

Golang中接口的底层结构体有两种，分别是`iface` 和 `eface`，其中`iface` 描述的接口包含方法，而 `eface` 则是不包含任何方法的空接口：`interface{}`。

### iface源码

```golang
type iface struct {
	tab  *itab  // 存放类型、方法等信息
	data unsafe.Pointer  //  指针指向的 iface 绑定对象的原始数据的副本
}

type itab struct {
	inter  *interfacetype
	_type  *_type
	link   *itab
	hash   uint32 // copy of _type.hash. Used for type switches.
	bad    bool   // type does not implement interface
	inhash bool   // has this itab been added to hash?
	unused [2]byte
	fun    [1]uintptr // variable sized
}
```

`iface` 内部维护两个指针，tab 中存放的是类型、方法等信息。data 指针指向的 iface 绑定对象的原始数据的副本。这里同样遵循 Go 的统一规则，值传递。tab 是 itab 类型的指针。

itab 中包含 5 个字段。inner 存的是 interface 自己的静态类型。_type 存的是 interface 对应具体对象的类型。itab 中的 _type 和 iface 中的 data 能简要描述一个变量。_type 是这个变量对应的类型，data 是这个变量的值。这里的 hash 字段和 _type 中存的 hash 字段是完全一致的，这么做的目的是为了类型断言(下文会提到)。fun 是一个函数指针，它指向的是具体类型的函数方法。虽然这里只有一个函数指针，但是它可以调用很多方法。在这个指针对应内存地址的后面依次存储了多个方法，利用指针偏移便可以找到它们。

由于 Go 语言是强类型语言，编译时对每个变量的类型信息做强校验，所以每个类型的元信息要用一个结构体描述。再者 Go 的反射也是基于类型的元信息实现的。_type 就是所有类型最原始的元信息。

整体结构如下：

![iface 结构体全景](https://golang.design/go-questions/interface/assets/0.png)



### eface源码

```golang
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

eface作为空的 inferface{} 是没有方法集的接口。所以不需要 itab 数据结构。它只需要存类型和类型对应的值即可。对应的数据结构如下：

从这个数据结构可以看出，只有当 2 个字段都为 nil，空接口才为 nil。空接口的主要目的有 2 个，一是实现“泛型”，二是使用反射。**所以空接口并不等于nil**，这是常见的犯错点。



### 关于type的结构体

```golang
type _type struct {
    // 类型大小
	size       uintptr
    ptrdata    uintptr
    // 类型的 hash 值
    hash       uint32
    // 类型的 flag，和反射相关
    tflag      tflag
    // 内存对齐相关
    align      uint8
    fieldalign uint8
    // 类型的编号，有bool, slice, struct 等等等等
	kind       uint8
	alg        *typeAlg
	// gc 相关
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```

Go 语言各种数据类型都是在 `_type` 字段的基础上，增加一些额外的字段来进行管理的，这些数据类型的结构体定义，也是反射实现的基础。

```golang
//数组类型
type arraytype struct {
	typ   _type
	elem  *_type
	slice *_type
	len   uintptr
}

//通道类型
type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}

//切片类型
type slicetype struct {
	typ  _type
	elem *_type
}

//结构体类型
type structtype struct {
	typ     _type
	pkgPath name
	fields  []structfield
}
```





## 参考链接

1.[iface与eface的区别是什么](https://golang.design/go-questions/interface/iface-eface/)

2.[深入研究 Go interface 底层实现](https://halfrost.com/go_interface/)