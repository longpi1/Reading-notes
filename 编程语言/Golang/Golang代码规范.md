#                                                             Golang代码规范

[TOC]

## 1 前言

本篇文章主要介绍代码编写时需要注意的基本规范，Golang代码安全相关规范可参考[Golang代码安全规范](https://blog.longpi1.com/2022/12/31/Golang%E4%BB%A3%E7%A0%81%E5%AE%89%E5%85%A8%E8%A7%84%E8%8C%83/)，大家有其他相关思路，欢迎提出；



## 2 编程规约

### 2.1 命名风格

* 【强制】代码中的命名均不能以**特殊字符**开始和结束，包含常见的中划线、下划线等。

    反例:

    ```golang
    _name; -name; __name; --name; $name; %name
    ```

* 【强制】参数名、局部变量都统一使用**lowerCamelCase**(**小驼峰**)风格。

    正例：

    ```golang
    func demo(ctx context.Context, name string) {
        var localVar string
        // other operations
    }
    ```

* 【推荐】全局变量统一使用**小驼峰**风格。包外引用需要提供相应的导出函数对外导出使用。

    正例：

    ```golang
    var globalVar string
    func Set(value string) {
        globalVar = value
    }
    ```

* 【强制】函数名和方法(method)的命名区分导出和非导出类型。对于包内非导出的函数或方法使用小驼峰风格。对于导出的函数或方法采用**UpperCamelCase**(**大驼峰**)风格。

    正例：

    ```golang
    func localFunction() {
        // do something
    }

    type sample struct {}

    // Show print hello
    func (s *sample) Show() {
        fmt.Println("Hello Blueking.")
    }
    ```

* 【强制】struct的命名区分导出和非导出类型。对于包内非导出的struct使用**小驼峰**风格。对于导出的struct采用**大驼峰**风格。

    正例：

    ```golang
    type unexportedSample struct {}

    // ExportedSample exported sample
    type ExportedSample struct {}
    ```

* 【强制】常量命名使用大驼峰风格，不要使用下划线分隔，力求语义表达完整清楚，不要嫌名字长。

    正例：

    ```golang
    MaxConnectionCount
    ```

    反例

    ```golang
    MAX_CNNECTION_COUNT
    ```

* 【推荐】接口(interface)采用**大驼峰**的风格命名。具体细分为以下三种情况：
  * 单个函数的接口名以“er”作为后缀，如Reader, Writer。而接口的实现则去掉“er”。

    正例：

    ```golang
    type Reader interface {
        Read(p []byte) (n int, err error)
    }
    ```

  * 两个函数的接口名缩合两个函数名

    正例：

    ```golang
    type WriteFlusher interface {
        Write([]byte) (int, error)
        Flush() error
    }
    ```

  * 三个以上函数的接口名，类似于结构体名

    正例：

    ```golang
    type Car interface {
        Start([]byte)
        Stop() error
        Recover()
    }
    ```

* 【强制】包名统一采用小写风格，使用短命名，不能包含特殊字符（下划线、中划线等）语义表达准确。建议最好是一个单词。使用多级目录来划分层级。

    正例：

    ```golang
    package client
    package clientset
    ```

    反例：

    ```golang
    package Demo
    package Demo_Case
    ```

* 【强制】杜绝不规范的缩写，避免望文不生意。不必要进行缩写时，避免进行缩写。**对于专 用名词如URL/CMDB/GSE等，在使用时要保证全名统一为大写或小写。不能出现部分大写和小写混用的情况。**

    正例：

    ```golang
    func setURL() {
        // do something
    }
    
    func GetURL() {
        // do something
    }
    ```

    反例：

    ```golang
    var sidecarUrl string
    
    func ResetUrl()  {
        // do something
    }
    ```

    

* 【推荐】如果包、struct、interface等使用了设计模式，在命名时需要体现出对应的设计模式。这有利于阅读者快速理解代码的设计架构和设计理念。

    正例：

    ```golang
    type ContainerFactory interface {
        Create(id string, config *configs.Config) (Container, error)
        Load(id string) (Container, error)
        StartInitialization() error
        Type() string
    }
    type PersonDecorator interface {
        SetName(name string) PersonDecorator
        SetAge(age uint) PersonDecorator
        Show()
    }
    ```

### 2.2 常量定义

* 【强制】对于多个具有枚举特性的类型，要求定义成为类型，并利用常量进行枚举。

    正例：

    ```golang
    type EventType string
    const (
        Create EventType = "create"
        Update EventType = "update"
        Get    EventType = "get"
        Delete EventType = "delete"
    )
    ```

* 【强制】**不允许任何未经定义的常量直接在代码中使用。**

### 2.3 代码格式

* 【强制】采用4个空格的缩进，每个tab也代表4个空格。这是唯一能够保证在所有环境下获得一致展现的方法。
* 【强制】运算符(:=, =等)的左右两侧必须要加一个空格（符合gofmt逻辑）。
* 【强制】作为输入参数或者数组下标时，运算符和运算数之间不需要空格，紧凑展示（符合gofmt逻辑）。
* 【强制】提交的代码必须经过gofmt格式化。很多IDE支持自动gofmt格式化。
* 【推荐】代码最大行宽为120列，超过换行。

### 2.4 控制语句

#### 2.4.1 if

* 【强制】if接受一个初始化语句，**对于返回参数不需要流入到下一个语句时，通过建立局部变量的方式构建if判断语句。**

    正例：

    ```golang
    if err := file.Chmod(0664); err != nil {
        return err
    }
    ```

    

* 【强制】对于遍历数据(如map)的场景，如果只使用第一项，则直接丢弃第二项。

    正例：

    ```golang
    for key := range mapper {
        codeUsing(key)
    }
    ```

    反例：

    ```golang
    for key, _ := range mapper {
        codeUsing(key)
    }
    ```

### 2.5 注释规约

* 【强制】注释必须是完整的句子，以句点作为结尾。

* 【强制】使用行间注释时，如果注释行与上一行不属于同一代码层级，不用空行。如果属于同行，则空一行再进行注释。

    正例：

    ```golang
    func demo() {
        // This is a start line of a new block, do not need a new line 
        // with the previous code.
        say("knock, knock!")
        
        // This is the same block with the previous code,
        // you should insert a new line before you start a comment.
        echo("who is there ?")
    }
    ```

* 【强制】使用 // 进行注释时, 和注释语句之间必须有一个空格。增加可读性。

    正例：

    ```golang
    // validator is used to validate dns's format.
    // should not contains dot, underscore character, etc.
    func validator(dns string) error {
        // do validate.

    }
    ```

* 【强制】**不要使用尾注释**

    反例：

    ```golang
    func show(name string) {
        display(name) // show a man's information
    }
    ```

* 【强制】使用/**/风格进行多行注释时，首/*和尾*/两行内容中不能包含注释内容，也不能包含额外的内容，如星号横幅等。

    正例：

    ```golang
    /*
    The syntax of the regular expressions accepted is:

        regexp:
            concatenation { '|' concatenation }
        concatenation:
            { closure }
        closure:
            term [ '*' | '+' | '?' ]
        term:
            '^'
            '$'
            '.'
            character
            '[' [ '^' ] character-ranges ']'
            '(' regexp ')'
    */
    ```

* 【强制】注释的单行长度最大不能超过120列，超过必须换行。一般以80列换行为宜。

* 【推荐】函数与方法的注释需要以函数或方法的名称作为开头。

    正例：

    ```golang
    // HasPrefix tests whether the string s begins with prefix.
    func HasPrefix(s, prefix string) bool {
        return len(s) >= len(prefix) && s[0:len(prefix)] == prefix
    }
    ```

* 【推荐】大段注释采用/**/风格进行注释。

* 【推荐】包中的每一个导出的函数、方法、结构体和常量都应该有相应的注释。

* 【推荐】对于特别复杂的包说明，可以单独创建doc.go文件来详细说明。

### 2.6 interface 的指针

您几乎不需要指向接口类型的指针。您应该将接口作为值进行传递，在这样的传递过程中，实质上传递的底层数据仍然可以是指针。

接口实质上在底层用两个字段表示：

1. 一个指向某些特定类型信息的指针。您可以将其视为"type"。
2. 数据指针。如果存储的数据是指针，则直接存储。如果存储的数据是一个值，则存储指向该值的指针。

如果希望接口方法修改基础数据，则必须使用指针传递 (将对象指针赋值给接口变量)。

```golang
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

// f1.f() 无法修改底层数据
// f2.f() 可以修改底层数据，给接口变量 f2 赋值时使用的是对象指针
var f1 F = S1{}
var f2 F = &S2{}
```



### 2.7 其它

#### 2.7.1 参数传递

* 【推荐】对于少量数据，不要通过指针传递。
* 【推荐】对于大量(>=4)的入参，考虑使用struct进行封装，并通过指针传递。
* 【强制】传参是map, slice, chan 不要使用指针进行传递。因为这三者是引用类型。

#### 2.7.2 接受者（receiver）

* 【推荐】名称统一采用1~3个字母，不宜太长。
* 【推荐】对于绝在多数可以使用指针接受者的场景，推荐使用指针接受者(point receiver)会更有效率。
* 【强制】如果接受者是map, slice, chan，不能使用指针接受者。

    正例：

    ```golang
    package main

    import (
        "fmt"
    )

    type queue chan interface{}

    func (q queue) Push(i interface{}) {
        q <- i
    }

    func (q queue) Pop() interface{} {
        return <-q
    }

    func main() {
        c := make(queue, 1)
        c.Push("i")
        fmt.Println(c.Pop())
    }
    ```

* 【强制】如果接受者是包含有锁(sync.Mutex等)，必须使用指针接受者。



#### 2.7.3 启动任何一个goroutine都要先想好如何退出



#### 2.7.4 return err的时候清理本无需启动的goroutine



#### 2.7.5 想好如何合理优雅Stop



#### 2.7.6 使用done chan退出goroutine的时候，不要做任何清理资源的操作。清理资源优先使用defer，因为当goroutine panic的时候， defer还能够清理资源，而done chan中的清理逻辑可能永远不会被执行



#### 2.7.7 不建议使用`time.after()`。使用`time.NewTicker()`代替，并及时清理ticker



#### 2.7.8 关闭(Stop)插件的时候注意关闭的顺序和资源释放， 不要造成其他goroutine夯住(例如供外部写的0容量的chan没有处理，但是读的goroutine退出了，导致外部写的goroutine一直阻塞)



### 2.8 在边界处拷贝 Slices 和 Maps

slices 和 maps 包含了指向底层数据的指针，因此在需要复制它们时要特别注意。

#### 2.8.1 接收 Slices 和 Maps

请记住，当 map 或 slice 作为函数参数传入时，如果您存储了对它们的引用，则用户可以对其进行修改。

正例

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// 这里我们修改 trips[0]，但不会影响到 d1.trips
trips[0] = ...
```



#### 2.8.2 返回 slices 或 maps

同样，请注意用户对暴露内部状态的 map 或 slice 的修改。

正例

```go
type Stats struct {
  mu sync.Mutex

  counters map[string]int
}

func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}

