# Golang内存管理和垃圾回收

现代高级编程语言管理内存的方式分自动和手动两种。手动管理内存的典型代表是C和C++，编写代码过程中需要主动申请或者释放内存；而PHP、Java 和Go等语言使用自动的内存管理系统，由内存分配器和垃圾收集器来代为分配和回收内存，其中垃圾收集器就是我们常说的GC。今天腾讯后台开发工程师汪汇向大家分享 Golang 垃圾回收算法。（当然，Rust 是另一种）

从Go v1.12版本开始，Go使用了**非分代的、并发的、基于三色标记清除的垃圾回收器**。相关标记清除算法可以参考C/C++，而Go是一种静态类型的编译型语言。因此，Go不需要VM，Go应用程序二进制文件中嵌入了一个小型运行时(Go runtime)，可以处理诸如垃圾收集(GC)、调度和并发之类的语言功能。首先让我们看一下Go内部的内存管理是什么样子的。



## **一、 Golang内存管理**

这里先简单介绍一下 Golang 运行调度。在 Golang 里面有三个基本的概念：G, M, P。

- G: Goroutine 执行的上下文环境。
- M: 操作系统线程。
- P: Processer。进程调度的关键，调度器，也可以认为约等于CPU。

一个 Goroutine 的运行需要G+P+M三部分结合起来。



![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe94Cjxng5VbT4M7FkUgyAfhpQu2e0dr5z6Za2b2aIw9peb8icIQyc29bC7VNuYfPh81ibaUdoSJg6ibicw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

 图源：《Golang---内存管理(内存分配)》

(http://t.zoukankan.com/zpcoding-p-13259943.html)



### **（一）TCMalloc**

Go将内存划分和分组为页（Page），这和Java的内存结构完全不同，没有分代内存，这样的原因是Go的内存分配器采用了TCMalloc的**设计思想**：

#### **1.Page**

与TCMalloc中的Page相同，x64下1个Page的大小是8KB。上图的最下方，1个浅蓝色的长方形代表1个Page。

#### **2.Span**

与TCMalloc中的Span相同，Span是内存管理的基本单位，代码中为mspan，一组连续的Page组成1个Span，所以上图一组连续的浅蓝色长方形代表的是一组Page组成的1个Span，另外，1个淡紫色长方形为1个Span。

#### **3.mcache**

mcache是提供给P（逻辑处理器）的高速缓存，用于存储小对象（对象大小<= 32Kb）。尽管这类似于线程堆栈，但它是堆的一部分，用于动态数据。所有类大小的mcache包含scan和noscan类型mspan。Goroutine可以从mcache没有任何锁的情况下获取内存，因为一次P只能有一个锁G。因此，这更有效。mcache从mcentral需要时请求新的span。

#### **4.mcentral**

mcentral与TCMalloc中的CentralCache类似，是所有线程共享的缓存，需要加锁访问，它按Span class对Span分类，串联成链表，当mcache的某个级别Span的内存被分配光时，它会向mcentral申请1个当前级别的Span。每个mcentral包含两个mspanList：

- empty：双向span链表，包括没有空闲对象的span或缓存mcache中的span。当此处的span被释放时，它将被移至non-empty span链表。
- non-empty：有空闲对象的span双向链表。当从mcentral请求新的span，mcentral将从该链表中获取span并将其移入empty span链表。

#### **5.mheap**

mheap与TCMalloc中的PageHeap类似，它是堆内存的抽象，也是垃圾回收的重点区域，把从OS申请出的内存页组织成Span，并保存起来。当mcentral的Span不够用时会向mheap申请，mheap的Span不够用时会向OS申请，向OS的内存申请是按页来的，然后把申请来的内存页生成Span组织起来，同样也是需要加锁访问的。

#### **6.栈**

这是栈存储区，每个Goroutine（G）有一个栈。在这里存储了静态数据，包括函数栈帧，静态结构，原生类型值和指向动态结构的指针。这与分配给每个P的mcache不是一回事。

### 7.TCMalloc为什么快：

1.使用了thread cache（线程cache），小块的内存分配都可以从cache中分配，这样再多线程分配内存的情况下，可以减少锁竞争。

2.tcmalloc会为每个线程分配本地缓存，小对象请求可以直接从本地缓存获取，如果没有空闲内存，则从central heap中一次性获取一连串小对象。大对象是直接使用页级分配器（page-level allocator）从Central page Heap中进行分配，即一个大对象总是按页对齐的。tcmalloc对于小内存，按8的整数次倍分配，对于大内存，按4K的整数次倍分配。

3.当某个线程缓存中所有对象的总大小超过2MB的时候，会进行垃圾收集。垃圾收集阈值会自动根据线程数量的增加而减少，这样就不会因为程序有大量线程而过度浪费内存。

4.tcmalloc为每个线程分配一个thread-local cache，小对象的分配直接从thread-local cache中分配。根据需要将对象从CentralHeap中移动到thread-local cache，同时定期的用垃圾回收器把内存从thread-local cache回收到Central free list中。

### 8.总结

ThreadCache（用于小对象分配）：线程本地缓存，每个线程独立维护一个该对象，多线程在并发申请内存时不会产生锁竞争。

CentralCache（Central free list，用于小对象分配）：全局cache，所有线程共享。当thread cache空闲链表为空时，会批量从CentralCache中申请内存；当thread cache总内存超过阈值，会进行内存垃圾回收，将空闲内存返还给CentralCache。

Page Heap（小/大对象）：全局页堆，所有线程共享。对于小对象，当centralcache为空时，会从page heap中申请一个span；当一个span完全空闲时，会将该span返还给page heap。对于大对象，直接从page heap中分配，用完直接返还给page heap。系统内存：当page cache内存用光后，会通过sbrk、mmap等系统调用向OS申请内存。

### **（二）内存分配**

Go 中的内存分类并不像TCMalloc那样分成小、中、大对象，但是它的小对象里又细分了一个Tiny对象，Tiny对象指大小在1Byte到16Byte之间并且不包含指针的对象。小对象和大对象只用大小划定，无其他区分。

**核心思想**：把内存分为多级管理，降低锁的粒度(只是去mcentral和mheap会申请锁), 以及多种对象大小类型，减少分配产生的内存碎片。

- **\*微小对象(Tiny)（size<16B\**\**）\***

使用mcache的微小分配器分配小于16个字节的对象，并且在单个16字节块上可完成多个微小分配。

- ***\*小对象（尺寸16B〜32KB）\****

大小在16个字节和32k字节之间的对象被分配在G运行所在的P的mcache的对应的mspan size class上。

- ***\*大对象（大小>32KB）\****

大于32 KB的对象直接分配在mheap的相应大小类上(size class)。

- 如果mheap为空或没有足够大的页面满足分配请求，则它将从操作系统中分配一组新的页（至少1MB）。
- 如果对应的大小规格在mcache中没有可用的块，则向mcentral申请。
- 如果mcentral中没有可用的块，则向mheap申请，并根据BestFit 算法找到最合适的mspan。如果申请到的mspan超出申请大小，将会根据需求进行切分，以返回用户所需的页数。剩余的页构成一个新的mspan放回mheap的空闲列表。
- 如果mheap中没有可用span，则向操作系统申请一系列新的页（最小 1MB）。Go 会在操作系统分配超大的页（称作arena）。分配一大批页会减少和操作系统通信的成本。

### **（三）内存回收**

go内存会分成堆区（Heap）和栈区（Stack）两个部分，程序在运行期间可以主动从堆区申请内存空间，这些内存由内存分配器分配并由垃圾收集器负责回收。栈区的内存由编译器自动进行分配和释放，栈区中存储着函数的参数以及局部变量，它们会随着函数的创建而创建，函数的返回而销毁。如果只申请和分配内存，内存终将枯竭。Go使用垃圾回收收集不再使用的span，把span释放交给mheap，mheap对span进行span的合并，把合并后的span加入scav树中，等待再分配内存时，由mheap进行内存再分配。**因此，Go堆是Go垃圾收集器管理的主要区域**。





## 二、常见的GC算法

### 引用计数法

根据对象自身的引用计数来回收，当引用计数归零时进行回收，但是计数频繁更新会带来更多开销，且无法解决循环引用的问题。

- 优点：简单直接，回收速度快
- 缺点：需要额外的空间存放计数，无法处理循环引用的情况；

### 可达性分析
对象引用链：通过一系列的称为"GCRoots"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链(Reference Chain) ，如果一个对象到GCRoots没有任何引用链相连，或者用图论的话来说，就是，从GCRoots到这个对象不可达时，则证明此对象是不可用的。

根对象在垃圾回收的术语中又叫做根集合，它是垃圾回收器在标记过程时最先检查的对象，包括：

1.全局变量：程序在编译期就能确定的那些存在于程序整个生命周期的变量。

2.执行栈：每个 goroutine 都包含自己的执行栈，这些执行栈上包含栈上的变量及指向分配的堆内存区块的指针。

3.寄存器：寄存器的值可能表示一个指针，参与计算的这些指针可能指向某些赋值器分配的堆内存区块。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801163955410.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMDA5MjYy,size_16,color_FFFFFF,t_70)

### 标记清除法

标记出所有不需要回收的对象，在标记完成后统一回收掉所有未被标记的对象。
![mark_clean](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/mark_clean.png)

- 优点：简单直接，速度快，适合可回收对象不多的场景
- 缺点：会造成不连续的内存空间（内存碎片），导致有大的对象创建的时候，明明内存中总内存是够的，但是空间不是连续的造成对象无法分配；

### 复制法

复制法将内存分为大小相同的两块，每次使用其中的一块，当这一块的内存使用完后，将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉
![copy_method](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/copy_method.png)

- 优点：解决了内存碎片的问题，每次清除针对的都是整块内存，但是因为移动对象需要耗费时间，效率低于标记清除法；
- 缺点：有部分内存总是利用不到，资源浪费，移动存活对象比较耗时，并且如果存活对象较多的时候，需要担保机制确保复制区有足够的空间可完成复制；

### 标记整理

标记过程同标记清除法，结束后将存活对象压缩至一端，然后清除边界外的内容
![mark_tidy](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/mark_tidy.png)

- 优点：解决了内存碎片的问题，也不像标记复制法那样需要担保机制，存活对象较多的场景也使适用；
- 缺点：性能低，因为在移动对象的时候不仅需要移动对象还要维护对象的引用地址，可能需要对内存经过几次扫描才能完成；

### 分代式

将对象根据存活时间的长短进行分类，存活时间小于某个值的为年轻代，存活时间大于某个值的为老年代，永远不会参与回收的对象为永久代。并根据分代假设（如果一个对象存活时间不长则倾向于被回收，如果一个对象已经存活很长时间则倾向于存活更长时间）对对象进行回收。

### Golang的垃圾回收（GC）算法

Golang的垃圾回收（GC）算法使用的是无分代（对象没有代际之分）、**不整理（回收过程中不对对象进行移动与整理）**、**并发（与用户代码并发执行）**的三色标记清扫算法。原因在于：

- **对象整理的优势是解决内存碎片问题以及“允许”使用顺序内存分配器。但 Go 运行时的分配算法基于`tcmalloc`，基本上没有碎片问题。 并且顺序内存分配器在多线程的场景下并不适用。G**o 使用的是基于`tcmalloc`的现代内存分配算法，对对象进行整理不会带来实质性的性能提升。
- 分代`GC`依赖分代假设，即`GC`将主要的回收目标放在新创建的对象上（存活时间短，更倾向于被回收），**而非频繁检查所有对象。**
- Go 的编译器会通过逃逸分析将大部分新生对象存储在栈上（栈直接被回收），只有那些需要长期存在的对象才会被分配到需要进行垃圾回收的堆中。也就是说，分代`GC`回收的那些存活时间短的对象在 Go 中是直接被分配到栈上，当`goroutine`死亡后栈也会被直接回收，不需要`GC`的参与，进而分代假设并没有带来直接优势。
- Go 的垃圾回收器与用户代码**并发执行**，**使得 STW 的时间与对象的代际、对象的 size 没有关系。**Go 团队更关注于如何更好地让 GC 与用户代码并发执行（使用适当的 CPU 来执行垃圾回收），而非减少停顿时间这一单一目标上。



## 三、Go的垃圾回收

垃圾回收(Garbage Collection，简称GC)是编程语言中提供的自动的内存管理机制，自动释放不需要的内存对象，让出存储器资源。GC过程中无需程序员手动执行。GC机制在现代很多编程语言都支持，GC能力的性能与优劣也是不同语言之间对比度指标之一。



Golang在GC的演进过程中也经历了很多次变革，Go V1.3之前的标记-清除(mark and sweep)算法，Go V1.3之前的标记-清扫(mark and sweep)的缺点

- Go V1.5的三色并发标记法
- Go V1.5的三色标记为什么需要STW
- Go V1.5的三色标记为什么需要屏障机制(“强-弱” 三色不变式、插入屏障、删除屏障 )
- Go V1.8混合写屏障机制
- Go V1.8混合写屏障机制的全场景分析

### （一）、Go V1.3之前的标记-清除(mark and sweep)算法

在Golang1.3之前的时候主要用的普通的标记-清除算法，此算法主要有两个主要的步骤：

- 标记(Mark phase)
- 清除(Sweep phase)

#### 1 标记清除算法的具体步骤

**第一步**，暂停程序业务逻辑, 分类出可达和不可达的对象，然后做上标记。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650787873045-d038fe47-4898-4b07-9e16-007bebb6fb9c.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_43%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)


