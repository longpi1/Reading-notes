#                               Golang内存泄露场景与定位方式

## 一、产生原因

Golang有自动垃圾回收机制，但是仍然可能会出现内存泄漏的情况。以下是Golang内存泄漏的常见可能原因：

1. **循环引用**：如果两个或多个对象相互引用，且没有其他对象引用它们，那么它们就会被垃圾回收机制误认为是仍在使用的对象，导致内存泄漏。
2. **全局变量**：在Golang中，全局变量的生命周期与程序的生命周期相同。如果一个全局变量被创建后一直存在于内存中，那么它所占用的内存就无法被回收，可能会导致内存泄漏。
3. **未关闭的文件句柄**：如果程序打开了文件句柄但没有关闭它们，那么这些文件句柄所占用的内存就无法被回收，可能会导致内存泄漏。
4. **大量的临时对象**：如果程序创建了大量的临时对象，但没有及时释放它们，那么这些对象所占用的内存就无法被回收，可能会导致内存泄漏。
5. **goroutine泄漏**：**常见的泄露场景**，例如协程发生阻塞，Go运行时并不会将处于永久阻塞状态的协程杀掉，因此永久处于阻塞状态的协程所占用的资源将永得不到释放。
6. **time.Ticker未关闭导致泄漏**：当一个`time.Timer`值不再被使用，一段时间后它将被自动垃圾回收掉。 但对于一个不再使用的`time.Ticker`值，我们必须调用它的`Stop`方法结束它，否则它将永远不会得到回收。



## 二、排查方式

如果出现内存泄漏，可以使用以下方式进行分析，找出内存泄漏的原因并进行修复。

1. 使用 Go 语言自带的 **pprof 工具**进行分析。pprof 可以生成程序的 CPU 和内存使用情况的报告，帮助开发者找出程序中的性能瓶颈和内存泄漏问题。可以通过在代码中添加 `import _ "net/http/pprof"` 和 `http.ListenAndServe("localhost:6060", nil)` 来开启 pprof 工具。
2. 使用 Golang 内置的 `runtime` 包进行分析。`runtime` 包提供了一些函数，包括 `SetFinalizer`、`ReadMemStats` 和 `Stack` 等，可以帮助开发者了解程序的内存使用情况和内存泄漏问题。
3. 使用第三方工具进行分析。例如，可以使用 `go-torch` 工具生成火焰图，帮助开发者找出程序中的性能瓶颈和内存泄漏问题。
4. 使用 `go vet` 工具进行静态分析。`go vet` 可以检查程序中的常见错误和潜在问题，包括内存泄漏问题。
5. 代码审查。开发者可以通过代码审查来找出程序中的潜在问题和内存泄漏问题。



## 三、通过 pprof 的命令排查内存泄露问题

### 3.1 通过 pprof 的命令行分析 heap

命令行执行命令: `go tool pprof -inuse_space [<http://127.0.0.1:9999/debug/pprof/heap>](<http://spark-master.x.upyun.com/debug/pprof/heap>)`

这个命令的作用是, 抓取当前程序已使用的 heap. 抓取后, 就可以进行类似于 gdb 的交互操作.

- top 命令, 默认能列出当前程序中内存占用排名前 10 的函数. 如图. 当时进行到这一步的时候, 我就非常惊讶, 因为 `time.NewTimer` 居然占据了 6 个多 G 的内存.

![img](https://pic4.zhimg.com/80/v2-2ef735b81234db0c7ba6c567135f123b_1440w.webp)

- `list <函数名>`, 展现函数内部的内存占用. 使用 `list time.NewTimer` 查看了该函数的内部, 真相大白了, 原来每次调用 `NewTimer` 都会创建一个 channel, 还会生成一个结构体 `runtimeTimer`, 应该就是这两个地方内存没有释放造成的内存泄露.

![img](https://pic2.zhimg.com/80/v2-16566847f6c80be09bd30ef5586b2915_1440w.webp)

### 3.2 **修改 `for ... select ... time.After` 造成的内存泄露**

原来程序中存在如下代码:

```go
for {
		select {

		case a := <-chanA:
			...

		case b := <-chanB:
			....

		case <-time.After(20*time.Minutes):
			return nil, errors.New("download timeout")
	}
```

`time.After` 就是封装了一层的 `NewTimer`, `time.After` 的源码:

```go
func After(d Duration) <-chan Time {
	return NewTimer(d).C
}
```

修复该错误, 只调用一次 `NewTimer`:

```go
downloadTimeout := time.NewTimer(20 * time.Minute)
// 添加关闭时退出操作
defer downloadTimeout.Stop()

for {
		select {

		case a := <-chanA:
			...

		case b := <-chanB:
			....

		case <-downloadTimeout.C:
			return nil, errors.New("download timeout")
	}
```



## 四、总结

通过这篇文章我们了解到Golang内存泄漏的常见可能原因有哪些：

1. **循环引用**：如果两个或多个对象相互引用，且没有其他对象引用它们，那么它们就会被垃圾回收机制误认为是仍在使用的对象，导致内存泄漏。
2. **全局变量**：在Golang中，全局变量的生命周期与程序的生命周期相同。如果一个全局变量被创建后一直存在于内存中，那么它所占用的内存就无法被回收，可能会导致内存泄漏。
3. **未关闭的文件句柄**：如果程序打开了文件句柄但没有关闭它们，那么这些文件句柄所占用的内存就无法被回收，可能会导致内存泄漏。
4. **大量的临时对象**：如果程序创建了大量的临时对象，但没有及时释放它们，那么这些对象所占用的内存就无法被回收，可能会导致内存泄漏。
5. **goroutine泄漏**：比较常见的泄露场景，例如协程发生阻塞，Go运行时并不会将处于永久阻塞状态的协程杀掉，因此永久处于阻塞状态的协程所占用的资源将永得不到释放。
6. **time.Ticker未关闭导致泄漏**：当一个`time.Timer`值不再被使用，一段时间后它将被自动垃圾回收掉。 但对于一个不再使用的`time.Ticker`值，我们必须调用它的`Stop`方法结束它，否则它将永远不会得到回收。

然后介绍了相关排查工具以及pprof如何排查内存泄露问题。



## 五、参考链接

1.[一些可能的内存泄漏场景](https://gfw.go101.org/article/memory-leaking.html)

2.[使用 pprof 排查 Golang 内存泄露](https://zhuanlan.zhihu.com/p/265080950)