// snapshot 现在是一个拷贝
snapshot := stats.Snapshot()
```

反例

```go
type Stats struct {
  mu sync.Mutex

  counters map[string]int
}

// Snapshot 返回当前状态。
func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  return s.counters
}

// snapshot 不再受互斥锁保护
// 因此对 snapshot 的任何访问都将受到数据竞争的影响
// 影响 stats.counters
snapshot := stats.Snapshot()
```



### 2.9 使用 defer 释放资源

使用 defer 释放资源，诸如文件和锁。

```go
正例：
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count

// 更可读
反例：
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// 当有多个 return 分支时，很容易遗忘 unlock
```

Defer 的开销非常小，只有在您可以证明函数执行时间处于纳秒级的程度时，才应避免这样做。使用 defer 提升可读性是值得的，因为使用它们的成本微不足道。尤其适用于那些不仅仅是简单内存访问的较大的方法，在这些方法中其他计算的资源消耗远超过 `defer`。



### 2.10 不要使用 panic

在生产环境中运行的代码必须避免出现 panic。panic 是 [级联失败](https://en.wikipedia.org/wiki/Cascading_failure) 的主要根源 。如果发生错误，该函数必须返回错误，并允许调用方决定如何处理它。

```go
正例：
func run(args []string) error {
  if len(args) == 0 {
    return errors.New("an argument is required")
  }
  // ...
  return nil
}

