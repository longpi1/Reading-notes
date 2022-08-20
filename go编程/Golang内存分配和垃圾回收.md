# Golang内存分配和垃圾回收

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

**1.Page**

与TCMalloc中的Page相同，x64下1个Page的大小是8KB。上图的最下方，1个浅蓝色的长方形代表1个Page。

**2.Span**

与TCMalloc中的Span相同，Span是内存管理的基本单位，代码中为mspan，一组连续的Page组成1个Span，所以上图一组连续的浅蓝色长方形代表的是一组Page组成的1个Span，另外，1个淡紫色长方形为1个Span。

**3.mcache**

mcache是提供给P（逻辑处理器）的高速缓存，用于存储小对象（对象大小<= 32Kb）。尽管这类似于线程堆栈，但它是堆的一部分，用于动态数据。所有类大小的mcache包含scan和noscan类型mspan。Goroutine可以从mcache没有任何锁的情况下获取内存，因为一次P只能有一个锁G。因此，这更有效。mcache从mcentral需要时请求新的span。

**4.mcentral**

mcentral与TCMalloc中的CentralCache类似，是所有线程共享的缓存，需要加锁访问，它按Span class对Span分类，串联成链表，当mcache的某个级别Span的内存被分配光时，它会向mcentral申请1个当前级别的Span。每个mcentral包含两个mspanList：

- empty：双向span链表，包括没有空闲对象的span或缓存mcache中的span。当此处的span被释放时，它将被移至non-empty span链表。
- non-empty：有空闲对象的span双向链表。当从mcentral请求新的span，mcentral将从该链表中获取span并将其移入empty span链表。

**5.mheap**

mheap与TCMalloc中的PageHeap类似，它是堆内存的抽象，也是垃圾回收的重点区域，把从OS申请出的内存页组织成Span，并保存起来。当mcentral的Span不够用时会向mheap申请，mheap的Span不够用时会向OS申请，向OS的内存申请是按页来的，然后把申请来的内存页生成Span组织起来，同样也是需要加锁访问的。

**6.栈**

这是栈存储区，每个Goroutine（G）有一个栈。在这里存储了静态数据，包括函数栈帧，静态结构，原生类型值和指向动态结构的指针。这与分配给每个P的mcache不是一回事。



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

## **二、 标记清除算法**

