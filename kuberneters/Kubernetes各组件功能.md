#                                  Kubernetes集群架构与组件介绍

## 一、集群架构

![集群架构](https://s2.loli.net/2023/04/05/NJErpDqHkXifKZb.png)

## 二、主要组件

### 1.kubelet

该组件运行在每个Kubernetes节点上，用于管理节点。用来接收、处理、上报kube-apiserver组件下发的任务。

主要负责所在节点上的Pod资源对象的管理，例如Pod资源对象的创建、修改、监控、删除、驱逐及Pod生命周期管理等。 kubelet组件会定期监控所在节点的资源使用状态并上报给kube-apiserver组件，这些资源数据可以帮助kube-scheduler调度器为Pod资源对象预选节点。kubelet也会对所在节点的镜像和容器做清理工作，保证节点上的镜像不会占满磁盘空间、删除的容器释放相关资源。 kubelet组件实现了3种开放接口：

- Container Runtime Interface：简称CRI（容器运行时接口）提供容器运行时通用插件接口服务。CRI定义了容器和镜像服务的接口。CRI将kubelet组件与容器运行时进行解耦，将原来完全面向Pod级别的内部接口拆分成面向Sandbox和Container的gRPC接口，并将镜像管理和容器管理分离给不同的服务。
- Container Network Interface：简称CNI（容器网络接口），提供网络通用插件接口服务。CNI定义了Kubernetes网络插件的基础，容器创建时通过CNI插件配置网络。
- Container Storage Interface：简称CSI（容器存储接口），提供存储通用插件接口服务。CSI定义了容器存储卷标准规范，容器创建时通过CSI插件配置存储卷。



### 2.client-go

client-go是从Kubernetes代码中抽离出来的包，作为官方提供的GO语言的客户端发挥作用。client-go简单、易用，Kubernetes系统的其他组件与Kubernetes API Server通信的方式也是基于client-go实现。



### 3.kube-apiserver

API 服务器是 Kubernetes [控制平面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane)的组件， 该组件公开了 Kubernetes API，负责处理接受请求的工作。 API 服务器是 Kubernetes 控制平面的前端。

它负责Kubernetes"资源组/资源版本/资源"以RESTful风格的形式对外暴露并提供服务。kube-apiserver组件也是集群中唯一与Etcd集群进行交互的核心组件。Kubernetes将所有的数据存储在Etcd集群中前缀为registry的目录下。 kube-apiserver的特性：

- 将Kubernetes系统中所有资源对象都封装成RESTful风格的API接口进行管理。
- 可进行集群状态管理和数据管理，是唯一与Etcd集群交互的组件。
- 拥有丰富的集群安  全访问机制，以及认证，授权及准入控制器。
- 提供了集群各组件的通信和交互功能。



### 4.kube-controller-manager

它负责管理Kubernetes集群中的节点（Node），Pod副本，服务，端点（Endpoint），命名空间（Namespace），服务账户（ServiceAcconut)，资源定额（ResourceQuota）等。

包括：

- 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
- 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
- 端点分片控制器（EndpointSlice controller）：填充端点分片（EndpointSlice）对象（以提供 Service 和 Pod 之间的链接）。
- 服务账号控制器（ServiceAccount controller）：为新的命名空间创建默认的服务账号（ServiceAccount）。



### 5.kube-scheduler

该组件是Kubernetes集群默认的调度器，它负责在Kubernetes集群中为一个Pod资源对象找到合适的节点并在该节点上运行。在调度的过程中每次只负责调度一个Pod资源对象。调度算法分为两种，分别为预选调度算法和优选调度算法。



### 6.kubectl

kubectl是Kubernetes官方提供的命令行工具CLI，用户可以通过命令行的方式与Kubernetes API Server进行操作，通信协议使用HTTP/JSON。



### 7.kube-proxy

该组件运行在Kubernetes集群中每个节点上，作为节点上的网络代理。它监控kube-apiserver的服务和端点资源变化，并通过iptables/ipvs等配置负载均衡器，为一组Pod提供统一的TCP/UDP流量转发和负载均衡功能。但是kube-proxy组件与其他负载均衡服务的区别在于kube-proxy代理只想Kubernetes服务及其后端Pod发出请求。



### 8.Add-ons

除了核心组件，还有一些推荐的Add-ons：

- kube-dns负责为整个集群提供DNS服务
- Ingress Controller为服务提供外网入口
- Heapster提供资源监控
- Dashboard提供GUI
- Federation提供跨可用区的集群
- Fluentd-elasticsearch提供集群日志采集、存储与查询



## 三、Kubernetes Project Layout 设计

| 源码目录      | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| cmd/          | 可执行文件的入口代码，每个可执行文件都会对应一个main函数     |
| pkg/          | 主要实现，核心库代码，并且可被项目内部或外部直接引用，       |
| vendor/       | 项目依赖的库代码，一般为第三方库代码                         |
| api/          | OpenAPI/Swagger的spce文件，包括JSON、Protocol的定义等        |
| build/        | 与构建相关的脚本                                             |
| test/         | 测试工具及测试数据                                           |
| docs/         | 设计或用户使用文档                                           |
| hack/         | 构建、测试等相关代码                                         |
| third_party   | 第三方工具、代码或者其他组件                                 |
| plugin/       | Kubernetes插件代码目录，例如认证、授权等相关插件             |
| staging/      | 部分核心库的暂存目录                                         |
| translations/ | i18n（国际化）语言包的相关文件，可以在不修改内部代码的情况下支持不同语言及地区 |



## 参考链接

1.https://raw.githubusercontent.com/kubernetes

2.https://juejin.cn/post/6999610929105240094