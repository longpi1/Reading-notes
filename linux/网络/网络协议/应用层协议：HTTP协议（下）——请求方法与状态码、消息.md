#    应用层协议：HTTP协议（下）——请求方法与状态码、消息

## 请求方法与状态码

### 请求方法

HTTP/1.1协议中共定义了八种方法（也叫“动作”）来以不同方式操作指定的资源：

- **GET**

  向指定的资源发出“显示”请求。使用GET方法应该只用在读取资料，而不应当被用于产生“[副作用](https://zh.wikipedia.org/wiki/超文本传输协议#副作用)”的操作中，例如在[网络应用程序](https://zh.wikipedia.org/wiki/网络应用程序)中。其中一个原因是GET可能会被[网络爬虫](https://zh.wikipedia.org/wiki/網路爬蟲)等随意访问。参见[安全方法](https://zh.wikipedia.org/wiki/超文本传输协议#安全方法)。浏览器直接发出的GET只能由一个url触发。GET上要在url之外带一些参数就只能依靠url上附带querystring。

- **HEAD**

  与GET方法一样，都是向服务器发出指定资源的请求。只不过服务器将不传回资源的本文部分。它的好处在于，使用这个方法可以在不必传输全部内容的情况下，就可以获取其中“关于该资源的信息”（元信息或称元数据）。

- **POST**

  向指定资源提交数据，请求服务器进行处理（例如提交表单或者上传文件）。数据被包含在请求本文中。这个请求可能会创建新的资源或修改现有资源，或二者皆有。每次提交，表单的数据被浏览器用编码到HTTP请求的body里。浏览器发出的POST请求的body主要有两种格式，一种是application/x-www-form-urlencoded用来传输简单的数据，大概就是"key1=value1&key2=value2"这样的格式。另外一种是传文件，会采用multipart/form-data格式。采用后者是因为application/x-www-form-urlencoded的编码方式对于文件这种二进制的数据非常低效。

- **PUT**

  向指定资源位置上传其最新内容。

- **DELETE**

  请求服务器删除Request-URI所标识的资源。

- **TRACE**

  回显服务器收到的请求，主要用于测试或诊断。

- **OPTIONS**

  这个方法可使服务器传回该资源所支持的所有HTTP请求方法。用'*'来代替资源名称，向Web服务器发送OPTIONS请求，可以测试服务器功能是否正常运作。

- **CONNECT**

  HTTP/1.1协议中预留给能够将连接改为隧道方式的代理服务器。通常用于SSL加密服务器的链接（经由非加密的HTTP代理服务器）。

方法名称是区分大小写的。当某个请求所针对的资源不支持对应的请求方法的时候，服务器应当返回[状态码405](https://zh.wikipedia.org/wiki/HTTP状态码#405)（Method Not Allowed），当服务器不认识或者不支持对应的请求方法的时候，应当返回[状态码501](https://zh.wikipedia.org/wiki/HTTP状态码#501)（Not Implemented）。

**HTTP服务器至少应该实现GET和HEAD方法**，其他方法都是可选的。当然，所有的方法支持的实现都应当符合下述的方法各自的语义定义。此外，除了上述方法，特定的HTTP服务器还能够扩展自定义的方法。例如：

- **PATCH**

  用于将局部修改应用到资源。

### 状态码

所有HTTP响应的第一行都是**状态行**，依次是当前HTTP版本号，3位数字组成的[状态代码](https://zh.wikipedia.org/wiki/HTTP状态码)，以及描述状态的短语，彼此由空格分隔。

状态代码的第一个数字代表当前响应的类型：

- [1xx消息](https://zh.wikipedia.org/wiki/HTTP状态码#1xx消息)——请求已被服务器接收，继续处理
- [2xx成功](https://zh.wikipedia.org/wiki/HTTP状态码#2xx成功)——请求已成功被服务器接收、理解、并接受
- [3xx重定向](https://zh.wikipedia.org/wiki/HTTP状态码#3xx重定向)——需要后续操作才能完成这一请求
- [4xx请求错误](https://zh.wikipedia.org/wiki/HTTP状态码#4xx请求错误)——请求含有词法错误或者无法被执行
- [5xx服务器错误](https://zh.wikipedia.org/wiki/HTTP状态码#5xx服务器错误)——服务器在处理某个正确请求时发生错误

## HTTP 消息

HTTP 消息是服务器和客户端之间交换数据的方式。有两种类型的消息：*请求*（request）——由客户端发送用来触发一个服务器上的动作；*响应*（response）——来自服务器的应答。

### 请求

![img](https://static001.geekbang.org/resource/image/85/c1/85ebb0396cbaa45ce00b505229e523c1.jpeg?wh=1920*1080)

HTTP 请求的一个例子：

![A basic HTTP request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview/http_request.png)

请求由以下元素组成：

- 一个 HTTP 的请求[方法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods)，经常是由一个动词像 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET)、[`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 或者一个名词像 [`OPTIONS`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/OPTIONS)、[`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 来定义客户端的动作行为。通常客户端的操作都是获取资源（GET 方法）或者发送 [HTML 表单](https://developer.mozilla.org/zh-CN/docs/Learn/Forms)（POST 方法），虽然在一些情况下也会有其他操作。
- 要获取的资源的路径，通常是上下文中就很明显的元素资源的 URL，它没有 [protocol](https://developer.mozilla.org/zh-CN/docs/Glossary/Protocol)（`http://`），[domain](https://developer.mozilla.org/zh-CN/docs/Glossary/Domain)（`developer.mozilla.org`），或是 TCP 的 [port (en-US)](https://developer.mozilla.org/en-US/docs/Glossary/Port)（HTTP 一般在 80 端口）。
- HTTP 协议版本号。
- 为服务端表达其他信息的可选[标头](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)。
- 对于一些像 POST 这样的方法，报文的主体（body）就包含了发送的资源，这与响应报文的主体类似。

#### [标头（Header）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Messages#标头（header）)

来自请求的 [HTTP 标头](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)遵循和 HTTP 标头相同的基本结构：不区分大小写的字符串，紧跟着的冒号（`':'`）和一个结构取决于标头的值。整个标头（包括值）由一行组成，这一行可以相当长。

有许多请求标头可用，它们可以分为几组：

- [通用标头（General header）](https://developer.mozilla.org/zh-CN/docs/Glossary/General_header)，例如 [`Via`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Via)，适用于整个消息。
- [请求标头（Request header）](https://developer.mozilla.org/zh-CN/docs/Glossary/Request_header)，例如 [`User-Agent`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/User-Agent)、[`Accept-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept-Type)，通过进一步的定义（例如 [`Accept-Language`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept-Language)）、给定上下文（例如 [`Referer`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Referer)）或者进行有条件的限制（例如 [`If-None`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-None)）来修改请求。
- [表示标头（Representation header）](https://developer.mozilla.org/zh-CN/docs/Glossary/Representation_header)，例如 [`Content-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type) 描述了消息数据的原始格式和应用的任意编码（仅在消息有主体时才存在）。

![Example of headers in an HTTP request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages/http_request_headers3.png)

#### [主体（Body）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Messages#主体（body）)

请求的最后一部分是它的主体。不是所有的请求都有一个主体：例如获取资源的请求，像 `GET`、`HEAD`、`DELETE` 和 `OPTIONS`，通常它们不需要主体。有些请求将数据发送到服务器以便更新数据：常见的的情况是 POST 请求（包含 HTML 表单数据）。

主体大致可分为两类：

- 单一资源（Single-resource）主体，由一个单文件组成。该类型的主体由两个标头定义：[`Content-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type) 和 [`Content-Length`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Length)。
- [多资源（Multiple-resource）主体](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types#multipartform-data)，由多部分主体组成，每一部分包含不同的信息位。通常是和 [HTML 表单](https://developer.mozilla.org/zh-CN/docs/Learn/Forms)连系在一起。





### 响应

![img](https://static001.geekbang.org/resource/image/6b/63/6bc37ddcb4e7a61ca3275790820f2263.jpeg?wh=1761*937)

HTTP 响应的一个例子：

![img](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview/http_response.png)

响应报文包含了下面的元素：

- HTTP 协议版本号。
- 一个状态码（[状态码（status code）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status)），来告知对应请求执行成功或失败，以及失败的原因。
- 一个状态信息，这个信息是非权威的状态码描述信息，可以由服务端自行设定。
- HTTP [标头](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)，与请求标头类似。
- 可选项，比起请求报文，响应报文中更常见地包含获取资源的主体。

#### [状态行](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Messages#状态行)

HTTP 响应的起始行被称作*状态行*（status line），包含以下信息：

1. *协议版本*，通常为 `HTTP/1.1`。
2. *状态码*（status code），表明请求是成功或失败。常见的状态码是 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200)、[`404`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/404) 或 [`302`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/302)。
3. *状态文本*（status text）。一个简短的，纯粹的信息，通过状态码的文本描述，帮助人们理解该 HTTP 消息。

一个典型的状态行看起来像这样：`HTTP/1.1 404 Not Found`。

#### [标头（Header）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Messages#标头（header）_2)

响应的 [HTTP 标头](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)遵循和任何其它标头相同的结构：不区分大小写的字符串，紧跟着的冒号（`':'`）和一个结构取决于标头类型的值。整个标头（包括其值）表现为单行形式。

许多不同的标头可能会出现在响应中。这些可以分为几组：

- [通用标头（General header）](https://developer.mozilla.org/zh-CN/docs/Glossary/General_header)，例如 [`Via`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Via)，适用于整个消息。
- [响应标头（Response header）](https://developer.mozilla.org/zh-CN/docs/Glossary/Response_header)，例如 [`Vary`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Vary) 和 [`Accept-Ranges`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept-Ranges)，提供有关服务器的其他信息，这些信息不适合状态行。
- [表示标头（Representation header）](https://developer.mozilla.org/zh-CN/docs/Glossary/Representation_header)，例如 [`Content-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type) 描述了消息数据的原始格式和应用的任意编码（仅在消息有主体时才存在）。

![Example of headers in an HTTP response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages/http_response_headers3.png)

#### [主体（Body）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Messages#主体（body）_2)

响应的最后一部分是主体。不是所有的响应都有主体：具有状态码（如 [`201`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/201) 或 [`204`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/204)）的响应，通常不会有主体。

主体大致可分为三类：

- 单资源（Single-resource）主体，由**已知**长度的单个文件组成。该类型主体由两个标头定义：[`Content-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type) 和 [`Content-Length`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Length)。
- 单资源（Single-resource）主体，由**未知**长度的单个文件组成。通过将 [`Transfer-Encoding`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Transfer-Encoding) 设置为 `chunked` 来使用分块编码。
- [多资源（Multiple-resource）主体](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types#multipartform-data)，由多部分 body 组成，每部分包含不同的信息段。但这是比较少见的。

## HTTP/2 帧

HTTP/1.x 消息有一些性能上的缺点：

- 与主体不同，标头不会被压缩。
- 两个消息之间的标头通常非常相似，但它们仍然在连接中重复传输。
- 无法多路复用。当在同一个服务器打开几个连接时：TCP 热连接比冷连接更加有效。

HTTP/2 引入了一个额外的步骤：它将 HTTP/1.x 消息分成帧并嵌入到流（stream）中。数据帧和报头帧分离，这将允许报头压缩。将多个流组合，这是一个被称为*多路复用*（multiplexing）的过程，它允许更有效的底层 TCP 连接。

![HTTP/2 modify the HTTP message to divide them in frames (part of a single stream), allowing for more optimization.](https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages/binary_framing2.png)

HTTP 帧现在对 Web 开发人员是透明的。在 HTTP/2 中，这是一个在 HTTP/1.1 和底层传输协议之间附加的步骤。Web 开发人员不需要在其使用的 API 中做任何更改来利用 HTTP 帧；当浏览器和服务器都可用时，HTTP/2 将被打开并使用。

## QUIC 协议

QUIC 协议通过基于 UDP 自定义的类似 TCP 的连接、重试、多路复用、流量控制技术，进一步提升性能。



## 参考链接

1.维基百科，https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE

2.web开发技术，https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Session

3.趣谈网络协议，https://time.geekbang.org/column/article/9410