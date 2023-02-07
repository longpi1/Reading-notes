#             入侵检测——如何实现反弹shell检测？

> 反弹shell的本质：就是控制端监听在某TCP/UDP端口，被控端发起请求到该端口，并将其命令行的输入输出转到控制端。reverse shell与telnet，ssh等标准shell对应，本质上是网络概念的客户端与服务端的角色反转。
>
> 反弹shell的结果：一个client上的bash进程 可以和 server上的进程通信。
>
> 而反弹shell的检测，本质上就是检测 shell进程（如bash）的输入输出是否来自于一个远程的server。

## 一、检测思路

## 1.进程 file descriptor 异常检测

### 1.1 检测 file descriptor 是否指向一个socket

以“重定向符”+"/dev/tcp网络通信"Bash反弹Shell这一类最经典的反弹Shell攻击方式为例，这类反弹shell的本质可以归纳为**file descriptor的重定向到一个socket句柄**。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215104034125-1531808234.png)

 ![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215104112969-1097291178.png) 

### 1.2 检测 file descriptor 是否指向一个管道符（pipe）

对于利用“管道符”传递指令的反弹shell攻击方式来说，这类反弹shell的本质可以归纳为**file descriptor的重定向到一个pipe句柄**。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215105358151-246000034.png)

更进一步地说，不管做了多少层的pipe，反弹shell的本质是将server的输入传递给client的bash，因此肯定存在socket连接。

我们只需要根据pid追溯pipe上游的进程，并判断其进程fd，检查是否是来自一个socket。

例如，跟踪pipe，发现pipe的进程建立了socket连接，那么就存在反弹shell的风险。 

## 2.netlink监控+fd异常检测

- 监听Netlink Socket，实时获取进程EXEC事件。
- 如果为Shell进程，检查进程启动打开的FD，
  - 打开了Socket
  - 未使用/dev/tty、/dev/pts/n、/dev/ptmx等终端
  - 则确认为反弹Shell 

**绕过风险：仅能通过进程执行文件名判断是否为Shell进程，上传可执行文件、拷贝Bash文件到其他路径等方法会绕过这个方法**。

例如通过将/bin/sh重命名为其他名字进行反弹shell。

## 3.脚本文件 && 应用程序 && 无文件（fileless）反弹shell检测

需要注意的是，操作系统是分层的，Bash只是一个应用程序的普通应用，其内部封装了调用glibc execve的功能而已，除了bash之外，白帽子还可以基于任意的应用层技术来实现反弹shell，例如：

- python/perl实现纯代码形式的反弹shell文件执行：文件脚本检测
- python/perl实现纯代码形式的反弹shell命令行指令（fileless）：纯命令行fileless检测
- C/C++实现纯代码形式的反弹shell：二进制文件检测

## 4. 特征检测

#### 4.1网络层反弹shell通信特征检测

反弹shell的通信会话中，会包含一些”cmdline shell特征“，例如”#root....“等，可以在网络侧进行NTA实时检测。



#### 4.2DNS反弹shell特征检测

针对DNS流量进行分析，判断关联进程是否开启/dev/net/tun，或者/dev/net/tap隧道等等。



#### 4.3 ICMP反弹shell特征检测

对于正常的ping命令产生的数据，有以下特点：

● 每秒发送的数据包个数比较少，通常每秒最多只会发送两个数据包；

● 请求数据包与对应的响应数据包内容一样；

● 数据包中payload的大小固定，windows下为32bytes，linux下为48bytes；

● 数据包中payload的内容固定，windows下为abcdefghijklmnopqrstuvwabcdefghi，linux下为!”#$%&’()+,-./01234567，如果指定ping发送的长度，则为不断重复的固定字符串；

● type类型只有2种，0和8。0为请求数据，8为响应数据。

对于ICMP隧道产生的数据，有以下特点：

● 每秒发送的数据包个数比较多，在同一时间会产生成百上千个 ICMP 数据包；

● 请求数据包与对应的响应数据包内容不一样；

● 数据包中 payload的大小可以是任意大小；

● 存在一些type为13/15/17的带payload的畸形数据包；

● 个别ICMP隧道工具产生的数据包内容前面会增加 ‘TUNL’ 标记以用于识别隧道。

因此，根据正常ping和ICMP隧道产生的数据包的特点，可以通过以下几点特征检测ICMP隧道:

● 检测同一来源数据包的数量。正常ping每秒只会发送2个数据包，而ICMP隧道可以每秒发送很多个；

● 检测数据包中 payload 的大小。正常ping产生的数据包payload的大小为固定，而ICMP隧道数据包大小可以任意；

