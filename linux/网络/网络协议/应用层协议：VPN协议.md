#                                                 应用层协议：VPN协议

## 概述

**VPN**，全名 **Virtual Private Network，虚拟专用网**，就是**利用开放的公众网络，建立专用数据传输通道**，将远程的分支机构、移动办公人员等连接起来。



## 工作机制

VPN 通过隧道技术在公众网络上仿真一条点到点的专线，是通过利用一种协议来传输另外一种协议的技术，这里面涉及三种协议：乘客协议、隧道协议和承载协议。

**以 IPsec VPN协议为例来说明。**

Internet Protocol Security (IPsec)是一个安全网络协议套件，用于保护互联网或公共网络传输的数据。它可以完成认证加密数据包，从而为IP层网络设备提供安全加密通信。IPSec还被用到了VPN中。

IPsec包括了用于在客户端之间开始会话前建立相互认证和会话所用秘钥分发的协议簇。IPsec可以保护数据流在端到端，网络到网络，端到网络之间。IPsec使用密码安全服务来保护IP网络中的通信。它支持网络层对等身份认证，数据源认证，数据加密和重放保护。

### 特点

- **私密性**，防止信息泄露给未经授权的个人，通过加密把数据从明文变成无法读懂的密文，从而确保数据的私密性。加密可以分为对称加密和非对称加密。对称加密速度快一些。而 VPN 一旦建立，需要传输大量数据，因而我们采取对称加密。但是同样，对称加密还是存在加密密钥如何传输的问题，这里需要用到因特网密钥交换（IKE，Internet Key Exchange）协议。
- **完整性**，数据没有被非法篡改，通过对数据进行 hash 运算，产生类似于指纹的数据摘要，以保证数据的完整性。
- **真实性**，数据确实是由特定的对端发出，通过身份认证可以保证数据的真实性。

**IPSec 不是一个协议，而是一套协议**，基于以上三个特性，组成了 IPsec VPN 的协议簇。

