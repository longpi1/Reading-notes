#                              传输层协议：TCP与UDP协议的区别

## TCP和UDP有哪些区别？

关于TCP与UDP协议两个协议的区别，大部分人会回答，TCP是面向连接的，UDP是面向无连接的。

什么叫面向连接，什么叫无连接呢？在互通之前，面向连接的协议会先建立连接。例如，TCP会三次握手，而UDP不会。为什么要建立连接呢？你TCP三次握手，我UDP也可以发三个包玩玩，有什么区别吗？

**所谓的建立连接，是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，用这样的数据结构来保证所谓的面向连接的特性。**

- **TCP提供可靠交付**。通过TCP连接传输的数据，无差错、不丢失、不重复、并且按序到达。而 **UDP继承了IP包的特性，不可靠，不保证不丢失，不保证按顺序到达。**
-  **TCP是面向字节流的**。发送的时候发的是一个流，没头没尾。IP包可不是一个流，而是一个个的IP包。之所以变成了流，这也是TCP自己的状态维护做的事情。而 **UDP继承了IP的特性，基于数据报的，一个一个地发，一个一个地收。**
-  **TCP是可以拥塞控制**。它意识到包丢弃了或者网络的环境不好了，就会根据情况调整自己的行为，看看是不是发快了，要不要发慢点。 **UDP就不会，应用让我发，我就发，管它洪水滔天，所以udp速度也更快**
- **TCP只支持单播传输**，每条TCP传输连接只能有两个端点，只能进行点对点的数据传输，不支持多播和广播传输方式。UDP 不止支持一对一的传输方式，同样支持一对多，多对多，多对一的方式，也就是说 **UDP 提供了单播，多播，广播的功能**。
- **TCP首部开销20字节;UDP的首部开销小，只有8个字节**

具体细节可见下表：

|          | TCP                              | UDP                            |
| :------- | :------------------------------- | ------------------------------ |
| 可靠性   | 可靠                             | 不可靠                         |
| 连接性   | 面向连接                         | 无连接                         |
| 报文     | 面向字节流                       | 面向报文                       |
| 资源占用 | 首部开销20字节                   | UDP的首部开销更小，只有8个字节 |
| 效率     | 传输效率低                       | 传输效率高                     |
| 传播方式 | 只支持单播                       | 一对一、一对多、多对一、多对多 |
| 流量控制 | 滑动窗口                         | 无                             |
| 拥塞控制 | 慢开始、拥塞避免、快重传、快恢复 | 无                             |
| 传输效率 | 慢                               | 快                             |

因而 **TCP其实是一个有状态服务**，通俗地讲就是有脑子的，里面精确地记着发送了没有，接收到没有，发送到哪个了，应该接收哪个了，错一点儿都不行。而 **UDP则是无状态服务**。通俗地说是没脑子的，天真无邪的，发出去就发出去了。

我们可以这样比喻，如果MAC层定义了本地局域网的传输行为，IP层定义了整个网络端到端的传输行为，这两层基本定义了这样的基因：网络传输是以包为单位的，二层叫帧，网络层叫包，传输层叫段。我们笼统地称为包。包单独传输，自行选路，在不同的设备封装解封装，不保证到达。基于这个基因，生下来的孩子UDP完全继承了这些特性，几乎没有自己的思想。



## 如何选择TCP还是UDP协议？

**根据两个协议的优点，我们可以在传输层有必要实现可靠传输的情况下用TCP协议；在那些对高速传输和实时性有较高要求的通信或者对准确性要求低的情况下用UDP协议。**TCP 和 UDP 应该根据应用的目的按需使用。

例如，当对网络通讯质量有要求的时候，比如：整个数据要准确无误的传递给对方，这往往用于一些要求可靠的应用，比如**HTTP、HTTPS、FTP**等传输文件的协议选择使用**TCP**协议，**POP、SMTP**等邮件传输的协议。当对网络通讯质量要求不高的时候，要求网络通讯速度能尽量的快，这时就可以使用**UDP**（如视频传输、实时通信等）。

![img](https://s6.51cto.com/oss/202105/14/26c5f19079fba62b5682f613b94c8df3.jpg)

## 参考链接

1.https://time.geekbang.org/column/article/8924

2.https://www.51cto.com/article/662343.html