图中表示是程序与对象的可达关系，目前程序的可达对象有对象1-2-3，对象4-7等五个对象。

**第二步**, 开始标记，程序找出它所有可达的对象，并做上标记。如下图所示：
![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650787891194-883ec541-5f13-4934-9274-080e5f44cf5e.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_44%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)


所以对象1-2-3、对象4-7等五个对象被做上标记。

**第三步**,  标记完了之后，然后开始清除未标记的对象. 结果如下。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650787913616-ecf21ee2-c247-4401-9d3e-5e2fa278726f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_38%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

操作非常简单，但是有一点需要额外注意：mark and sweep算法在执行的时候，需要程序暂停！即 `STW(stop the world)`，STW的过程中，CPU不执行用户代码，全部用于垃圾回收，这个过程的影响很大，所以STW也是一些回收机制最大的难题和希望优化的点。所以在执行第三步的这段时间，程序会暂定停止任何工作，卡在那等待回收执行完毕。

**第四步**, 停止暂停，让程序继续跑。然后循环重复这个过程，直到process程序生命周期结束。

以上便是标记-清除（mark and sweep）回收的算法。

#### 2 标记-清除(mark and sweep)的缺点

标记清除算法明了，过程鲜明干脆，但是也有非常严重的问题。

- STW，stop the world；让程序暂停，程序出现卡顿 **(重要问题)**；
- 标记需要扫描整个heap；
- 清除数据会产生heap碎片。

Go V1.3版本之前就是以上来实施的,  在执行GC的基本流程就是首先启动STW暂停，然后执行标记，再执行数据回收，最后停止STW，如图所示。





![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650787936233-9002040d-220b-4af6-8e51-75d7887569b4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_69%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



从上图来看，全部的GC时间都是包裹在STW范围之内的，这样貌似程序暂停的时间过长，影响程序的运行性能。所以Go V1.3 做了简单的优化,将STW的步骤提前, 减少STW暂停的时间范围.如下所示

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650788071197-26a29703-0fb5-43f4-afc5-87a35fc78a4b.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_69%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10) 
上图主要是将STW的步骤提前了一步，因为在Sweep清除的时候，可以不需要STW停止，因为这些对象已经是不可达对象了，不会出现回收写冲突等问题。



但是无论怎么优化，Go V1.3都面临这个一个重要问题，就是**mark-and-sweep 算法会暂停整个程序** 。



Go是如何面对并这个问题的呢？接下来G V1.5版本 就用**三色并发标记法**来优化这个问题.



### （二）、Go V1.5的三色并发标记法

为了解决标记清除算法带来的STW问题，Go和Java都会实现三色可达性分析标记算法的变种以缩短STW的时间。三色可达性分析标记算法按“是否被访问过”将程序中的对象分成白色、黑色和灰色：

