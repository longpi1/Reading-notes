#                            传输层协议：套接字Socket

## 介绍

socket是一种操作系统提供的进程间通信机制。

在[操作系统](https://zh.m.wikipedia.org/wiki/作業系統)中，通常会为应用程序提供一组[应用程序接口](https://zh.m.wikipedia.org/wiki/應用程式介面)（API），称为**套接字接口**（英语：socket API）。应用程序可以通过套接字接口，来使用网络套接字，以进行资料交换。最早的套接字接口来自于4.2 BSD，因此现代常见的套接字接口大多源自[Berkeley套接字](https://zh.m.wikipedia.org/wiki/Berkeley套接字)（Berkeley sockets）标准。在套接字接口中，以[IP地址](https://zh.m.wikipedia.org/wiki/IP地址)及[端口](https://zh.m.wikipedia.org/wiki/通訊埠)组成**套接字地址**（socket address）。远程的套接字地址，以及本地的套接字地址完成连线后，再加上使用的协议（protocol），这个五元组（five-element tuple），作为**套接字对**（socket pairs），之后就可以彼此交换资料。例如，在同一台计算机上，TCP协议与UDP协议可以同时使用相同的port而互不干扰。 操作系统根据套接字地址，可以决定应该将资料送达特定的[行程](https://zh.m.wikipedia.org/wiki/行程)或[线程](https://zh.m.wikipedia.org/wiki/執行緒)。这就像是[电话](https://zh.m.wikipedia.org/wiki/電話)系统中，以[电话号码](https://zh.m.wikipedia.org/wiki/電話號碼)加上分机号码，来决定通话对象一般。

有下面几种特征：

- 本地接口地址，由本地ip地址和（包括TCP，UDP）端口号
- 传输协议，例如TCP、UDP、raw IP协议

一个已经创建连接的接口双方都有整数形式的接口描述符，用来唯一表示该接口。操作系统根据对方接口发过来的IP以及传输协议头信息来提取接口的地址信息，并且将应用数据去除头信息之后提交给相应的应用程序。 在很多网络协议、教科书以及本文中，接口指的是有一个独一无二的接口号的实体。在一些其他的文章[[来源请求\]](https://zh.m.wikipedia.org/wiki/Wikipedia:列明来源)当中，接口被叫做本地接口地址，比如"ip和端口的结合"。在一RFC147标准中，这个定义与1971的ARPA网有关，接口指的是一个32位数字，其中偶数的是接收接口，奇数的是发送接口，但是今天通信已经可以实现双向传输，在一个接口中，可以发送的同时还可以接收。

在类UNIX系统和Windows系统，命令行工具netstat和ss可用以查看当前系统的接口情况。



## 类型

### 数据报套接字（SOCK_DGRAM）—— 基于UDP的Socket

数据报套接字是一种无连套接字接字，使用[用户数据报协议](https://zh.m.wikipedia.org/wiki/用户数据报协议)（UDP）传输数据。每一个数据包都单独寻址和路由。这导致了接收端接收到的数据可能是乱序的，有一些数据甚至可能会在传输过程中丢失。不过得益于数据报套接字并不需要创建并维护一个稳定的连接，数据报套接字所占用的计算机和系统资源较小。

### 流套接字（SOCK_STREAM）—— 基于TCP的Socket

连接导向式通信套接字，使用[传输控制协议](https://zh.m.wikipedia.org/wiki/传输控制协议)（TCP）、[流控制传输协议](https://zh.m.wikipedia.org/wiki/流控制传输协议)（SCTP）或者[数据拥塞控制协议](https://zh.m.wikipedia.org/wiki/数据拥塞控制协议)（DCCP）传输数据。流套接字提供可靠并且有序的数据传输服务。在互联网上，流套接字通常使用TCP实现，以便应用可以在任何使用TCP/IP协议的网络上运行。

### 原始套接字

[原始套接字](https://zh.m.wikipedia.org/wiki/原始套接字)是一种[网络套接字](https://zh.m.wikipedia.org/wiki/网络套接字)。允许直接发送和接受IP数据包并且**不需要任何传输层协议格式**。原始套接字主要用于一些协议的开发，可以进行比较底层的操作。

原始套接字用于安全相关的应用程序，如[nmap](https://zh.m.wikipedia.org/wiki/Nmap)。原始套接字一种可能的用例是在[用户空间](https://zh.m.wikipedia.org/wiki/使用者空間)实现新的传输层协议。[[1\]](https://zh.m.wikipedia.org/zh-cn/原始套接字#cite_note-1) 原始套接字常在网络设备上用于[路由协议](https://zh.m.wikipedia.org/wiki/路由协议)，例如[IGMP](https://zh.m.wikipedia.org/wiki/因特网组管理协议)v4、[开放式最短路径优先](https://zh.m.wikipedia.org/wiki/开放式最短路径优先)协议 (OSPF)、[互联网控制消息协议](https://zh.m.wikipedia.org/wiki/互联网控制消息协议) (ICMP)。[Ping](https://zh.m.wikipedia.org/wiki/Ping)就是发送一个ICMP响应请求包然后接收ICMP响应回复



相关的性能分析可参考[Unix Domain Socket 性能分析](https://blog.longpi1.com/2022/11/25/UnixDomainSocket%E6%80%A7%E8%83%BD%E5%88%86%E6%9E%90/) 



## 流套接字（SOCK_STREAM）—— 基于TCP的Socket

### TCP 协议的 Socket 程序函数调用过程如下：

1. TCP 的服务端要先监听一个端口，一般是先调用 bind 函数，给这个 Socket 赋予一个 IP 地址和端口。为什么需要端口呢？要知道，你写的是一个应用程序，当一个网络包来的时候，内核要通过 TCP 头里面的这个端口，来找到你这个应用程序，把包给你。为什么要 IP 地址呢？有时候，一台机器会有多个网卡，也就会有多个 IP 地址，你可以选择监听所有的网卡，也可以选择监听一个网卡，这样，只有发给这个网卡的包，才会给你。
2. 当服务端有了 IP 和端口号，就可以调用 listen 函数进行监听。在 TCP 的状态图里面，有一个 listen 状态，当调用这个函数之后，服务端就进入了这个状态，这个时候客户端就可以发起连接了。
3. 在内核中，为每个 Socket 维护两个队列。一个是已经建立了连接的队列，这时候连接三次握手已经完毕，处于 established 状态；一个是还没有完全建立连接的队列，这个时候三次握手还没完成，处于 syn_rcvd 的状态。
4. 接下来，服务端调用 accept 函数，拿出一个已经完成的连接进行处理。如果还没有完成，就要等着。
5. 在服务端等待的时候，客户端可以通过 connect 函数发起连接。先在参数中指明要连接的 IP 地址和端口号，然后开始发起三次握手。内核会给客户端分配一个临时的端口。一旦握手成功，服务端的 accept 就会返回另一个 Socket。这是需要注意，就是监听的 Socket 和真正用来传数据的 Socket 是两个，一个叫作**监听 Socket**，一个叫作**已连接 Socket**。
6. 连接建立成功之后，双方开始通过 read 和 write 函数来读写数据，就像往一个文件流里面写东西一样。

如下图所示：

![img](https://static001.geekbang.org/resource/image/87/ea/87c8ae36ae1b42653565008fc47aceea.jpg?wh=1626*2172)



## 数据报套接字（SOCK_DGRAM）—— 基于UDP的Socket

### 基于 UDP 协议的 Socket 程序函数调用过程：

对于 UDP 来讲，过程有些不一样。UDP 是没有连接的，所以不需要三次握手，也就不需要调用 listen 和 connect，但是，UDP 的交互仍然需要 IP 和端口号，因而也需要 bind。UDP 是没有维护连接状态的，因而不需要每对连接建立一组 Socket，而是只要有一个 Socket，就能够和多个客户端通信。也正是因为没有连接状态，每次通信的时候，都调用 sendto 和 recvfrom，都可以传入 IP 地址和端口。

如下图所示：

![img](https://static001.geekbang.org/resource/image/6b/31/6bbe12c264f5e76a81523eb8787f3931.jpg?wh=1245*1261)

# 参考链接

1.维基百科，https://zh.m.wikipedia.org/zh-cn/%E7%B6%B2%E8%B7%AF%E6%8F%92%E5%BA%A7

2.趣谈网络协议，https://time.geekbang.org/column/article/9293