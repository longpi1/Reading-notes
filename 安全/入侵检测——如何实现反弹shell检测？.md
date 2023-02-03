#             入侵检测——如何实现反弹shell检测？

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

除此以外，还可以对特定场景进行分析，进程对应的fd是否异常并且外联；



## 三、总结

这篇文章主要介绍了常见的反弹shell检测思路，以及实现举例，欢迎大家进行补充与分享！



## 四、参考链接

1. [郑瀚Andrew](https://home.cnblogs.com/u/LittleHann/)的 [反弹Shell原理及检测技术研究](https://www.cnblogs.com/LittleHann/p/12038070.html)

