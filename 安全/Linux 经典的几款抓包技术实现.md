# Linux 经典的几款抓包技术实现

> 装载自网络安全研发随想，原文链接：https://z.itpub.net/article/detail/E7282132F901DCA52E32EB06F25D6CF2.

本文列举四个比较经典的 Linux 抓包技术实现，如果还有其他你觉得ok的可以留言。这四个分别是：

- libpcap/libpcap-mmap
- PF_RING
- DPDK
- xdp

## libpcap

libpcap的包捕获机制是在数据链路层增加一个旁路处理，不干扰系统自身的网路协议栈的处理，对发送和接收的数据包通过Linux内核做过滤和缓冲处理，后直接传递给上层应用程序。

1. 数据包到达网卡设备。
2. 网卡设备依据配置进行DMA操作。（ **「第1次拷贝」** ：网卡寄存器->内核为网卡分配的缓冲区ring buffer）
3. 网卡发送中断，唤醒处理器。
4. 驱动软件从ring buffer中读取，填充内核skbuff结构（ **「第2次拷贝」** ：内核网卡缓冲区ring buffer->内核专用数据结构skbuff）
5. 接着调用netif_receive_skb函数：

- 5.1 如果有抓包程序，由网络分接口进入BPF过滤器，将规则匹配的报文拷贝到系统内核缓存 （ **「第3次拷贝」** ）。BPF为每一个要求服务的抓包程序关联一个filter和两个buffer。BPF分配buffer 且通常情况下它的额度是4KB the store buffer 被使用来接收来自适配器的数据；the hold buffer被使用来拷贝包到应用程序。
- 5.2 处理数据链路层的桥接功能；
- 5.3 根据skb->protocol字段确定上层协议并提交给网络层处理，进入网络协议栈，进行高层处理。

libpcap绕过了Linux内核收包流程中协议栈部分的处理，使得用户空间API可以直接调用套接字PF_PACKET从链路层驱动程序中获得数据报文的拷贝，将其从内核缓冲区拷贝至用户空间缓冲区（ **「第4次拷贝」** ）

## libpcap-mmap

libpcap-mmap是对旧的libpcap实现的改进，新版本的libpcap基本都采用packet_mmap机制。PACKET_MMAP通过mmap，减少一次内存拷贝（ **「第4次拷贝没有了」** ），减少了频繁的系统调用，大大提高了报文捕获的效率。

## PF_RING

我们看到之前libpcap有4次内存拷贝。libpcap_mmap有3次内存拷贝。PF_RING提出的核心解决方案便是减少报文在传输过程中的拷贝次数。

我们可以看到，相对与libpcap_mmap来说，pfring允许用户空间内存直接和rx_buffer做mmap。这又减少了一次拷贝 （ **「libpcap_mmap的第2次拷贝」** ：rx_buffer->skb）

PF-RING ZC实现了DNA（Direct NIC Access 直接网卡访问）技术，将用户内存空间映射到驱动的内存空间，使用户的应用可以直接访问网卡的寄存器和数据。

通过这样的方式，避免了在内核对数据包缓存，减少了一次拷贝（ **「libpcap的第1次拷贝」** ，DMA到内核缓冲区的拷贝）。这就是完全的零拷贝。

其缺点是，只有一个 应用可以在某个时间打开DMA ring（请注意，现在的网卡可以具有多个RX / TX队列，从而就可以在每个队列上同时一个应用程序），换而言之，用户态的多个应用需要彼此沟通才能分发数据包。

## DPDK

pf-ring zc和dpdk均可以实现数据包的零拷贝，两者均旁路了内核，但是实现原理略有不同。pf-ring zc通过zc驱动（也在应用层）接管数据包，dpdk基于UIO实现。

### 1 UIO+mmap 实现零拷贝（zero copy）

UIO（Userspace I/O）是运行在用户空间的I/O技术。Linux系统中一般的驱动设备都是运行在内核空间，而在用户空间用应用程序调用即可，而UIO则是将驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能。采用Linux提供UIO机制，可以旁路Kernel，将所有报文处理的工作在用户空间完成。

### 2 UIO+PMD 减少中断和CPU上下文切换

DPDK的UIO驱动屏蔽了硬件发出中断，然后在用户态采用主动轮询的方式，这种模式被称为PMD（Poll Mode Driver）。

与DPDK相比，pf-ring（no zc）使用的是NAPI polling和应用层polling，而pf-ring zc与DPDK类似，仅使用应用层polling。

### 3 HugePages 减少TLB miss

在操作系统引入MMU（Memory Management Unit）后，CPU读取内存的数据需要两次访问内存。次要查询页表将逻辑地址转换为物理地址，然后访问该物理地址读取数据或指令。

为了减少页数过多，页表过大而导致的查询时间过长的问题，便引入了TLB(Translation Lookaside Buffer)，可翻译为地址转换缓冲器。TLB是一个内存管理单元，一般存储在寄存器中，里面存储了当前可能被访问到的一小部分页表项。

引入TLB后，CPU会首先去TLB中寻址，由于TLB存放在寄存器中，且其只包含一小部分页表项，因此查询速度非常快。若TLB中寻址成功（TLB hit），则无需再去RAM中查询页表；若TLB中寻址失败（TLB miss），则需要去RAM中查询页表，查询到后，会将该页更新至TLB中。

而DPDK采用HugePages ，在x86-64下支持2MB、1GB的页大小，大大降低了总页个数和页表的大小，从而大大降低TLB miss的几率，提升CPU寻址性能。

### 4 其它优化

- SNA（Shared-nothing Architecture），软件架构去中心化，尽量避免全局共享，带来全局竞争，失去横向扩展的能力。NUMA体系下不跨Node远程使用内存。
- SIMD（Single Instruction Multiple Data），从早的mmx/sse到新的avx2，SIMD的能力一直在增强。DPDK采用批量同时处理多个包，再用向量编程，一个周期内对所有包进行处理。比如，memcpy就使用SIMD来提高速度。
- cpu affinity：即 CPU 亲和性

## XDP

xdp代表eXpress数据路径，使用ebpf 做包过滤，相对于dpdk将数据包直接送到用户态，用用户态当做快速数据处理平面，xdp是在驱动层创建了一个数据快速平面。在数据被网卡硬件dma到内存，分配skb之前，对数据包进行处理。

请注意，XDP并没有对数据包做Kernel bypass，它只是提前做了一点预检而已。

相对于DPDK，XDP具有以下优点：

- 无需第三方代码库和许可
- 同时支持轮询式和中断式网络
- 无需分配大页
- 无需专用的CPU
- 无需定义新的安全网络模型

XDP的使用场景包括：

- DDoS防御
- 防火墙
- 基于XDP_TX的负载均衡
- 网络统计
- 复杂网络采样
- 高速交易平台

