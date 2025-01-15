# 详解Golang内存管理

> 主要内容转载自[一站式Golang内存洗髓经](https://learnku.com/articles/68142#replies)，阅读之前建议先了解GMP原理：https://github.com/longpi1/Reading-notes/blob/main/golang/Golang%E7%9A%84%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6%E5%99%A8%E5%8E%9F%E7%90%86%E5%8F%8AGMP%E8%AE%BE%E8%AE%A1%E6%80%9D%E6%83%B3.md

在了解Golang的内存管理之前，一定要了解的基本申请内存模式，即TCMalloc（Thread Cache Malloc）。Golang的内存管理就是基于TCMalloc的核心思想来构建的。本节将介绍TCMalloc的基础理念和结构。

### 1.1 TCMalloc

TCMalloc最大优势就是每个线程都会独立维护自己的内存池。在之前章节介绍的自定义实现的Golang内存池版BufPool实则是所有Goroutine或者所有线程共享的内存池，其关系如图1所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651132777839-f6077cf7-f8e4-40d0-9fa0-1167208508da.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_58%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 1 BufPool内存池与线程Thread的关系

这种内存池的设计缺点显而易见，应用方全部的内存申请均需要和全局的BufPool交互，为了线程的并发安全，那么频繁的BufPool的内存申请和退还需要加互斥和同步机制，影响了内存的使用的性能。

TCMalloc则是为每个Thread预分配一块缓存，每个Thread在申请内存时首先会先从这个缓存区ThreadCache申请，且所有ThreadCache缓存区还共享一个叫CentralCache的中心缓存。这里假设目前Golang的内存管理用的是原生TCMalloc模式，那么线程与内存的关系将如图2所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651132869540-a130e8b3-1f7d-45ec-8413-52bba81426a0.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_57%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 2 TCMalloc内存池与线程Thread的关系

这样做的好处其一是ThreadCache做为每个线程独立的缓存，能够明显的提高Thread获取高命中的数据，其二是ThreadCache也是从堆空间一次性申请，即只触发一次系统调用即可。每个ThreadCache还会共同访问CentralCache，这个与BufPool的类似，但是设计更为精细一些。CentralCache是所有线程共享的缓存，当ThreadCache的缓存不足时，就会从CentralCache获取，当ThreadCache的缓存充足或者过多时，则会将内存退还给CentralCache。但是CentralCache由于共享，那么访问一定是需要加锁的。ThreadCache作为线程独立的第一交互内存，访问无需加锁，CentralCache则作为ThreadCache临时补充缓存。

TCMalloc的构造不仅于此，提供了ThreadCache和CentralCache可以解决小对象内存块的申请，但是对于大块内存Cache显然是不适合的。       TCMalloc将内存分为三类，如表1所示。

###### 表1 TCMalloc的内存分离

| **对象** | **容量**     |
| -------- | ------------ |
| 小对象   | (0,256KB]    |
| 中对象   | (256KB, 1MB] |
| 大对象   | (1MB, +∞)    |

所以为了解决中对象和大对象的内存申请，TCMalloc依然有一个全局共享内存堆PageHeap，如图3所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133032054-ea888b96-0fb0-46ea-ac26-4c38abc2b66f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_89%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 3 TCMalloc中的PageHeap

PageHeap也是一次系统调用从虚拟内存中申请的，PageHeap很明显是全局的，所以访问一定是要加锁。其作用是当CentralCache没有足够内存时会从PageHeap取，当CentralCache内存过多或者充足，则将低命中内存块退还PageHeap。如果Thread需要大对象申请超过的Cache容纳的内存块单元大小，也会直接从PageHeap获取。

### 1.2 TCMalloc模型相关基础结构

在了解TCMalloc的一些内部设计结构时，首要了解的是一些TCMalloc定义的基本名词Page、Span和Size Class。

#### 1.Page

TCMalloc中的Page与之前章节介绍操作系统对虚拟内存管理的MMU定义的物理页有相似的定义，TCMalloc将虚拟内存空间划分为多份同等大小的Page，每个Page默认是8KB。

对于TCMalloc来说，虚拟内存空间的全部内存都按照Page的容量分成均等份，并且给每份Page标记了ID编号，如图4所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133095495-53138cb4-89b8-4833-ac41-7957a1c19354.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_82%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 4 TCMalloc将虚拟内存平均分层N份Page

将Page进行编号的好处是，可以根据任意内存的地址指针，进行固定算法偏移计算来算出所在的Page。

#### 2.Span

多个连续的Page称之为是一个Span，其定义含义有操作系统的管理的页表相似，Page和Span的关系如图5所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133180209-09abdb85-cf15-40d4-8e7c-c9acb973e107.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_81%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 5 TCMalloc中Page与Span的关系

TCMalloc是以Span为单位向操作系统申请内存的。每个Span记录了第一个起始Page的编号Start，和一共有多少个连续Page的数量Length。

为了方便Span和Span之间的管理，Span集合是以双向链表的形式构建，如图6所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133255704-7c07cb59-d879-468f-a925-d3494454cb7d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_82%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 6 TCMalloc中Span存储形式

#### 3.Size Class

参考表3-3所示，在256KB以内的小对象，TCMalloc会将这些小对象集合划分成多个内存刻度[[6\]](#_ftn6)，同属于一个刻度类别下的内存集合称之为属于一个Size Class。这与之前章节自定义实现的内存池，将Buf划分多个刻度的BufList类似。

每个Size Class都对应一个大小比如8字节、16字节、32字节等。在申请小对象内存的时候，TCMalloc会根据使用方申请的空间大小就近向上取最接近的一个Size Class的Span（由多个等空间的Page组成）内存块返回给使用方。

如果将Size Class、Span、Page用一张图来表示，则具体的抽象关系如图7所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133299709-8c33bad3-a31f-4844-b07c-ad54a0dc64d4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_67%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 7 TCMalloc中Size Class、Page、Span的结构关系

接下来剖析一下ThreadCache、CentralCache、PageHeap的内存管理结构。

### 1.3 ThreadCache

在TCMalloc中每个线程都会有一份单独的缓存，就是ThreadCache。ThreadCache中对于每个Size Class都会有一个对应的FreeList，FreeList表示当前缓存中还有多少个空闲的内存可用，具体的结构布局如图8所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133403346-3d07b578-45df-41b1-880e-d1a591d106ff.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_97%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 8 TCMalloc中ThreadCache

使用方对于从TCMalloc申请的小对象，会直接从TreadCache获取，实则是从FreeList中返回一个空闲的对象，如果对应的Size Class刻度下已经没有空闲的Span可以被获取了，则ThreadCache会从CentralCache中获取。当使用方使用完内存之后，归还也是直接归还给当前的ThreadCache中对应刻度下的的FreeList中。

整条申请和归还的流程是不需要加锁的，因为ThreadCache为当前线程独享，但如果ThreadCache不够用，需要从CentralCache申请内存时，这个动作是需要加锁的。不同Thread之间的ThreadCache是以双向链表的结构进行关联，是为了方便TCMalloc统计和管理。

### 1.4 CentralCache

CentralCache是各个线程共用的，所以与CentralCache获取内存交互是需要加锁的。CentralCache缓存的Size Class和ThreadCache的一样，这些缓存都被放在CentralFreeList中，当ThreadCache中的某个Size Class刻度下的缓存小对象不够用，就会向CentralCache对应的Size Class刻度的CentralFreeList获取，同样的如果ThreadCache有多余的缓存对象也会退还给响应的CentralFreeList，流程和关系如图9所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133530033-bd9265dc-fd49-4a77-a845-f175ab317ea9.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_97%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 9 TCMalloc中CentralCache

CentralCache与PageHeap的角色关系与ThreadCache与CentralCache的角色关系相似，当CentralCache出现Span不足时，会从PageHeap申请Span，以及将不再使用的Span退还给PageHeap。

### 1.5 PageHeap

PageHeap是提供CentralCache的内存来源。PageHead与CentralCache不同的是CentralCache是与ThreadCache布局一模一样的缓存，主要是起到针对ThreadCache的一层二级缓存作用，且只支持小对象内存分配。而PageHeap则是针对CentralCache的三级缓存。弥补对于中对象内存和大对象内存的分配，PageHeap也是直接和操作系统虚拟内存衔接的一层缓存，当ThreadCache、CentralCache、PageHeap都找不到合适的Span，PageHeap则会调用操作系统内存申请系统调用函数来从虚拟内存的堆区中取出内存填充到PageHeap当中，具体的结构如图10所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133596465-fd16a3cc-256a-464c-a066-b896043a9f63.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_101%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 10 TCMalloc中PageHeap

PageHeap内部的Span管理，采用两种不同的方式，对于128个Page以内的Span申请，每个Page刻度都会用一个链表形式的缓存来存储。对于128个Page以上内存申请，PageHeap是以有序集合（C++标准库STL中的Std::Set容器）来存放。

### 1.6 TCMalloc的小对象分配

上述已经将TCMalloc的几种基础结构介绍了，接下来总结一下TCMalloc针对小对象、中对象和大对象的分配流程。小对象分配流程如图11所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133672724-0ac13b26-1623-444a-8c81-0b2120b2e2fa.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_80%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 11 TCMalloc小对象分配流程

小对象为占用内存小于等于256KB的内存，参考图中的流程，下面将介绍详细流程步骤：

（1）Thread用户线程应用逻辑申请内存，当前Thread访问对应的ThreadCache获取内存，此过程不需要加锁。

（2）ThreadCache的得到申请内存的SizeClass（一般向上取整，大于等于申请的内存大小），通过SizeClass索引去请求自身对应的FreeList。

（3）判断得到的FreeList是否为非空。

（4）如果FreeList非空，则表示目前有对应内存空间供Thread使用，得到FreeList第一个空闲Span返回给Thread用户逻辑，流程结束。

（5）如果FreeList为空，则表示目前没有对应SizeClass的空闲Span可使用，请求CentralCache并告知CentralCache具体的SizeClass。

（6）CentralCache收到请求后，加锁访问CentralFreeList，根据SizeClass进行索引找到对应的CentralFreeList。

（7）判断得到的CentralFreeList是否为非空。

（8）如果CentralFreeList非空，则表示目前有空闲的Span可使用。返回多个Span，将这些Span（除了第一个Span）放置ThreadCache的FreeList中，并且将第一个Span返回给Thread用户逻辑，流程结束。

（9）如果CentralFreeList为空，则表示目前没有可用是Span可使用，向PageHeap申请对应大小的Span。

（10）PageHeap得到CentralCache的申请，加锁请求对应的Page刻度的Span链表。

（11）PageHeap将得到的Span根据本次流程请求的SizeClass大小为刻度进行拆分，分成N份SizeClass大小的Span返回给CentralCache，如果有多余的Span则放回PageHeap对应Page的Span链表中。

（12）CentralCache得到对应的N个Span，添加至CentralFreeList中，跳转至第（8）步。

综上是TCMalloc一次申请小对象的全部详细流程，接下来分析中对象的分配流程。

### 1.7 TCMalloc的中对象分配

中对象为大于256KB且小于等于1MB的内存。对于中对象申请分配的流程TCMalloc与处理小对象分配有一定的区别。对于中对象分配，Thread不再按照小对象的流程路径向ThreadCache获取，而是直接从PageHeap获取，具体的流程如图12所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133803693-486e9b4a-ffb1-4932-a989-1df013b601c1.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_71%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 12 TCMalloc中对象分配流程

PageHeap将128个Page以内大小的Span定义为小Span，将128个Page以上大小的Span定义为大Span。由于一个Page为8KB，那么128个Page即为1MB，所以对于中对象的申请，PageHeap均是按照小Span的申请流程，具体如下：

（1）Thread用户逻辑层提交内存申请处理，如果本次申请内存超过256KB但不超过1MB则属于中对象申请。TCMalloc将直接向PageHeap发起申请Span请求。

（2）PageHeap接收到申请后需要判断本次申请是否属于小Span（128个Page以内），如果是，则走小Span，即中对象申请流程，如果不是，则进入大对象申请流程，下一节介绍。

（3）PageHeap根据申请的Span在小Span的链表中向上取整，得到最适应的第K个Page刻度的Span链表。

（4）得到第K个Page链表刻度后，将K作为起始点，向下遍历找到第一个非空链表，直至128个Page刻度位置，找到则停止，将停止处的非空Span链表作为提供此次返回的内存Span，将链表中的第一个Span取出。如果找不到非空链表，则当错本次申请为大Span申请，则进入大对象申请流程。

（5）假设本次获取到的Span由N个Page组成。PageHeap将N个Page的Span拆分成两个Span，其中一个为K个Page组成的Span，作为本次内存申请的返回，给到Thread，另一个为N-K个Page组成的Span，重新插入到N-K个Page对应的Span链表中。

综上是TCMalloc对于中对象分配的详细流程。

### 1.8 TCMalloc的大对象分配

对于超过128个Page（即1MB）的内存分配则为大对象分配流程。大对象分配与中对象分配情况类似，Thread绕过ThreadCache和CentralCache，直接向PageHeap获取。详细的分配流程如图13所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133987470-28f3feb2-8a9e-45be-a41b-596b1bd54e8d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_81%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 13 TCMalloc大对象分配流程

进入大对象分配流程除了申请的Span大于128个Page之外，对于中对象分配如果找不到非空链表也会进入大对象分配流程，大对象分配的具体流程如下：

（1）Thread用户逻辑层提交内存申请处理，如果本次申请内存超过1MB则属于大对象申请。TCMalloc将直接向PageHeap发起申请Span      。

（2）PageHeap接收到申请后需要判断本次申请是否属于小Span（128个Page以内），如果是，则走小Span中对象申请流程（上一节已介绍），如果不是，则进入大对象申请流程。

（3）PageHeap根据Span的大小按照Page单元进行除法运算，向上取整，得到最接近Span的且大于Span的Page倍数K，此时的K应该是大于128。如果是从中对象流程分过来的（中对象申请流程可能没有非空链表提供Span），则K值应该小于128。

（4）搜索Large Span Set集合，找到不小于K个Page的最小Span（N个Page）。如果没有找到合适的Span，则说明PageHeap已经无法满足需求，则向操作系统虚拟内存的堆空间申请一堆内存，将申请到的内存安置在PageHeap的内存结构中，重新执行（3）步骤。

（5）将从Large Span Set集合得到的N个Page组成的Span拆分成两个Span，K个Page的Span直接返回给Thread用户逻辑，N-K个Span退还给PageHeap。其中如果N-K大于128则退还到Large Span Set集合中，如果N-K小于128，则退还到Page链表中。

综上是TCMalloc对于大对象分配的详细流程。

## 2 Golang堆内存管理

本章节将介绍Golang的内存管理模型，看本章节之前强烈建议读者将上述章节均阅读理解完成，更有助于理解Golang的内存管理机制。

### 2Golang内存管理模型的逻辑层次全景图，如图14所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134285363-999c7495-7834-4785-a6ea-c44b4615ff19.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_58%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 14 Golang内存管理模块关系

Golang内存管理模型与TCMalloc的设计极其相似。基本轮廓和概念也几乎相同，只是一些规则和流程存在差异，接下来分析一下Golang内存管理模型的基本层级模块组成概念。

### 2.2 Golang内存管理单元相关概念

Golang内存管理中依然保留TCMalloc中的Page、Span、Size Class等概念。

#### 1.Page

与TCMalloc的Page一致。Golang内存管理模型延续了TCMalloc的概念，一个Page的大小依然是8KB。Page表示Golang内存管理与虚拟内存交互内存的最小单元。操作系统虚拟内存对于Golang来说，依然是划分成等分的N个Page组成的一块大内存公共池，如图3.21所示。

#### 2.mSpan

与TCMalloc中的Span一致。mSpan概念依然延续TCMalloc中的Span概念，在Golang中将Span的名称改为mSpan，依然表示一组连续的Page。

#### 3.Size Class相关

Golang内存管理针对Size Class对衡量内存的的概念又更加详细了很多，这里面介绍一些基础的有关内存大小的名词及算法。

（1）Object Size，是只协程应用逻辑一次向Golang内存申请的对象Object大小。Object是Golang内存管理模块针对内存管理更加细化的内存管理单元。一个Span在初始化时会被分成多个Object。比如Object Size是8B（8字节）大小的Object，所属的Span大小是8KB（8192字节），那么这个Span就会被平均分割成1024（8192/8=1024）个Object。逻辑层向Golang内存模型取内存，实则是分配一个Object出去。为了更好的让读者理解，这里假设了几个数据来标识Object Size 和Span的关系，如图15所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134337384-3b5b18a9-63a2-41eb-89fb-d7a030b1e569.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_79%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 15 Object Size与Span的关系

上图中的Num Of Object表示当前Span中一共存在多少个Object。注意 Page是Golang内存管理与操作系统交互衡量内存容量的基本单元，Golang内存管理内部本身用来给对象存储内存的基本单元是Object。

（2）Size Class，Golang内存管理中的Size Class与TCMalloc所表示的设计含义是一致的，都表示一块内存的所属规格或者刻度。Golang内存管理中的Size Class是针对Object Size来划分内存的。也是划分Object大小的级别。比如Object Size在1Byte~8Byte之间的Object属于Size Class 1级别，Object Size 在8B~16Byte之间的属于Size Class 2级别。

（3）Span Class，这个是Golang内存管理额外定义的规格属性，是针对Span来进行划分的，是Span大小的级别。一个Size Class会对应两个Span Class，其中一个Span为存放需要GC扫描的对象（包含指针的对象），另一个Span为存放不需要GC扫描的对象（不包含指针的对象），具体Span Class与Size Class的逻辑结构关系如图16所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134377320-3f71752d-65fa-4081-a255-09c387f23a65.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_70%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 16 Span Class与Size Class的逻辑结构关系



其中Size Class和Span Class的对应关系计算方式可以参考Golang源代码，如下：

```
//usr/local/go/src/runtime/mheap.go

type spanClass uint8 

//……(省略部分代码)

func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}

//……(省略部分代码)
```

这里makeSpanClass()函数为通过Size Class来得到对应的Span Class，其中第二个形参noscan表示当前对象是否需要GC扫描，不难看出来Span Class 和Size Class的对应关系公式如表5所示。

###### 表5 TCMalloc的内存分离

| **对象**     | **Size Class** **与** **Span Class****对应公式** |
| ------------ | ------------------------------------------------ |
| 需要GC扫描   | Span Class = Size Class * 2 + 0                  |
| 不需要GC扫描 | Span Class = Size Class * 2 + 1                  |

#### 4.Size Class明细

如果再具体一些，则通过Golang的源码可以看到，Golang给内存池中固定划分了66[[7\]](#_ftn7)个Size Class，这里面列举了详细的Size Class和Object大小、存放Object数量，以及每个Size Class对应的Span内存大小关系，代码如下：

```
//usr/local/go/src/runtime/sizeclasses.go

package runtime

// 标题Title解释：
// [class]: Size Class
// [bytes/obj]: Object Size，一次对外提供内存Object的大小
// [bytes/span]: 当前Object所对应Span的内存大小
// [objects]: 当前Span一共有多少个Object
// [tail wastre]: 为当前Span平均分层N份Object，会有多少内存浪费
// [max waste]: 当前Size Class最大可能浪费的空间所占百分比

// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0        87.50%
//     2         16        8192      512           0        43.75%
//     3         32        8192      256           0        46.88%
//     4         48        8192      170          32        31.52%
//     5         64        8192      128           0        23.44%
//     6         80        8192      102          32        19.07%
//     7         96        8192       85          32        15.95%
//     8        112        8192       73          16        13.56%
//     9        128        8192       64           0        11.72%
//    10        144        8192       56         128        11.82%
//    11        160        8192       51          32        9.73%
//    12        176        8192       46          96        9.59%
//    13        192        8192       42         128        9.25%
//    14        208        8192       39          80        8.12%
//    15        224        8192       36         128        8.15%
//    16        240        8192       34          32        6.62%
//    17        256        8192       32           0        5.86%
//    18        288        8192       28         128        12.16%
//    19        320        8192       25         192        11.80%
//    20        352        8192       23          96        9.88%
//    21        384        8192       21         128        9.51%
//    22        416        8192       19         288        10.71%
//    23        448        8192       18         128        8.37%
//    24        480        8192       17          32        6.82%
//    25        512        8192       16           0        6.05%
//    26        576        8192       14         128        12.33%
//    27        640        8192       12         512        15.48%
//    28        704        8192       11         448        13.93%
//    29        768        8192       10         512        13.94%
//    30        896        8192        9         128        15.52%
//    31       1024        8192        8           0        12.40%
//    32       1152        8192        7         128        12.41%
//    33       1280        8192        6         512        15.55%
//    34       1408       16384       11         896        14.00%
//    35       1536        8192        5         512        14.00%
//    36       1792       16384        9         256        15.57%
//    37       2048        8192        4           0        12.45%
//    38       2304       16384        7         256       12.46%
//    39       2688        8192        3         128        15.59%
//    40       3072       24576        8           0        12.47%
//    41       3200       16384        5         384        6.22%
//    42       3456       24576        7         384        8.83%
//    43       4096        8192        2           0        15.60%
//    44       4864       24576        5         256        16.65%
//    45       5376       16384        3         256        10.92%
//    46       6144       24576        4           0        12.48%
//    47       6528       32768        5         128        6.23%
//    48       6784       40960        6         256        4.36%
//    49       6912       49152        7         768        3.37%
//    50       8192        8192        1           0        15.61%
//    51       9472       57344        6         512        14.28%
//    52       9728       49152        5         512        3.64%
//    53      10240       40960        4           0        4.99%
//    54      10880       32768        3         128        6.24%
//    55      12288       24576        2           0        11.45%
//    56      13568       40960        3         256        9.99%
//    57      14336       57344        4           0        5.35%
//    58      16384       16384        1           0        12.49%
//    59      18432       73728        4           0        11.11%
//    60      19072       57344        3         128        3.57%
//    61      20480       40960        2           0        6.87%
//    62      21760       65536        3         256        6.25%
//    63      24576       24576        1           0        11.45%
//    64      27264       81920        3         128        10.00%
//    65      28672       57344        2           0        4.91%
//    66      32768       32768        1           0        12.50%
```

下面分别解释一下每一列的含义：

（1）Class列为Size Class规格级别。

（2）bytes/obj列为Object Size，即一次对外提供内存Object的大小（单位为Byte），可能有一定的浪费，比如业务逻辑层需要2B的数据，实则会定位到Size Class为1，返回一个Object即8B的内存空间。

（3）bytes/span列为当前Object所对应Span的内存大小（单位为Byte）。

（4）objects列为当前Span一共有多少个Object，该字段是通过bytes/span和bytes/obj相除计算而来。

（5）tail waste列为当前Span平均分层N份Object，会有多少内存浪费，这个值是通过bytes/span对bytes/obj求余得出，即span%obj。

（6）max waste列当前Size Class最大可能浪费的空间所占百分比。这里面最大的情况就是一个Object保存的实际数据刚好是上一级Size Class的Object大小加上1B。当前Size Class的Object所保存的真实数据对象都是这一种情况，这些全部空间的浪费再加上最后的tail waste就是max waste最大浪费的内存百分比，具体如图20所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134515410-7716dd9f-cc6c-410e-954e-9c0819076f34.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_90%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 20 Max Waste最大浪费空间计算公式

图中以Size Class 为7的Span为例，通过源代码runtime/sizeclasses.go的详细Size Class数据可以得知具体Span细节如下：

```bash
// class  bytes/obj  bytes/span  objects  tail waste  max waste

// … …
//     6         80        8192      102          32        19.07%
//     7         96        8192       85          32        15.95%
// … …
```



从图20可以看出，Size Class为7的Span如果每个Object均超过Size Class为7中的Object一个字节。那么就会导致Size Class为7的Span出现最大空间浪费情况。综上可以得出计算最大浪费空间比例的算法公式如下：

```bash
(本级Object Size – (上级Object Size + 1)*本级Object数量) / 本级Span Size
```

### 2.3 MCache

从概念来讲MCache与TCMalloc的ThreadCache十分相似，访问mcache依然不需要加锁而是直接访问，且MCache中依然保存各种大小的Span。

虽然MCache与ThreadCache概念相似，二者还是存在一定的区别的，MCache是与Golang协程调度模型GPM中的P所绑定，而不是和线程绑定。因为Golang调度的GPM模型，真正可运行的线程M的数量与P的数量一致，即GOMAXPROCS个，所以MCache与P进行绑定更能节省内存空间使用，可以保证每个G使用MCache时不需要加锁就可以获取到内存。而TCMalloc中的ThreadCache随着Thread的增多，ThreadCache的数量也就相对成正比增多，二者绑定关系的区别如图21所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134611972-cc01f19f-d74a-40aa-b79c-a04e994f9824.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_94%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 21 ThreadCache与mcache的绑定关系区别

如果将图35的mcache展开，来看mcache的内部构造，则具体的结构形式如图22所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134640677-2a153c96-b7e8-46bc-86f3-dfaf50087329.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_84%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 22 MCache内部构造

协程逻辑层从mcache上获取内存是不需要加锁的，因为一个P只有一个M在其上运行，不可能出现竞争，由于没有锁限制，mcache则其到了加速内存分配。

MCache中每个Span Class都会对应一个MSpan，不同Span Class的MSpan的总体长度不同，参考runtime/sizeclasses.go的标准规定划分。比如对于Span Class为4的MSpan来说，存放内存大小为1Page，即8KB。每个对外提供的Object大小为16B，共存放512个Object。其他Span Class的存放方式类似。当其中某个Span Class的MSpan已经没有可提供的Object时，MCache则会向MCentral申请一个对应的MSpan。

在图3.36中应该会发现，对于Span Class为0和1的，也就是对应Size Class为0的规格刻度内存，mcache实际上是没有分配任何内存的。因为Golang内存管理对内存为0的数据申请做了特殊处理，如果申请的数据大小为0将直接返回一个固定内存地址，不会走Golang内存管理的正常逻辑，相关Golang源代码如下：

```
//usr/local/go/src/runtime/malloc.go

// Al Allocate an object of size bytes.                                     
// Sm Small objects are allocated from the per-P cache's free lists.        
// La Large objects (> 32 kB) are allocated straight from the heap.         
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {                        
// ……（省略部分代码）

if size == 0 {
return unsafe.Pointer(&zerobase)
}

//……（省略部分代码）
}
```

上述代码可以看见，如果申请的size为0，则直接return一个固定地址zerobase。下面来测试一下有关0空间申请的情况，在Golang中如[0]int、 struct{}所需要大小均是0，这也是为什么很多开发者在通过Channel做同步时，发送一个struct{}数据，因为不会申请任何内存，能够适当节省一部分内存空间，测试代码如下：

```
//第一篇/chapter3/MyGolang/zeroBase.go
package main

import (
"fmt"
)

func main() {
var (
//0内存对象
a struct{}
b [0]int

//100个0内存struct{}
c [100]struct{}

//100个0内存struct{},make申请形式
d = make([]struct{}, 100)
)

fmt.Printf("%p\n", &a)
fmt.Printf("%p\n", &b)
fmt.Printf("%p\n", &c[50])    //取任意元素
fmt.Printf("%p\n", &(d[50]))  //取任意元素
}
```

运行结果如下：

```
$ go run zeroBase.go 
0x11aac78
0x11aac78
0x11aac78
0x11aac78
```

从结果可以看出，全部的0内存对象分配，返回的都是一个固定的地址。

### 2.4 MCentral

MCentral与TCMalloc中的Central概念依然相似。向MCentral申请Span是同样是需要加锁的。当MCache中某个Size Class对应的Span被一次次Object被上层取走后，如果出现当前Size Class的Span空缺情况，MCache则会向MCentral申请对应的Span。Goroutine、MCache、MCentral、MHeap互相交换的内存单位是不同，具体如图23所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134740690-c1fc15a5-af2a-474c-adaa-ddc4bb05a5e3.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_89%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 23 Golang内存管理各层级内存交换单位

其中协程逻辑层与MCache的内存交换单位是Object，MCache与MCentral的内存交换单位是Span，而MCentral与MHeap的内存交换单位是Page。

MCentral与TCMalloc中的Central不同的是MCentral针对每个Span Class级别有两个Span链表，而TCMalloc中的Central只有一个。MCentral的内部构造如图24所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134816735-43c615ce-3c3c-485a-9ae2-e36dca963f95.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_88%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 24 MCentral的内部构造

MCentral与MCCache不同的是，每个级别保存的不是一个Span，而是一个Span List链表。与TCMalloc中的Central不同的是，MCentral每个级别都保存了两个Span List。



**注意 图24中MCentral是表示一层抽象的概念，实际上每个Span Class对应的内存数据结构是一个mcentral，即在MCentral这层数据管理中，实际上有Span Class个mcentral小内存管理单元。**



**1）NonEmpty Span List**

表示还有可用空间的Span链表。链表中的所有Span都至少有1个空闲的Object空间。如果MCentral上游MCache退还Span，会将退还的Span加入到NonEmpty Span List链表中。

**2）Empty Span List**

表示没有可用空间的Span链表。该链表上的Span都不确定否还有有空闲的Object空间。如果MCentral提供给一个Span给到上游MCache，那么被提供的Span就会加入到Empty List链表中。



**注意 在Golang 1.16版本之后，MCentral中的NonEmpty Span List 和 Empty Span List**

**均由链表管理改成集合管理，分别对应Partial Span Set 和 Full Span Set。虽然存储的数据结构有变化，但是基本的作用和职责没有区别。**



下面是MCentral层级中其中一个Size Class级别的MCentral的定义Golang源代码（V1.14版本）：

```
//usr/local/go/src/runtime/mcentral.go  , Go V1.14

// Central list of free objects of a given size.
// go:notinheap
type mcentral struct {
lock      mutex      //申请MCentral内存分配时需要加的锁

spanclass spanClass //当前哪个Size Class级别的

// list of spans with a free object, ie a nonempty free list
// 还有可用空间的Span 链表
nonempty  mSpanList 

// list of spans with no free objects (or cached in an mcache)
// 没有可用空间的Span链表，或者当前链表里的Span已经交给mcache
empty     mSpanList 

// nmalloc is the cumulative count of objects allocated from
// this mcentral, assuming all spans in mcaches are
// fully-allocated. Written atomically, read under STW.
// nmalloc是从该mcentral分配的对象的累积计数
// 假设mcaches中的所有跨度都已完全分配。
// 以原子方式书写，在STW下阅读。
nmalloc uint64
}
```

新版本的改进是将List变成了两个Set集合，Partial集合与NonEmpty Span List责任类似，Full集合与Empty Span List责任类似。可以看见Partial和Full都是一个[2]spanSet类型，也就每个Partial和Full都各有两个spanSet集合，这是为了给GC垃圾回收来使用的，其中一个集合是已扫描的，另一个集合是未扫描的。

### 2.5 MHeap

Golang内存管理的MHeap依然是继承TCMalloc的PageHeap设计。MHeap的上游是MCentral，MCentral中的Span不够时会向MHeap申请。MHeap的下游是操作系统，MHeap的内存不够时会向操作系统的虚拟内存空间申请。访问MHeap获取内存依然是需要加锁的。

MHeap是对内存块的管理对象，是通过Page为内存单元进行管理。那么用来详细管理每一系列Page的结构称之为一个HeapArena，它们的逻辑层级关系如图25所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135097610-8b18c759-6207-435d-80d4-616ce62de8d3.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_52%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 25 MHeap内部逻辑层级构造

一个HeapArena占用内存64MB[[8\]](#_ftn8)，其中里面的内存的是一个一个的mspan，当然最小单元依然是Page，图中没有表示出mspan，因为多个连续的page就是一个mspan。所有的HeapArena组成的集合是一个Arenas，也就是MHeap针对堆内存的管理。MHeap是Golang进程全局唯一的所以访问依然加锁。图中又出现了MCentral，因为MCentral本也属于MHeap中的一部分。只不过会优先从MCentral获取内存，如果没有MCentral会从Arenas中的某个HeapArena获取Page。

如果再详细剖析MHeap里面相关的数据结构和指针依赖关系，可以参考图26，这里不做过多解释，如果更像详细理解MHeap建议研读源代码/usr/local/go/src/runtime/mheap.go文件。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135153113-4db62f09-063b-4470-9fa9-1229150f703c.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_99%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 26 MHeap数据结构引用依赖

MHeap中HeapArena占用了绝大部分的空间，其中每个HeapArean包含一个bitmap，其作用是用于标记当前这个HeapArena的内存使用情况。其主要是服务于GC垃圾回收模块，bitmap共有两种标记，一个是标记对应地址中是否存在对象，一个是标记此对象是否被GC模块标记过，所以当前HeapArena中的所有Page均会被bitmap所标记。

ArenaHint为寻址HeapArena的结构，其有三个成员：

（1）addr，为指向的对应HeapArena首地址。

（2）down，为当前的HeapArena是否可以扩容。

（3）next，指向下一个HeapArena所对应的ArenaHint首地址。

从图3.40中可以看出，MCentral实际上就是隶属于MHeap的一部分，从数据结构来看，每个Span Class对应一个MCentral，而之前在分析Golang内存管理中的逻辑分层中，是将这些MCentral集合统一归类为MCentral层。

### 1.6 Tiny对象分配流程

TCMalloc将对象分为了小对象、中对象、和大对象，而Golang内存管理将对象的分类进行了更细的一些划分，具体的划分区别对比如表6所示。

###### 表6 Golang内存与TCMalloc对内存的分类对比

| **TCMalloc** | **Golang** |
| ------------ | ---------- |
| 小对象       | Tiny对象   |
| 中对象       | 小对象     |
| 大对象       | 大对象     |

针对Tiny微小对象的分配，实际上Golang做了比较特殊的处理，之前在介绍MCache的时候并没有提及有关Tiny的存储和分配问题，MCache中不仅保存着各个Span Class级别的内存块空间，还有一个比较特殊的Tiny存储空间，如图26所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135206587-7442cb63-77a7-4b87-8822-db91367db164.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_84%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 26 MCache中的Tiny空间

Tiny空间是从Size Class = 2（对应Span Class = 4 或5）中获取一个16B的Object，作为Tiny对象的分配空间。对于Golang内存管理为什么需要一个Tiny这样的16B空间，原因是因为如果协程逻辑层申请的内存空间小于等于8B，那么根据正常的Size Class匹配会匹配到Size Class = 1（对应Span Class = 2或3），所以像 int32、 byte、 bool 以及小字符串等经常使用的Tiny微小对象，也都会使用从Size Class = 1申请的这8B的空间。但是类似bool或者1个字节的byte，也都会各自独享这8B的空间，进而导致有一定的内存空间浪费，如图27所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135248063-4376ca60-ef0e-4463-a648-e87ffe2d7e51.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_25%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 27 如果微小对象不存在Tiny空间中

可以看出来这样当大量的使用微小对象可能会对Size Class = 1的Span造成大量的浪费。所以Golang内存管理决定尽量不使用Size Class = 1的Span，而是将申请的Object小于16B的申请统一归类为Tiny对象申请。具体的申请流程如图28所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135299397-38b04b81-3179-44e4-81ca-bb476b69f22f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_73%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 28 MCache中Tiny微小对象分配流程

MCache中对于Tiny微小对象的申请流程如下：

（1）P向MCache申请微小对象如一个Bool变量。如果申请的Object在Tiny对象的大小范围则进入Tiny对象申请流程，否则进入小对象或大对象申请流程。

（2）判断申请的Tiny对象是否包含指针，如果包含则进入小对象申请流程（不会放在Tiny缓冲区，因为需要GC走扫描等流程）。

（3）如果Tiny空间的16B没有多余的存储容量，则从Size Class = 2（即Span Class = 4或5）的Span中获取一个16B的Object放置Tiny缓冲区。

（4）将1B的Bool类型放置在16B的Tiny空间中，以字节对齐的方式。

Tiny对象的申请也是达不到内存利用率100%的，就上述图43为例，当前Tiny缓冲16B的内存利用率为，而如果不用Tiny微小对象的方式来存储，那么内存的布局将如图29所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135361669-940339e3-48fd-4e7a-88d7-0f3d6dc0f681.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_70%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 29 不用Tiny缓冲存储情况

可以算出利用率为。Golang内存管理通过Tiny对象的处理，可以平均节省20%左右的内存。

### 2.7 小对象分配流程

上节已经介绍了分配在1B至16B的Tiny对象的分配流程，那么对于对象在16B至32B的内存分配，Golang会采用小对象的分配流程。

分配小对象的标准流程是按照Span Class规格匹配的。在之前介绍MCache的内部构造已经介绍了，MCache一共有67份Size Class其中Size Class 为0的做了特殊的处理直接返回一个固定的地址。Span Class为Size Class的二倍，也就是从0至133共134个Span Class。

当协程逻辑层P主动申请一个小对象的时候，Golang内存管理的内存申请流程如图30所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135402859-c6404c6a-f0fd-4bb9-bbef-0c5286b0b2a4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_100%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 30 Golang小对象内存分配流程

下面来分析一下具体的流程过程：

（1）首先协程逻辑层P向Golang内存管理申请一个对象所需的内存空间。

（2）MCache在接收到请求后，会根据对象所需的内存空间计算出具体的大小Size。

（3）判断Size是否小于16B，如果小于16B则进入Tiny微对象申请流程，否则进入小对象申请流程。

（4）根据Size匹配对应的Size Class内存规格，再根据Size Class和该对象是否包含指针，来定位是从noscan Span Class 还是 scan Span Class获取空间，没有指针则锁定noscan。

（5）在定位的Span Class中的Span取出一个Object返回给协程逻辑层P，P得到内存空间，流程结束。

（6）如果定位的Span Class中的Span所有的内存块Object都被占用，则MCache会向MCentral申请一个Span。

（7）MCentral收到内存申请后，优先从相对应的Span Class中的NonEmpty Span List（或Partial Set，Golang V1.16+）里取出Span（多个Object组成），NonEmpty Span List没有则从Empty List（或 Full Set Golang V1.16+）中取，返回给MCache。

（8）MCache得到MCentral返回的Span，补充到对应的Span Class中，之后再次执行第（5）步流程。

（9）如果Empty Span List（或Full Set）中没有符合条件的Span，则MCentral会向MHeap申请内存。

（10）MHeap收到内存请求从其中一个HeapArena从取出一部分Pages返回给MCentral，当MHeap没有足够的内存时，MHeap会向操作系统申请内存，将申请的内存也保存到HeapArena中的mspan中。MCentral将从MHeap获取的由Pages组成的Span添加到对应的Span Class链表或集合中，作为新的补充，之后再次执行第（7）步。

（11）最后协程业务逻辑层得到该对象申请到的内存，流程结束。

### 2.8 大对象分配流程

小对象是在MCache中分配的，而大对象是直接从MHeap中分配。对于不满足MCache分配范围的对象，均是按照大对象分配流程处理。

大对象分配流程是协程逻辑层直接向MHeap申请对象所需要的适当Pages，从而绕过从MCaceh到MCentral的繁琐申请内存流程，大对象的内存分配流程相对比较简单，具体的流程如图31所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135463380-9ee93382-7deb-48c0-ab38-519d679101e4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_90%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 31 Golang大对象内存分配流程

下面来分析一下具体的大对象内存分配流程：

（1）协程逻辑层申请大对象所需的内存空间，如果超过32KB，则直接绕过MCache和MCentral直接向MHeap申请。

（2）MHeap根据对象所需的空间计算得到需要多少个Page。

（3）MHeap向Arenas中的HeapArena申请相对应的Pages。

（4）如果Arenas中没有HeapA可提供合适的Pages内存，则向操作系统的虚拟内存申请，且填充至Arenas中。

（5）MHeap返回大对象的内存空间。

（6）协程逻辑层P得到内存，流程结束。