func main() {
  if err := run(os.Args[1:]); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}

反例：
func run(args []string) {
  if len(args) == 0 {
    panic("an argument is required")
  }
  // ...
}

func main() {
  run(os.Args[1:])
}
```

panic/recover 不是错误处理策略。仅当发生不可恢复的事情（例如：nil 引用）时，程序才必须 panic。程序初始化是一个例外：程序启动时应使程序中止的不良情况可能会引起 panic。



### 2.11 避免可变全局变量

使用选择依赖注入方式避免改变全局变量。 既适用于函数指针又适用于其他值类型

```go
正例：
// sign.go
type signer struct {
  now func() time.Time
}
func newSigner() *signer {
  return &signer{
    now: time.Now,
  }
}
func (s *signer) Sign(msg string) string {
  now := s.now()
  return signWithTime(msg, now)
}

反例：
// sign.go
var _timeNow = time.Now
func sign(msg string) string {
  now := _timeNow()
  return signWithTime(msg, now)
}

```



### 2.12 避免使用 `init()`

尽可能避免使用`init()`。当`init()`是不可避免或可取的，代码应先尝试：

1. 无论程序环境或调用如何，都要完全确定。
2. 避免依赖于其他`init()`函数的顺序或副作用。虽然`init()`顺序是明确的，但代码可以更改， 因此`init()`函数之间的关系可能会使代码变得脆弱和容易出错。
3. 避免访问或操作全局或环境状态，如机器信息、环境变量、工作目录、程序参数/输入等。
4. 避免`I/O`，包括文件系统、网络和系统调用。

```go
正例：
var _defaultFoo = Foo{
    // ...
}
// or，为了更好的可测试性：
var _defaultFoo = defaultFoo()
func defaultFoo() Foo {
    return Foo{
        // ...
    }
}

