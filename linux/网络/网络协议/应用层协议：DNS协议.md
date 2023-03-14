#                                          应用层协议：DNS协议

## **概述**

**网域名称系统**（英语：**D**omain **N**ame **S**ystem，缩写：**DNS**）是[互联网](https://zh.wikipedia.org/wiki/互联网)的一项服务。它作为将[域名](https://zh.wikipedia.org/wiki/域名)和[IP地址](https://zh.wikipedia.org/wiki/IP地址)相互[映射](https://zh.wikipedia.org/wiki/映射)的一个[分布式数据库](https://zh.wikipedia.org/wiki/分布式数据库)，能够使人更方便地访问[互联网](https://zh.wikipedia.org/wiki/互联网)。DNS 协议运行在 UDP 之上，DNS使用[TCP](https://zh.wikipedia.org/wiki/传输控制协议)和[UDP](https://zh.wikipedia.org/wiki/用户数据报协议)[端口](https://zh.wikipedia.org/wiki/TCP/UDP端口列表)53[[1\]](https://zh.wikipedia.org/zh-hans/域名系统#cite_note-1)。

在网络世界，也是这样的。你肯定记得住网站的名称，但是很难记住网站的 IP 地址，因而也需要一个地址簿，就是 **DNS 服务器**，它是一个由分层的 DNS 服务器(DNS server)实现的分布式数据库;它还是一个使得主机能够查询分布式数据库的应用层协议。

![img](https://s4.51cto.com/oss/202101/19/66e6aabb9a6b65ba6a46e9873b5be7cf.png)

DNS 最早的设计是只有一台 DNS 服务器。这台服务器会包含所有的 DNS 映射。这是一种集中式的设计，这种设计并不适用于当今的互联网，因为互联网有着数量巨大并且持续增长的主机，这种集中式的设计会存在以下几个问题

- 单点故障(a single point of failure)，如果 DNS 服务器崩溃，那么整个网络随之瘫痪。
- 通信容量(traaffic volume)，单个 DNS 服务器不得不处理所有的 DNS 查询，这种查询级别可能是上百万上千万级
- 远距离集中式数据库(distant centralized database)，单个 DNS 服务器不可能 邻近 所有的用户，假设在美国的 DNS 服务器不可能临近让澳大利亚的查询使用，其中查询请求势必会经过低速和拥堵的链路，造成严重的时延。
- 维护(maintenance)，维护成本巨大，而且还需要频繁更新。

所以 DNS 不可能集中式设计，它完全没有可扩展能力，因此需要采用**分布式设计**，接下来介绍它的相关特点

## 报文结构

DNS 有两种报文，一种是查询报文，一种是响应报文，并且这两种报文有着相同的格式，下面是 DNS 的报文格式

![img](https://s4.51cto.com/oss/202101/19/c2fc42808a2a73456010cc52c3b79efb.png)

上图显示了 DNS 的报文格式，其中事务 ID、标志、问题数量、回答资源记录数、权威名称服务器计数、附加资源记录数这六个字段是 DNS 的报文段首部，报文段首部一共有 12 个字节。

## 特点

### 分层结构

![img](https://s5.51cto.com/oss/202101/19/188ab4b953d416645db6b828557deb50.png)

- 根 DNS 服务器 ：返回顶级域 DNS 服务器的 IP 地址
- 顶级域 DNS 服务器：返回权威 DNS 服务器的 IP 地址
- 权威 DNS 服务器 ：返回相应主机的 IP 地址



### 记录类型

共同实现 DNS 分布式数据库的所有 DNS 服务器存储了资源记录(Resource Record, RR)，RR 提供了主机名到 IP 地址的映射。每个 DNS 回答报文中会包含一条或多条资源记录。RR 记录用于回复客户端查询。

资源记录是一个包含了下列字段的 4 元组

```shell
(Name, Value, Type, TTL) 
```

DNS系统中，**RR**常见的资源记录类型有：

- 主机记录（A记录）：RFC 1035定义，A记录是用于名称解析的重要记录，它将特定的主机名映射到对应主机的IP地址上。
- 别名记录（CNAME记录）: RFC 1035定义，CNAME记录用于将某个别名指向到某个A记录上，这样就不需要再为某个新名字另外创建一条新的A记录。
- IPv6主机记录（AAAA记录）: RFC 3596定义，与A记录对应，用于将特定的主机名映射到一个主机的[IPv6](https://zh.wikipedia.org/wiki/IPv6)地址。
- 服务位置记录（SRV记录）: RFC 2782定义，用于定义提供特定服务的服务器的位置，如主机（hostname），端口（port number）等。
- 域名服务器记录（NS记录） ：用来指定该域名由哪个DNS服务器来进行解析。 您注册域名时，总有默认的DNS服务器，每个注册的域名都是由一个DNS域名服务器来进行解析的，DNS服务器NS记录地址一般以以下的形式出现： ns1.domain.com、ns2.domain.com等。 简单的说，NS记录是指定由哪个DNS服务器解析你的域名。
- NAPTR记录：RFC 3403定义，它提供了[正则表达式](https://zh.wikipedia.org/wiki/正则表达式)方式去映射一个域名。NAPTR记录非常著名的一个应用是用于[ENUM](https://zh.wikipedia.org/w/index.php?title=ENUM&action=edit&redlink=1)查询。



### 解析流程

**通常情况下，DNS 的查找会经历下面这些步骤**

1. 用户在浏览器中输入网址 www.example.com 并点击回车后，查询会进入网络，并且由 DNS 解析器进行接收。
2. DNS 解析器会向根域名发起查询请求，要求返回顶级域名的地址。
3. 根 DNS 服务器会注意到请求地址的前缀并向 DNS 解析器返回 com 的顶级域名服务器(TLD) 的 IP 地址列表。
4. 然后，DNS 解析器会向 TLD 服务器发送查询报文
5. TLD 服务器接收请求后，会根据域名的地址把权威 DNS 服务器的 IP 地址返回给 DNS 解析器。
6. 最后，DNS 解析器将查询直接发送到权威 DNS 服务器
7. 权威 DNS 服务器将 IP 地址返回给 DNS 解析器
8. DNS 解析器将会使用 IP 地址响应 Web 浏览器

一旦 DNS 查找的步骤返回了 example.com 的 IP 地址，浏览器就可以请求网页了。

**解析流程如下：**

![img](https://static001.geekbang.org/resource/image/71/e8/718e3a1a1a7927302b6a0f836409e8e8.jpg?wh=1456*1212)



## 参考链接

1.[万字长文爆肝 DNS 协议！](https://www.51cto.com/article/641655.html)

2.维基百科：https://zh.wikipedia.org/zh-hans/%E5%9F%9F%E5%90%8D%E7%B3%BB%E7%BB%9F

3.[趣谈网络协议](https://time.geekbang.org/column/article/9492)