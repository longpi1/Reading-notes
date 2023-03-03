#                    Go 使用 fsnotify 监听文件

> 主要内容转载自**Golang语言开发栈**的[Go 语言跨平台文件监听库 fsnotify 怎么使用？](https://mp.weixin.qq.com/s/tJ1LvDf14EKg-qQlJUQapQ)

### **01** 介绍

Go 语言作为静态编译型语言，每次修改配置文件后，我们都需要重新编译，修改的配置信息才可以生效，而动态编译型语言修改配置文件可以自动生效，相对来说更方便一些。

但是，我们可以使用三方开源库 `fsnotify`，这是一款非常流行的文件系统监听库，很多开源的三方库也都使用该库实现监听文件变更，比如流行的管理配置信息开源库 `viper`。

另外，fsnotify也可以用作于安全场景，用于监听系统中敏感文件是否被未授权访问以及更新。

### **02** fsnotify 源码解读

**NewWatcher 函数：**

`fsnotify` 提供了 `NewWatcher` 函数，使用该函数可以创建一个监听器。

```go
// NewWatcher creates a new Watcher.
func NewWatcher() (*Watcher, error) {
 // 省略代码 ...

 w := &Watcher{
  // 省略代码 ...
  Events:       make(chan Event),
  Errors:       make(chan error),
  // 省略代码 ...
 }

 go w.readEvents()
 return w, nil
}
```

阅读 `NewWatcher` 函数的源码，我们可以发现，该函数返回一个 `*Watcher`。

并且我们可以发现该结构体的两个公开字段 `Events` 和 `Errors` 分别是 `Event` 类型和 `error` 类型的 `channel`。

**事件：**

`Event` 类型的字段 `Events`。

```go
type Event struct {
 Name string
 Op Op
}

type Op uint32

const (
 Create Op = 1 << iota
 Write
 Remove
 Rename
 Chmod
)
```

阅读上面这段代码，我们可以发现 `Event` 包含两个字段，分别表示事件名称和操作类型，其中，事件操作类型有 5 个，分别是 `Create`、`Write`、`Remove`、`Rename` 和 `Chmod`。

我们可以启动一个协程，使用 `for ... select` 监听 `watcher` 的 `Events` 和 `Errors` 通道并输出事件信息和错误信息。

`Event` 包含 2 个方法，分别是 `Has` 和 `String`，`Has` 用于判断事件是否包含给定操作，源码如下：

```go
// Has reports if this event has the given operation.
func (e Event) Has(op Op) bool { return e.Op.Has(op) }
```

**监听器：**

`Watcher` 包含 4 个公共方法，分别是 `Add`、`Close`、`Remove` 和 `WatchList`。

- Add - 用于指定监听目录或监听文件，需要注意的是，**指定目录仅能监听该目录中的所有文件，无法监听该目录中子目录的文件。**
- Close - 删除所有监听，并关闭 `Events` 通道。
- Remove - 停止监视指定目录或指定文件的变更，需要注意的是，指定目录仅代表当前目录，指定目录中的子目录需单独停止监听。删除未被监听的目录或文件，将会返回错误。
- WatchList - 返回尚未被删除的所有使用 `Add` 添加的目录或文件。



### **03** fsnotify 使用原理

![img](https://pic1.zhimg.com/80/v2-23048f6ebd328b9f631285c06c0ff5fc_1440w.webp)

如上图所示 fsnotify作为“**后端**“，负责接收文件事件，它被作为”**前端**“、和listener直接交互的dnotify, inotify和fanotify所共享。

每一个前端instance被抽象为一个"group"（在代码中由"fsnotify_group"结构体表示），每个group都有自己的notification queue（以下简称"nq"），用于向listener传递事件。

从效率的角度，fsnotify不会把收到的事件依次放到每个group的"nq"上，而是只维护一个event queue，根据各个group配置的mask，在其对应的"nq"里存放指针，指向event queue中感兴趣的事件。



### **04** fsnotify 使用示例

在了解完 `fsnotify` 源码之后，我们再介绍一下 `fsnotify` 的使用示例。

```golang
func main() {
 // 创建一个监听器
 watcher, err := fsnotify.NewWatcher()
 if err != nil {
  log.Fatal(err)
 }
 // 关闭监听器
 defer watcher.Close()
 // 开始监听事件
 go func() {
  for {
   select {
   case event, ok := <-watcher.Events:
    if !ok {
     return
    }
    if event.Has(fsnotify.Write) {
     // 自动加载文件内容
     f, _ := os.Open("log.txt")
     _, _ = io.Copy(os.Stdout, f)
    }
  }
 }()
 // 添加监听目录
 err = watcher.Add("./")
 if err != nil {
  log.Fatal(err)
 }
 // 永久阻塞 main goroutine
 <-make(chan struct{})
}
```

阅读上面这段示例代码，我们可以发现，使用 `fsnotify` 非常简单。

首先，使用 `NewWatcher` 函数创建一个 `watcher`，然后，使用 `Add` 方法添加监听目录或文件，最后，使用 `defer` 调用 `Close` 方法，关闭监听器，释放系统资源。

示例代码中，启动一个 `goroutine` 循环输出事件通道中的事件，发现 `Write` 操作类型的事件时，将 `log.txt` 中的文件内容拷贝到标准输出。

我们可以在运行该程序后，修改 `log.txt` 中的内容，终端将会打印该文件修改后的最新内容。

**我们可以使用该特性，自动监听应用程序的配置文件，避免修改配置信息后，还需要重新编译并启动应用才可以生效。**

### **05** 总结

本文我们介绍了跨平台文件监听库 `fsnotify`，它主要用于自动监听文件中的内容变更。

我们通过 `fsnotify` 源码和示例代码，介绍了该库支持的功能和使用方式。

建议感兴趣的朋友，继续阅读参考链接中该库的官方文档和源码，了解在不同系统平台中使用的注意事项，并有效运用在自己的项目中。



### 06 **参考资料：**

1. https://pkg.go.dev/github.com/fsnotify/fsnotify
2. https://github.com/fsnotify/fsnotify
3. https://zhuanlan.zhihu.com/p/186027813