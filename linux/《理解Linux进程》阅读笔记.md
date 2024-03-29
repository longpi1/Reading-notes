#             《理解Linux进程》阅读笔记

> 本篇文章主要是阅读《理解Linux进程》对相关知识点的简单的介绍与扩展

## 1.退出码

任何进程退出时，都会留下退出码，操作系统根据退出码可以知道进程是否正常运行。

退出码是0到255的整数，通常0表示正常退出，其他数字表示不同的错误。

**不管是正常退出还是异常退出，进程都结束了这个退出码有意义吗？**

有意义，我们在写Bash脚本时，可以根据前一个命令的退出码选择是否执行下一个命令。例如Docker使用Dockerfile来构建镜像，这是类似Bash的领域定义语言(DSL)，每一行执行一个命令，如果命令的进程退出码不为0，构建镜像的流程就会中止，证明Dockerfile有异常，方便用户排查问题。



## 2.进程状态转换

![进程状态转换图](https://s2.loli.net/2023/03/01/FOuhP5nHI1tQU8T.png)

### 2.1 查看状态

通过`ps aux`可以看到进程的状态。

O：进程正在处理器运行,这个状态从来没有见过.
S：休眠状态（sleeping）
R：等待运行（runable）R Running or runnable (on run queue) 进程处于运行或就绪状态
I：空闲状态（idle）
Z：僵尸状态（zombie）
T：跟踪状态（Traced）
B：进程正在等待更多的内存页
D: 不可中断的深度睡眠，一般由IO引起，同步IO在做读或写操作时，cpu不能做其它事情，只能等待，这时进程处于这种状态，如果程序采用异步IO，这种状态应该就很少见到了

其中就绪状态表示进程已经分配到除CPU以外的资源，等CPU调度它时就可以马上执行了。运行状态就是正在运行了，获得包括CPU在内的所有资源。等待状态表示因等待某个事件而没有被执行，这时候不耗CPU时间，而这个时间有可能是等待IO、申请不到足够的缓冲区或者在等待信号。



## 3.活锁

### 3.1 活锁实例

举个很简单的例子，两个人相向过独木桥，他们同时向一边谦让，这样两个人都过不去，然后二者同时又移到另一边，这样两个人又过不去了。如果不受其他因素干扰，两个人一直同步在移动，但外界看来两个人都没有前进，这就是活锁。

活锁会导致CPU耗尽的，解决办法是引入随机变量、增加重试次数等。

所以活锁也是程序设计上可能存在的问题，导致进程都没办法运行下去了，还耗CPU。

## 4.POSIX简介

POSIX(Portable Operation System Interface)就是一种操作系统的接口标准

### 4.1 POSIX进程

我们运行Hello World程序时，操作系统通过POSIX定义的`fork`和`exec`接口创建起一个POSIX进程，这个进程就可以使用通用的IPC、信号等机制。

### 4.2 POSIX线程

POSIX也定义了线程的标准，包括创建和控制线程的API，在Pthreads库中实。



## 5.Nohup命令

每个开发者都会躺过这个坑，在命令行跑一个后台程序，关闭终端后发现进程也退出了，网上搜一下发现要用`nohup`，究竟什么原因呢？

普通进程运行时默认会绑定TTY(虚拟终端)，关闭终端后系统会给上面所有进程发送TERM信号，这时普通进程也就退出了。当然还有些进程不会退出，这就是后面将会提到的守护进程。

`Nohup`的原理也很简单，终端关闭后会给此终端下的每一个进程发送SIGHUP信号，而使用`nohup`运行的进程则会忽略这个信号，因此终端关闭后进程也不会退出。



## 6.进程锁

这里的进程锁与线程锁、互斥量、读写锁和自旋锁不同，它是通过记录一个PID文件，避免两个进程同时运行的文件锁。

进程锁的作用之一就是可以协调进程的运行，例如[crontab使用进程锁解决冲突](http://www.live-in.org/archives/1036.html)提到，使用crontab限定每一分钟执行一个任务，但这个进程运行时间可能超过一分钟，如果不用进程锁解决冲突的话两个进程一起执行就会有问题。后面提到的项目实例Run也有类似的问题，通过进程锁可以解决进程间同步的问题。

使用PID文件锁还有一个好处，方便进程向自己发停止或者重启信号。Nginx编译时可指定参数`--pid-path=/var/run/nginx.pid`，进程起来后就会把当前的PID写入这个文件，当然如果这个文件已经存在了，也就是前一个进程还没有退出，那么Nginx就不会重新启动。进程管理工具Supervisord也是通过记录进程的PID来停止或者拉起它监控的进程的。



## 7.孤儿进程与僵尸进程

### 7.1 孤儿进程

孤儿进程指的是在其父进程执行完成或被终止后仍继续运行的一类进程。

孤儿进程与僵尸进程是完全不同的，孤儿进程借用了现实中孤儿的概念，也就是父进程不在了，子进程还在运行，这时我们就把子进程的PPID设为1。前面讲PID提到，操作系统会创建进程号为1的init进程，它没有父进程也不会退出，可以收养系统的孤儿进程。

### 7.2 孤儿进程用途

在现实中用户可能刻意使进程成为孤儿进程，这样就可以让它与父进程会话脱钩，例如守护进程。

### 7.3 僵尸进程

当一个进程完成它的工作终止之后，它的父进程需要调用wait()或者waitpid()系统调用取得子进程的终止状态。

一个进程使用fork创建子进程，如果子进程退出，而父进程并没有调用wait或waitpid获取子进程的状态信息，那么子进程的进程描述符仍然保存在系统中。这种进程称之为僵死进程。



## 8.进程间通信

IPC全称Interprocess Communication，指进程间协作的各种方法，当然包括共享内存，信号量或Socket等。

### 8.1 管道(Pipe)

管道是进程间通信最简单的方式，任何进程的标准输出都可以作为其他进程的输入。

### 8.2 信号(Signal)

我们知道信号是进程间通信的其中一种方法，当然也可以是内核给进程发送的消息，注意信息只是告诉进程发生了什么事件，而不会传递任何数据。

简单介绍几个我们最常用的，在命令行中止一个程序我们一般摁Ctrl+c，这就是发送SIGINT信号，而使用kill命令呢？默认是SIGTERM，加上`-9`参数才是SIGKILL。

### 8.3 消息队列(Message)

和传统消息队列类似，但是在内核实现的。

### 8.4 共享内存(Shared Memory)

不同进程之间内存空间是独立的，也就是说进程不能访问也不会干扰其他进程的内存。如果两个进程希望通过共享内存的方式通信呢？可以通过`mmap()`系统调用实现。

### 8.5 信号量(Semaphore)

信号量本质上是一个整型计数器，**调用`wait`时计数减一，减到零开始阻塞进程，从而达到进程、线程间协作的作用。**

### 8.6 套接口(Socket)

也就是通过网络来通信，这也是**最通用的IPC**，不要求进程在同一台服务器上。



## 9.文件描述符

Linux很重要的设计思想就是一切皆文件，网络是文件，键盘等外设也是文件？于是所有资源都有了统一的接口，开发者可以像写文件那样通过网络传输数据，我们也可以通过`/proc/`的文件看到进程的资源使用情况。

内核给每个访问的文件分配了**文件描述符(File Descriptor)**，它本质是一个非负整数，在打开或新建文件时返回，以后读写文件都要通过这个文件描述符了。

### 9.1 应用

操作系统打开的文件这么多，不可能他们共用一套文件描述符整数吧？其实Linux实现时这个fd是一个索引值，指向每个进程打开文件的记录表。

POSIX已经定义了**STDIN_FILENO、STDOUT_FILENO和STDERR_FILENO三个常量，也就是0、1、2**。这三个文件描述符是每个进程都有的，这也解释了为什么每个进程都有编号为0、1、2的文件而不会与其他进程冲突。

文件描述符帮助应用找到这个文件，而文件的打开模式等上下文信息存储在文件对象中，这个对象直接与文件描述符关联。



## 10.Epoll

### 10.1 简介

Epoll是poll的改进版，更加高效，能同时处理大量文件描述符，跟高并发有关，Nginx就是充分利用了epoll的特性。讲这些没用，我们先了解poll是什么。

### 10.2 Poll

Poll本质上是Linux系统调用，其接口为`int poll(struct pollfd *fds,nfds_t nfds, int timeout)`，作用是监控资源是否可用。

举个例子，一个Web服务器建了多个socket连接，它需要知道里面哪些连接传输发了请求需要处理，功能与`select`系统调用类似，不过`poll`不会清空文件描述符集合，因此检测大量socket时更加高效。

### 10.3 Epoll

我们重点看看epoll，它大幅提升了高并发服务器的资源使用率，相比poll而言哦。前面提到poll会轮询整个文件描述符集合，而epoll可以做到只查询被内核IO事件唤醒的集合，当然它还提供边沿触发(Edge Triggered)等特性。

不知大家是否了解C10K问题，指的是服务器如何支持同时一万个连接的问题。如果是一万个连接就有至少一万个文件描述符，poll的效率也随文件描述符的更加而下降，epoll不存在这个问题是因为它仅关注活跃的socket。

### 10.4 实现

这是怎么做到的呢？简单来说epoll是基于文件描述符的callback函数来实现的，只有发生IO时间的socket会调用callback函数，然后加入epoll的Ready队列。更多实现细节可以参考Linux源码，

### 10.5 Mmap

无论是select、poll还是epoll，他们都要把文件描述符的消息送到用户空间，这就存在内核空间和用户空间的内存拷贝。其中epoll使用mmap来共享内存，提高效率。

Mmap不是进程的概念，这里提一下是因为epoll使用了它，这是一种共享内存的方法，而Go语言的设计宗旨是"**不要通过共享来通信，通过通信来共享**"，所以我们也可以思考下进程的设计，是使用mmap还是Go提供的channel机制呢。



## 11.写时复制(Copy On Write)

一般我们运行程序都是Fork一个进程后马上执行Exec加载程序，而Fork的是否实际上用的是父进程的堆栈空间，Linux通过Copy On Write技术极大地减少了Fork的开销。

**Copy On Write的含义是只有真正写的时候才把数据写到子进程的数据，Fork时只会把页表复制到子进程，这样父子进程都指向同一个物理内存页，只有再写子进程的时候才会把内存页的内容重新复制一份。**



## 12.Cgroups与Namespaces

### 12.1　Cgroups

**Cgroups**全称Control Groups，是Linux内核用于资源隔离的技术。目前Cgroups可以控制CPU、内存、磁盘访问。

### 12.２　Namespaces

Linux Namespaces是资源隔离技术，在2.6.23合并到内核，而在3.12内核加入对用户空间的支持。

Namespaces是容器技术的基础，因为有了命名空间的隔离，才能限制容器之间的进程通信，像虚拟内存对于物理内存那样，开发者无需针对容器修改已有的代码。



## 13.捕获SIGKILL

SIGKILL是常见的Linux信号，我们使用`kill`命令杀掉进程也就是像进程发送SIGKILL信号。

和其他信号不同，**[SIGKILL](https://en.wikipedia.org/wiki/Unix_signal#SIGKILL)和SIGSTOP是不可被Catch的，**因此下面的代码是能编译通过但也是无效的，更多细节可以参考[golang/go#9463](https://github.com/golang/go/issues/9463).

```
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGKILL, syscall.SIGSTOP)
```



## 14.系统调用sendfile

[Sendfile](http://man7.org/linux/man-pages/man2/sendfile.2.html)是Linux实现的系统调用，可以通过避免文件在内核态和用户态的拷贝来优化文件传输的效率。

其中大名鼎鼎的分布式消息队列服务Kafka就使用sendfile来优化效率，具体用法可参见其[官方文档](http://kafka.apache.org/documentation.html)。



## 参考书籍

- [理解Linux进程](https://github.com/tobegit3hub/understand_linux_process)
- [理解Unix进程](http://book.douban.com/subject/24298701/)
- [Unix编程艺术](http://book.douban.com/subject/1467587/)
- [Unix环境高级编程](http://book.douban.com/subject/1788421/)
- [Go Web编程](http://book.douban.com/subject/24316255/)
- [Go并发编程实战](http://book.douban.com/subject/26244729/)