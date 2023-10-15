## gRPC vs HTTP

### 性能

gRPC 消息使用 [Protobuf](https://developers.google.com/protocol-buffers/docs/overview)（一种高效的二进制消息格式）进行序列化。 Protobuf 在服务器和客户端上可以非常快速地序列化。 Protobuf 序列化产生的有效负载较小，这在移动应用等带宽有限的方案中很重要。

gRPC 专为 HTTP/2（HTTP 的主要版本）而设计，与 HTTP 1.x 相比，HTTP/2 具有巨大性能优势：

- 二进制组帧和压缩。 HTTP/2 协议在发送和接收方面均紧凑且高效。
- 在单个 TCP 连接上多路复用多个 HTTP/2 调用。 多路复用可消除[队头阻塞](https://en.wikipedia.org/wiki/Head-of-line_blocking)。

HTTP/2 不是 gRPC 独占的。 许多请求类型（包括具有 JSON 的 HTTP API）都可以使用 HTTP/2，并受益于其性能改进。

#### 应用：

例如ETCT版本v2升级至v3中watch机制实现，通过etcd v2 Watch机制实现中，使用的是HTTP/1.x协议，实现简单、兼容性好，每个watcher对应一个TCP连接。client通过HTTP/1.1协议长连接定时轮询server，获取最新的数据变化事件。

然而当你的watcher成千上万的时，即使集群空负载，大量轮询也会产生一定的QPS，server端会消耗大量的socket、内存等资源，导致etcd的扩展性、稳定性无法满足Kubernetes等业务场景诉求。

etcd v3的Watch机制的设计实现并非凭空出现，它正是吸取了etcd v2的经验、教训而重构诞生的。

在etcd v3中，为了解决etcd v2的以上缺陷，使用的是基于HTTP/2的gRPC协议，双向流的Watch API设计，实现了连接多路复用。

HTTP/2协议为什么能实现多路复用呢？

[![img](https://camo.githubusercontent.com/d7fd163ed1a40b1a9bf90977acd99473482ac65512e0a4d16478800f0bd16d3b/68747470733a2f2f7374617469633030312e6765656b62616e672e6f72672f7265736f757263652f696d6167652f62652f37342f62653361303139626561663133313064323134653563393934386363396337342e706e673f77683d313738342a353334)](https://camo.githubusercontent.com/d7fd163ed1a40b1a9bf90977acd99473482ac65512e0a4d16478800f0bd16d3b/68747470733a2f2f7374617469633030312e6765656b62616e672e6f72672f7265736f757263652f696d6167652f62652f37342f62653361303139626561663133313064323134653563393934386363396337342e706e673f77683d313738342a353334)

在HTTP/2协议中**，HTTP消息被分解独立的帧（Frame），交错发送，帧是最小的数据单位。每个帧会标识属于哪个流（Stream），流由多个数据帧组成，每个流拥有一个唯一的ID，一个数据流对应一个请求或响应包。**通过以上机制，HTTP/2就解决了HTTP/1的请求阻塞、连接无法复用的问题，实现了多路复用、乱序发送。



### 代码生成

所有 gRPC 框架都为代码生成提供一流支持。 [`.proto` 文件](https://developers.google.com/protocol-buffers/docs/proto3)是 gRPC 开发的核心文件，它定义 gRPC 服务和消息的协定。 通过此文件，gRPC 框架生成服务基类、消息和完整的客户端。

通过在服务器和客户端之间共享 `.proto` 文件，可以端到端生成消息和客户端代码。 客户端的代码生成消除了客户端和服务器上的消息重复，并为你创建强类型客户端。 无需编写客户端可在具有许多服务的应用程序中节省大量开发时间。



### 严格规范

具有 JSON 的 HTTP API 没有正式规范。 开发人员为 URL、HTTP 谓词和响应代码的最佳格式争论不休。

[gRPC 规范](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)对 gRPC 服务必须遵循的格式进行了规定。 gRPC 消除了争论并为开发人员节省了时间，因为 gRPC 在各个平台和实现中都是一致的。



### 流式处理

HTTP/2 为长期实时通信流提供基础。 gRPC 为通过 HTTP/2 进行流式传输提供一流支持。

gRPC 服务支持所有流式传输组合：

- 一元（无流式传输）
- 服务器到客户端流式传输
- 客户端到服务器流式传输
- 双向流式传输



## gRPC 建议方案

gRPC 非常适合以下方案：

- 微服务：gRPC 设计用于低延迟和高吞吐量通信。 gRPC 对于效率至关重要的轻量级微服务非常有用。
- 点对点实时通信：gRPC 对双向流式传输提供出色的支持。 gRPC 服务可以实时推送消息而无需轮询。
- 多语言环境：gRPC 工具支持所有常用的开发语言，因此，gRPC 是多语言环境的理想选择。
- 网络受限环境：gRPC 消息使用 Protobuf（一种轻量级消息格式）进行序列化。 gRPC 消息始终小于等效的 JSON 消息。
- **进程间通信 (IPC)** ：IPC 传输（如 Unix 域套接字和命名管道）可与 gRPC 一起用于同一台计算机上的应用间通信。 有关详细信息，请参阅[使用 gRPC 进行进程间通信](https://learn.microsoft.com/zh-cn/aspnet/core/grpc/interprocess?view=aspnetcore-7.0)。



## gRPC 弱点

### 浏览器支持受限

当前无法通过浏览器直接调用 gRPC 服务。 gRPC 大量使用 HTTP/2 功能，且没有浏览器在 Web 请求中提供支持 gRPC 客户端所需的控制级别。 例如，浏览器不允许调用方要求使用 HTTP/2，也不提供对 HTTP/2 基础框架的访问。

ASP.NET Core 上的 gRPC 提供两种兼容浏览器的解决方案：

- gRPC-Web 允许浏览器应用通过 gRPC-Web 客户端和 Protobuf 调用 gRPC 服务。 gRPC-Web 要求浏览器应用生成 gRPC 客户端。 gRPC-Web 允许浏览器应用从 gRPC 的高性能和低网络使用率获益。

  .NET 提供对 gRPC-Web 的内置支持。 有关详细信息，请参阅 [ASP.NET Core gRPC 应用中的 gRPC-Web](https://learn.microsoft.com/zh-cn/aspnet/core/grpc/grpcweb?view=aspnetcore-7.0)。

- gRPC JSON 转码允许浏览器应用调用 gRPC 服务，就像它们是使用 JSON 的 RESTful API 一样。 浏览器应用不需要生成 gRPC 客户端或了解 gRPC 的任何信息。 通过使用 HTTP 元数据注释 `.proto` 文件，可从 gRPC 服务自动创建 RESTful API。 转码使得应用可以同时支持 gRPC 和 JSON Web API，而无需重复为两者生成单独的服务。

  .NET 对从 gRPC 服务创建 JSON Web API 提供了内置支持。 有关详细信息，请参阅 [ASP.NET Core gRPC 应用中的 gRPC JSON 转码](https://learn.microsoft.com/zh-cn/aspnet/core/grpc/json-transcoding?view=aspnetcore-7.0)。