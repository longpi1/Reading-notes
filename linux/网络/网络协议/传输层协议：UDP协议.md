#                               传输层协议：UDP协议

## 简介

**用户数据报协议**（英语：**U**ser **D**atagram **P**rotocol，缩写：**UDP**；又称**用户数据包协议**）是一个简单的面向[数据包](https://zh.wikipedia.org/wiki/資料包)的[通信协议](https://zh.wikipedia.org/wiki/通信协议)，位于[OSI模型](https://zh.wikipedia.org/wiki/OSI模型)的[传输层](https://zh.wikipedia.org/wiki/传输层)。该协议由[David P. Reed](https://zh.wikipedia.org/w/index.php?title=David_P._Reed&action=edit&redlink=1)在1980年设计且在[RFC 768](https://tools.ietf.org/html/rfc768)中被规范。典型网络上的众多使用UDP协议的关键应用在一定程度上是相似的。

是一个与 [IP 协议](https://developer.mozilla.org/zh-CN/docs/Glossary/IPv6) 一起使用的长期[协议](https://developer.mozilla.org/zh-CN/docs/Glossary/Protocol)，用于在传输速度和效率比安全性和可靠性更重要的场合下发送数据。

## UDP的报头是怎么样的？

当我们发送 UDP 包到达目标机器后，发现 MAC 地址匹配，于是就取下来，将剩下的包传给处理 IP 层的代码。把 IP 头取下来，发现目标 IP 匹配，接下来这里面的数据包是给谁呢？

发送的时候，我知道我发的是一个 UDP 的包，收到的那台机器咋知道的呢？所以在 IP 头里面有个 8 位协议，这里会存放，数据里面到底是 TCP 还是 UDP，当然这里是 UDP。所以，如果我们知道 UDP 头的格式，就能从数据里面，将它解析出来。解析出来以后呢？数据给谁处理呢？

**UDP报头结构如下**：

![报头分组结构](https://s2.loli.net/2023/01/07/ZcromS8aTj7Cyqw.png)

UDP报头包括4个字段，每个字段占用2个字节（即16个二进制位）。在IPv4中，“来源连接端口”和“校验和”是可选字段（以粉色背景标出）。在IPv6中，只有来源连接端口是可选字段。 各16[bit](https://zh.wikipedia.org/wiki/位元)的来源端口和目的端口用来标记发送和接受的应用进程。因为UDP不需要应答，所以来源端口是可选的，如果来源端口不用，那么置为零。在目的端口后面是长度固定的以字节为单位的长度域，用来指定UDP数据报包括数据部分的长度，长度最小值为8byte。首部剩下地16bit是用来对首部和数据部分一起做[校验和](https://zh.wikipedia.org/wiki/校验和)（Checksum）的，这部分是可选的，但在实际应用中一般都使用这一功能。

- 报文长度

  该字段指定UDP报头和数据总共占用的长度。可能的最小长度是8字节，因为UDP报头已经占用了8字节。由于这个字段的存在，UDP报文总长不可能超过65535字节（包括8字节的报头，和65527字节的数据）。实际上通过[IPv4](https://zh.wikipedia.org/wiki/IPv4)协议传输时，由于IPv4的头部信息要占用20字节，因此数据长度不可能超过65507字节（65,535 − 8字节UDP报头 − 20字节[IP头部](https://zh.wikipedia.org/wiki/IPv4#.E9.A6.96.E9.83.A8)）。在IPv6的[jumbogram](https://zh.wikipedia.org/w/index.php?title=Jumbogram&action=edit&redlink=1)中，是有可能传输超过65535字节的UDP数据包的。依据[RFC](https://zh.wikipedia.org/wiki/RFC) [2675](https://tools.ietf.org/html/rfc2675)，如果这种情况发生，报文长度应被填写为0。

- 校验和

  [校验和](https://zh.wikipedia.org/wiki/校验和)字段可以用于发现头部信息和数据中的传输错误。该字段在IPv4中是可选的，在IPv6中则是强制的。如果不使用校验和，该字段应被填充为全0。

## 主要特点

- **无连接**，即发送数据之前不需要建立连接(发送数据结束时也没有连接可释放)，减少了开销和发送数据之前的时延
- **尽最大努力交付**，即不保证可靠交付，主机不需要维持复杂的连接状态表，例如流媒体，实时多人游戏和IP语音（[VoIP](https://zh.wikipedia.org/wiki/VoIP)）是经常使用UDP的应用程序。 在这些特定应用中，丢包通常不是重大问题。如果应用程序需要高度可靠性，则可以使用诸如[TCP](https://zh.wikipedia.org/wiki/传输控制协议)之类的协议。
- **面向报文**，发送方的 UDP 对应用程序交下来的报文，在添加首部后就向下交付 IP 层。UDP 对应用层交下来的报文，既不合并，也不拆分，而是保留这些报文的边界
- 支持**一对一、一对多、多对一和多对多的交互通信**
- **首部开销小**，只有8个字节，比 TCP 的20个字节的首部要短
- 缺乏[拥塞控制](https://zh.wikipedia.org/wiki/拥塞控制)，所以需要基于网络的机制来减少因失控和高速UDP流量负荷而导致的拥塞崩溃效应。换句话说，因为UDP发送端无法检测拥塞，所以像使用包队列和丢弃技术的路由器之类的网络基础设备会被用于降低UDP过大流量。[数据拥塞控制协议](https://zh.wikipedia.org/wiki/数据拥塞控制协议)（DCCP）设计成通过在诸如流媒体类型的高速率UDP流中增加主机拥塞控制，来减小这个潜在的问题。

## 应用场景

许多关键的互联网应用程序使用UDP[[2\]](https://zh.wikipedia.org/wiki/用户数据报协议#cite_note-kuroseross-2)，包括：

- [域名系统](https://zh.wikipedia.org/wiki/域名系统)（DNS），其中查询阶段必须快速，并且只包含单个请求，后跟单个回复数据包；
- [动态主机配置协议](https://zh.wikipedia.org/wiki/动态主机配置协议)（DHCP），用于动态分配[IP地址](https://zh.wikipedia.org/wiki/IP地址)；
- [简单网络管理协议](https://zh.wikipedia.org/wiki/简单网络管理协议)（SNMP）；
- [路由信息协议](https://zh.wikipedia.org/wiki/路由信息协议)（RIP）；
- [网络时间协议](https://zh.wikipedia.org/wiki/網路時間協定)（NTP）。

[流媒体](https://zh.wikipedia.org/wiki/串流媒體)、[在线游戏](https://zh.wikipedia.org/wiki/線上遊戲)流量通常使用UDP传输。 实时视频流和音频流应用程序旨在处理偶尔丢失、错误的数据包，因此只会发生质量轻微下降，而避免了重传[数据包](https://zh.wikipedia.org/wiki/数据包)带来的高[延迟](https://zh.wikipedia.org/wiki/延遲)。 由于TCP和UDP都在同一网络上运行，因此一些企业发现来自这些实时应用程序的UDP流量影响了使用TCP的应用程序的性能，例如[销售](https://zh.wikipedia.org/wiki/销售)、[会计](https://zh.wikipedia.org/wiki/会计)和[数据库系统](https://zh.wikipedia.org/wiki/数据库系统)。 当TCP检测到数据包丢失时，它将限制其数据速率使用率。由于实时和业务应用程序对企业都很重要，因此一些人认为开发[服务质量](https://zh.wikipedia.org/wiki/服务质量)解决方案至关重要。[[3\]](https://zh.wikipedia.org/wiki/用户数据报协议#cite_note-3)

一些[VPN](https://zh.wikipedia.org/wiki/VPN)应用（如[OpenVPN](https://zh.wikipedia.org/wiki/OpenVPN)）使用UDP并可以在[应用程序](https://zh.wikipedia.org/wiki/应用程序)级别实现可靠连接和错误检查。

## 参考链接

1.https://zh.wikipedia.org/wiki/%E7%94%A8%E6%88%B7%E6%95%B0%E6%8D%AE%E6%8A%A5%E5%8D%8F%E8%AE%AE