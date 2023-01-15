#    应用层协议：HTTP协议（上）——概述、主要特点、发展

## 概述

HTTP是一个客户端（用户）和服务端（网站）之间请求和应答的标准，通常使用[TCP协议](https://zh.wikipedia.org/wiki/传输控制协议)，在 Web 上进行数据交换的基础，是一种 client-server 协议。通过使用[网页浏览器](https://zh.wikipedia.org/wiki/網頁瀏覽器)、[网络爬虫](https://zh.wikipedia.org/wiki/网络爬虫)或者其它的工具，客户端发起一个HTTP请求到服务器上指定端口（默认[端口](https://zh.wikipedia.org/wiki/通訊埠)为80）。我们称这个客户端为用户代理程序（user agent）。应答的服务器上存储着一些资源，比如HTML文件和图像。我们称这个应答服务器为源服务器（origin server）。在用户代理和源服务器中间可能存在多个“中间层”，比如[代理服务器](https://zh.wikipedia.org/wiki/代理伺服器)、[网关](https://zh.wikipedia.org/wiki/网关)或者[隧道](https://zh.wikipedia.org/wiki/隧道)（tunnel）。

HTTP 是一个拓展性非常好的协议。它依赖于资源或统一资源定位符（URI）的概念、一个简单的消息结构和一个客户端——服务器结构的通信流。在这些基础概念之上，近年来已经出现了许多拓展，以增加新的 HTTP 方法或首部的方式为 HTTP 协议增加了新的功能和语义。



## 主要特点

- **简单快速**：客户向服务器请求服务时，只需传送请求方法和路径。请求方法常用的有GET、HEAD、POST。每种方法规定了客户与服务器联系的类型不同。由于HTTP协议简单，使得HTTP服务器的程序规模小，因而通信速度很快。虽然下一代 HTTP/2 协议将 HTTP 消息封装到了帧（frame）中，HTTP 大体上还是被设计得简单易读。HTTP 报文能够被人读懂，还允许简单测试，降低了门槛，对新人很友好。
- **可扩展的**：在 HTTP/1.0 中出现的 [HTTP 标头（header）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)让协议扩展变得非常容易。只要服务端和客户端就新标头达成语义一致，新功能就可以被轻松加入进来。
- **无连接、有会话的**：HTTP 是无状态的：在同一个连接中，两个执行成功的请求之间是没有关系的。这就带来了一个问题，用户没有办法在同一个网站中进行连续的交互，比如在一个电商网站里，用户把某个商品加入到购物车，切换一个页面后再次添加了商品，这两次添加商品的请求之间没有关联，浏览器无法知道用户最终选择了哪些商品。而使用 HTTP 的标头扩展，HTTP Cookie 就可以解决这个问题。把 Cookie 添加到标头中，创建一个会话让每次请求都能共享相同的上下文信息，达成相同的状态。
- **灵活**：HTTP允许传输任意类型的数据对象。正在传输的类型由Content-Type加以标记。



## 发展过程

超文本传输协议已经演化出了很多版本，它们中的大部分都是[向下兼容](https://zh.wikipedia.org/wiki/向下兼容)的。在[RFC 2145](https://tools.ietf.org/html/rfc2145)中描述了HTTP版本号的用法。**客户端在请求的开始告诉服务器它采用的协议版本号，而后者则在响应中采用相同或者更早的协议版本。**

### HTTP/0.9 -- 单行协议

已过时。只接受GET一种请求方法，请求由单行指令构成，以唯一可用方法 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 开头，其后跟目标资源的路径（一旦连接到服务器，协议、服务器、端口号这些都不是必须的）没有在通讯中指定版本号，且不支持请求头。由于该版本不支持POST方法，因此客户端无法向服务器传递太多信息。

### HTTP/1.0 -- 构建可扩展性

这是第一个在通讯中指定版本号的HTTP协议版本。主要改进以及特点如下：

- 协议版本信息现在会随着每个请求发送（`HTTP/1.0` 被追加到了 `GET` 行）。
- 状态码会在响应开始时发送，使浏览器能了解请求执行成功或失败，并相应调整行为（如更新或使用本地缓存）。
- 引入了 HTTP 标头的概念，无论是对于请求还是响应，允许传输元数据，使协议变得非常灵活，更具扩展性。
- 在新 HTTP 标头的帮助下，具备了传输除纯文本 HTML 文件以外其他类型文档的能力（凭借 [`Content-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type) 标头）。

### HTTP/1.1  -- 标准化的协议

默认采用持续连接（Connection: keep-alive），能很好地配合代理服务器工作。还支持以[管道方式](https://zh.wikipedia.org/wiki/HTTP管线化)在同时发送多个请求，以便降低线路负载，提高传输速度。

HTTP/1.1相较于HTTP/1.0协议的区别主要体现在：

- 连接可以复用，节省了多次打开 TCP 连接加载网页文档资源的时间。
- 增加管线化技术，允许在第一个应答被完全发送之前就发送第二个请求，以降低通信延迟。
- 缓存处理，引入额外的缓存控制机制。
- 带宽优化及网络连接的使用
- 错误通知的管理
- 消息在网络中的发送
- 引入内容协商机制，包括语言、编码、类型等。
- 互联网地址的维护
- 安全性及完整性

### HTTP/2 -- 更优异的表现

当前版本，于2015年5月作为互联网标准正式发布。[[5\]](https://zh.wikipedia.org/wiki/超文本传输协议#cite_note-5)

HTTP/2 在 HTTP/1.1 有几处基本的不同：

- HTTP/2 是二进制协议而不是文本协议。不再可读，也不可无障碍的手动创建，改善的优化技术现在可被实施。
- 这是一个多路复用协议。并行的请求能在同一个链接中处理，移除了 HTTP/1.x 中顺序和阻塞的约束。
- 压缩了标头。因为标头在一系列请求中常常是相似的，其移除了重复和传输重复数据的成本。
- 其允许服务器在客户端缓存中填充数据，通过一个叫服务器推送的机制来提前请求。

### HTTP/3 -- 基于QUIC的HTTP

最新版本，于2022年6月6日标准化为RFC9114。传输层抛弃使用TCP，通过UDP上使用QUIC来承载应用层数据。

截至2023年一月份，主要版本使用比例如下：

**HTTP/2 is used by 39.8%** of all the websites.

**HTTP/3 is used by 25.0%** of all the websites.

数据来源于[w3techs](https://w3techs.com/technologies/details/ce-http2)，感兴趣的可以自行查看



## 参考链接

1.维基百科，https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE

2.web开发技术，https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Session

3.趣谈网络协议，https://time.geekbang.org/column/article/9410