● 检测响应数据包中 payload 跟请求数据包是否不一致。正常ping产生的数据包请求响应内容一致，而ICMP隧道请求响应数据包可以一致，也可以不一致；

● 检测数据包中 payload 的内容。正常ping产生的payload为固定字符串，ICMP隧道的payload可以为任意；

● 检测 ICMP 数据包的type是否为0和8。正常ping产生的带payload的数据包，type只有0和8，ICMP隧道的type可以为13/15/17。

![图片名称](https://blog.riskivy.com/wp-content/uploads/2019/04/p6.png)

具体实现可参考https://blog.riskivy.com/%E5%9F%BA%E4%BA%8E%E7%BB%9F%E8%AE%A1%E5%88%86%E6%9E%90%E7%9A%84icmp%E9%9A%A7%E9%81%93%E6%A3%80%E6%B5%8B%E6%96%B9%E6%B3%95%E4%B8%8E%E5%AE%9E%E7%8E%B0/

## 二、具体实现举例

### 1.监听Netlink Socket 并轮询处理

```go
func (ns *NetlinkSocket) ReceiveFrom() ([]syscall.NetlinkMessage, syscall.Sockaddr, error) {
	nr, from, err := syscall.Recvfrom(ns.fd, ns.buf, 0)
	if err != nil {
		return nil, from, err
	}
	if nr < syscall.NLMSG_HDRLEN {
		return nil, from, fmt.Errorf("Got short response from netlink")
	}

	msg, err := syscall.ParseNetlinkMessage(ns.buf[:nr])
	return msg, from, err
}

```

### 2.实时处理进程EXEC事件

```go

func (p *Probe) netLinkHandler(e *netlinkProcEvent) {
	switch e.Event {
	case netlink.PROC_EVENT_EXEC:
		p.handleProcExec(e.Pid, false) // pid
	}
}
```

### 3.根据反弹shell的基本原理，判断进程 file descriptor 是否异常

```go
//根据反弹shell的基本原理，判断进程 file descriptor 是否异常（也就是是否相等）
func (p *Probe) checkReverseShell(pid int)  {
    //获取对应的inode
   inodeStdin, err := osutil.GetFDSocketInode(pid, 0)
   if err != nil {
      return nil
   }
   inodeStdout, err := osutil.GetFDSocketInode(pid, 1)
   if err != nil {
      return nil
   }
   if inodeStdin != inodeStdout || inodeStdin == 0 {
      return nil
   }
   return osutil.GetProcessConnection(pid, nil, utils.NewSet(inodeStdin))
}

//获取进程对应的连接
func GetProcessConnection(pid int, clientPort *share.CLUSProtoPort, inodes utils.Set) *Connection {
	var err error
	if inodes == nil {
		inodes, err = GetProcessSocketInodes(pid)
		if err != nil {
			return nil
		}
	}
	if inodes.Cardinality() == 0 {
		return nil
	}
	var sport uint16
	if clientPort != nil {
		sport = clientPort.Port
	}
	pidDir := global.SYS.ContainerProcFilePath(pid, "/")
	if clientPort == nil || clientPort.IPProto == syscall.IPPROTO_TCP {
		if conn := getConnectionByFile(pidDir+"net/tcp", inodes, true, sport); conn != nil {
			return conn
		}
		if conn := getConnectionByFile(pidDir+"net/tcp6", inodes, true, sport); conn != nil {
			return conn
		}
	}
	if clientPort == nil || clientPort.IPProto == syscall.IPPROTO_UDP {
		if conn := getConnectionByFile(pidDir+"net/udp", inodes, false, sport); conn != nil {
			return conn
		}
		if conn := getConnectionByFile(pidDir+"net/udp6", inodes, false, sport); conn != nil {
			return conn
		}
	}
	return nil
}
```

**以上方式的缺点为：仅能通过进程执行文件名判断是否为Shell进程，上传可执行文件、拷贝Bash文件到其他路径等方法会绕过这个方法**。

除此以外，还可以对特定场景进行分析，进程对应的fd是否异常并且外联，开源代码可参考：https://github.com/zhanghaoyil/seesaw；



## 三、总结

这篇文章主要介绍了常见的反弹shell检测思路，以及实现举例，欢迎大家进行补充与分享！



## 四、参考链接

1. [郑瀚Andrew](https://home.cnblogs.com/u/LittleHann/)的 [反弹Shell原理及检测技术研究](https://www.cnblogs.com/LittleHann/p/12038070.html)

   