![img](https://static001.geekbang.org/resource/image/e4/c2/e43f13e5c68c9a5455b3793fb530a4c2.jpeg?wh=1920*742)

这个协议簇内容比较丰富。在这个协议簇里面，有两种协议，这两种协议的区别在于封装网络包的格式不一样。

- 一种协议称为 **AH（Authentication Header）**，只能进行数据摘要 ，不能实现数据加密。
- 还有一种 **ESP（Encapsulating Security Payload）**，能够进行数据加密和数据摘要。

在这个协议簇里面，还有两类算法，分别是**加密算法和摘要算法**。这个协议簇还包含两大组件，一个用于 VPN 的双方要进行对称密钥的交换的 IKE 组件，另一个是 VPN 的双方要对连接进行维护的 SA（Security Association）组件。

### 建立过程

> SA(Security Association)：是两个IPSec通信实体之间经协商建立起来的一种共同协定，它规定了通信双方使用哪种IPSec协议保护数据安全、应用的算法标识、加密和验证的密钥取值以及密钥的生存周期等等安全属性值。通过使用安全关联(SA) ， IPSec能够区分对不同的数据流提供的安全服务。
>

关于 **IPsec VPN** 的建立过程，主要分两个阶段。

- 一阶段：**建立 IKE 自己的 SA**，为双方进一步的IKE通信提供机密性、数据完整性以及数据源认证服务；
- 二阶段：**建立IPsec SA**，为数据交换提供`IPsec`服务；

### 运行方式

IPSec 有两种不同的运行方式：**隧道模式和传输模式**。两者之间的区别在于 IPSec 如何处理数据包报头。

- 在隧道模式下加密和验证整个 IP数据包（包括 IP 标头和有效负载），并附加一个新的报头；
- 在传输模式下，IPSec 仅加密（或验证）数据包的有效负载，但或多或少地保留现有的数据报头数据；

IPSec传输模式和隧道模式的区别在于：

1. 从安全性来讲，隧道模式优于传输模式。它可以完全对原始IP数据包进行验证和加密。隧道模式下可以隐藏内部IP地址、协议类型和端口。
2. 从性能来讲，隧道模式因为有一个额外的IP头，所以它将比传输模式占用更多带宽。
3. 从场景来讲，传输模式主要应用于两台主机或一台主机和一台VPN网关之间通信；隧道模式主要应用于两台VPN网关之间或一台主机与一台VPN网关之间的通信。



## 应用场景

以下是当今使用的**最受欢迎的VPN协议**：

1. **OpenVPN** － OpenVPN是为多种身份验证方法开发的开源项目。它是一种非常通用的协议，可以在具有不同功能的许多不同设备上使用，并可以通过UDP或TCP在任何端口上使用。OpenVPN使用OpenSSL库和TLS协议提供出色的性能和强大的加密。
2. **WireGuard** － WireGuard是一种较新的VPN协议，与现有VPN协议相比，旨在提供更高的安全性和更好的性能。默认情况下，WireGuard在隐私方面存在一些问题，尽管大多数支持WireGuard的VPN已经克服这些问题。
3. **IKEv2 / IPSec** －带有Internet密钥交换版本2（IPSec / IKEv2）的Internet协议安全性是一种快速且安全的VPN协议。它已在许多操作系统（例如Windows、Mac OS和iOS）中自动进行预配置。它特别适合与移动设备重新建立连接。IKEv2的一个缺点是它是由Cisco和Microsoft开发的，不是像OpenVPN这样的开源项目。对于需要快速、轻量级VPN（该VPN安全且可以暂时断开连接以快速重新连接）的移动用户而言，IKEv2 / IPSec是一个不错的选择。
4. **L2TP / IPSec** －具有Internet协议安全性也是不错选择。该协议比PPTP更安全，但是由于数据包是双重封装的，因此它并不总能提供最佳响应速度。它通常与移动设备一起使用，并内置在许多操作系统中。
5. **PPTP** －点对点隧道协议是一种基本的较旧的VPN协议，内置在许多操作系统中。不过，PPTP具有已知的安全漏洞，出于隐私和安全原因，就不太建议选择。

针对VPN加密，目前AES（高级加密标准）是当今使用的最常见的加密密码之一。大多数VPN使用密钥长度为128位或256位的AES加密。即便是在量子计算方面取得进步，AES-128也被认为是安全的。

### 分类标准

1、按VPN的协议分类：
VPN的隧道协议主要有三种，PPTP、L2TP和IPSec，其中PPTP和L2TP协议工作在OSI模型的第二层，又称为二层隧道协议；IPSec是第三层隧道协议。
2、按VPN的应用分类：
（1）Access VPN（远程接入VPN）：客户端到网关，使用公网作为骨干网在设备之间传输VPN数据流量；
（2）Intranet VPN（内联网VPN）：网关到网关，通过公司的网络架构连接来自同公司的资源；
（3）Extranet VPN（外联网VPN）：与合作伙伴企业网构成Extranet，将一个公司与另一个公司的资源进行连接。

3、按所用的设备类型进行分类：
网络设备提供商针对不同客户的需求，开发出不同的VPN网络设备，主要为交换机、路由器和防火墙：
（1）路由器式VPN：路由器式VPN部署较容易，只要在路由器上添加VPN服务即可；
（2）交换机式VPN：主要应用于连接用户较少的VPN网络；
（3）防火墙式VPN：防火墙式VPN是最常见的一种VPN的实现方式，许多厂商都提供这种配置类型
4、按照实现原理划分：
（1）重叠VPN：此VPN需要用户自己建立端节点之间的VPN链路，主要包括：GRE、L2TP、IPSec等众多技术。
（2）对等VPN：由网络运营商在主干网上完成VPN通道的建立，主要包括MPLS、VPN技术。

### 不同VPN实现对比

|             | L2TP                               | SSL VPN                           | IPSec                                            |
| :---------- | :--------------------------------- | :-------------------------------- | :----------------------------------------------- |
| 发起方      | 远端PC                             | 远端PC                            | 总部、分支、远端PC                               |
| 身份验证    | 基于PPP认证                        | 用户名+口令+证书                  | 地址或名字+口令或证书                            |
| 加密保证    | 无                                 | 有                                | 有                                               |
| 地址分配    | 有                                 | 有                                | 无                                               |
| 客户端      | XP自带或第三方                     | 免安装式浏览器插件                | 通常为厂家设备内部实现，PC使用厂家专用客户端     |
| 兼容互通性  | 优良，基本上各个厂家都兼容XP客户端 | PC使用浏览器打开VPN，与浏览器相关 | 优良，已经达到工业化标准成都，大部分厂家能够互通 |
| 资源消耗    | 轻                                 | 重                                | 中                                               |
| 性能加速    | 无                                 | 加密卡硬件加速                    | 加密卡硬件加速                                   |
| 隧道穿越NAT | 可以                               | 可以                              | 可以                                             |

从上表可以看出，L2TP和SSL VPN确实更适合远端PC，特别是SSL VPN就是专门为远端PC开发的VPN技术。而IPSec更适合在分支和总部之间搭建VPN隧道。



## 参考链接

1. [VPN](https://zh.wikipedia.org/wiki/虛擬私人網路)
2. [PPTP(Point to Point Tunneling Protocol)](https://en.wikipedia.org/wiki/Point-to-Point_Tunneling_Protocol)
3. [L2TP(Layer 2 Tunneling Protocol)](https://en.wikipedia.org/wiki/Layer_2_Tunneling_Protocol)
4. [IPSec](https://en.wikipedia.org/wiki/IPsec)
5. [SSL/TLS(Secure Sockets Layer / Transport Layer Security)](https://en.wikipedia.org/wiki/Transport_Layer_Security)
6. [Client/Serveur SSL/TLS multiplateformes avec OpenSSL](https://www.asafety.fr/projects-and-tools/c-client-serveur-ssl-tls-multiplateformes-avec-openssl/)
7. [OSI 7 LAYER MODEL](http://www.escotal.com/osilayer.html)
8. [VPN协议与加密](https://zhuanlan.zhihu.com/p/369908696)
9. [趣谈网络协议](https://time.geekbang.org/column/article/10386)

