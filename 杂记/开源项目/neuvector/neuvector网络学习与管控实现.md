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

### agent侧进程间通信

dp/ctrl.c中的dp_ctrl_loop方法与agent/dp/dp.go中的listenDP和connectDP方法监（"/tmp/dp_listen.sock"和"/tmp/ctrl_listen.sock"）两个socket以进行与dp侧数据通信。

#### 相关的通信图如下：  

![image-20231219224817158](C:/Users/longp/AppData/Roaming/Typora/typora-user-images/image-20231219224817158.png)

#### 具体代码如下：

##### 1.agent侧

1.1通过listenDP方法监听socket = "/tmp/ctrl_listen.sock" 从ctrl.c:dp_ctrl_loop获取对应的信息,然后更新威胁日志、域名转换ip、网络连
接、应用的mac地址更新  

![image-20231219225118807](C:/Users/longp/AppData/Roaming/Typora/typora-user-images/image-20231219225118807.png)

1.2 通过monitorDP方法监听 DPServer string = "/tmp/dp_listen.sock"， agent侧在service.go中可以通过dp侧获取到相关数据转化成
CLUSDatapathCounter、CLUSMetry、CLUSSessionCounter、CLUSSession、CLUSMeter等数据， Controller侧通过grpc的方法调用
service的方法获取到上述数据信息。  

![image-20231219225239683](C:/Users/longp/AppData/Roaming/Typora/typora-user-images/image-20231219225239683.png)

##### 2.dp侧  

2.1.dp/ctrl.c中的dp_ctrl_loop方法通过与agent监听socket（/tmp/dp_listen.soc）实现从agent侧获取对应的数据。（agent侧最终会将对应key的数据通过dpSendMsgExSilent等方法传输给socket）。  

![image-20231219225415608](C:/Users/longp/AppData/Roaming/Typora/typora-user-images/image-20231219225415608.png)

2.2 dp_ctrl_loop方法收到socket侧的数据后调用dp_ctrl_handler(g_ctrl_fd)函数进行遍历处理，通过识别对应的key采取对应的措施。  

![image-20231219225508951](C:/Users/longp/AppData/Roaming/Typora/typora-user-images/image-20231219225508951.png)