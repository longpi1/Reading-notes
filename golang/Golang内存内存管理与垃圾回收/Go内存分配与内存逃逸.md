# Go内存分配与内存逃逸

## 内存为什么需要管理

当存储的东西越来越多，也就发现物理内存的容量依然是不够用，那么对物理内存的利用率和合理的分配，管理就变得非常的重要。

（1）操作系统就会对内存进行非常详细的管理。

（2）基于操作系统的基础上，不同语言的内存管理机制也应允而生，有的一些语言并没有提供自动的内存管理模式，有的语言就已经提供了自身程序的内存管理模式，如表2所示。

###### 表2 自动与非自动内存管理的语言

| **内存自动管理的语言（部分）** | **内存非自动管理的语言（部分）** |
| ------------------------------ | -------------------------------- |
| Golang                         | C                                |
| Java                           | C++                              |
| Python                         | Rust                             |

所以为了降低内存管理的难度，像C、C++这样的编程语言会完全将分配和回收内存的权限交给开发者，而Rust则是通过生命周期限定开发者对非法权限内存的访问来自动回收，因而并没有提供自动管理的一套机制。但是像Golang、Java、Python这类为了完全让开发则关注代码逻辑本身，语言层提供了一套管理模式。因为Golang编程语言给开发者提供了一套内存管理模式，所以开发者有必要了解一下Golang做了哪些助力的功能。

在理解Golang语言层内存管理之前，应先了解操作系统针对物理内存做了哪些管理的方式。当插上内存条之后，通过操作系统是如何将软件存放在这个绿色的物理内存条中去的。



## **为什么需要关心内存分配问题**

------

每个工程师的时间都如此宝贵，在继续读这篇文章之前，需要你先回答几个问题，如果得到的答案是否定的，那可能本文章里写的内容对你并没有什么帮助。但是，如果你遇到了因内存分配而导致的性能问题，可能这篇文章能带你理解 Golang 的内存分配的冰山一角，带你入个门。

问题如下：

- 你的程序是性能敏感型吗？
- GC 带来的延迟影响到了你的程序性能吗？
- 你的程序有过多的堆内存分配吗？

如果你命中上面问题的其中一个或两个，那这篇文章适合你继续读下去。或你根本不知道如何回答这些问题，可能去了解下 go 性能观测相关的知识（pprof 的使用等）对你更有帮助。

**下面正文开始。**



## **Golang 简要内存划分**

------

