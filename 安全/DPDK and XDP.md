# DPDK and XDP

> 装载自腾讯云，原文链接：https://cloud.tencent.com/developer/article/1484793

# 预备知识

## tcp/ip

### TCP/IP网络模型

- 链路层：负责封装和解封装IP报文，发送和接受ARP/RARP报文等。
- 网络层：负责路由以及把分组报文发送给目标网络或主机。
- 传输层：负责对报文进行分组和重组，并以TCP或UDP协议格式封装报文。
- 应用层：负责向用户提供应用程序，比如HTTP、FTP、Telnet、DNS、SMTP等。

### 封装

![img](https://ask.qcloudimg.com/http-save/1319879/w8h4cbei2s.png?imageView2/2/w/1620);



## iptables/netfilter

iptables是一个配置Linux内核防火墙的[命令行工具](https://cloud.tencent.com/product/cli?from=3346)，它基于内核的netfilter机制。新版本的内核（3.13+）也提供了nftables，用于取代iptables

![img](https://ask.qcloudimg.com/http-save/1319879/kcj3j4dupy.png?imageView2/2/w/1620)

## iptables的问题和改进方案

iptables规则逐渐增加，遍历iptables效率变得很低，一个表现就是kube-proxy，他是Kubernetes的一个组件，[容器](https://cloud.tencent.com/product/tke?from=3346)要使用iptables和-j DNAT规则为服务提供[负载均衡](https://cloud.tencent.com/product/clb?from=3346)。随着服务增加，iptable的规则列表指数增长。随着服务数量的增长，网络延迟和性能严重下降。iptables的还有一个缺点，无法实现增量更新。每次添加新规则时，必须更新整个规则列表。一个例子：装配2万个Kubernetes服务产生16万条的iptables规则需要耗时5个小时。

在容器环境下还有一个问题：容器的生命周期可能很多，可能一个容器的生命周期只有几秒，意味着iptables规则需要被快速更新，这也使得依靠使用IP地址进行安全过滤的系统受到压力，因为集群中的所有节点都必须始终知道最新的IP到容器的映射。

一个解决方案是BPF，[Cilium](https://github.com/cilium/cilium)项目就利用了这种技术.

![img](https://ask.qcloudimg.com/http-save/1319879/dhj7kcmvcr.png?imageView2/2/w/1620)

利用BPF构建的bpfilter性能远高于iptables和nftables, linux内核社区的Florian Westphal提出了一个运行在bpfilter上框架，通过框架并将nftables转换为BPF。框架允许保持特定领域nftables语句，而且还可以带有JIT编译器，硬件卸载和工具集等BPF运行时的所有优点。

## linux网络为什么慢

linux协议栈是在20世纪90年代作为一个通用操作系统实现的，想要支持现代的高速网络，必须要做优化.

[dog250](https://blog.csdn.net/dog250) 把linux协议栈重新"分层", 指出了其中的"门", 即那些会严重影响性能的门槛

![img](https://ask.qcloudimg.com/http-save/1319879/cnz7nsptwu.png?imageView2/2/w/1620);

目前有两个比较火的方案：DPDK和XDP，两种方案分别在用户层和内核层直接处理数据包，避免了用户、内核态切换k开销。

# DPDK

DPDK由intel支持，DPDK的加速方案原理是完全绕开内核实现的协议栈，把数据包直接从网卡拉到用户态，依靠Intel自身处理器的一些专门优化，来高速处理数据包。**使用DPDK前提需要能支持 DPDK 的网卡配合使用。**

Intel DPDK全称Intel Data Plane Development Kit，是intel提供的数据平面开发工具集，为Intel architecture（IA）处理器架构下用户空间高效的数据包处理提供库函数和驱动的支持，它不同于Linux系统以通用性设计为目的，而是专注于网络应用中数据包的高性能处理。DPDK应用程序是运行在用户空间上利用自身提供的数据平面库来收发数据包，绕过了Linux内核协议栈对数据包处理过程。Linux内核将DPDK应用程序看作是一个普通的用户态进程，包括它的编译、连接和加载方式和普通程序没有什么两样。DPDK程序启动后只能有一个主线程，然后创建一些子线程并绑定到指定CPU核心上运行。

![img](https://ask.qcloudimg.com/http-save/1319879/uw20zpb9p1.png?imageView2/2/w/1620);

# XDP

## eBPF

eBPF（extended Berkeley Packet Filter）起源于BPF，它提供了内核的数据包过滤机制。

- BPF is a highly flexible and efficient virtual machine-like construct in the Linux kernel allowing to execute bytecode at various hook points in a safe manner. It is used in a number of Linux kernel subsystems, most prominently networking, tracing and security (e.g. sandboxing).
- BPF的最原始版本为cBPF，曾用于tcpdump
- Berkeley Packet Filter 尽管名字的也表明最初是设计用于packet filtering，但是现在已经远不止networking上面的用途了.

![img](https://ask.qcloudimg.com/http-save/1319879/vxsewsqpf0.png?imageView2/2/w/1620);

基于bpf 这个[项目](https://github.com/iovisor/bcc)  开发了很多有用的小工具, 具体如下图

![img](https://ask.qcloudimg.com/http-save/1319879/67xpxaoubn.png?imageView2/2/w/1620)

更多关于bpf的历史和设计、概念可以参考[这篇文章](https://www.ibm.com/developerworks/cn/linux/l-lo-eBPF-history/index.html)

## XDP

XDP的意思是eXpress Data Path，它能够在网络包进入用户态直接对网络包进行过滤或者处理。XDP依赖eBPF技术。

![img](https://ask.qcloudimg.com/http-save/1319879/gun42kp75l.png?imageView2/2/w/1620);

![img](https://ask.qcloudimg.com/http-save/1319879/konghu5766.png?imageView2/2/w/1620);

相对于DPDK，XDP具有以下优点

- 无需第三方代码库和许可
- 同时支持轮询式和中断式网络
- 无需分配大页
- 无需专用的CPU
- 无需定义新的安全网络模型

XDP的使用场景包括

- DDoS防御
- 防火墙
- 基于XDP_TX的负载均衡
- 网络统计
- 复杂网络采样
- 高速交易平台

## 实战

这个例子使用bcc，参考 [这个例子](https://github.com/iovisor/bcc/blob/master/examples/networking/xdp/xdp_drop_count.py) 和 [这个例子](https://github.com/iovisor/bcc/blob/master/examples/networking/xdp/xdp_drop_count.py)改造而成

这个例子中，实现一个简单的ddos防火墙，当包很小，两个包的时间距离很小的时候，认为这是一个ddos攻击，直接丢包，否则让其通过。（这是一个极度的简化例子，真实的ddos防御工具要复杂得多）

> 注意需要在内核4.6以上系统测试，笔者使用ubuntu 18.04

```js
#!/usr/bin/python
#
# xdp_drop_count.py Drop incoming packets on XDP layer and count for which
#                   protocol type
#
# Copyright (c) 2016 PLUMgrid
# Copyright (c) 2016 Jan Ruth
# Licensed under the Apache License, Version 2.0 (the "License")

from bcc import BPF
import pyroute2
import time
import sys

flags = 0
def usage():
    print("Usage: {0} [-S] <ifdev>".format(sys.argv[0]))
    print("       -S: use skb mode\n")
    print("e.g.: {0} eth0\n".format(sys.argv[0]))
    exit(1)

if len(sys.argv) < 2 or len(sys.argv) > 3:
    usage()

if len(sys.argv) == 2:
    device = sys.argv[1]

if len(sys.argv) == 3:
    if "-S" in sys.argv:
        # XDP_FLAGS_SKB_MODE
        flags |= 2 << 0

    if "-S" == sys.argv[1]:
        device = sys.argv[2]
    else:
        device = sys.argv[1]

mode = BPF.XDP
ctxtype = "xdp_md"

# load BPF program
b = BPF(text = """
#define KBUILD_MODNAME "foo"
#include <uapi/linux/bpf.h>
#include <linux/in.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h>
#include <linux/if_vlan.h>
#include <linux/ip.h>
#include <linux/ipv6.h>

// how to determin ddos
#define MAX_NB_PACKETS 1000
#define LEGAL_DIFF_TIMESTAMP_PACKETS 1000000

// store data, data can be accessd in kernel and user namespace
BPF_HASH(rcv_packets);
BPF_TABLE("percpu_array", uint32_t, long, dropcnt, 256);

static inline int parse_ipv4(void *data, u64 nh_off, void *data_end) {
    struct iphdr *iph = data + nh_off;
    if ((void*)&iph[1] > data_end)
        return 0;
    return iph->protocol;
}
static inline int parse_ipv6(void *data, u64 nh_off, void *data_end) {
    struct ipv6hdr *ip6h = data + nh_off;
    if ((void*)&ip6h[1] > data_end)
        return 0;
    return ip6h->nexthdr;
}

// determine ddos
static inline int detect_ddos(){
    // Used to count number of received packets
    u64 rcv_packets_nb_index = 0, rcv_packets_nb_inter=1, *rcv_packets_nb_ptr;
    // Used to measure elapsed time between 2 successive received packets
    u64 rcv_packets_ts_index = 1, rcv_packets_ts_inter=0, *rcv_packets_ts_ptr;
    int ret = 0;
    
    rcv_packets_nb_ptr = rcv_packets.lookup(&rcv_packets_nb_index);
    rcv_packets_ts_ptr = rcv_packets.lookup(&rcv_packets_ts_index);
    if(rcv_packets_nb_ptr != 0 && rcv_packets_ts_ptr != 0){
        rcv_packets_nb_inter = *rcv_packets_nb_ptr;
        rcv_packets_ts_inter = bpf_ktime_get_ns() - *rcv_packets_ts_ptr;
        if(rcv_packets_ts_inter < LEGAL_DIFF_TIMESTAMP_PACKETS){
            rcv_packets_nb_inter++;
        } else {
            rcv_packets_nb_inter = 0;
        }
        if(rcv_packets_nb_inter > MAX_NB_PACKETS){
            ret = 1;
        }
    }
    rcv_packets_ts_inter = bpf_ktime_get_ns();
    rcv_packets.update(&rcv_packets_nb_index, &rcv_packets_nb_inter);
    rcv_packets.update(&rcv_packets_ts_index, &rcv_packets_ts_inter);
    return ret;
}

// determine and recode by proto
int xdp_prog1(struct CTXTYPE *ctx) {
    void* data_end = (void*)(long)ctx->data_end;
    void* data = (void*)(long)ctx->data;
    struct ethhdr *eth = data;
    // drop packets
    int rc = XDP_PASS; // let pass XDP_PASS or redirect to tx via XDP_TX
    long *value;
    uint16_t h_proto;
    uint64_t nh_off = 0;
    uint32_t index;
    nh_off = sizeof(*eth);
    if (data + nh_off  > data_end)
        return rc;
    h_proto = eth->h_proto;
    // parse double vlans
    if (detect_ddos() == 0){
        return rc;
    }
    rc = XDP_DROP;
    #pragma unroll
    for (int i=0; i<2; i++) {
        if (h_proto == htons(ETH_P_8021Q) || h_proto == htons(ETH_P_8021AD)) {
            struct vlan_hdr *vhdr;
            vhdr = data + nh_off;
            nh_off += sizeof(struct vlan_hdr);
            if (data + nh_off > data_end)
                return rc;
                h_proto = vhdr->h_vlan_encapsulated_proto;
        }
    }
    if (h_proto == htons(ETH_P_IP))
        index = parse_ipv4(data, nh_off, data_end);
    else if (h_proto == htons(ETH_P_IPV6))
       index = parse_ipv6(data, nh_off, data_end);
    else
        index = 0;
    value = dropcnt.lookup(&index);
    if (value)
        *value += 1;
    return rc;
}
""", cflags=["-w", "-DCTXTYPE=%s" % ctxtype])

fn = b.load_func("xdp_prog1", mode)
b.attach_xdp(device, fn, flags)


dropcnt = b.get_table("dropcnt")
prev = [0] * 256
print("Printing drops per IP protocol-number, hit CTRL+C to stop")
while 1:
    try:
        for k in dropcnt.keys():
            val = dropcnt.sum(k).value
            i = k.value
            if val:
                delta = val - prev[i]
                prev[i] = val
                print("{}: {} pkt/s".format(i, delta))
        time.sleep(1)
    except KeyboardInterrupt:
        print("Removing filter from device")
        break;

b.remove_xdp(device, flags)
```

运行命令, 测试

```js
# 运行 ddos_firewall
/usr/local/share/bcc/examples/networking/xdp/ddos_firewall.py -S lo

# 在另一个窗口用ab测试；本地80有一个正在运行的nginx
ab -n 1000 -c 10 "http://127.0.0.1/"

# 发现有部分请求被判定成了ddos, 平均耗时3.3ms (不开防火墙耗时约1ms)
6: 28 pkt/s
6: 27 pkt/s
6: 26 pkt/s

# 使用hping3测试更准确
# 使用 --fast测试正常，并无drop
hping3 -c 10000 -d 120 -S -w 64 -p 80 --fast --rand-source localhost

# 使用 --faster测试，出现大量drop
hping3 -c 10000 -d 120 -S -w 64 -p 80 --faster --rand-source localhost
```

参考:

- https://tonydeng.github.io/sdn-handbook 
- https://blog.csdn.net/dog250/article/details/77993218
- https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/
- https://kuaibao.qq.com/s/20180420A06HIB00?refer=spider
- https://www.ibm.com/developerworks/cn/linux/l-lo-eBPF-history/index.html
- [https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2019/02/25/ebpf-introduction-1.html](https://www.lijiaocn.com/技巧/2019/02/25/ebpf-introduction-1.html)