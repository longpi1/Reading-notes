# neuvector网络学习与管控实现

## 一、具体实现

相关介绍：

| packetmmap： https://sites.google.com/site/packetmmap/ | 、 https://docs.kernel.org/networking/packet_mmap.html | 、 http |
| ------------------------------------------------------ | ------------------------------------------------------ | ------- |
| s://man7.org/linux/man-pages/man7/packet.7.html        |                                                        |         |

tcp复位攻击： https://juejin.cn/post/7036535891094929438 、 https://segmentfault.com/a/1190000022954874 、 https://zh.m.
wikipedia.org/zh-hans/TCP%E9%87%8D%E7%BD%AE%E6%94%BB%E5%87%BB
nfqueue： http://just4coding.com/2017/03/27/nfqueue/
TC流量控制： https://www.cnblogs.com/yhp-smarthome/p/11182683.html
sidecar流量劫持： https://jimmysong.io/kubernetes-handbook/usecases/understand-sidecar-injection-and-traffic-hijack-in-istioservice-mesh.html
需要了解的重要逻辑：
1.Netfilter与iptables
2.TC流量控制
下图为网络数据包通过Netfilter时的工作流向.
3.集群的网络工作原理：

![image-20231219225721479](C:/Users/longp/AppData/Roaming/Typora/typora-user-images/image-20231219225721479.png)