反例：

type Foo struct {
    // ...
}
var _defaultFoo Foo
func init() {
    _defaultFoo = Foo{
        // ...
    }
}
```

不能满足这些要求的代码可能属于要作为`main()`调用的一部分（或程序生命周期中的其他地方）， 或者作为`main()`本身的一部分写入。特别是，打算由其他程序使用的库应该特别注意完全确定性， 而不是执行“init magic”



### 2.13 追加时优先指定切片容量

在尽可能的情况下，在初始化要追加的切片时为`make()`提供一个容量值。

```go
正例：
for n := 0; n < b.N; n++ {
  data := make([]int, 0, size)
  for k := 0; k < size; k++{
    data = append(data, k)
  }
}

反例：
for n := 0; n < b.N; n++ {
  data := make([]int, 0)
  for k := 0; k < size; k++{
    data = append(data, k)
  }
}

```



### 2.14 不要一劳永逸地使用 goroutine

Goroutines 是轻量级的，但它们不是免费的： 至少，它们会为堆栈和 CPU 的调度消耗内存。 虽然这些成本对于 Goroutines 的使用来说很小，但当它们在没有受控生命周期的情况下大量生成时会导致严重的性能问题。 具有非托管生命周期的 Goroutines 也可能导致其他问题，例如防止未使用的对象被垃圾回收并保留不再使用的资源。

因此，不要在代码中泄漏 goroutine。 使用 [go.uber.org/goleak](https://pkg.go.dev/go.uber.org/goleak) 来测试可能产生 goroutine 的包内的 goroutine 泄漏。

一般来说，每个 goroutine:

- 必须有一个可预测的停止运行时间； 或者
- 必须有一种方法可以向 goroutine 发出信号它应该停止

在这两种情况下，都必须有一种方式代码来阻塞并等待 goroutine 完成。

```go
正例：
var (
  stop = make(chan struct{}) // 告诉 goroutine 停止
  done = make(chan struct{}) // 告诉我们 goroutine 退出了
)
go func() {
  defer close(done)
  ticker := time.NewTicker(delay)
  defer ticker.Stop()
  for {
    select {
    case <-tick.C:
      flush()
    case <-stop:
      return
    }
  }
}()
// 其它...
close(stop)  // 指示 goroutine 停止
<-done       // and wait for it to exit

