#                                 client-go架构与原理介绍

## 一、架构展示

client-go 库中的各种组件架构如下图所示：

![img](https://pic2.zhimg.com/80/v2-ebd721c9c5860db9b64865e6aaa01ffd_1440w.webp)



## 二、目录结构

client-go 是用 Golang 语言编写的官方编程式交互客户端库，提供对 Kubernetes API server 服务的交互访问。

其源码目录结构如下：

```bash
.
├── discovery                   # 定义DsicoveryClient客户端。作用是用于发现k8s所支持GVR(Group, Version, Resources)。
├── dynamic                     # 定义DynamicClient客户端。可以用于访问k8s Resources(如: Pod, Deploy...)，也可以访问用户自定义资源(即: CRD)。
├── informers                   # k8s中各种Resources的Informer机制的实现。
├── kubernetes                  # 定义ClientSet客户端。它只能用于访问k8s Resources。每一种资源(如: Pod等)都可以看成是一个客端，而ClientSet是多个客户端的集合，它对RestClient进行了封装，引入了对Resources和Version的管理。通常来说ClientSet是client-gen来自动生成的。
├── listers                     # 提供对Resources的获取功能。对于Get()和List()而言，listers提供给二者的数据都是从缓存中读取的。
├── pkg                         
├── plugin                      # 提供第三方插件。如：GCP, OpenStack等。
├── rest                        # 定义RestClient，实现了Restful的API。同时会支持Protobuf和Json格式数据。
├── scale                       # 定义ScalClient。用于Deploy, RS, RC等的扩/缩容。
├── tools                       # 定义诸如SharedInformer、Reflector、DealtFIFO和Indexer等常用工具。实现client查询和缓存机制，减少client与api-server请求次数，减少api-server的压力。
├── transport
└── util                        # 提供诸如WorkQueue、Certificate等常用方法。
```

### 2.1 RESTClient 客户端

RESTful Client 是最基础的客户端，它主要是对 HTTP 请求进行了封装，并且支持 JSON 和 Protobuf 格式数据。

### 2.2 DynamicClient 客户端

DynamicClient 是一种动态客户端，它可以动态的指定资源的组，版本和资源。因此它可以对任意 K8S 资源进行 RESTful 操作，包括 CRD 自定义资源。它封装了 RESTClient。所以同样提供 RESTClient 的各种方法。

具体使用方法，可参考官方示例：[dynamic-create-update-delete-deployment](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/tree/master/examples/dynamic-create-update-delete-deployment)。

**注意**: 该官方示例是基于集群外的环境，如果你需要在集群内部使用（例如你需要在 container 中访问），你将需要调用 `rest.InClusterConfig()` 生成一个 configuration。具体的示例请参考 [in-cluster-client-configuration](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/tree/master/examples/in-cluster-client-configuration)。

### 2.3 ClientSet 客户端

ClientSet 客户端在 RESTClient 的基础上封装了对资源和版本的管理方法。每个资源可以理解为一个客户端，而 ClientSet 则是多个客户端的集合，每一个资源和版本都以函数的方式暴露给开发者。

具体使用方法，可参考官方示例：[create-update-delete-deployment](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/tree/master/examples/create-update-delete-deployment)。

### 2.4 DiscoveryClient 客户端

DiscoveryClient 是一个发现客户端，它主要用于发现 K8S API Server 支持的资源组，资源版本和资源信息。所以开发者可以通过使用 DiscoveryClient 客户端查看所支持的资源组，资源版本和资源信息。

### 2.5 ClientSet VS DynamicClient

类型化 `ClientSets` 使得使用预先生成的本地 API 对象与 API 服务器通信变得简单，从而获得类似 `RPC` 的编程体验。类型化客户端使用程序编译来强制执行数据安全性和一些验证。然而，在使用类型化客户端时，程序被迫与所使用的版本和类型紧密耦合。

而 `DynamicClient` 则使用 `unstructured.Unstructured` 表示来自 API Server 的所有对象值。`Unstructured` 类型是一个嵌套的 `map[string]inferface{}` 值的集合来创建一个内部结构，该结构和服务端的 REST 负载非常相似。

`DynamicClient` 将所有数据绑定推迟到运行时，这意味着程序运行之前，使用 `DynamicClient` 的的程序将不会获取到类型验证的任何好处。对于某些需要强数据类型检查和验证的应用程序来说，这可能是一个问题。

然而，松耦合意味着当客户端 API 发生变化时，使用 `DynamicClient` 的程序不需要重新编译。客户端程序在处理 API 表面更新时具有更大的灵活性，而无需提前知道这些更改是什么。

## 三、组件介绍

下面对图中每个组件进行简单介绍：

​    **client-go 组件：**

1. **Reflector**: 定义在 [/tools/cache 包内的 Reflector 类型](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go) 中的 reflector 监视 Kubernetes API 以获取指定的资源类型 (Kind)。完成此操作的函数是 ListAndWatch。监视可以用于内建资源，也可以用于自定义资源。当 reflector 通过监视 API 的收到关于新资源实例存在的通知时，它使用相应的 listing API 获取新创建的对象，并将其放入 watchHandler 函数内的 Delta Fifo 队列中。

2. **Informer**: 在 [/tools/cache 包内的基础 controller](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/blob/master/tools/cache/controller.go) 中定义的一个 informer 从 Delta FIFO 队列中弹出对象。完成此操作的函数是 processLoop。这个基础 controller 的任务是保存对象以供以后检索，并调用 controller 将对象传递给它。

3. **Indexer**: indexer 为对象提供索引功能。它定义在 [/tools/cache 包内的 Indexer 类型](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/blob/master/tools/cache/index.go)。一个典型的索引用例是基于对象标签创建索引。Indexer 可以基于多个索引函数维护索引。Indexer 使用线程安全的数据存储来存储对象及其键值。在 [/tools/cache 包内的 Store 类型](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/blob/master/tools/cache/store.go) 定义了一个名为 `MetaNamespaceKeyFunc` 的默认函数，该函数为该对象生成一个名为 `<namespace>/<name>` 组合的对象键值。

   **Custom Controller 组件：**

4. **Informer reference**: 这是一个知道如何使用自定义资源对象的 Informer 实例的引用。您的自定义控制器代码需要创建适当的 Informer。

5. **Indexer reference**: 这是一个知道如何使用自定义资源对象的 Indexer 实例的引用。您的自定义控制器代码需要创建这个。您将使用此引用检索对象，以便稍后处理。

6. **Resource Event Handlers**: 当 Informer 想要分发一个对象给你的控制器时，会调用这些回调函数。编写这些函数的典型模式是获取已分配对象的键值，并将该键值放入一个工作队列中进行进一步处理。

7. **Work queue**: 这是在控制器代码中创建的队列，用于将对象的分发与处理解耦。编写 Resource Event Handler 函数来提取所分发对象的键值并将其添加到工作队列中。

8. **Process Item**: 这是在代码中创建的处理 work queue 中的 items 的函数。可以有一个或多个其他函数来执行实际的处理。这些函数通常使用 [Indexer 引用](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go%23L73) 或 Listing wrapper 来获取与键值对应的对象。

## 四、Custom Controller 

**自定义controller实现流程如下：**

![img](https://blogstatic.haohtml.com/uploads/2023/04/d2b5ca33bd970f64a6301fa75ae2eb22-2.png?x-oss-process=image%2Fformat,webp)

相关实现案例可参考：https://github.com/trstringer/k8s-controller-custom-resource



## 五、CRD资源与Controller和operator关系

CRD（Custom Resource Definition）资源是一种自定义资源类型，允许用户在 Kubernetes 中定义自己的 API 对象。CRD 可以定义自己的 API 对象，这些对象可以像 Kubernetes 原生资源一样进行管理和操作。

在 Kubernetes 中，CRD 资源和 Controller 之间存在一种父子关系。CRD 资源定义了自己的 API 对象，而 Controller 可以通过监视这些对象来控制它们的状态。当 CRD 资源中的对象状态发生变化时，Controller 会根据变化的状态来执行相应的操作，以确保资源的状态与期望的状态一致。

例如，如果用户定义了一个名为 "MyResource" 的 CRD 资源，并创建了一个名为 "my-resource" 的对象，那么 Controller 可以监视这个对象，并在对象状态发生变化时执行相应的操作。如果对象状态发生变化，Controller 可以更新对象的状态，或者执行其他操作来确保资源的状态与期望的状态一致。

而Operator是一种在Kubernetes中使用Controller的特定类型。Operator是一种自动化Kubernetes管理的方式，它使用自定义控制器来管理应用程序的状态。Operator可以监视CRD资源，并在资源状态发生变化时自动执行操作。例如，如果您有一个CRD资源来表示您的应用程序的特定部署，Operator可以监视该资源，并在部署发生更改时自动更新应用程序的状态。Operator可以使用自定义控制器来执行这些操作，这些控制器可以使用Kubernetes API来管理Pod和其他资源的状态。

因此，CRD资源可以与Controller和Operator一起使用。您可以使用CRD来定义自己的自定义资源，然后使用Controller来管理这些资源的状态。或者，您可以使用Operator来监视CRD资源，并在资源状态发生变化时自动执行操作。无论您选择哪种方法，CRD资源都可以帮助您更好地管理Kubernetes中的应用程序。



## 六、总结

kubernetes 的设计理念是通过各种控制器将系统的实际运行状态协调到声明 API 中的期待状态。而这种协调机制就是基于 client-go 实现的。同样，kubernetes 对于 ETCD 存储的缓存处理也使用到了 client-go 中的 Reflector 机制。所以学好 client-go，等于迈入了 Kubernetes 的大门！相关源码分析可查看源码分析目录。



## 七、参考链接

1. [client-go 源码学习总结](https://zhuanlan.zhihu.com/p/202611841)