![img](https://ask.qcloudimg.com/http-save/5469577/b1e2510bc404791b9a0909da0a0f1a99.webp?imageView2/2/w/1620/format/jpg)

可以简单的认为 Golang 程序在启动时，会向操作系统申请一定区域的内存，分为栈（Stack）和堆（Heap）。栈内存会随着函数的调用分配和回收；堆内存由程序申请分配，由垃圾回收器（Garbage Collector）负责回收。性能上，栈内存的使用和回收更迅速一些；尽管Golang 的 GC 很高效，但也不可避免的会带来一些性能损耗。因此，Go 优先使用栈内存进行内存分配。在不得不将对象分配到堆上时，才将特定的对象放到堆中。



## **内存分配过程分析**

------

本部分，将以代码的形式，分别介绍栈内存分配、指针作为参数情况下的栈内存分配、指针作为返回值情况下的栈内存分配并逐步引出逃逸分析和几个内存逃逸的基本原则。

正文开始，Talk is cheap，show me the code。

## **栈内存分配**

我将以一段简单的代码作为示例，分析这段代码的内存分配过程。

```javascript
package main
import "fmt"
func main() {  n := 4  n2 := square(n)  fmt.Println(n2)}
func square(n int) int{  return n * n}
```

复制

代码的功能很简单，一个 main 函数作为程序入口，定义了一个变量n，定义了另一个函数 squire ，返回乘方操作后的 int 值。最后，将返回的值打印到控制台。程序输出为16。

下面开始逐行进行分析，解析调用时，go 运行时是如何对内存进行分配的。

![img](https://ask.qcloudimg.com/http-save/5469577/479e83aa67b17d920bd71f6625afe1c9.webp?imageView2/2/w/1620/format/jpg)

当代码运行到第6行，进入 main 函数时，会在栈上创建一个 Stack frame，存放本函数中的变量信息。包括函数名称，变量等。

![img](https://ask.qcloudimg.com/developer-images/article/5469577/h4jok0khik.png?imageView2/2/w/1620)

当代码运行到第7行时，go 会在栈中压入一个新的 Stack Frame，用于存放调用 square 函数的信息；包括函数名、变量 n 的值等。此时，计算4 * 4 的值，并返回。

![img](https://ask.qcloudimg.com/http-save/5469577/e595bc8e8421e68646c147ea1cb411ab.webp?imageView2/2/w/1620/format/jpg)

当 square 函数调用完成，返回16到 main 函数后，将16赋值给 n2变量。注意，原来的 stack frame 并不会被 go 清理掉，而是如栈左侧的箭头所示，被标记为不合法。上图夹在红色箭头和绿色箭头之间的横线可以理解为 go 汇编代码中的 SP 栈寄存器的值，当程序申请或释放栈内存时，只需要修改 SP 寄存器的值，这种栈内存分配方式省掉了清理栈内存空间的耗时【1】。



![img](https://ask.qcloudimg.com/developer-images/article/5469577/m6ls2qtdbl.png?imageView2/2/w/1620)

接下来，调用 fmt.Println 时，SP 寄存器的值会进一步增加，覆盖掉原来 square 函数的 stack frame，完成 print 后，程序正常退出。

## **指针作为参数情况下的栈内存分配**

还是同样的过程，看如下这段代码。

```javascript
package main
import "fmt"
func main() {  n := 4  increase(&n)  fmt.Println(n)}
func increase(i *int) {  *i++}
```

main 作为程序入口，声明了一个变量 n，赋值为4。声明了一个函数  increase，使用一个 int 类型的指针 i 作为参数，increase 函数内，对指针 i 对应的值进行自增操作。最后 main 函数中打印了 n 的值。程序输出为5。

![img](https://ask.qcloudimg.com/developer-images/article/5469577/7snaopjz26.png?imageView2/2/w/1620)

当程序运行到 main 函数的第6行时，go 在栈上分配了一个 stack frame ，对变量 n 进行了赋值，n 在内存中对应的地址为0xc0008771，此时程序将继续向下执行，调用 increase 函数。

![img](https://ask.qcloudimg.com/developer-images/article/5469577/nzbfzfdkup.png?imageView2/2/w/1620)

这时，increase 函数对应的 stack fream 被创建，i 被赋值为变量 n对应的地址值0xc0008771，然后进行自增操作。

![img](https://ask.qcloudimg.com/developer-images/article/5469577/u14njxdglz.png?imageView2/2/w/1620)

当 increase 函数运行结束后，SP 寄存器会上移，将之前分配的 stack freme 标记为不合法。此时，程序运行正常，并没有因为 SP 寄存器的改动而影响程序的正确性，内存中的值也被正确的修改了。

## **指针作为返回值情况下的栈内存分配**

文章之前的部分分别介绍了普通变量作为参数和将指针作为参数情况下的栈内存使用，本部分来介绍将指针作为返回值，返回给调用方的情况下，内存是如何分配的，并引出内存逃逸相关内容。来看这段代码：

```javascript
package main
import "fmt"
func main() {  n := initValue()  fmt.Println(*n/2)}
func initValue() *int {  i := 4  return &i}
```

main 函数中，调用了 initValue 函数，该函数返回一个 int 指针并赋值给 n，指针对应的值为4。随后，main 函数调用 fmt.Println 打印了指针 n / 2对应的值。程序输出为2。

![img](https://ask.qcloudimg.com/developer-images/article/5469577/945gf1ih2a.png?imageView2/2/w/1620)

程序调用 initValue 后，将 i 的地址赋值给变量 n 。注意，如果这时，变量 i 的位置在栈上，则可能会随时被覆盖掉。

![img](https://ask.qcloudimg.com/developer-images/article/5469577/8ydk3bi2lv.png?imageView2/2/w/1620)

在调用 fmt.Println 时，Stack Frame 会被重新创建，变量 i 被赋值为*n/2也就是2，会覆盖掉原来 n 所指向的变量值。这会导致及其严重的问题。在面对 sharing up 场景时，go 通常会将变量分配到堆中，如下图所示：

![img](https://ask.qcloudimg.com/http-save/5469577/16ab1617ecd4832e038e07e9a612fd69.webp?imageView2/2/w/1620/format/jpg)

通过上面的分析，可以看到在面对被调用的函数返回一个指针类型时将对象分配到栈上会带来严重的问题，因此 Go 将变量分配到了堆上。这种分配方式保证了程序的安全性，但也不可避免的增加了堆内存创建，并需要在将来的某个时候，需要 GC 将不再使用的内存清理掉。



## **内存分配原则**

------

经过上述分析，可以简单的归纳几条原则。

- Sharing down typically stays on the stack 在调用方创建的变量或对象，通过参数的形式传递给被调用函数，这时，在调用方创建的内存空间通常在栈上。这种在调用方创建内存，在被调用方使用该内存的“内存共享”方式，称之为 Sharing down。
- Sharing up typically escapes to the heap 在被调用函数内创建的对象，以指针的形式返回给调用方的情况下，通常，创建的内存空间在堆上。这种在被调用方创建，在调用方使用的“内存共享”方式，称之为 Sharing up。
- Only the compiler knows 之所以上面两条原则都加了通常，因为具体的分配方式，是由编译器确定的，一些编译器后端优化，可能会突破这两个原则，因此，具体的分配逻辑，只有编译器（或开发编译器的人）知道。



## **使用 go build 命令确定内存逃逸情况**

------

值得注意的是，Go 在判断一个变量或对象是否需要逃逸到堆的操作，是在编译器完成的；也就是说，当代码写好后，经过编译器编译后，会在二进制中进行特定的标注，声明指定的变量要被分配到堆或栈。可以使用如下命令在编译期打印出内存分配逻辑，来具体获知特定变量或对象的内存分配位置。

查看 go help 可以看到 go build 其实是在调用 go tool compile。

```javascript
go help build ... -gcflags '[pattern=]arg list'        arguments to pass on each go tool compile invocation....
```

```javascript
go tool compile -h...-m    print optimization decisions...-l    disable inlining...
```

其中，需要关心的参数有两个，

- -m 显示优化决策
- -l 禁止使用内联【2】

代码如下：

```javascript
package main
func main() {  n := initValue()  println(*n / 2)
  o := initObj()  println(o)
  f := initFn()  println(f)
  num := 5  result := add(num)  println(result)}
func initValue() *int {  i := 3                // ./main.go:19:2: moved to heap: i  return &i}
type Obj struct {  i int}
func initObj() *Obj {  return &Obj{i: 3}      // ./main.go:28:9: &Obj literal escapes to heap}
func initFn() func() {  return func() {       // ./main.go:32:9: func literal escapes to heap    println("I am a function")  }}
func add(i int) int {  return i + 1}
```

完整的构建命令和输出如下：

```javascript
go build -gcflags="-m -l" 
# _/Users/rocket/workspace/stack-or-heap./main.go:19:2: moved to heap: i./main.go:24:9: &Obj literal escapes to heap./main.go:28:9: func literal escapes to heap
```

可以看到，sharing up 的情况（initValue，initObj，initFn）内存空间被分配到了堆上。sharing down 的情况（add）内存空间在栈上。

这里给读者留个问题，大家可以研究下 moved to heap 和 escapes to heap 的区别。



# 内存逃逸

## **怎么答** 

`golang程序变量`会携带有一组校验数据，用来证明它的整个生命周期是否在运行时完全可知。如果变量通过了这些校验，它就可以在`栈上`分配。否则就说它 `逃逸` 了，必须在`堆上分配`。

能引起变量逃逸到堆上的**典型情况**：

- **在方法内把局部变量指针返回** 局部变量原本应该在栈中分配，在栈中回收。但是由于返回时被外部引用，因此其生命周期大于栈，则溢出。
- **发送指针或带有指针的值到 channel 中。** 在编译时，是没有办法知道哪个 [goroutine](https://www.zhihu.com/search?q=goroutine&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"145468000"}) 会在 channel 上接收数据。所以编译器没法知道变量什么时候才会被释放。
- **在一个切片上存储指针或带指针的值。** 一个典型的例子就是 []*string 。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。
- **slice 的背后数组被重新分配了，因为 append 时可能会超出其容量( cap )。** slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。
- **在 interface 类型上调用方法。** 在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r , 调用 r.Read(b) 会使得 r 的值和切片b 的背后存储都逃逸掉，所以会在堆上分配。

## **举例** 

- 通过一个例子加深理解，接下来尝试下怎么通过 `go build -gcflags=-m` 查看逃逸的情况。

```go
package main
import "fmt"
type A struct {
 s string
}
// 这是上面提到的 "在方法内把局部变量指针返回" 的情况
func foo(s string) *A {
 a := new(A) 
 a.s = s
 return a //返回局部变量a,在C语言中妥妥野指针，但在go则ok，但a会逃逸到堆
}
func main() {
 a := foo("hello")
 b := a.s + " world"
 c := b + "!"
 fmt.Println(c)
}
```

执行`go build -gcflags=-m main.go`

```go
go build -gcflags=-m main.go
# command-line-arguments
./main.go:7:6: can inline foo
./main.go:13:10: inlining call to foo
./main.go:16:13: inlining call to fmt.Println
/var/folders/45/qx9lfw2s2zzgvhzg3mtzkwzc0000gn/T/go-build409982591/b001/_gomod_.go:6:6: can inline init.0
./main.go:7:10: leaking param: s
./main.go:8:10: new(A) escapes to heap
./main.go:16:13: io.Writer(os.Stdout) escapes to heap
./main.go:16:13: c escapes to heap
./main.go:15:9: b + "!" escapes to heap
./main.go:13:10: main new(A) does not escape
./main.go:14:11: main a.s + " world" does not escape
./main.go:16:13: main []interface {} literal does not escape
<autogenerated>:1: os.(*File).close .this does not escape
```

- `./main.go:8:10: new(A) escapes to heap` 说明 `new(A)` 逃逸了,符合上述提到的常见情况中的第一种。
- `./main.go:14:11: main a.s + " world" does not escape` 说明 `b` 变量没有逃逸，因为它只在方法内存在，会在方法结束时被回收。
- `./main.go:15:9: b + "!" escapes to heap` 说明 `c` 变量逃逸，通过`fmt.Println(a ...interface{})`打印的变量，都会发生逃逸，感兴趣的朋友可以去查查为什么。
- 以上操作其实就叫**逃逸分析**

## 如何利用逃逸分析提升性能

### 传值 VS 传指针

传值会拷贝整个对象，而传指针只会拷贝指针地址，指向的对象是同一个。传指针可以减少值的拷贝，但是会导致内存分配逃逸到堆中，增加垃圾回收(GC)的负担。在对象频繁创建和删除的场景下，传递指针导致的 GC 开销可能会严重影响性能。

一般情况下，对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的性能。



## **总结**

------

1.因为栈比堆更高效，不需要 GC，因此 Go 会尽可能的将内存分配到栈上。

2.当分配到栈上可能引起非法内存访问等问题后，会使用堆，主要场景有：

1. 当一个值可能在函数被调用后访问，这个值极有可能被分配到堆上。
2. 当编译器检测到某个值过大，这个值会被分配到堆上。
3. 当编译时，编译器不知道这个值的大小（slice、map...）这个值会被分配到堆上。

3.Sharing down typically stays on the stack

4.Sharing up typically escapes to the heap

5.Don't guess, Only the compiler knows

6.Golang中一个函数内局部变量，不管是不是动态new出来的，它会被分配在堆还是栈，是由编译器做逃逸分析之后做出的决定。



# **参考文献**

【1】Go语言设计与实现：[https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/#%E5%AF%84%E5%AD%98%E5%99%A8](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/#寄存器)

【2】Inlining optimisations in Go：https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go

【3】Golang FAQ：https://golang.org/doc/faq#stack_or_heap

【4】知乎：https://zhuanlan.zhihu.com/p/145468000