反例：
go func() {
  for {
    flush()
    time.Sleep(delay)
  }
}()


```



### 2.15 减少嵌套

代码应通过尽可能先处理错误情况/特殊情况并尽早返回或继续循环来减少嵌套。减少嵌套多个级别的代码的代码量。

```go
正例：
for _, v := range data {
  if v.F1 != 1 {
    log.Printf("Invalid v: %v", v)
    continue
  }

  v = process(v)
  if err := v.Call(); err != nil {
    return err
  }
  v.Send()
}
反例：
for _, v := range data {
  if v.F1 == 1 {
    v = process(v)
    if err := v.Call(); err == nil {
      v.Send()
    } else {
      return err
    }
  } else {
    log.Printf("Invalid v: %v", v)
  }
}
```



### 2.16  不必要的 else

如果在 if 的两个分支中都设置了变量，则可以将其替换为单个 if。

```go
正例：
a := 10
if b {
  a = 100
}
反例：
var a int
if b {
  a = 100
} else {
  a = 10
}

```



### 2.17 初始化 Maps

对于空 map 请使用 `make(..)` 初始化， 并且 map 是通过编程方式填充的。 这使得 map 初始化在表现上不同于声明，并且它还可以方便地在 make 后添加大小提示。

```go
正例：
var (
  // m1 读写安全;
  // m2 在写入时会 panic
  m1 = make(map[T1]T2)
  m2 map[T1]T2
)
反例：
var (
  // m1 读写安全;
  // m2 在写入时会 panic
  m1 = map[T1]T2{}
  m2 map[T1]T2
)

声明和初始化看起来差别非常大。
```



## 3 性能

性能方面的特定准则只适用于高频场景。

### 3.1 优先使用 strconv 而不是 fmt

将原语转换为字符串或从字符串转换时，`strconv`速度比`fmt`快。

```go
正例：
for i := 0; i < b.N; i++ {
  s := strconv.Itoa(rand.Int())
}
BenchmarkStrconv-4    64.2 ns/op    1 allocs/op
反例：
for i := 0; i < b.N; i++ {
  s := fmt.Sprint(rand.Int())
}
BenchmarkFmtSprint-4    143 ns/op    2 allocs/op
```



### 3.2 避免字符串到字节的转换

不要反复从固定字符串创建字节 slice。相反，请执行一次转换并捕获结果。

```go
正例：
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
  w.Write(data)
}
BenchmarkGood-4  500000000   3.25 ns/op
反例：
for i := 0; i < b.N; i++ {
  w.Write([]byte("Hello world"))
}
BenchmarkBad-4   50000000   22.2 ns/op
```



### 3.3 指定容器容量

尽可能指定容器容量，以便为容器预先分配内存。这将在添加元素时最小化后续分配（通过复制和调整容器大小）。

在尽可能的情况下，在使用 `make()` 初始化的时候提供容量信息

向`make()`提供容量提示会在初始化时尝试调整 map 的大小，这将减少在将元素添加到 map 时为 map 重新分配内存。

注意，与 slices 不同。map capacity 提示并不保证完全的抢占式分配，而是用于估计所需的 hashmap bucket 的数量。 因此，在将元素添加到 map 时，甚至在指定 map 容量时，仍可能发生分配。

```go
正例：
files, _ := os.ReadDir("./files")

m := make(map[string]os.FileInfo, len(files))
for _, f := range files {
    m[f.Name()] = f
}
m 是有大小提示创建的；在运行时可能会有更少的分配。
反例：
m := make(map[string]os.FileInfo)

files, _ := os.ReadDir("./files")
for _, f := range files {
    m[f.Name()] = f
}
m 是在没有大小提示的情况下创建的； 在运行时可能会有更多分配。