- **白色对象 — 对象尚未被垃圾收集器访问过，在可达性分析刚开始的阶段，所有的对象都是白色的，若在分析结束阶段，仍然是白色的对象，即代表不可达。**
- **黑色对象 — 表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经被扫描过，黑色的对象代表已经被扫描过而且是安全存活的，如果有其他对象只想黑色对象无需再扫描一遍，黑色对象不可能直接（不经过灰色对象）指向某个白色对象。**
- **灰色对象 — 表示对象已经被垃圾收集器访问过，但是这个对象上至少存在一个引用还没有被扫描过，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象。**

三色可达性分析算法大致的流程是（初始状态所有对象都是白色）：

**1.从GC Roots开始枚举，它们所有的直接引用变为灰色（移入灰色集合），GC Roots变为黑色。**

**2.从灰色集合中取出一个灰色对象进行分析：**

- **将这个对象所有的直接引用变为灰色，放入灰色集合中；**
- **将这个对象变为黑色。**

**3.重复步骤2，一直重复直到灰色集合为空。**

**4.分析完成，仍然是白色的对象就是GC Roots不可达的对象，可以作为垃圾被清理。**



Golang中的垃圾回收主要应用三色标记法，GC过程和其他用户goroutine可并发运行，但需要一定时间的**STW(stop the world)**，所谓**三色标记法**实际上就是通过三个阶段的标记来确定清楚的对象都有哪些？我们来看一下具体的过程。

**第一步** , 每次新创建的对象，默认的颜色都是标记为“白色”，如图所示。



