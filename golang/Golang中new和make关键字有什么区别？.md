#               Golang中new和make关键字有什么区别？

## 前言

在golang中， 我们经常使用new 和 make 来创建并分配相应类型的内存空间。在我们定义变量的时候，那他们有什么区别呢？其实他们的规则很简单，new 对相应类型进行内存分配，而 make 只能用于 slice、map 和 channel 的初始化，下面我们就来详细介绍一下。



## new

在golang中，内置函数 `new` 可以对类型进行内存分配。**其返回值是所创建类型的指针引用**。

代码示例如下：

```go
type T struct {
	Name string
    Age  int
}

func main() {
	t := new(T)
	t.Name = "test"
    t.Age = 18
}
```

 `new` 函数在日常工程代码中是比较少见的，我们通常直接用 `T{}` 来进行初始化，这种初始化方式更方便。



## make

在golang中，内置函数 `make` 仅支持 `slice`、`map`、`channel` 三种数据类型的内存创建，**其返回值是所创建类型的本身，而不是新的指针引用**。不仅可以开辟一个内存，还能给这个内存的类型初始化其零值。

代码示例如下：

```go
func main() {
	s := make([]int, 1, 5)
	m := make(map[int]bool, 5)
	ch := make(chan int, 1)
}
```



## 小结

- make和new都是golang用来分配内存的內建函数，且在堆上分配内存，make 即分配内存，也初始化内存。new只是将内存清零，并没有初始化内存。
- make返回的创建类型的本身；而new返回的是指向类型的指针。
- make只能用来分配及初始化类型为slice，map，channel的数据；new可以分配任意类型的数据。