```



### 3.4 字符串拼接用strings.Builder

综合易用性和性能，一般推荐使用 `strings.Builder` 来拼接字符串。

参考链接：https://geektutu.com/post/hpg-string-concat.html



### 3.5 结构体集合用for还是range遍历

```go

type Item struct {
	id  int
	val [4096]byte
}

func BenchmarkForStruct(b *testing.B) {
	var items [1024]Item
	for i := 0; i < b.N; i++ {
		length := len(items)
		var tmp int
		for k := 0; k < length; k++ {
			tmp = items[k].id
		}
		_ = tmp
	}
}

func BenchmarkRangeIndexStruct(b *testing.B) {
	var items [1024]Item
	for i := 0; i < b.N; i++ {
		var tmp int
		for k := range items {
			tmp = items[k].id
		}
		_ = tmp
	}
}

func BenchmarkRangeStruct(b *testing.B) {
	var items [1024]Item
	for i := 0; i < b.N; i++ {
		var tmp int
		for _, item := range items {
			tmp = item.id
		}
		_ = tmp
	}
}

BenchmarkForStruct-8             3769580               324 ns/op
BenchmarkRangeIndexStruct-8      3597555               330 ns/op
BenchmarkRangeStruct-8              2194            467411 ns/op
```

- 仅遍历下标的情况下，for 和 range 的性能几乎是一样的。
- `items` 的每一个元素的类型是一个结构体类型 `Item`，`Item` 由两个字段构成，一个类型是 int，一个是类型是 `[4096]byte`，也就是说每个 `Item` 实例需要申请约 4KB 的内存。
- 在这个例子中，for 的性能大约是 range (同时遍历下标和值) 的 2000 倍。

参考链接：https://geektutu.com/post/hpg-range.html

### 3.6 避免使用反射

使用反射赋值，效率非常低下，如果有替代方案，尽可能避免使用反射，特别是会被反复调用的热点代码。例如 RPC 协议中，需要对结构体进行序列化和反序列化，这个时候避免使用 Go 语言自带的 `json` 的 `Marshal` 和 `Unmarshal` 方法，因为标准库中的 json 序列化和反序列化是利用反射实现的。可选的替代方案有 [easyjson](https://github.com/mailru/easyjson)，在大部分场景下，相比标准库，有 5 倍左右的性能提升。

参考链接：https://geektutu.com/post/hpg-reflect.html



### 3.7 空结构体 struct{} 的使用

因为空结构体不占据内存空间，因此被广泛作为各种场景下的占位符使用。一是节省资源，二是空结构体本身就具备很强的语义，即这里不需要任何值，仅作为占位符。



### 3.8 [内存对齐对性能的影响](https://geektutu.com/post/hpg-struct-alignment.html)



## 4 异常

### 4.1 异常处理

* 【强制】程序中出现的任何异常都必须处理，不能忽略。
* 【强制】错误描述必须为小写，不需要标点结尾。
* 【推荐】程序中尽量避免使用panic来进行异常处理。对于必须要使用panic进行异常处理的场景，应该保证能够在单元测试或集成测试中覆盖到此场景。同时要在非测试场景下，启用recover能力。

### 4.2 日志规约

* 【推荐】日志采用分级打印方式，包含Info, Warning，Error和自定义等级。统一使用blog日志包进行日志的管理。
* 【强制】日志的内容要详尽，至少包含这几个要素：谁在什么情况下，因为什么原因，出现了什么异常，会引起什么问题。方便异常的定位。
* 【强制】Info级别用于打印在程序运行过程中必须要打印的日志信息。不能包含调试等日志信息。
* 【强制】Warning级别用于打印程序运行过程中出现的异常，但不影响程序的正常运行，需要通过Warning级别日志进行提示。
* 【强制】Error级别用于打印程序运行过程中出现的会影响业务正常运行逻辑的异常。
* 【推荐】对于自定义等级的日志，默认3级为debug日志。自定义日志的级别可根据自身需求进行调整。
* 【推荐】底层公共库中的异常应该抛出，不建议在公共库中打印相关的日志信息，应该由上层的逻辑层处理异常，并打印日志信息。

## 5 单元测试

* 【强制】单元测试必须遵守 AIR 原则，即具有自动化、独立性、可重复执行的特点。
* 【强制】单元测试应该是全自动执行的，并且非交互式的。测试用例通常是被定期执行的，执行过程必须完全自动化才有意义。输出结果禁止进行人工验证。
* 【强制】保持单元测试的独立性。为了保证单元测试稳定可靠且便于维护，单元测试用例之间禁止互相调用，也不能依赖执行的先后次序。
* 【强制】新增代码及时补充单元测试，如果新增代码影响了原有单元测试，请及时修正。
* 【推荐】对于不可测的代码建议做必要的重构，使代码变得可测，避免为了达到测试要求而编写不规范测试代码。
* 【推荐】在设计评审阶段，开发人员需要和测试人员一起确定单元测试范围，单元测试最好 覆盖所有测试用例。



### 5.1 用例编写原则 

当测试逻辑是重复的时候，通过 [subtests](https://blog.golang.org/subtests) 使用 table 驱动的方式编写 case 代码看上去会更简洁。

```go
正例：
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
  {
    give:     ":8000",
    wantHost: "",
    wantPort: "8000",
  },
  {
    give:     "1:8",
    wantHost: "1",
    wantPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
反例：
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

很明显，使用 test table 的方式在代码逻辑扩展的时候，比如新增 test case，都会显得更加的清晰。

我们遵循这样的约定：将结构体切片称为`tests`。 每个测试用例称为`tt`。



## 6 工程结构

* 【推荐】项目名可以通过中划线来连接多个单词。
* 【强制】工程目录名称只能使用英文小写字母
* 【推荐】保持 package 的名字和目录一致，包名应该为小写单词，不要使用下划线或者混合大小写，使用多级目录来划分层级。
* 【强制】文件名只能使用英文小写字母，如果有多个单词，中间可以使用下划线进行分隔。命名尽量望文生义。
* 【强制】当命名包时，请按下面规则选择一个名称：

    - 全部小写。没有大写或下划线。
    - 大多数使用命名导入的情况下，不需要重命名。
    - 简短而简洁。请记住，在每个使用的地方都完整标识了该名称。
    - 不用复数。例如`net/url`，而不是`net/urls`。
    - 不要用“common”，“util”，“shared”或“lib”。这些是不好的，信息量不足的名称。

    另请参阅 [Go 包命名规则](https://blog.golang.org/package-names) 和 [Go 包样式指南](https://rakyll.org/style-packages/).

    
* 【强制】工程中引入(import)的包，不能使用相对路径。必须使用相对于GOPATH的完整路径。

    反例：

    ```golang
    import (
        "fmt"
        "errors"

        "../../apimachinery/discovery"
    )
    ```

* 【强制】如果程序包名称与导入路径的最后一个元素不匹配，则必须使用导入别名。

    ```
    import (
      "net/http"
    
      client "example.com/client-go"
      trace "example.com/trace/v2"
    )
    ```
    
* 【强制】工程中引入的包，需要按照“**标准库包、工程内部包、第三方包”**的顺序进行组织。三种包之间用空行进行分隔，这样在gofmt时不会打乱三种包之间的顺序。

    正例：

    ```golang
    import (
        "fmt"
        "context"
        "errors"
    
        "configcenter/src/apimachinery/discovery"
        "configcenter/src/apimachinery/rest"
        "configcenter/src/apimachinery/util"
    
        "github.com/juju/ratelimit"
    )
    ```

## 参考文档

* [https://golang.org/doc/effective_go.html](https://golang.org/doc/effective_go.html)
* [https://github.com/golang/go/wiki/CodeReviewComments](https://github.com/golang/go/wiki/CodeReviewComments)
* https://github.com/TencentBlueKing/bk-bcs/blob/master/docs/specification/blueking-golang-code-conduct1.0.1.md
* https://github.com/xxjwxc/uber_go_guide_cn#%E4%BC%98%E5%85%88%E4%BD%BF%E7%94%A8-strconv-%E8%80%8C%E4%B8%8D%E6%98%AF-fmt
* [Effective Go](https://golang.org/doc/effective_go.html)
* [Go Common Mistakes](https://github.com/golang/go/wiki/CommonMistakes)
* [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)