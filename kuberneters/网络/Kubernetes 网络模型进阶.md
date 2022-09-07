#                                                   Kubernetes 网络模型进阶

## Underlay Network Model

### 什么是Underlay Network

底层网络 *Underlay Network* 顾名思义是指网络设备基础设施，如交换机，路由器, *DWDM* 使用网络介质将其链接成的物理网络拓扑，负责网络之间的数据包传输。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyxjoDlKBvyPVoThkJ42pKhe4t3iaE0U1VgCcn0jybn1TCx6yicMB1efjA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                  **图：Underlay network topology**

*Source：*https://community.cisco.com/t5/data-center-switches/understanding-underlay-and-overlay-networks/td-p/4295870



*underlay network* 可以是二层，也可以是三层；二层 *underlay network* 的典型例子是以太网 *Ethernet*，三层是 *underlay network* 的典型例子是互联网 *Internet*。

而工作与二层的技术是 *vlan*，工作在三层的技术是由 *OSPF*, *BGP* 等协议组成

### kubernetes中的underlay network

在kubernetes中，*underlay network* 中比较典型的例子是通过将宿主机作为路由器设备，Pod 的网络则通过学习成路由条目从而实现跨节点通讯。

![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyqjHDK7dv8WtgBpibmbozUC7wo5zWdEDlIkoK03vxpv5QoR7ciaVNKzTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



​                                                                                  **图：underlay network topology in kubernetes**

这种模型下典型的有 *flannel* 的 *host-gw* 模式与 *calico* *BGP* 模式。

#### flannel host-gw [1]

*flannel host-gw* 模式中每个Node需要在同一个二层网络中，并将Node作为一个路由器，跨节点通讯将通过路由表方式进行，这样方式下将网络模拟成一个*underlay network*。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyDLPDv7ZTCW14W1wtUs1SbR9yOcibm3ncwCd6dJV7C076XoSovLehaZA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                         **图：layer2 ethernet topology**

*Source：*https://www.auvik.com/franklyit/blog/layer-3-switches-layer-2/

> Notes：因为是通过路由方式，集群的cidr至少要配置16，因为这样可以保证，跨节点的Node作为一层网络，同节点的Pod作为一个网络。如果不是这种用情况，路由表处于相同的网络中，会存在网络不可达

#### Calico BGP [2]

BGP（*Border Gateway Protocol*）是去中心化自治路由协议。它是通过维护IP路由表或'前缀'表来实现AS （*Autonomous System*）之间的可访问性，属于向量路由协议。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyGwaHm71fY1JAaOyWcdvfX0gjcO0e61aB56kskOgdjsLSmiayBoaLeQg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                                  **图：BGP network topology**

*Source：*https://infocenter.nokia.com/public/7705SAR214R1A/index.jsp?topic=%2Fcom.sar.routing_protocols%



与 *flannel* 不同的是，*Calico* 提供了的 *BGP* 网络解决方案，在网络模型上，*Calico* 与 *Flannel host-gw* 是近似的，但在软件架构的实现上，*flannel* 使用 *flanneld* 进程来维护路由信息；而 *Calico* 是包含多个守护进程的，其中 *Brid* 进程是一个 *BGP* 的客户端 与路由反射器(*Router Reflector*)，*BGP* 客户端负责从 *Felix* 中获取路由并分发到其他 *BGP Peer*，而反射器在BGP中起了优化的作用。在同一个IBGP中，BGP客户端仅需要和一个 *RR* 相连，这样减少了*AS*内部维护的大量的BGP连接。通常情况下，*RR* 是真实的路由设备，而 *Bird* 作为 *BGP* 客户端工作。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyfEXWkGJTBS8TSt3pksPGe18LTicxXWDvCnTZXPibJrJ5Og5oE1wOHibrQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                                       **图：Calico Network Architecture**

*Source：*https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/cisco-nx-os-calico-network-design.html



#### IPVLAN & MACVLAN [4]

*IPVLAN* 和 *MACVLAN* 是一种网卡虚拟化技术，两者之间的区别为， *IPVLAN* 允许一个物理网卡拥有多个IP地址，并且所有的虚拟接口用同一个MAC地址；而 *MACVLAN* 则是相反的，其允许同一个网卡拥有多个MAC地址，而虚拟出的网卡可以没有IP地址。

因为是网卡虚拟化技术，而不是网络虚拟化技术，本质上来说属于 *Overlay network*，这种方式在虚拟化环境中与*Overlay network* 相比最大的特点就是可以将Pod的网络拉平到Node网络同级，从而提供更高的性能、低延迟的网络接口。本质上来说其网络模型属于下图中第二个。

- 虚拟网桥：创建一个虚拟网卡对(veth pair)，一头栽容器内，一头栽宿主机的root namespaces内。这样一来容器内发出的数据包可以通过网桥直接进入宿主机网络栈，而发往容器的数据包也可以经过网桥进入容器。
- 多路复用：使用一个中间网络设备，暴露多个虚拟网卡接口，容器网卡都可以介入这个中间设备，并通过MAC/IP地址来区分packet应该发往哪个容器设备。
- 硬件交换，为每个Pod分配一个虚拟网卡，这样一来，Pod与Pod之间的连接关系就会变得非常清晰，因为近乎物理机之间的通信基础。如今大多数网卡都支持SR-IOV功能，该功能将单一的物理网卡虚拟成多个VF接口，每个VF接口都有单独的虚拟PCIe通道，这些虚拟的PCIe通道共用物理网卡的PCIe通道。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyXl7hJSbrKicdnPZLbNhads4BrWPwDS78RZJIAtodUb5v8m0mWtkmiaDQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                           **图：Virtual networking modes: bridging, multiplexing and SR-IOV**

*Source：*https://thenewstack.io/hackers-guide-kubernetes-networking/



在kubernetes中 *IPVLAN* 这种网络模型下典型的CNI有，multus 与 danm。

##### multus

*multus* 是 intel 开源的CNI方案，是由传统的 *cni* 与 *multus* 组成，并且提供了 SR-IOV CNI 插件使 K8s pod 能够连接到 SR-IOV VF 。这是使用了 *IPVLAN/MACVLAN* 的功能。

当创建新的Pod后，SR-IOV 插件开始工作。配置 VF 将被移动到新的 CNI 名称空间。该插件根据 CNI 配置文件中的 “name” 选项设置接口名称。最后将VF状态设置为UP。

下图是一个 Multus 和 SR-IOV CNI 插件的网络环境，具有三个接口的 pod。

- *eth0* 是 *flannel* 网络插件，也是作为Pod的默认网络
- VF 是主机的物理端口 *ens2f0* 的实例化。这是英特尔X710-DA4上的一个端口。在Pod端的 VF 接口名称为 *south0* 。
- 这个VF使用了 DPDK 驱动程序，此 VF 是从主机的物理端口 *ens2f1* 实例化出的。这个是英特尔® X710-DA4上另外一个端口。Pod 内的 VF 接口名称为 *north0*。该接口绑定到 DPDK 驱动程序 *vfio-pci* 。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxy1fqm6eakBu9XZT59bervsUvIFp2pF4fteTOULSaV24NIaSTFaTCuYA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                      **图：Mutus networking Architecture overlay and SR-IOV**

*Source：*https://builders.intel.com/docs/networkbuilders/enabling_new_features_in_kubernetes_for_NFV.pdf



> Notes：terminology
>
> - NIC：network interface card，网卡
> - SR-IOV：single root I/O virtualization，硬件实现的功能，允许各虚拟机间共享PCIe设备。
> - VF：Virtual Function，基于PF，与PF或者其他VF共享一个物理资源。
> - PF：PCIe Physical Function，拥有完全控制PCIe资源的能力
> - DPDK：Data Plane Development Kit

于此同时，也可以将主机接口直接移动到Pod的网络名称空间，当然这个接口是必须存在，并且不能是与默认网络使用同一个接口。这种情况下，在普通网卡的环境中，就直接将Pod网络与Node网络处于同一个平面内了。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyBpQH5b5UoWxWhM0YKPIVBW25oZLowbR6BuCu8ZGf0zXeiatS7zxudTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                         **图：Mutus networking Architecture overlay and ipvlan**

*Source：*https://devopstales.github.io/kubernetes/multus/



##### danm

DANM是诺基亚开源的CNI项目，目的是将电信级网络引入kubernetes中，与multus相同的是，也提供了SR-IOV/DPDK 的硬件技术，并且支持IPVLAN.

## Overlay Network Model

### 什么是Overlay

叠加网络是使用网络虚拟化技术，在 *underlay* 网络上构建出的虚拟逻辑网络，而无需对物理网络架构进行更改。本质上来说，*overlay network* 使用的是一种或多种隧道协议 (*tunneling*)，通过将数据包封装，实现一个网络到另一个网络中的传输，具体来说隧道协议关注的是数据包（帧）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyyGkhSJE7WhbUna4s0mvzghkvGDCgPsPNtmibTUAtIWYfCRLdkGSasGQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图：overlay network topology

*Source：*https://www.researchgate.net/figure/Example-Overlay-Network-built-on-top-of-an-Internet-style-Underlay_fig4_230774628



### 常见的网络隧道技术

- 通用路由封装 ( *Generic Routing Encapsulation* ) 用于将来自 IPv4/IPv6的数据包封装为另一个协议的数据包中，通常工作与L3网络层中。
- VxLAN (*Virtual Extensible LAN*)，是一个简单的隧道协议，本质上是将L2的以太网帧封装为L4中UDP数据包的方法，使用 4789 作为默认端口。*VxLAN* 也是 *VLAN* 的扩展对于 4096（212 位 *VLAN ID*） 扩展为1600万（224 位 *VNID* ）个逻辑网络。

这种工作在 *overlay* 模型下典型的有 *flannel* 与 *calico* 中的的 *VxLAN*, *IPIP* 模式。

### IPIP

*IP in IP* 也是一种隧道协议，与 *VxLAN* 类似的是，*IPIP* 的实现也是通过Linux内核功能进行的封装。*IPIP* 需要内核模块 `ipip.ko` 使用命令查看内核是否加载IPIP模块`lsmod | grep ipip` ；使用命令`modprobe ipip` 加载。



![图片](https://cdn.jsdelivr.net/gh/longpi1/blog-img/640)

图：A simple IPIP network workflow

*Source：*https://ssup2.github.io/theory_analysis/IPIP_GRE_Tunneling/



Kubernetes中 *IPIP* 与 *VxLAN* 类似，也是通过网络隧道技术实现的。与 *VxLAN* 差别就是，*VxLAN* 本质上是一个 UDP包，而 *IPIP* 则是将包封装在本身的报文包上。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxybmxBiantAKWVD2nlCUNBHAIQkHSOJZcQeM4znchxeRqicRNvh3pcPIMw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                                         **图：IPIP in kubernetes**

![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyNfkHJrC9xJw6SzZRFND1XdRacXJ6A7utD0RYvRyj7qOwJPzM60YiaeA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                         **图：IPIP packet with wireshark unpack**



> Notes：公有云可能不允许IPIP流量，例如Azure

### VxLAN

kubernetes中不管是 *flannel* 还是 *calico* VxLAN的实现都是使用Linux内核功能进行的封装，Linux 对 vxlan 协议的支持时间并不久，2012 年 Stephen Hemminger 才把相关的工作合并到 kernel 中，并最终出现在 kernel 3.7.0 版本。为了稳定性和很多的功能，你可以会看到某些软件推荐在 3.9.0 或者 3.10.0 以后版本的 kernel 上使用 *VxLAN*。

![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyVuvdFPpJgI1QG5U2MHUib3DbBGia1HVB6sicRiadIptJxM0B7nUXaCSqYQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                             **图：A simple VxLAN network topology**



在kubernetes中vxlan网络，例如 *flannel*，守护进程会根据kubernetes的Node而维护 *VxLAN*，名称为 `flannel.1` 这是 *VNID*，并维护这个网络的路由，当发生跨节点的流量时，本地会维护对端 *VxLAN* 设备的MAC地址，通过这个地址可以知道发送的目的端，这样就可以封包发送到对端，收到包的对端 VxLAN设备 `flannel.1` 解包后得到真实的目的地址。

查看 *Forwarding database* 列表

```
$ bridge fdb 26:5e:87:90:91:fc dev flannel.1 dst 10.0.0.3 self permanent
```



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyib50Tia4cxibibR5uhmL4eO4m158hQFxZsiaWaqYE9vH2Fflee6aEaEACJg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                                      **图：VxLAN in kubernetes**

![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxyCSReVTSz26R2z2ibGa2HvNuTjwKI8tQHHv14amJr1eoOTw05gpMc5mg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                       **图：VxLAN packet with wireshark unpack**

> Notes：VxLAN使用的4789端口，wireshark应该是根据端口进行分析协议的，而flannel在linux中默认端口是8472，此时抓包仅能看到是一个UDP包。

通过上述的架构可以看出，隧道实际上是一个抽象的概念，并不是建立的真实的两端的隧道，而是通过将数据包封装成另一个数据包，通过物理设备传输后，经由相同的设备（网络隧道）进行解包实现网络的叠加。

### weave vxlan [3]

weave也是使用了 *VxLAN* 技术完成的包的封装，这个技术在 *weave* 中称之为 *fastdp (fast data path)*，与 *calico* 和 *flannel* 中用到的技术不同的，这里使用的是 Linux 内核中的 *openvswitch datapath module*，并且weave对网络流量进行了加密。



![图片](https://mmbiz.qpic.cn/mmbiz_png/D727NicjCjMOfSoHgRPZfL1ZzWoqxIyxy1r4xqNRVbh6Ua8kaalhWPbicCYYI0CcbC3tLeuoMGHxLX6zLqmEOiawA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​                                                                                           **图：weave fastdp network topology**

*Source：*https://www.weave.works/docs/net/latest/concepts/fastdp-how-it-works/

> Notes：fastdp工作在Linux 内核版本 3.12 及更高版本，如果低于此版本的例如CentOS7，weave将工作在用户空间，weave中称之为 *sleeve mode*



Reference

[1] flannel host-gw

[2] calico bgp networking

[3] calico bgp networking

[4] sriov network

[5] danm

作者：Cylon

出处：https://www.cnblogs.com/Cylon/p/16595820.html