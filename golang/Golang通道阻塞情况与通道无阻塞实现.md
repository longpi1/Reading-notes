#                           Golang通道阻塞情况与通道无阻塞实现

## 一、通道阻塞原理

在Go语言中，通道会在以下情况下发生阻塞：

1. 如果通道已满，并且没有协程在读取通道中的数据，那么任何试图将数据写入通道的协程都会被阻塞，直到有空间可用为止。
2. 如果通道为空，并且没有协程在等待从通道中读取数据，那么任何试图从通道中读取数据的协程都会被阻塞，直到有数据可用为止。



## 二、通道阻塞场景

在channel中，**无论是有缓存通道、无缓冲通道都存在阻塞的情况**。阻塞场景共4个，有缓存和无缓冲各2个。

### 2.1 无缓冲通道

**无缓冲通道**的特点是，发送的数据需要被读取后，发送才会完成，它**阻塞场景**：

1. 通道中无数据，但执行读通道。
2. 通道中无数据，向通道写数据，但无协程读取。

```go
// 场景1
func ReadNoDataFromNoBufCh() {
    noBufCh := make(chan int)

    <-noBufCh
    fmt.Println("read from no buffer channel success")

    // Output:
    // fatal error: all goroutines are asleep - deadlock!
}

// 场景2
func WriteNoBufCh() {
    ch := make(chan int)

    ch <- 1
    fmt.Println("write success no block")
    
    // Output:
    // fatal error: all goroutines are asleep - deadlock!
}
```

*注：示例代码中的Output注释代表函数的执行结果*

每一个函数都由于阻塞在通道操作而无法继续向下执行，最后报了死锁错误。

### 2.2 有缓存通道

**有缓存通道**的特点是，有缓存时可以向通道中写入数据后直接返回，缓存中有数据时可以从通道中读到数据直接返回，这时有缓存通道是不会阻塞的，它**阻塞场景是**：

1. 通道的缓存无数据，但执行读通道。
2. 通道的缓存已经占满，向通道写数据，但无协程读。

```go
// 场景1
func ReadNoDataFromBufCh() {
    bufCh := make(chan int, 1)

    <-bufCh
    fmt.Println("read from no buffer channel success")

    // Output:
    // fatal error: all goroutines are asleep - deadlock!
}

// 场景2
func WriteBufChButFull() {
    ch := make(chan int, 1)
    // make ch full
    ch <- 100

    ch <- 1
    fmt.Println("write success no block")
    
    // Output:
    // fatal error: all goroutines are asleep - deadlock!
}
```



## 三、通道无阻塞读写

### 3.1 Select实现无阻塞读写

下面**示例代码是使用select修改后的无缓冲通道和有缓冲通道的读写**，以下函数可以直接通过main函数调用；

```go
// 1.select结构实现通道读
func ReadWithSelect(ch chan int) (x int, err error) {
    select {
    case x = <-ch:
        return x, nil
    default:
        return 0, errors.New("channel has no data")
    }
}

// 无缓冲通道读
func ReadNoDataFromNoBufChWithSelect() {
    bufCh := make(chan int)

    if v, err := ReadWithSelect(bufCh); err != nil {
        fmt.Println(err)
    } else {
        fmt.Printf("read: %d\n", v)
    }

    // Output:
    // channel has no data
}

// 有缓冲通道读
func ReadNoDataFromBufChWithSelect() {
    bufCh := make(chan int, 1)

    if v, err := ReadWithSelect(bufCh); err != nil {
        fmt.Println(err)
    } else {
        fmt.Printf("read: %d\n", v)
    }

    // Output:
    // channel has no data
}

// 2. select结构实现通道写
func WriteChWithSelect(ch chan int) error {
    select {
    case ch <- 1:
        return nil
    default:
        return errors.New("channel blocked, can not write")
    }
}

// 无缓冲通道写
func WriteNoBufChWithSelect() {
    ch := make(chan int)
    if err := WriteChWithSelect(ch); err != nil {
        fmt.Println(err)
    } else {
        fmt.Println("write success")
    }

    // Output:
    // channel blocked, can not write
}

// 有缓冲通道写
func WriteBufChButFullWithSelect() {
    ch := make(chan int, 1)
    // make ch full
    ch <- 100
    if err := WriteChWithSelect(ch); err != nil {
        fmt.Println(err)
    } else {
        fmt.Println("write success")
    }

    // Output:
    // channel blocked, can not write
}
```

*注：示例代码中的Output注释代表函数的执行结果*

从结果能看出，在通道不可读或者不可写的时候，不再阻塞等待，而是直接返回。

### 3.2 使用Select+超时改善无阻塞读写

**使用default实现的无阻塞通道阻塞有一个缺陷：当通道不可读或写的时候，会即可返回**。实际场景，更多的需求是，我们希望尝试读一会数据，或者尝试写一会数据，如果实在没法读写再返回，程序继续做其它的事情。

**使用定时器替代default**可以解决这个问题，**给通道增加读写数据的容忍时间**，如果500ms内无法读写，就即刻返回。示例代码修改一下会是这样：

```go
func ReadWithSelect(ch chan int) (x int, err error) {
    timeout := time.NewTimer(time.Microsecond * 500)

    select {
    case x = <-ch:
        return x, nil
    case <-timeout.C:
        return 0, errors.New("read time out")
    }
}

func WriteChWithSelect(ch chan int) error {
    timeout := time.NewTimer(time.Microsecond * 500)

    select {
    case ch <- 1:
        return nil
    case <-timeout.C:
        return errors.New("write time out")
    }
}
```

结果就会变成超时返回：

```plaintext
read time out
write time out
read time out
write time out
```



## 四、总结

本篇文章介绍了在Go语言中，通道会在以下情况下发生阻塞：

1. 如果通道已满，并且没有协程在读取通道中的数据，那么任何试图将数据写入通道的协程都会被阻塞，直到有空间可用为止。
2. 如果通道为空，并且没有协程在等待从通道中读取数据，那么任何试图从通道中读取数据的协程都会被阻塞，直到有数据可用为止。

以及解决阻塞的2种办法：

1. 使用select的default语句，在channel不可读写时，即可返回
2. 使用select+定时器，在超时时间内，channel不可读写，则返回



## 五、参考链接

1. [大彬](https://link.segmentfault.com/?enc=I8bBYkFvu%2FrXuHxcGZ0Hzg%3D%3D.cLfDBqOXcGkKHQP6HNyDPb3gkSx3iH9%2FT4kLKXRpQVI%3D)，http://lessisbetter.site/2018/11/03/Golang-channel-read-and-write-without-blocking/
2. https://studygolang.com/articles/6024