当成功区分出 Go 垃圾收集器管理区域的存活对象和死亡对象后，Go 垃圾收集器接下来的任务就是执行GC，释放无用对象占用的内存空间，以便有足够的可用内存空间为新对象分配内存。目前常见的垃圾回收算法在上篇《[自动的内存管理系统实操手册——Java垃圾回收篇](http://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247518133&idx=1&sn=186ef93a6add47b01cac2f355211f6cd&chksm=eaa89de5dddf14f39b4172adb6bf1f964252a244f9887286a6cae30aefe2b27833db9407d58f&scene=21#wechat_redirect)》一文中的“垃圾收集算法”部分已有介绍，而**Go使用的是标记清除算法**，这是一种非常基础和常见的垃圾收集算法，于1960年被J.McCarthy等人提出。

当堆空间被耗尽的时，就会STW(也被称为stop the world)，其执行过程可以分成标记和清除两个阶段。Go 垃圾收集器从根结点开始遍历，执行可达性分析算法，递归标记所有被引用的对象为存活状态；标记阶段结束后，垃圾收集器会依次遍历堆中的对象并清除其中的未被标记为存活的对象。

由于用户程序在垃圾收集的过程中也不能执行（STW）。在可达性分析算法中，Go 的GC Roots一般为全局变量和G Stack中的引用指针，和整堆的对象相比只是极少数，因此它带来的停顿是非常短暂且相对固定的，不随堆容量增长。在从GC Roots往下遍历对象的过程，堆越大，存储对象越多，递归遍历越复杂，要标记更多对象而产生的停顿时间自然就更长。因此我们需要用到更复杂的机制来解决STW的问题。

## **三、三色可达性分析**

为了解决标记清除算法带来的STW问题，Go和Java都会实现三色可达性分析标记算法的变种以缩短STW的时间。三色可达性分析标记算法按“是否被访问过”将程序中的对象分成白色、黑色和灰色：

- 白色对象 — 对象尚未被垃圾收集器访问过，在可达性分析刚开始的阶段，所有的对象都是白色的，若在分析结束阶段，仍然是白色的对象，即代表不可达。
- 黑色对象 — 表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经被扫描过，黑色的对象代表已经被扫描过而且是安全存活的，如果有其他对象只想黑色对象无需再扫描一遍，黑色对象不可能直接（不经过灰色对象）指向某个白色对象。
- 灰色对象 — 表示对象已经被垃圾收集器访问过，但是这个对象上至少存在一个引用还没有被扫描过，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象。

三色可达性分析算法大致的流程是（初始状态所有对象都是白色）：

1.从GC Roots开始枚举，它们所有的直接引用变为灰色（移入灰色集合），GC Roots变为黑色。

2.从灰色集合中取出一个灰色对象进行分析：

- 将这个对象所有的直接引用变为灰色，放入灰色集合中；
- 将这个对象变为黑色。

3.重复步骤2，一直重复直到灰色集合为空。

4.分析完成，仍然是白色的对象就是GC Roots不可达的对象，可以作为垃圾被清理。



具体例子如下图所示，经过三色可达性分析，最后白色H为不可达的对象，是需要垃圾回收的对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe94Cjxng5VbT4M7FkUgyAfhpTg82w6bpdH5FmHRajnIIJTbqbEAqJUZYYUw59vkeOzoxYkHjhOPTCg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

三色标记清除算法本身是不可以并发或者增量执行的，它需要STW，而如果并发执行，用户程序可能在标记执行的过程中修改对象的指针。

![图片](https://mmbiz.qpic.cn/mmbiz_png/VY8SELNGe94Cjxng5VbT4M7FkUgyAfhpDGBI8liaibcvyXxOjP7kowzG1TnVmgAJefhegPo2IJiabXQ6IxnRdVqPQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**这种异常情况一般会有2种：**

1.一种是把原本应该垃圾回收的死亡对象错误的标记为存活。虽然这不好，但是不会导致严重后果，只不过产生了一点逃过本次回收的浮动垃圾而已，下次清理就可以，比如上图所示的三色标记过程中，用户程序取消了从B对象到E对象的引用，但是因为B到E已经被标记完成不会继续执行步骤2，所以E对象最终会被错误的标记成黑色，不会被回收，这个D就是浮动垃圾，会在下次垃圾收集中清理。

2.一种是把原本存活的对象错误的标记为已死亡，导致“对象消失”，这在内存管理中是非常严重的错误。比如上图所示的三色标记过程中，用户程序建立了从B对象到H对象的引用(例如**B.next =H**)，接着执行**D.next=nil**，但是因为B到H中不存在灰色对象，因此在这之间不会继续执行三色并发标记中的步骤2，D到H之间的链接被断开，所以H对象最终会被标记成白色，会被垃圾收集器错误地回收。我们将这种错误称为**悬挂指针**，即指针没有指向特定类型的合法对象，影响了内存的安全性。



## **四、屏障技术**

为了解决上述的“对象消失”的现象，Wilson于1994年在理论上证明了，当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：

- 赋值器插入了一条或多条从黑色对象到白色对象的新引用；
- 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。

因此为了我们要解决并发扫描时的对象消失问题，保证垃圾收集算法的正确性，只需破坏这两个条件的任意一个即可，**屏障技术**就是在并发或者增量标记过程中保证三色不变性的重要技术。

内存屏障技术是一种屏障指令，它可以让CPU或者编译器在执行内存相关操作时遵循特定的约束，目前多数的现代处理器都会乱序执行指令以最大化性能，但是该技术能够保证内存操作的顺序性，在内存屏障前执行的操作一定会先于内存屏障后执行的操作。垃圾收集中的屏障技术更像是一个钩子方法，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，根据操作类型的不同，我们可以将它们分成读屏障（Read barrier）和写屏障（Write barrier）两种，因为读屏障需要在读操作中加入代码片段，对用户程序的性能影响很大，所以编程语言往往都会采用写屏障保证三色不变性

**（一）插入写屏障**

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

由于栈上的对象在垃圾回收中被认为是根对象，并没有写屏障，那么导致黑色的栈可能指向白色的堆对象，例如上图1中Root2指向H，且删除了由D指向H的引用，由于没有写屏障，那么H将会被删除。为了保障内存安全，Dijkstra必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序，垃圾收集算法的设计者需要在这两者之前做出权衡。

**（二）删除写屏障**

Yuasa在1990年的论文Real-time garbage collection on general-purpose machines 中提出了删除写屏障，因为一旦该写屏障开始工作，它会保证开启写屏障时堆上所有对象的可达。起始时STW扫描所有的goroutine栈，保证所有堆上在用的对象都处于灰色保护下，所以也被称作快照垃圾收集（Snapshot GC），这是破坏了“对象消失”的第二个条件（赋值器删除了全部从灰色对象到该白色对象的直接或间接引用）。

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

由于原始快照的原因，起始也是执行STW，删除写屏障不适用于栈特别大的场景，栈越大，STW扫描时间越长。

**（三）混合写屏障**

在 Go 语言 v1.7版本之前，运行时会使用Dijkstra插入写屏障保证强三色不变性，但是运行时并没有在所有的垃圾收集根对象上开启插入写屏障。因为应用程序可能包含成百上千的Goroutine，而垃圾收集的根对象一般包括全局变量和栈对象，如果运行时需要在几百个Goroutine的栈上都开启写屏障，会带来巨大的额外开销，所以 Go 团队在v1.8结合上述2种写屏障构成了混合写屏障，实现上选择了在标记阶段完成时暂停程序、将所有栈对象标记为灰色并重新扫描。

Go 语言在v1.8组合Dijkstra插入写屏障和Yuasa删除写屏障构成了如下所示的混合写屏障，该写屏障会将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色：

```

writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

为了移除栈的重扫描过程，除了引入混合写屏障之外，在垃圾收集的标记阶段，我们还需要将创建的所有新对象都标记成黑色，防止新分配的栈内存和堆内存中的对象被错误地回收，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。总结来说主要有这几点：

- GC开始将栈上的对象全部扫描并标记为黑色；
- GC期间，任何在栈上创建的新对象，均为黑色；
- 被删除的堆对象标记为灰色；
- 被添加的堆对象标记为灰色。



## **五、GC演进过程**

v1.0 — 完全串行的标记和清除过程，需要暂停整个程序；

v1.1 — 在多核主机并行执行垃圾收集的标记和清除阶段；

v1.3 — 运行时**基于只有指针类型的值包含指针**的假设增加了对栈内存的精确扫描支持，实现了真正精确的垃圾收集；将unsafe.Pointer类型转换成整数类型的值认定为不合法的，可能会造成悬挂指针等严重问题；

v1.5 — 实现了基于**三色标记清扫的并发**垃圾收集器：

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
      gcStart(gcTrigger{kind: gcTriggerTime, now: nanotime()})
   }
}
```

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

8. [《腾讯妹子图解Golang内存分配和垃圾回收》](https://mp.weixin.qq.com/s/iAy9ReQhnmCYUFvwYroGPA)