![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651035738281-051f7a89-e07f-418c-ad0e-7cb94ef1a3b8.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_61%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



上图所示，我们的程序可抵达的内存对象关系如左图所示，右边的标记表，是用来记录目前每个对象的标记颜色分类。这里面需要注意的是，所谓“程序”，则是一些对象的根节点集合。所以我们如果将“程序”展开，会得到类似如下的表现形式，如图所示。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651035821416-b0ad644e-ef8e-440a-bbf4-b9e24a7e0257.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

**第二步**, 每次GC回收开始, 会从根节点开始遍历所有对象，把遍历到的对象从白色集合放入“灰色”集合如图所示。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651035842467-7341846f-6dee-4f8b-ad37-dc9723aa6407.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
这里 要注意的是，本次遍历是一次遍历，非递归形式，是从程序抽次可抵达的对象遍历一层，如上图所示，当前可抵达的对象是对象1和对象4，那么自然本轮遍历结束，对象1和对象4就会被标记为灰色，灰色标记表就会多出这两个对象。

**第三步**, 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合，如图所示。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651035859950-96053775-24f7-4bdc-a1fb-295747055b3e.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
这一次遍历是只扫描灰色对象，将灰色对象的第一层遍历可抵达的对象由白色变为灰色，如：对象2、对象7. 而之前的灰色对象1和对象4则会被标记为黑色，同时由灰色标记表移动到黑色标记表中。

**第四步**, 重复**第三步**, 直到灰色中无任何对象，如图所示。
![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651035907012-927d6cbc-686b-4f81-a1de-097ac7598a8e.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651035916208-9c293dc0-8988-4180-a9b7-412e2599af0e.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



当我们全部的可达对象都遍历完后，灰色标记表将不再存在灰色对象，目前全部内存的数据只有两种颜色，黑色和白色。那么黑色对象就是我们程序逻辑可达（需要的）对象，这些数据是目前支撑程序正常业务运行的，是合法的有用数据，不可删除，白色的对象是全部不可达对象，目前程序逻辑并不依赖他们，那么白色对象就是内存中目前的垃圾数据，需要被清除。

**第五步**: 回收所有的白色标记表的对象. 也就是回收垃圾，如图所示。

以上我们将全部的白色对象进行删除回收，![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651035960263-e50436a6-4a3c-48f9-82cb-bb5729d71116.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)剩下的就是全部依赖的黑色对象。



三色标记清除算法本身是不可以并发或者增量执行的，**它需要STW**，**而如果并发执行，用户程序可能在标记执行的过程中修改对象的指针。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe94Cjxng5VbT4M7FkUgyAfhpDGBI8liaibcvyXxOjP7kowzG1TnVmgAJefhegPo2IJiabXQ6IxnRdVqPQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### **没有STW的异常情况一般会有2种：**

1.一种是把原本应该垃圾回收的死亡对象错误的标记为存活。虽然这不好，但是不会导致严重后果，只不过产生了一点逃过本次回收的浮动垃圾而已，下次清理就可以，比如上图所示的三色标记过程中，用户程序取消了从B对象到E对象的引用，但是因为B到E已经被标记完成不会继续执行步骤2，所以E对象最终会被错误的标记成黑色，不会被回收，这个D就是**浮动垃圾**，会在下次垃圾收集中清理。

2.一种是把原本存活的对象错误的标记为已死亡，导致“对象消失”，这在内存管理中是非常严重的错误。比如上图所示的三色标记过程中，用户程序建立了从B对象到H对象的引用(例如**B.next =H**)，接着执行**D.next=nil**，但是因为B到H中不存在灰色对象，因此在这之间不会继续执行三色并发标记中的步骤2，D到H之间的链接被断开，所以H对象最终会被标记成白色，会被垃圾收集器错误地回收。我们将这种错误称为**悬挂指针**，即指针没有指向特定类型的合法对象，影响了内存的安全性。



### 没有STW的三色标记法情况下  --- 悬挂指针的具体介绍

先抛砖引玉，我们加入如果没有STW，那么也就不会再存在性能上的问题，那么接下来我们假设如果三色标记法不加入STW会发生什么事情？
我们还是基于上述的三色并发标记法来说, 他是一定要依赖STW的. 因为如果不暂停程序, 程序的逻辑改变对象引用关系, 这种动作如果在标记阶段做了修改，会影响标记结果的正确性，我们来看看一个场景，如果三色标记法, 标记过程不使用STW将会发生什么事情?



1.我们把初始状态设置为已经经历了第一轮扫描，目前黑色的有对象1和对象4， 灰色的有对象2和对象7，其他的为白色对象，且对象2是通过指针p指向对象3的，如图所示。
![no_STW_1](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_1.png)



2.现在如何三色标记过程不启动STW，那么在GC扫描过程中，任意的对象均可能发生读写操作，如图所示，在还没有扫描到对象2的时候，已经标记为黑色的对象4，此时创建指针q，并且指向白色的对象3。

![no_STW_2](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_2.png)

3. 与此同时灰色的对象2将指针p移除，那么白色的对象3实则就是被挂在了已经扫描完成的黑色的对象4下，如图所示。

![no_STW_3](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_3.png)





4.然后我们正常指向三色标记的算法逻辑，将所有灰色的对象标记为黑色，那么对象2和对象7就被标记成了黑色，如图所示。![no_STW_4](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_4.png)



5.那么就执行了三色标记的最后一步，将所有白色对象当做垃圾进行回收，如图所示。

![no_STW_5](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_5.png)
但是最后我们才发现，本来是对象4合法引用的对象3，却被GC给“误杀”回收掉了。

**可以看出，有两种情况，在三色标记法中，是不希望被发生的。**

- 条件1: 一个白色对象被黑色对象引用**(白色被挂在黑色下)**
- 条件2: 灰色对象与它之间的可达关系的白色对象遭到破坏**(灰色同时丢了该白色)**
  如果当以上两个条件同时满足时，就会出现对象丢失现象!

并且，如图所示的场景中，如果示例中的白色对象3还有很多下游对象的话, 也会一并都清理掉。

为了防止这种现象的发生，最简单的方式就是STW，直接禁止掉其他用户程序对对象引用关系的干扰，但是**STW的过程有明显的资源浪费，对所有的用户程序都有很大影响**。那么是否可以在保证对象不丢失的情况下合理的尽可能的提高GC效率，减少STW时间呢？答案是可以的，我们只要使用一种机制，尝试去破坏上面的两个必要条件就可以了。



## **四、屏障技术**

为了解决上述的“对象消失”的现象，Wilson于1994年在理论上证明了，当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：

- 赋值器插入了一条或多条从黑色对象到白色对象的新引用；
- 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。

因此为了我们要解决并发扫描时的对象消失问题，保证垃圾收集算法的正确性，只需破坏这两个条件的任意一个即可，**屏障技术**就是在并发或者增量标记过程中保证**三色不变性**的重要技术。

内存屏障技术是一种屏障指令，它可以让CPU或者编译器在执行内存相关操作时遵循特定的约束，目前多数的现代处理器都会乱序执行指令以最大化性能，但是该技术能够保证内存操作的顺序性，在内存屏障前执行的操作一定会先于内存屏障后执行的操作。垃圾收集中的屏障技术更像是一个**钩子方法**，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，根据操作类型的不同，我们可以将它们分成**读屏障（Read barrier）**和写屏障（Write barrier）两种，**因为读屏障需要在读操作中加入代码片段，对用户程序的性能影响很大，所以编程语言往往都会采用写屏障保证三色不变性。**

**重点**：

1. **写屏障的代码在编译期间生成好，之后不会再变化；**
2. 堆上对象赋值才会生成写屏障；
3. 哪些对象分配在栈上，哪些分配在堆上？也是编译期间由编译器决定，这个过程叫做“逃逸分析”；

#### 原理分析

下面的例子使用的是 go1.13.3。

##### 示例分析代码

写屏障是编译器生成的，先形象看下代码样子：

```
  1 package main
  2 
  3 type BaseStruct struct {
  4     name string
  5     age  int
  6 }
  7 
  8 type Tstruct struct {
  9     base   *BaseStruct
 10     field0 int
 11 }
 12 
 13 func funcAlloc0 (a *Tstruct) {
 14     a.base = new(BaseStruct)    // new 一个BaseStruct结构体，赋值给 a.base 字段
 15 }
 16 
 17 func funcAlloc1 (b *Tstruct) {
 18     var b0 Tstruct
 19     b0.base = new(BaseStruct)  // new 一个BaseStruct结构体，赋值给 b0.base 字段
 20 }
 21 
 22 func main() {
 23     a := new(Tstruct)    // new 一个Tstruct 结构体
 24     b := new(Tstruct)   // new 一个Tstruct 结构体
 25     
 26     go funcAlloc0(a)
 27     go funcAlloc1(b)
 28 }
```

这里例子，可以用来观察两个东西：

1. 逃逸分析
2. 编译器插入内存屏障的时机

##### 逃逸分析

只有堆上对象的写才会可能有写屏障，因为如果对栈上的写做拦截，那么流程代码会非常复杂，并且性能下降会非常大，得不偿失。根据局部性的原理来说，其实我们程序跑起来，大部分的其实都是操作在栈上，函数参数啊、函数调用导致的压栈出栈啊、局部变量啊，协程栈，这些如果也弄起写屏障，那么可想而知了，根本就不现实，复杂度和性能就是越不过去的坎。

继续看逃逸什么意思？就是内存分配到堆上。golang 可以在编译的时候使用 `-m` 参数支持把这个可视化出来：

```
$ go build -gcflags "-N -l -m" ./test_writebarrier0.go 
# command-line-arguments
./test_writebarrier0.go:13:18: funcAlloc0 a does not escape
./test_writebarrier0.go:14:17: new(BaseStruct) escapes to heap
./test_writebarrier0.go:17:18: funcAlloc1 b does not escape
./test_writebarrier0.go:19:18: funcAlloc1 new(BaseStruct) does not escape
./test_writebarrier0.go:23:13: new(Tstruct) escapes to heap
./test_writebarrier0.go:24:13: new(Tstruct) escapes to heap
```

**先说逃逸分析两点原则**：

1. 在保证程序正确性的前提下，尽可能的把对象分配到栈上，这样性能最好；
   1. 栈上的对象生命周期就跟随 goroutine ，协程终结了，它就没了
2. 明确一定要分配到堆上对象，或者不确定是否要分配在堆上的对象，那么就全都分配到堆上；
   1. 这种对象的生命周期始于业务程序的创建，终于垃圾回收器的回收

我们看到源代码，有四次 new 对象的操作，经过编译器的“逃逸分析”之后，实际分配到堆上的是三次：

1. 14 行 —— 触发逃逸（分配到堆上）

   1. 这个必须得分配到堆上，因为除了这个 goroutine 还要存活呢

2. 19 行 —— 无 （分配到栈上）

   1. 这个虽然也是 new，单就分配到栈上就行，因为 b0 这个对象就是一个纯粹的栈对象

3. 23 行 —— 触发逃逸 （分配到堆上）

   1. 这个需要分配到堆上，因为分配出来的对象需要传递到其他协程使用

4. 24 行 —— 触发逃逸 （分配到堆上）

     1.这次必须注意下，其实站在我们上帝视角，这次的分配其实也可以分配到栈上。这种情况编译器就简单处理了，直接给分配到堆上。这种就属于编译器它摸不准的，那么分配到堆上就对了，反正也就性能有点影响，功能不会有问题，不然的话你真分配到栈上了，一旦栈被回收就出问题了

##### 写屏障真实的样子

再看下编译器汇编的代码：

![](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-07-11-gc2/8F95309B-9CD8-44E2-A65A-BAAA96CC0BE8.png)

从这个地方我们需要知道一个事情，go 的关键字语法呀，其实在编译的时候，都会对应到一个特定的函数，比如 new 这个关键字就对应了 `newobject` 函数，go 这个关键字对应的是 `newproc` 函数。贴一张比较完整的图：

![](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-07-11-gc2/Image.png)

从这个汇编代码我们也确认了，23，24行的对象分配确实是在堆上。我们再看下函数 `funcAlloc0` 和 `funcAlloc1` 这两个。

**`main.funcAlloc0`**

```
 13 func funcAlloc0 (a *Tstruct) {
 14     a.base = new(BaseStruct)    // new 一个BaseStruct结构体，赋值给 a.base 字段
 15 }
```

![](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-07-11-gc2/23DEE999-886F-4298-BCAE-EDB4F7A0B454.png)

简单的注释解析：

```
(gdb) disassemble 
Dump of assembler code for function main.funcAlloc0:
   0x0000000000456b10 <+0>:     mov    %fs:0xfffffffffffffff8,%rcx
   0x0000000000456b19 <+9>:     cmp    0x10(%rcx),%rsp
   0x0000000000456b1d <+13>:    jbe    0x456b6f <main.funcAlloc0+95>
   0x0000000000456b1f <+15>:    sub    $0x20,%rsp
   0x0000000000456b23 <+19>:    mov    %rbp,0x18(%rsp)
   0x0000000000456b28 <+24>:    lea    0x18(%rsp),%rbp
   0x0000000000456b2d <+29>:    lea    0x1430c(%rip),%rax        # 0x46ae40
   0x0000000000456b34 <+36>:    mov    %rax,(%rsp)
   0x0000000000456b38 <+40>:    callq  0x40b060 <runtime.newobject>
   # newobject的返回值在 0x8(%rsp) 里，golang 的参数和返回值都是通过栈传递的。这个跟 c 程序不同，c 程序是溢出才会用到栈，这里先把返回值放到寄存器 rax
   0x0000000000456b3d <+45>:    mov    0x8(%rsp),%rax           
   0x0000000000456b42 <+50>:    mov    %rax,0x10(%rsp)
   # 0x28(%rsp) 就是 a 的地址：0xc0000840b0
=> 0x0000000000456b47 <+55>:    mov    0x28(%rsp),%rdi         
   0x0000000000456b4c <+60>:    test   %al,(%rdi)
   # 这里判断是否开启了屏障（垃圾回收的扫描并发过程，才会把这个标记打开，没有打开的情况，对于堆上的赋值只是多走一次判断开销）
   0x0000000000456b4e <+62>:    cmpl   $0x0,0x960fb(%rip)        # 0x4ecc50 <runtime.writeBarrier>
   0x0000000000456b55 <+69>:    je     0x456b59 <main.funcAlloc0+73>
   0x0000000000456b57 <+71>:    jmp    0x456b68 <main.funcAlloc0+88>
   # 赋值 a.base = xxxx
   0x0000000000456b59 <+73>:    mov    %rax,(%rdi)
   0x0000000000456b5c <+76>:    jmp    0x456b5e <main.funcAlloc0+78>
   0x0000000000456b5e <+78>:    mov    0x18(%rsp),%rbp
   0x0000000000456b63 <+83>:    add    $0x20,%rsp
   0x0000000000456b67 <+87>:    retq   
   # 如果是开启了屏障，那么完成 a.base = xxx 的赋值就是在 gcWriteBarrier 函数里面了
   0x0000000000456b68 <+88>:    callq  0x44d170 <runtime.gcWriteBarrier>
   0x0000000000456b6d <+93>:    jmp    0x456b5e <main.funcAlloc0+78>
   0x0000000000456b6f <+95>:    callq  0x44b370 <runtime.morestack_noctxt>
   0x0000000000456b74 <+100>:   jmp    0x456b10 <main.funcAlloc0>
End of assembler dump.
```

**所以，从上面简单的汇编代码，我们印证得出几个小知识点**：

1. golang 传参和返回参数都是通过栈来传递的（可以思考下优略点，有点是逻辑简单了，也能很好的支持多返回值的实现，缺点是比寄存器的方式略慢，但是这种损耗在程序的运行下可以忽略）；
2. 写屏障是一段编译器插入的特殊代码，在编译期间插入，代码函数名字叫做 `gcWriteBarrier` ；
3. 屏障代码并不是直接运行，也是要条件判断的，并不是只要是堆上内存赋值就会运行gcWriteBarrier 代码，而是要有一个条件判断。这里提前透露下，这个条件判断是垃圾回收器扫描开始前，stw 程序给设置上去的；
   1. 所以平时对于堆上内存的赋值，多了一次写操作；

伪代码如下：

```
if runtime.writeBarrier.enabled {
    runtime.gcWriteBarrier(ptr, val)
} else {
    *ptr = val
}
```

`runtime·gcWriteBarrier` 函数干啥的，这个函数是用纯汇编写的，举一个特定cpu集合的例子，在 asm_amd64.s 里的实现。这个函数只干两件事：

1. 执行写请求
2. 处理 GC 相关的逻辑

下面简单理解下 `runtime·gcWriteBarrier` 这个函数：

```
TEXT runtime·gcWriteBarrier(SB),NOSPLIT,$120

        get_tls(R13)
        MOVQ    g(R13), R13
        MOVQ    g_m(R13), R13
        MOVQ    m_p(R13), R13
        MOVQ    (p_wbBuf+wbBuf_next)(R13), R14

        LEAQ    16(R14), R14
        MOVQ    R14, (p_wbBuf+wbBuf_next)(R13)
    // 检查 buffer 队列是否满？
        CMPQ    R14, (p_wbBuf+wbBuf_end)(R13)

    // 赋值的前后两个值都会被入队

        // 把 value 存到指定 buffer 位置
        MOVQ    AX, -16(R14)   // Record value

    // 把 *slot 存到指定 buffer 位置
        MOVQ    (DI), R13
        MOVQ    R13, -8(R14)

    // 如果 wbBuffer 队列满了，那么就下刷处理，比如置灰，置黑等操作
        JEQ     flush
ret:
    // 赋值：*slot = val 
        MOVQ    104(SP), R14
        MOVQ    112(SP), R13
        MOVQ    AX, (DI)
        RET

flush:
    。。。

        //  队列满了，统一处理，这个其实是一个批量优化手段
        CALL    runtime·wbBufFlush(SB)

    。。。

        JMP     ret
```

**思考下：不是说把 `\*slot = value` 直接置灰色，置黑色，就完了嘛，这里搞得这么复杂？**

最开始还真不是这样的，这个也是一个优化的过程，这里是利用批量的一个思想做的一个优化。我们再理解下最本质的东西，触发了写屏障之后，我们的核心目的是为了能够把赋值的前后两个值记录下来，以便 GC 垃圾回收器能得到通知，从而避免错误的回收。记录下来是最本质的，但是并不是要立马处理，所以这里做的优化就是，攒满一个 buffer ，然后批量处理，这样效率会非常高的。

wbBuf 结构如下： |————————————-| | 8 | 8 | 8 * 512 | 4 | |————————————-|

每个 P 都有这么个 wbBuf 队列。

我们看到 `CALL runtime·wbBufFlush(SB)` ，这个函数 wbBufFlush 是 golang 实现的，本质上是调用 `wbBufFlush1` 。这个函数才是 hook 写操作想要做的事情，精简了下代码如下：

```
func wbBufFlush1(_p_ *p) {
        start := uintptr(unsafe.Pointer(&_p_.wbBuf.buf[0]))
        n := (_p_.wbBuf.next - start) / unsafe.Sizeof(_p_.wbBuf.buf[0])
        ptrs := _p_.wbBuf.buf[:n]

        _p_.wbBuf.next = 0

        gcw := &_p_.gcw
        pos := 0
    // 循环批量处理队列里的值，这个就是之前在 gcWriteBarrier 赋值的
        for _, ptr := range ptrs {
                if ptr < minLegalPointer {
                        continue
                }
                obj, span, objIndex := findObject(ptr, 0, 0)
                if obj == 0 {
                        continue
                }

                mbits := span.markBitsForIndex(objIndex)
                if mbits.isMarked() {
                        continue
                }
                mbits.setMarked()
                if span.spanclass.noscan() {
                        gcw.bytesMarked += uint64(span.elemsize)
                        continue
                }
                ptrs[pos] = obj
                pos++
        }

        // 置灰色（投入灰色的队列），这就是我们的目的，对象在这里面我们就不怕了，我们要扫描的就是这个队列；
        gcw.putBatch(ptrs[:pos])

        _p_.wbBuf.reset()
}
```

所以我们总结下，写屏障到底做了什么：

1. hook 写操作
2. hook 住了写操作之后，把赋值语句的前后两个值都记录下来，投入 buffer 队列
3. buffer 攒满之后，批量刷到扫描队列（置灰）（这是 GO 1.10 左右引入的优化）

**`main.funcAlloc1`**

```
 17 func funcAlloc1 (b *Tstruct) {
 18     var b0 Tstruct
 19     b0.base = new(BaseStruct)  // new 一个BaseStruct结构体，赋值给 b0.base 字段
 20 }
```

![](https://cdn.jsdelivr.net/gh/liqingqiya/liqingqiya.github.io/images/posts/2020-07-11-gc2/FB0B964D-0A0F-4650-8700-45C4278E704E.png)

最后，再回顾看下 `main.funcAlloc1` 函数，这个函数是只有栈操作，非常简单。





####  “强-弱” 三色不变式

我们让GC回收器，满足下面两种情况之一时，即可保对象不丢失。  这两种方式就是**“强三色不变式”和“ 弱三色不变式”**。

- 强三色不变式

不存在黑色对象引用到白色对象的指针。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036383192-cb6b9fe9-4946-47da-bb9a-643f0c38a654.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)


弱三色不变色实际上是强制性的不允许黑色对象引用白色对象，这样就不会出现有白色对象被误删的情况。

- 弱三色不变式

所有被黑色对象引用的白色对象都处于灰色保护状态（允许黑色对象指向白色对象，但必须保证一个前提，这个白色对象必须处于灰色对象的保护下）。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036404003-e0ea569e-7a8a-4d9f-a08f-4bb9ed5c64ed.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)


弱三色不变式强调，黑色对象可以引用白色对象，但是这个白色对象必须存在其他灰色对象对它的引用，或者可达它的链路上游存在灰色对象。 这样实则是黑色对象引用白色对象，白色对象处于一个危险被删除的状态，但是上游灰色对象的引用，可以保护该白色对象，使其安全。



为了遵循上述的两个方式，GC算法演进到两种屏障方式，他们**“插入写屏障”, “删除写屏障”**。

**插入写屏障：**

`具体操作`: 在A对象引用B对象的时候，B对象被标记为灰色。(将B挂在A下游，B必须被标记为灰色)

`满足`: **强三色不变式**. (不存在黑色对象引用白色对象的情况了， 因为白色会强制变成灰色)

**删除写屏障：**

`具体操作`: 被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。

`满足`: **弱三色不变式**. (保护灰色对象到白色对象的路径不会断)



### **（一）插入写屏障**

Dijkstra在1978年提出了插入写屏障，也被叫做增量更新，通过如下所示的写屏障，破坏上述第一个条件（赋值器插入了一条或多条从黑色对象到白色对象的新引用）：

```
func DijkstraWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) 
     shade(ptr)  //先将新下游对象 ptr 标记为灰色
     *slot = ptr
}

//说明：
添加下游对象(当前下游对象slot, 新下游对象ptr) { 
 //step 1
 标记灰色(新下游对象ptr) 
 
 //step 2
 当前下游对象slot = 新下游对象ptr 
}

//场景：
A.添加下游对象(nil, B) //A 之前没有下游， 新添加一个下游对象B， B被标记为灰色
A.添加下游对象(C, B) //A 将下游对象C 更换为B， B被标记为灰色
```



上述伪代码非常好理解，当黑色对象（slot）插入新的指向白色对象（ptr）的引用关系时，就尝试使用shade函数将这个新插入的引用（ptr）标记为灰色。



![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe94Cjxng5VbT4M7FkUgyAfhpj99S1E3KkyG9kbgAWz9mcJeJthjrVDZZ47DHBs3IgiaicSxjVvhlKUsw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



假设我们上图的例子并发可达性分析中使用插入写屏障：

1.GC 将根对象Root2指向的B对象标记成黑色并将B对象指向的对象D标记成灰色；

2.用户程序修改指针，**B.next=H**这时触发写屏障将H对象标记成灰色；

3.用户程序修改指针**D.next=null**；

4.GC依次遍历程序中的H和D将它们分别标记成黑色。



### 关于栈没有写屏障的原因

 黑色对象的内存槽有两种位置, `栈`和`堆`. 栈空间的特点是**容量小**,但是**要求响应速度快,因为函数调用弹出频繁使用**, 所以“插入屏障”机制,在**栈空间的对象操作中不使用**. 而仅仅使用在堆空间对象的操作中.

**由于栈上的对象在垃圾回收中被认为是根对象，并没有写屏障，那么导致黑色的栈可能指向白色的堆对象。为了保障内存安全，Dijkstra必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之前做出权衡。**



​		接下来，我们用几张图，来模拟整个一个详细的过程， 希望您能够更可观的看清晰整体流程。

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036442131-91f36e55-5c94-4931-a140-58ff5627c681.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036449149-2fb53d7c-d351-4305-84a8-7a1b51806ce4.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036456806-6b1aeb27-831d-43d9-a79e-4dad49fea07d.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036465710-e260440e-b53d-4f76-a826-842e28666efe.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036474130-755abe1f-d070-47e6-93cf-7aa129489206.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036481384-c4e44929-09e4-4a05-81bb-b5e9ed195982.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



​		但是如果栈不添加,当全部三色标记扫描之后,栈上有可能依然存在白色对象被引用的情况(如上图的对象9).  所以要对栈重新进行三色标记扫描, 但这次为了对象不丢失, 要对本次标记扫描启动STW暂停. 直到栈空间的三色标记结束.

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036522462-5e0c1ea9-e136-45c8-9648-bf691b270431.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036531031-d37d4239-9b13-4d0e-a9cc-d7bc230d56a8.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036538543-d84895c0-451d-4c49-9c67-f77dcf5a3ae9.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

​		最后将栈和堆空间 扫描剩余的全部 白色节点清除.  这次STW大约的时间在10~100ms间.



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036559017-4564c417-9059-415c-aa81-d9504ac4e00b.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)







### **（二）删除写屏障**



Yuasa在1990年的论文Real-time garbage collection on general-purpose machines 中提出了删除写屏障，因为一旦该写屏障开始工作，它会保证开启写屏障时堆上所有对象的可达。起始时STW扫描所有的goroutine栈，保证所有堆上在用的对象都处于灰色保护下，所以也被称作**快照垃圾收集或者原始快照**（Snapshot GC），这是破坏了“对象消失”的第二个条件（赋值器删除了全部从灰色对象到该白色对象的直接或间接引用）

原始快照(Snapshot At The Beginning，SATB)。当某个时刻 的 GC Roots 确定后，当时的对象图就已经确定了。当赋值器（业务线程）从灰色或者白色对象中删除白色指针时候，写屏障会捕捉这一行为，将这一行为通知给回收器。这样，基于起始快照的解决方案保守地将其目标对象当作存活的对象，这样就绝对不会有被误回收的对象，但是有扫描工作量浮动放大的风险。术语叫做追踪波面的回退。这个操作在「修改操作前」进行，JVM中 的 G1 垃圾回收器用的也是这个思路。

```

// 黑色赋值器 Yuasa 屏障
func YuasaWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(*slot) 先将*slot标记为灰色
    *slot = ptr
}

//说明：
添加下游对象(当前下游对象slot， 新下游对象ptr) {
  //step 1
  if (当前下游对象slot是灰色 || 当前下游对象slot是白色) {
          标记灰色(当前下游对象slot)     //slot为被删除对象， 标记为灰色
  }  
  //step 2
  当前下游对象slot = 新下游对象ptr
}

//场景
A.添加下游对象(B, nil)   //A对象，删除B对象的引用。B被A删除，被标记为灰(如果B之前为白)
A.添加下游对象(B, C)     //A对象，更换下游B变成C。B被A删除，被标记为灰(如果B之前为白)
```

上述代码会在老对象的引用被删除时，将白色的老对象涂成灰色，这样删除写屏障就可以保证弱三色不变性，老对象引用的下游对象一定可以被灰色对象引用。

但是这样也会导致一个问题，由于会将**有存活可能的对象都标记成灰色**，因此最后可能会导致应该回收的对象未被回收，这个对象只有在下一个循环才会被回收，比如下图的D对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe94Cjxng5VbT4M7FkUgyAfhpTuHVXfE7fSIbu8yNpJt877FyhAQuBB96eYr2wH7QcxKBIVxNrssIyQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**由于原始快照的原因，起始也是执行STW，删除写屏障不适用于栈特别大的场景，栈越大，STW扫描时间越长。**



接下来，我们用几张图，来模拟整个一个详细的过程， 希望您能够更可观的看清晰整体流程。


![delete_barrier_1](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_1.png)

![delete_barrier_2](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_2.png)

![delete_barrier_3](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_3.png)

![delete_barrier_4](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_4.png)

![delete_barrier_5](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_5.png)

![delete_barrier_6](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_6.png)

![delete_barrier_7](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_7.png)

这种方式的回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉。

**插入写屏障和删除写屏障的短板：**

-  插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活； 
-  删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。 



### **（三）混合写屏障**

在 Go 语言 v1.7版本之前，运行时会使用Dijkstra插入写屏障保证强三色不变性，但是运行时并没有在所有的垃圾收集根对象上开启插入写屏障。因为应用程序可能包含成百上千的Goroutine，而垃圾收集的根对象一般包括全局变量和栈对象，如果运行时需要在几百个Goroutine的栈上都开启写屏障，会带来巨大的额外开销，所以 Go 团队在v1.8结合上述2种写屏障构成了混合写屏障，实现上选择了在标记阶段完成时暂停程序、将所有栈对象标记为灰色并重新扫描。

Go 语言在v1.8组合Dijkstra插入写屏障和Yuasa删除写屏障构成了如下所示的混合写屏障，该写屏障会将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色：

```

writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

避免了对栈re-scan的过程，极大的减少了STW的时间。结合了两者的优点。

最本质的区别就是：**内存屏障其实就是编译器帮你生成的一段 hook 代码**，这三个屏障的本质区别就是 hook 的时机不同而已。

------

#### (1) 混合写屏障规则

`具体操作`:

1、GC开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)，

2、GC期间，任何在栈上创建的新对象，均为黑色。

3、被删除的对象标记为灰色。

4、被添加的对象标记为灰色。

`满足`: 变形的**弱三色不变式**.

伪代码：

```go
添加下游对象(当前下游对象slot, 新下游对象ptr) {
  	//1 
	标记灰色(当前下游对象slot)    //只要当前下游对象被移走，就标记灰色
  	
  	//2 
  	标记灰色(新下游对象ptr)
  		
  	//3
  	当前下游对象slot = 新下游对象ptr
}
```

这里需要注意， 屏障技术是不在栈上应用的，因为要保证栈的运行效率。

#### (2) 混合写屏障的具体场景分析

接下来，我们用几张图，来模拟整个一个详细的过程， 希望您能够更可观的看清晰整体流程。

注意混合写屏障是Gc的一种屏障机制，所以只是当程序执行GC的时候，才会触发这种机制。

##### GC开始：扫描栈区，将可达对象全部标记为黑



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036708530-f7c50de5-6a63-45dc-baef-f53b1b42eb62.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036716310-65729a9c-d8df-40ce-9c2b-d35228278791.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

------

##### 场景一： 对象被一个堆对象删除引用，成为栈对象的下游

伪代码

```go
//前提：堆对象4->对象7 = 对象7；  //对象7 被 对象4引用
栈对象1->对象7 = 堆对象7；  //将堆对象7 挂在 栈对象1 下游
堆对象4->对象7 = null；    //对象4 删除引用 对象7
```



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036737874-a2f71441-c4f9-4f74-8c8a-c5a53bd35d4c.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036745104-24b7bf17-27b9-4531-97b7-48c5b7e64fac.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



##### 场景二： 对象被一个栈对象删除引用，成为另一个栈对象的下游

伪代码

```go
new 栈对象9；
对象8->对象3 = 对象3；      //将栈对象3 挂在 栈对象9 下游
对象2->对象3 = null；      //对象2 删除引用 对象3
```



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036778055-bda31c21-45dc-4602-9241-11a33b6393a6.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036785024-0edb665e-7b4b-46e3-b8cf-1d4ff02e73cd.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036791814-78eed337-a9ac-42d9-bcd8-99a21c01111c.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



##### 场景三：对象被一个堆对象删除引用，成为另一个堆对象的下游

伪代码

```go
堆对象10->对象7 = 堆对象7；       //将堆对象7 挂在 堆对象10 下游
堆对象4->对象7 = null；         //对象4 删除引用 对象7
```



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036826144-893174fb-0111-4838-9f7d-38fe2f89648a.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036833484-a18064d9-1329-42d7-8687-8a029542e85e.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036840569-f50df9db-5219-48fe-83ff-c3545ed4dec4.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



##### 场景四：对象从一个栈对象删除引用，成为另一个堆对象的下游

伪代码

```go
堆对象10->对象7 = 堆对象7；       //将堆对象7 挂在 堆对象10 下游
堆对象4->对象7 = null；         //对象4 删除引用 对象7
```



![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036859560-21a75ea4-ee66-46ae-81bc-ce4e697c3814.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036864959-929ec428-e8d8-48a9-aaeb-e2589723ec62.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/26269664/1651036876957-976a0ac6-6c82-4eca-88f3-10180782281c.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_55%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



​		Golang中的混合写屏障满足`弱三色不变式`，结合了删除写屏障和插入写屏障的优点，只需要在开始时并发扫描各个goroutine的栈，使其变黑并一直保持，这个过程不需要STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行re-scan操作了，减少了STW的时间。

## **五、GC演进过程**

v1.0 — 完全串行的标记和清除过程，需要暂停整个程序；

v1.1 — 在多核主机并行执行垃圾收集的标记和清除阶段；

v1.3 — 运行时**基于只有指针类型的值包含指针**的假设增加了对栈内存的精确扫描支持，实现了真正精确的垃圾收集；将unsafe.Pointer类型转换成整数类型的值认定为不合法的，可能会造成悬挂指针等严重问题；

v1.5 — 实现了基于**三色标记清扫的并发**垃圾收集器（插入写屏障）：

- 大幅度降低垃圾收集的延迟从几百 ms 降低至 10ms 以下；
- 计算垃圾收集启动的合适时间并通过并发加速垃圾收集的过程；

v1.6 — 实现了去中心化的垃圾收集协调器：

- 基于显式的状态机使得任意Goroutine都能触发垃圾收集的状态迁移；
- 使用密集的位图替代空闲链表表示的堆内存，降低清除阶段的CPU占用;

v1.7 — 通过**并行栈收缩**将垃圾收集的时间缩短至2ms以内；

v1.8 — 使用**混合写屏障**将垃圾收集的时间缩短至0.5ms以内；

v1.9 — 彻底移除暂停程序的重新扫描栈的过程；

v1.10 — 更新了垃圾收集调频器（Pacer）的实现，分离软硬堆大小的目标；

v1.12 — 使用**新的标记终止算法**简化垃圾收集器的几个阶段；

v1.13 — 通过新的 Scavenger 解决瞬时内存占用过高的应用程序向操作系统归还内存的问题；

v1.14 — 使用全新的页分配器**优化内存分配的速度**；

v1.15 — 改进编译器和运行时内部的CL 226367，它使编译器可以将更多的x86寄存器用于垃圾收集器的写屏障调用；

v1.16 — Go runtime默认使用MADV_DONTNEED更积极的将不用的内存释放给OS。

## **六、GC过程**

Golang GC 相关的代码在**runtime/mgc.go**文件下，可以看见GC总共分为4个阶段(翻译自Golang v1.16版本源码)：

**1.sweep termination（清理终止）**

- 暂停程序，触发STW。所有的P（处理器）都会进入safe-point（安全点）；

- 清理未被清理的 span 。如果当前垃圾收集是强制触发的，需要处理还未被清理的内存管理单元；

**2.the mark phase（标记阶段）**

- 将**GC状态gcphase从_GCoff改成_GCmark**、开启写屏障、启用协助线程（mutator assists）、将根对象入队；
- 恢复程序执行，标记进程（mark workers）和协助程序会开始并发标记内存中的对象，写屏障会覆盖的重写指针和新指针（标记成灰色），而所有新创建的对象都会被直接标记成黑色；
- GC执行根节点的标记，这包括扫描所有的栈、全局对象以及不在堆中的运行时数据结构。扫描goroutine栈会导致goroutine停止，并对栈上找到的所有指针加置灰，然后继续执行goroutine；
- GC遍历灰色对象队列，会将灰色对象变成黑色，并将该指针指向的对象置灰；
- 由于GC工作分布在本地缓存中，GC会使用分布式终止算法（distributed termination algorithm）来检测何时不再有根标记作业或灰色对象，如果没有了GC会转为mark termination（标记终止）。

**3. mark termination（标记终止）**

-  STW；
- 将GC状态gcphase切换至_GCmarktermination，关闭gc工作线程和协助程序；
- 执行housekeeping，例如刷新mcaches。

**4. the sweep phase（清理阶段）**

- 将GC状态gcphase切换至_GCoff来准备清理阶段，初始化清理阶段并关闭写屏障；
- 恢复用户程序，从现在开始，所有新创建的对象会标记成白色；如果有必要，在使用前分配清理spans；
- 后台并发清理所有的内存管理类单元。

**GC过程代码示例**

```

func gcfinished() *int {
  p := 1
  runtime.SetFinalizer(&p, func(_ *int) {
    println("gc finished")
  })
  return &p
}
func allocate() {
  _ = make([]byte, int((1<<20)*0.25))
}
func main() {
  f, _ := os.Create("trace.out")
  defer f.Close()
  trace.Start(f)
  defer trace.Stop()
  gcfinished()
  // 当完成 GC 时停止分配
  for n := 1; n < 50; n++ {
    println("#allocate: ", n)
    allocate()
  }
  println("terminate")
}
```

运行程序

```

hewittwang@HEWITTWANG-MB0 rtx % GODEBUG=gctrace=1 go run new1.go  
gc 1 @0.015s 0%: 0.015+0.36+0.043 ms clock, 0.18+0.55/0.64/0.13+0.52 ms cpu, 4->4->0 MB, 5 MB goal, 12 P
gc 2 @0.024s 1%: 0.045+0.19+0.018 ms clock, 0.54+0.37/0.31/0.041+0.22 ms cpu, 4->4->0 MB, 5 MB goal, 12 P
....
```

栈分析

```

gc 2      : 第一个GC周期
@0.024s   : 从程序开始运行到第一次GC时间为0.024 秒
1%        : 此次GC过程中CPU 占用率

wall clock
0.045+0.19+0.018 ms clock
0.045 ms  : STW，Marking Start, 开启写屏障
0.19 ms   : Marking阶段
0.018 ms  : STW，Marking终止，关闭写屏障

CPU time
0.54+0.37/0.31/0.041+0.22 ms cpu
0.54 ms   : STW，Marking Start
0.37 ms  : 辅助标记时间
0.31 ms  : 并发标记时间
0.041 ms   : GC 空闲时间
0.22 ms   : Mark 终止时间

4->4->0 MB， 5 MB goal
4 MB      ：标记开始时，堆大小实际值
4 MB      ：标记结束时，堆大小实际值
0 MB      ：标记结束时，标记为存活对象大小
5 MB      ：标记结束时，堆大小预测值

12 P      ：本次GC过程中使用的goroutine 数量
```



## **七、GC触发条件**

运行时会通过runtime.gcTrigger.test方法决定是否需要触发垃圾收集，当满足触发垃圾收集的基本条件（即满足_GCoff阶段的退出条件）时——允许垃圾收集、程序没有崩溃并且没有处于垃圾收集循环，该方法会根据三种不同方式触发进行不同的检查：

```

//mgc.go 文件 runtime.gcTrigger.test
 func (t gcTrigger) test() bool {
    //测试是否满足触发垃圾手机的基本条件
    if !memstats.enablegc || panicking != 0 || gcphase != _GCoff {
       return false
    }
    switch t.kind {
      case gcTriggerHeap:    //堆内存的分配达到达控制器计算的触发堆大小
         // Non-atomic access to gcController.heapLive for performance. If
         // we are going to trigger on this, this thread just
         // atomically wrote gcController.heapLive anyway and we'll see our
         // own write.
         return gcController.heapLive >= gcController.trigger
      case gcTriggerTime:      //如果一定时间内没有触发，就会触发新的循环，该出发条件由 `runtime.forcegcperiod`变量控制，默认为 2 分钟；
         if gcController.gcPercent < 0 {
            return false
        }
         lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
         return lastgc != 0 && t.now-lastgc > forcegcperiod
      case gcTriggerCycle:      //如果当前没有开启垃圾收集，则触发新的循环；
         // t.n > work.cycles, but accounting for wraparound.
         return int32(t.n-work.cycles) > 0
    }
    return true
 }
```

用于开启垃圾回收的方法为runtime.gcStart，因此所有调用该函数的地方都是触发GC的代码：

- runtime.mallocgc申请内存时根据堆大小触发GC
- runtime.GC用户程序手动触发GC
- runtime.forcegchelper后台运行定时检查触发GC

**（一）申请内存触发runtime.mallocgc**

Go运行时会将堆上的对象按大小分成微对象、小对象和大对象三类，这三类对象的创建都可能会触发新的GC。

1.当前线程的内存管理单元中不存在空闲空间时，创建微对象(noscan &&size<maxTinySize)和小对象需要调用 runtime.mcache.nextFree从中心缓存或者页堆中获取新的管理单元，这时如果span满了就会导致返回的shouldhelpgc=true，就可能触发垃圾收集；

2.当用户程序申请分配32KB以上的大对象时，一定会构建 runtime.gcTrigger结构体尝试触发垃圾收集。

```

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    省略代码 ...
    shouldhelpgc := false  
  dataSize := size
  c := getMCache()       //尝试获取mCache。如果没启动或者没有P,返回nil；
 
    省略代码 ...
    if size <= maxSmallSize {  
       if noscan && size < maxTinySize { // 微对象分配
  省略代码 ...
          v := nextFreeFast(span)
          if v == 0 {
             v, span, shouldhelpgc = c.nextFree(tinySpanClass)
          }
      省略代码 ...
      } else {      //小对象分配
         省略代码 ...
          if v == 0 {
             v, span, shouldhelpgc = c.nextFree(spc)
          }
        省略代码 ...
      }
    } else {
       shouldhelpgc = true
       省略代码 ...
    }
  省略代码 ...
    if shouldhelpgc {      //是否应该触发gc
      if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {   //如果满足gc触发条件就调用gcStart()
          gcStart(t)
      }
    }
  省略代码 ...
    return x
 }
