# golang项目可用性问题排查

> 转载自**叶性敏**的[《竟然是"你"偷走了那0.001的服务可用性》](https://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247494890&idx=1&sn=8a0755b38de9b6a1ea6074eff4c2c0c7&chksm=cf2f29cff858a0d9b3c0721ef312e619f913011239521d59b190ffaecb938d1148a09448f4da&scene=21#wechat_redirect)

# **背景**

前段时间同学反映我们活动项目某个服务可用性出现抖动，偶尔低于0.999。虽然看起来3个9的可用性相当高，但是对于一个 10w+ qps 的服务来讲，影响面就会被放大到不可接受的状态。最大的影响就是调用预约接口在流量高峰期有不少超时。预约接口是一个qps相对高的接口，超时就显得更加严重，而且这个问题有一段时间，所以需要尽快排查清楚原因，进行解决。服务可用性监控如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4jKiaz7ic5eFAfwdCD28XpIlz1rHpibAlvEDPvMw0PyIUXFicQvE2fRBMIQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个服务承载了很多活动逻辑，用了很多技术栈如redis,mc,mysql,taishan,es,mongodb,databus等，是一个特大单体。所以业务与组件的复杂给排查问题带来不少挑战。

## **猜想与否定**

了解基本情况后，知道可用性降低是由于超时导致，非其他错误。进行简要分析，能够想出一些可能的原因，例如某些业务写法导致性能问题，异常流量，系统调用，网络问题，cpu throttle，中间件问题（如redis,mysql），go调度，gc问题等。至于是这8名嫌疑犯还是另有其人，需要结合实际现象进行排除与论证，所以需要针对性的收集一些线索。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx45u3aeicwvs40ia0fYNuAFSBOYxusFuTC6fWDGDgX86xJ6dsB96Xc6jrA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4Szcee5FtIbW4deMFo4kFALJibhJQ9pf0eXZ6oPMdEytMZxl3VDGOFSg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4R9BMiaubGFACYuPo9QsxyUjibcD29Hxa1o2J5UPuuz526Jcyh36XOibxw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上图可以看出，这段时间流量比较规律性，并没有出现异常波动，似乎这个问题与流量没什么直接关系（背景中提到上游反馈高峰时段超时,可能只是高峰期放大现象），所以排除是异常流量导致的。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4oz3LHBQELwGnEicvv5QbHdaD9icoJaY2PEjAKnicuTld62Asr64zZu8UQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看出很多接口的999分位都有同样的问题。如果只是某个业务写法有问题，仅仅影响该业务的接口。或者可能是业务写法有问题，影响了runtime，那就具体再看runtime的表现，所以当时并没有深入看业务代码。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4QnpPD9E8Wmges6rQicibg5dETWQyKziciaXswqGsUjqwlWg3ibgb2siaCdrw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其实cpu throttle并不高，也问过运维，没啥异常，不太像是导致超时的原因。中间有个小插曲：当时有同学从cpu throttle导致超时这个猜想出发，发现预约业务内存cache占用量比较大（占用大的话可能影响内存的分配与释放），尝试减少预约业务内存cache占用量。观察一段时间，cpu throttle稍微有点降低，但可用性问题依然没有好转。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4ya3qCpxccF3aZTdY6Tfh5GduOBLs7caEIUeQAZEHWHwnxkDwib8t1VQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4tZLQ3MApgvqTKHVUq6yxqACsRTEsDsNLubI8UUIAicOlZ4RKgj4WOKQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4iaxnO4jIIeOWqm0Arx9oQs4wbOzFFGMyiaSIIzSibIyOOeKWlQr6IN0WQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4YicHtosn87DQTgLVtGLaO1lIKQSEyBulEMMzIHIyonLZIfqfr7mvmRg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

后续通过查看trace，发现段时间mysql与redis均有超时，细看给mysql的查询时间只有0.01ms，mysql说这锅我不背的。

那redis层呢，给了21.45ms，似乎给比较充足的查询时间，且redis有毛刺（不过毛刺时间点与可用性抖动点对不上），但是redis内心一万个不服啊。那行我们找对应时间段，再多看几个超时请求的trace，发现这些请求给redis的查询时间都比较短，如下图：

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4KbEz2W0RWZQzakpJrianJNfJCT7LMqTyA4qqmm8QRQOuDxXyeppic7Wg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

好吧，redis也可以鼓起勇气说，这锅我也不背。

其他组件也同样看过，都没啥异常。那么问题来了，组件们都不背，那到底谁来背？那网络，系统调用，go调度与gc，你们自己看着办吧。

网络说你再仔细想想，不对啊，一个请求至少给了250ms的time_quota，你们最后只留给我和组件们那么点时间，redis 0.04ms，mysql 0.01ms。请问扣除这点时间，剩余是谁“消费”了，应该心知肚明了吧。说到这，go调度，系统调用与gc主动跳出来了。



# **排查思路**

现在的矛头指向go runtime与系统调用。根据以往的经验有以下几种主要手段辅助定位：

1. 采集pprof，用cpu profiler查看cpu占用，memory profiler查看gc问题
2. 开启GODEBUG=gctrace=1 ，可查看程序运行过程中的GC信息。如果觉得是gc问题，可查看服务可用性抖动时间段gctrace是否异常，进一步确认问题
3. 添加fgprof，辅助排查off-cpu可能性，off-cpu例如I/O、锁、计时器、页交换等，具体详看鸟窝大佬文章：分析Go程序的Off-CPU性能（*https://colobu.com/2020/11/12/analyze-On-CPU-in-go/*）
4. 采集go trace，查看go调度问题，例如gc、抢占调度等，真实反映当时调度情况
5. linux strace查看go运行时调用了哪些系统调用导致超时



# **分析**

## **gctrace分析**

根据以往一些经验，gc有时候会影响系统的延时，所以先用gctrace看看这块有没有问题。



![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4Fcl5b6mzRuqnicOtYibyNUzicQiahFjQCwPnysLczibdiaIibDkSasbcia90CA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从gctrace上可以看出，并发标记和扫描的时间占用了860ms（图中红色框0.8+860+0.0668 ms中的860，一般gc问题通常看这块区域），并发标记和扫描比较占用cpu时间，这样可能导致这段时间大多数cpu时间用于扫描，业务协程只能延迟被调度。后续与可用性未抖动时间段的gctrace对比，发现并发标记与扫描时间并没有明显增多，但从860ms的时长上看，感觉也不是很合理，但没有证据证明它就能够导致超时抖动，这块异常先记着。



##  **strace分析**

并未发现异常，都是一些正常的系统调用，看不出有明显系统导致goroutine超时，所以"系统调用"这个嫌疑也暂时排除。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4hShaso8ib1LFOgEugwqUHHwdGw8Lk7aXKO616iaKPI4ZDPv2LgyyHWRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4tFYrZCTNlZmPKyODEf2EXX9lQYyNLcFKekLeQY1BHhaewYNBic1cFqg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





## **fgprof分析**

未见异常，结论同strace一样，未出现off-cpu的协程堵塞。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4PwbaiauIJjchSr6wicPXOeqL9T8Lxr46f6vGIluUy6A0ibLt6q5mIumXw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##  **go trace分析**

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4CmERUgAQSNVFor9SBiczZOGRFRCuNk8ItCkrySdTE7j59nfZ0mXWk0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4kycBVr3EGsdMgJBS4szL7lKxJxiajqxUj5R9ic2D9uC8bglG8lOMNLdQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



尝试多次(超过20次)抓取go trace文件进行分析。从trace文件上，可以明显看到发生MARK ASSIST了，顿时心中有谱。多抓trace看看，还是有明显的MARK ASSIST现象，gc问题应该比较明显了。关于MARK ASSIST现象（**如果在垃圾收集过程中，P1 在堆内存达到极限之前无法完成标记工作（因为应用程序可能在大量分配内存），应用程序 `goroutine` 成为 `Mark Assist`（协助标记）中的时间长度与它申请的堆内存成正比。`Mark Assist` 有助于更快地完成垃圾收集。**）

## **go heap 分析**

如果是gc问题，那就和heap息息相关了。抓一段低峰期的heap，进行分析。

**inuse_space：**

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4tRtb3CMWN9o0XPzViat6bn9mxt4GdHyhJnkzx4zsc1sogZdD3kHAEYw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



可见grpc连接占用了很大的一块内存，但这其实一般不太直接影响gc标记和扫描，主要影响内存的释放。gc标记和扫描还得看inuse_objects。

**inuse_objects：**

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx45DRjicwKyzQc9kNR1gdc0IZDvtWWKKW89LnOWkLFdAGX8KtHS5DlYvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



可以看到gcache中LFU生产的object数量高达100w+，而总共的object才300w。这块明显有问题，那很可能就是它导致的问题。



# **解决**

我们找到最大的嫌疑-gcache（该包引入项目已一年多）。看了一下业务中使用gcache的情况及LFU的使用处



![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx4GWBxInwmxFkL9dIIJewKxHv6bJqNvXiaRfRVcPedwkM5vwyk4yBqnCA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



从用法上，未发现有什么问题。便把问题转向gcache包本身。百度，google一顿搜索，源码浅看了一下，也没发现异常。github.com issue一搜，发现有人提过类似问题*https://github.com/bluele/gcache/issues/71*。gcache LFU策略的Get方法存在内存泄露（内存大概泄露100M，占总内存占用量2.5%，主要是产生大量指针对象）。具体bug是如何产生的，由于篇幅原因，这里不进行赘述，可参考issue（*https://github.com/bluele/gcache/issues/71*）。后续将gcahce版本从v0.0.1升级至v0.0.2，该问题得以解决。

gcache竟然是你啊，偷走了我0.001的服务可用性。

![Image](https://mmbiz.qpic.cn/mmbiz_png/1BMf5Ir754R6wJEh2mzA360Ro8iaxLMx48mzyudLRto8sygaOBg9jU4W4xXsc5qTuX8kBvC4RywzgydSRBXXqcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



# **总结**

至此问题排查与解决都告一段落了，但有些问题还需总结与复盘。

从上面看，你可能想这问题好像也很容易就排查到了，实际排查过程中并不顺利。在进行trace及heap分析时也是抓取了很多次，才抓到有效信息。后面发现某些gcache的过期时间只有5分钟，过了这5分钟，现场就没了（如果能有自动抓取能力，那该多方便），这让怀疑是gc问题的我们一段时间在自我肯定与否定中。中间产生更多猜想，例如怀疑定时器使用过多(业务代码里面比较多后台刷新配置的定时器)，导致业务逻辑调度延迟；grpc客户端连接过多(2w+)，占用较大内存，产生较多对象，影响gc；也有猜测是机器问题；常驻内存cache过多，内部指针较多，影响gc扫描；甚至想用go ballast 丝滑的控制内存等。

关于系统稳定性这块的小启示：

1. 第三方库的引入还需慎重。本次问题是第三方包bug导致的，所以引入包时需要考虑合理性，避免引入不稳定因素。第三方包的安全漏洞问题大家一般比较重视，bug却常常被忽视。可制定第三方包的引入标准、编写工具监测第三方包issue的提出与解决，通知开发人员评估风险及更新包版本，从而做到第三方包的合理引入，快速发现issue以及修复。
2. 关于系统稳定性这块，基本上都是尽可能的添加监控(包括系统，组件，go runtime维度)，通过报警及时发现问题或隐患。至于go程序运行时内部的现场？似乎只能出问题时在容器内部或者借助公司自研平台手动抓取pprof，但多少存在一定的滞后性，有时候甚至现场都没了。当然也有人定时抓取pprof，但多少有点影响性能，姿势也不够优雅。holmes(MOSN 社区性能分析利器)就显得很有必要。通过相应的策略，例如协程暴涨，cpu异常升高，内存OOM等情况，自动抓取，达到近似"无人值守"。既能够较好的保留事故现场，也能提前暴漏一些隐患。



# **参考文献**

- <<分析Go程序的Off-CPU性能>> *https://colobu.com/2020/11/12/analyze-On-CPU-in-go/*
- <<Holmes 原理浅析>> *https://mosn.io/blog/posts/mosn-holmes-design/*
- holmes github：*https://github.com/mosn/holmes*
- gcache github: *https://github.com/bluele/gcache*
- <<深度解密Go语言之 pprof>> *https://www.cnblogs.com/qcrao-2018/p/11832732.html*