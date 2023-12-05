#                      Golang 的channel底层实现

> 转载自RyuGou的[图解Go的channel底层实现](https://i6448038.github.io/2019/04/11/go-channel/)

## channel的整体结构图

![img](https://i6448038.github.io/img/channel/hchan.png)

简单说明：

- `buf`是有缓冲的channel所特有的结构，用来存储缓存数据。是个循环链表
- `sendx`和`recvx`用于记录`buf`这个循环链表中的发送或者接收的index
- `lock`是个互斥锁。
- `recvq`和`sendq`分别是接收(<-channel)或者发送(channel <- xxx)的goroutine抽象出来的结构体(sudog)的队列。是个双向链表

源码位于`/runtime/chan.go`中(目前版本：1.11)。结构体为`hchan`。

```go
type hchan struct {
	// chan 里元素数量
	qcount   uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf      unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed   uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx    uint   // send index
	// 已接收元素在循环数组中的索引
	recvx    uint   // receive index
	// 等待接收的 goroutine 队列
	recvq    waitq  // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq    waitq  // list of send waiters

	// 保护 hchan 中所有字段
    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

下面我们来详细介绍`hchan`中各部分是如何使用的。

## 先从创建开始

我们首先创建一个channel。

```
ch := make(chan int, 3)
```

![img](https://i6448038.github.io/img/channel/hchan1.png)

创建channel实际上就是在内存中实例化了一个`hchan`的结构体，并返回一个ch指针，我们使用过程中channel在函数之间的传递都是用的这个指针，这就是为什么函数传递中无需使用channel的指针，而直接用channel就行了，因为channel本身就是一个指针。

## channel中发送send(ch <- xxx)和recv(<- ch)接收

先考虑一个问题，如果你想让goroutine以先进先出(FIFO)的方式进入一个结构体中，你会怎么操作？
加锁！对的！channel就是用了一个锁。hchan本身包含一个互斥锁`mutex`

### channel中队列是如何实现的

channel中有个缓存buf，是用来缓存数据的(假如实例化了带缓存的channel的话)队列。我们先来看看是如何实现“队列”的。
还是刚才创建的那个channel

```
ch := make(chan int, 3)
```

![img](https://i6448038.github.io/img/channel/hchan_gif1.png)

当使用`send (ch <- xx)`或者`recv ( <-ch)`的时候，首先要锁住`hchan`这个结构体。

![img](https://i6448038.github.io/img/channel/hchan_gif2.png)

然后开始`send (ch <- xx)`数据。
一

```
ch <- 1
```

二

```
ch <- 1
```

三

```
ch <- 1
```

这时候满了，队列塞不进去了
动态图表示为：
![img](https://i6448038.github.io/img/channel/send.gif)

然后是取`recv ( <-ch)`的过程，是个逆向的操作，也是需要加锁。

![img](https://i6448038.github.io/img/channel/hchan_gif6.png)

然后开始`recv (<-ch)`数据。
一

```
<-ch
```

二

```
<-ch
```

三

```
<-ch
```

图为：
![img](https://i6448038.github.io/img/channel/recv.gif)

注意以上两幅图中`buf`和`recvx`以及`sendx`的变化，`recvx`和`sendx`是根据循环链表`buf`的变动而改变的。
至于为什么channel会使用循环链表作为缓存结构，我个人认为是在缓存列表在动态的`send`和`recv`过程中，定位当前`send`或者`recvx`的位置、选择`send`的和`recvx`的位置比较方便吧，只要顺着链表顺序一直旋转操作就好。

缓存中按链表顺序存放，取数据的时候按链表顺序读取，符合FIFO的原则。

### send/recv的具体操作

注意：缓存链表中以上每一步的操作，都是需要加锁操作的！

每一步的操作的细节可以细化为：

- 第一，加锁
- 第二，把数据从goroutine中copy到“队列”中(或者从队列中copy到goroutine中）。
- 第三，释放锁

每一步的操作总结为动态图为：(发送过程)
![img](https://i6448038.github.io/img/channel/send_single.gif)

或者为：(接收过程)
![img](https://i6448038.github.io/img/channel/recv_single.gif)

所以不难看出，Go中那句经典的话：`Do not communicate by sharing memory; instead, share memory by communicating.`的具体实现就是利用channel把数据从一端copy到了另一端！
还真是符合`channel`的英文含义：

![img](https://i6448038.github.io/img/channel/hchan_channl.gif)

### 当channel缓存满了之后会发生什么？这其中的原理是怎样的？

使用的时候，我们都知道，当channel缓存满了，或者没有缓存的时候，我们继续send(ch <- xxx)或者recv(<- ch)会阻塞当前goroutine，但是，是如何实现的呢？

我们知道，Go的goroutine是用户态的线程(`user-space threads`)，用户态的线程是需要自己去调度的，Go有运行时的scheduler去帮我们完成调度这件事情。关于Go的调度模型GMP模型我在此不做赘述，如果不了解，可以看我另一篇文章([Go调度原理](https://i6448038.github.io/2017/12/04/golang-concurrency-principle/))

goroutine的阻塞操作，实际上是调用`send (ch <- xx)`或者`recv ( <-ch)`的时候主动触发的，具体请看以下内容：

```
//goroutine1 中，记做G1

ch := make(chan int, 3)

ch <- 1
ch <- 1
ch <- 1
```

![img](https://i6448038.github.io/img/channel/hchan_block.png)

![img](https://i6448038.github.io/img/channel/hchan_block1.png)

这个时候G1正在正常运行,当再次进行send操作(ch<-1)的时候，会主动调用Go的调度器,让G1等待，并从让出M，让其他G去使用

![img](https://i6448038.github.io/img/channel/hchan_block2.png)

同时G1也会被抽象成含有G1指针和send元素的`sudog`结构体保存到hchan的`sendq`中等待被唤醒。

![img](https://i6448038.github.io/img/channel/hchan_blok3.gif)

那么，G1什么时候被唤醒呢？这个时候G2隆重登场。

![img](https://i6448038.github.io/img/channel/hchan_block4.png)

G2执行了recv操作`p := <-ch`，于是会发生以下的操作：

![img](https://i6448038.github.io/img/channel/hchan_block5.gif)

G2从缓存队列中取出数据，channel会将等待队列中的G1推出，将G1当时send的数据推到缓存中，然后调用Go的scheduler，唤醒G1，并把G1放到可运行的Goroutine队列中。

![img](https://i6448038.github.io/img/channel/hchan_block6.gif)

### 假如是先进行执行recv操作的G2会怎么样？

你可能会顺着以上的思路反推。首先：

![img](https://i6448038.github.io/img/channel/hchan_block7_1.png)

这个时候G2会主动调用Go的调度器,让G2等待，并从让出M，让其他G去使用。
G2还会被抽象成含有G2指针和recv空元素的`sudog`结构体保存到hchan的`recvq`中等待被唤醒

![img](https://i6448038.github.io/img/channel/hchan_block7.gif)

此时恰好有个goroutine G1开始向channel中推送数据 `ch <- 1`。
此时，非常有意思的事情发生了：

![img](https://i6448038.github.io/img/channel/hchan_block8.gif)

G1并没有锁住channel，然后将数据放到缓存中，而是直接把数据从G1直接copy到了G2的栈中。
这种方式非常的赞！在唤醒过程中，G2无需再获得channel的锁，然后从缓存中取数据。减少了内存的copy，提高了效率。

之后的事情显而易见：
![img](https://i6448038.github.io/img/channel/hchan_block9.gif)