```

这个时候调用t.test()执行的是gcTriggerHeap情况，只需要判断gcController.heapLive >= gcController.trigger的真假就可以了。 heapLive表示垃圾收集中存活对象字节数，trigger表示触发标记的堆内存大小的；当内存中存活的对象字节数大于触发垃圾收集的堆大小时，新一轮的垃圾收集就会开始。

1.heapLive — 为了减少锁竞争，运行时只会在中心缓存分配或者释放内存管理单元以及在堆上分配大对象时才会更新；

2.trigger — 在标记终止阶段调用runtime.gcSetTriggerRatio更新触发下一次垃圾收集的堆大小，它能够决定触发垃圾收集的时间以及用户程序和后台处理的标记任务的多少，利用反馈控制的算法根据堆的增长情况和垃圾收集CPU利用率确定触发垃圾收集的时机。

**（二）手动触发runtime.GC**

用户程序会通过runtime.GC函数在程序运行期间主动通知运行时执行，该方法在调用时会阻塞调用方直到当前垃圾收集循环完成，在垃圾收集期间也可能会通过STW暂停整个程序：

```

func GC() {
    //在正式开始垃圾收集前，运行时需要通过runtime.gcWaitOnMark等待上一个循环的标记终止、标记和清除终止阶段完成；
    n := atomic.Load(&work.cycles)
    gcWaitOnMark(n)
 
  //调用 `runtime.gcStart` 触发新一轮的垃圾收集
    gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})
 
    //`runtime.gcWaitOnMark` 等待该轮垃圾收集的标记终止阶段正常结束；
    gcWaitOnMark(n + 1)
 
    // 持续调用 `runtime.sweepone` 清理全部待处理的内存管理单元并等待所有的清理工作完成
    for atomic.Load(&work.cycles) == n+1 && sweepone() != ^uintptr(0) {
        sweep.nbgsweep++
        Gosched()  //等待期间会调用 `runtime.Gosched` 让出处理器
    }
 
    //
    for atomic.Load(&work.cycles) == n+1 && !isSweepDone() {
        Gosched()
    }
 
    // 完成本轮垃圾收集的清理工作后，通过 `runtime.mProf_PostSweep` 将该阶段的堆内存状态快照发布出来，我们可以获取这时的内存状态
    mp := acquirem()
    cycle := atomic.Load(&work.cycles)
    if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {   //仅限于没有启动其他标记终止过程
        mProf_PostSweep()
    }
    releasem(mp)
}
```



**（三）后台运行定时检查触发runtime.forcegchelper**

运行时会在应用程序启动时在后台开启一个用于强制触发垃圾收集的Goroutine，该Goroutine调用runtime.gcStart尝试启动新一轮的垃圾收集：

```

