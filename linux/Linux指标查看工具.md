#                          Linux指标查看工具
> 本文笔记来自：「极客时间  Linux 性能优化实战」，原文链接：https://time.geekbang.org/column/article/80898

## 性能工具速查


**在选择性能工具时，除了要考虑性能指标这个目的外，还要结合待分析的环境来综合考虑**。比如，实际环境是否允许安装软件包，是否需要新的内核版本等。

性能工具谱图如下：

![img](https://static001.geekbang.org/resource/image/b0/01/b07ca95ef8a3d2c89b0996a042d33901.png?wh=3000*2100)

（图片来自 [brendangregg.com](http://www.brendangregg.com/linuxperf.html)）

这张图从 Linux 内核的各个子系统出发，汇总了对各个子系统进行性能分析时，你可以选择的工具。

**从性能指标出发，根据性能指标的不同，将性能工具划分为不同类型**。比如，最常见的就是可以根据 CPU、内存、磁盘 I/O 以及网络的各类性能指标，将这些工具进行分类。

## CPU性能工具

首先，从 CPU 的角度来说，主要的性能指标就是 CPU 的使用率、上下文切换以及 CPU Cache 的命中率等。下面这张图就列出了常见的 CPU 性能指标。

![img](https://static001.geekbang.org/resource/image/9a/69/9a211905538faffb5b3221ee01776a69.png?wh=1241*1212)

从这些指标出发，再把 CPU 使用率，划分为系统和进程两个维度，工具表如下：

![img](https://static001.geekbang.org/resource/image/28/b0/28cb85011289f83804c51c1fb275dab0.png?wh=1707*2563)

## 内存性能工具

接着我们来看内存方面。从内存的角度来说，主要的性能指标，就是系统内存的分配和使用、进程内存的分配和使用以及 SWAP 的用量。下面这张图列出了常见的内存性能指标。

![img](https://static001.geekbang.org/resource/image/ee/c0/ee36f73b9213063b3bcdaed2245944c0.png?wh=1581*1760)

内存性能工具速查表如下：

![img](https://static001.geekbang.org/resource/image/79/f8/79ad5caf0a2c105b7e9ce77877d493f8.png?wh=1653*2198)

注：最后一行pcstat的源码链接为 [https://github.com/tobert/pcstat](https://github.com/tobert/pcstat)

## 磁盘I/O性能工具

接下来，从文件系统和磁盘 I/O 的角度来说，主要性能指标，就是文件系统的使用、缓存和缓冲区的使用，以及磁盘 I/O 的使用率、吞吐量和延迟等。下面这张图列出了常见的 I/O 性能指标。

![img](https://static001.geekbang.org/resource/image/72/3b/723431a944034b51a9ef13a8a1d4d03b.png?wh=2631*808)

文件系统和磁盘 I/O 性能工具速查表如下：

![img](https://static001.geekbang.org/resource/image/c2/a3/c232dcb4185f7b7ba95c126889cf6fa3.png?wh=1714*2424)

## 网络性能工具

最后，从网络的角度来说，主要性能指标就是吞吐量、响应时间、连接数、丢包数等。根据 TCP/IP 网络协议栈的原理，我们可以把这些性能指标，进一步细化为每层协议的具体指标。

![img](https://static001.geekbang.org/resource/image/37/a4/37d04c213acfa650bd7467e3000356a4.png?wh=1983*1104)

网络性能工具速查表如下：

![img](https://static001.geekbang.org/resource/image/5d/5d/5dde213baffd7811ab73c82883b2a75d.png?wh=1709*2462)

## 基准测试工具

除了性能分析外，很多时候，我们还需要对系统性能进行基准测试。比如，

- 在文件系统和磁盘 I/O 模块中，我们使用 fio 工具，测试了磁盘 I/O 的性能。

- 在网络模块中，我们使用 iperf、pktgen 等，测试了网络的性能。

- 而在很多基于 Nginx 的案例中，我们则使用 ab、wrk 等，测试 Nginx 应用的性能。


Brendan Gregg 整理的 Linux 基准测试工具图谱如下：

![img](https://static001.geekbang.org/resource/image/f0/e9/f094f489049602e1058e02edc708e6e9.png?wh=1500*1050)

（图片来自 [brendangregg.com](http://www.brendangregg.com/linuxperf.html)）

## 小结

常见的性能工具，并从 CPU、内存、文件系统和磁盘 I/O、网络以及基准测试等不同的角度，汇总了各类性能指标所对应的性能工具速查表。

当分析性能问题时，大的来说，主要有这么两个步骤：

- 第一步，从性能瓶颈出发，根据系统和应用程序的运行原理，确认待分析的性能指标。

- 第二步，根据这些图表，选出最合适的性能工具，然后了解并使用工具，从而更快观测到需要的性能数据。


虽然 Linux 的性能指标和性能工具都比较多，但熟悉了各指标含义后，自然就会发现这些工具同性能指标间的关联。顺着这个思路往下走，掌握这些工具的选用其实并不难。

当然，不要把性能工具当成性能分析和优化的全部。

- 一方面，性能分析和优化的核心，是对系统和应用程序运行原理的掌握，而性能工具只是辅助你更快完成这个过程的帮手。

- 另一方面，完善的监控系统，可以提供绝大部分性能分析所需的基准数据。从这些数据中，你很可能就能大致定位出性能瓶颈，也就不用再去手动执行各类工具了。