// start forcegc helper goroutine
func init() {
   go forcegchelper()
}
 
func forcegchelper() {
   forcegc.g = getg()
   lockInit(&forcegc.lock, lockRankForcegc)
   for {
      lock(&forcegc.lock)
      if forcegc.idle != 0 {
         throw("forcegc: phase error")
      }
      atomic.Store(&forcegc.idle, 1)
      
     //该 Goroutine 会在循环中调用runtime.goparkunlock主动陷入休眠等待其他 Goroutine 的唤醒
      goparkunlock(&forcegc.lock, waitReasonForceGCIdle, traceEvGoBlock, 1)
       
      if debug.gctrace > 0 {
         println("GC forced")
      }
      // Time-triggered, fully concurrent.
      gcStart(gcTrigger{kind: gcTriggerTime, n
      ow: nanotime()})
   }
}
```

## 八、问题思考

1.为什么删除写屏障的时候要原始快照？

2.删除写屏障出现已扫描黑色对象新增白色对象的怎么处理？

3.关于内存管理，gc整体流程，go如何将代码转化为二进制？



**参考文献**

  1.《Go语言设计与实现》

(https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/)

  2.《一个专家眼中的Go与Java垃圾回收算法大对比》

(https://blog.csdn.net/u011277123/article/details/53991572)

  3.《Go语言问题集》

(https://www.bookstack.cn/read/qcrao-Go-Questions/spilt.19.GC-GC.md)

   4.《CMS垃圾收集器》

(https://juejin.cn/post/6844903782107578382)

  5.《Golang v 1.16版本源码》

(https://github.com/golang/go)

  6.《Golang---内存管理(内存分配)》

(http://t.zoukankan.com/zpcoding-p-13259943.html)

  7.《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》—机械工业出版社

  8.《腾讯妹子图解Golang内存分配和垃圾回收》](https://mp.weixin.qq.com/s/iAy9ReQhnmCYUFvwYroGPA)

  9.[《Golang修养之路》](https://www.yuque.com/aceld/golang/zhzanb)

10. https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/barrier/

11. https://liqingqiya.github.io/golang/gc/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/2020/06/02/gc2.html
