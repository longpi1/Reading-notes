# 源码分析 kubernetes scheduler 核心调度器的实现原理

> 主要内容转载自https://github.com/rfyiamcool/notes

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301251502086.png)



> 基于 kubernetes `v1.27.0` 源码分析 scheduler 调度器

k8s scheduler 的主要职责是为新创建的 pod 寻找一个最合适的 node 节点, 然后进行 bind node 绑定, 后面 kubelet 才会监听到并创建真正的 pod.

那么问题来了, 如何为 pod 寻找最合适的 node ? 调度器需要经过 predicates 预选和 priority 优选.

- 预选就是从集群的所有节点中根据调度算法筛选出所有可以运行该 pod 的节点集合
- 优选则是按照算法对预选出来的节点进行打分，找到分值最高的节点作为调度节点.

选出最优节点后, 对 apiserver 发起 pod 节点 bind 操作, 其实就是对 pod 的 spec.NodeName 赋值最优节点.

**源码基本调用关系**

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202212/k8s-scheduler.jpg)

## k8s scheduler 启动入口

k8s scheduler 在启动时会调用在 cobra 注册的 `setup` 来初始化 scheduler 调度器对象.

代码位置: `cmd/kube-scheduler/app/server.go`

```go
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	// 获取默认配置
	if cfg, err := latest.Default(); err != nil {
		return nil, nil, err
	} else {
		opts.ComponentConfig = cfg
	}

	// 验证 scheduler 的配置参数
	if errs := opts.Validate(); len(errs) > 0 {
		return nil, nil, utilerrors.NewAggregate(errs)
	}

	c, err := opts.Config()
	if err != nil {
		return nil, nil, err
	}

	// 配置中填充和调整
	cc := c.Complete()

	...

	// 构建 scheduler 对象
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		cc.DynInformerFactory,
		recorderFactory,
		ctx.Done(),
		...
	)
	if err != nil {
		return nil, nil, err
	}
	...

	return &cc, sched, nil
}
```

实例化 kubernetes scheduler 对象, 初始化流程直接看下面代码.

代码位置: `pkg/scheduler/scheduler.go`

```go
// New returns a Scheduler
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {


	// 构建 registry 对象, 默认集成了一堆的插件
	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	// profiles 用来保存不同调度器的 framework 框架, framework 则用来存放 plugin.
	profiles, err := profile.NewMap(options.profiles, registry, recorderFactory, stopCh,
		frameworkruntime.WithComponentConfigVersion(options.componentConfigVersion),
		frameworkruntime.WithClientSet(client),
		frameworkruntime.WithKubeConfig(options.kubeConfig),
		frameworkruntime.WithInformerFactory(informerFactory),
		...
		...
	)

	// 实例化快照
	snapshot := internalcache.NewEmptySnapshot()

	// 实例化 queue, 该 queue 为 PriorityQueue.
	podQueue := internalqueue.NewSchedulingQueue(
		profiles[options.profiles[0].SchedulerName].QueueSortFunc(),
		informerFactory,
		...
	)

	// 实例化 cache 缓存
	schedulerCache := internalcache.New(durationToExpireAssumedPod, stopEverything)

	// 实例化 scheduler 对象
	sched := &Scheduler{
		Cache:                    schedulerCache,
		client:                   client,
		nodeInfoSnapshot:         snapshot,
		NextPod:                  internalqueue.MakeNextPodFunc(podQueue),
		StopEverything:           stopEverything,
		SchedulingQueue:          podQueue,
	}
	sched.applyDefaultHandlers()

	// 在 informer 里注册自定义的事件处理方法
	addAllEventHandlers(sched, informerFactory, dynInformerFactory, unionedGVKs(clusterEventMap))

	return sched, nil
}

func (s *Scheduler) applyDefaultHandlers() {
	s.SchedulePod = s.schedulePod
	s.FailureHandler = s.handleSchedulingFailure
}
```

## 注册在 scheduler informer 的回调方法

在 informer 里注册 pod 和 node 资源的回调方法，监听 pod 事件对 queue 和 cache 做回调处理. 监听 node 事件对 cache 做处理.

```go
func addAllEventHandlers(
	sched *Scheduler,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	gvkMap map[framework.GVK]framework.ActionType,
) {
	// 监听 pod 事件，并注册增删改回调方法, 其操作是对 cache 的增删改
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				...
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToCache,
				UpdateFunc: sched.updatePodInCache,
				DeleteFunc: sched.deletePodFromCache,
			},
		},
	)

	// 监听 pod 事件，并注册增删改方法, 对 queue 插入增删改事件.
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				...
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
		},
	)

	// 监听 node 事件，注册回调方法，该方法在 cache 里对 node 的增删改查.
	informerFactory.Core().V1().Nodes().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.addNodeToCache,
			UpdateFunc: sched.updateNodeInCache,
			DeleteFunc: sched.deleteNodeFromCache,
		},
	)
}
```

## scheduler 的选举实现

`kube-scheduler` 跟 k8s 的其他主控组件一样, 也会通过选举 `leaderelection` 机制保证集群只有一个 leader 实例运行调度器, 其他 follower 实例则尝试轮询抢锁直到成功.

源码位置: `cmd/kube-scheduler/app/server.go`

```go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	...

	waitingForLeader := make(chan struct{})
	isLeader := func() bool {
		select {
		case _, ok := <-waitingForLeader:
			// if channel is closed, we are leading
			return !ok
		default:
			// channel is open, we are waiting for a leader
			return false
		}
	}


	// 启动 informers, 这里只有 pod 和 node.
	cc.InformerFactory.Start(ctx.Done())

	// 同步 informer 的数据到本地缓存
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

	// 如果在配置中启动了选举, 创建选举对象, 注册事件方法, 并启用选举.
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				// 当选举拿到 leader 时, 启动 scheduler 调度器
				close(waitingForLeader)
				sched.Run(ctx)
			},
			OnStoppedLeading: func() {
				// 当选举成功但后面又丢失 leader 后, 则退出进程.
				// 进程退出后, 会被 docker 或 systemd 重新拉起, 尝试拿锁.
				select {
				case <-ctx.Done():
					// We were asked to terminate. Exit 0.
					os.Exit(0)
				default:
					// We lost the lock.
					klog.ErrorS(nil, "Leaderelection lost")
					klog.FlushAndExit(klog.ExitFlushTimeout, 1)
				}
			},
		}

		// 构建 leaderelection 对象
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		// 启动选举
		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}

	// 如果没有开启选举, 则直接启动 scheduler 调度器.
	close(waitingForLeader)
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```

关于 client-go LeaderElection 选举的实现原理, 请点击下面连接. 

[https://github.com/rfyiamcool/notes/blob/main/kubernetes_leader_election_code.md](https://github.com/rfyiamcool/notes/blob/main/kubernetes_leader_election_code.md)

## scheudler 启动入口

`Run()` 方法是 k8s scheduler 的启动运行入口, 其流程是先启动 queue 队列的 Run 方法, 再异步启动一个协程处理核心调度方法 `scheduleOne`.

`schedulingQueue` 的 `Run()` 方法用来监听内部的延迟任务, 把到期的任务放到 activeQ 中.

而 `scheduleOne` 方法用来从优先级队列里获取由 informer 插入的 pod 对象, 调用 `schedulingCycle` 为 pod 选择最优的 node 节点. 如果找到了合适的 node 节点, 则调用 `bindingCycle` 方法来发起 pod 和 node 绑定.

源码位置: `pkg/scheduler/scheduler.go`

```go
func (sched *Scheduler) Run(ctx context.Context) {
	sched.SchedulingQueue.Run()

	go wait.UntilWithContext(ctx, sched.scheduleOne, 0)

	<-ctx.Done()
	sched.SchedulingQueue.Close()
}

func (sched *Scheduler) scheduleOne(ctx context.Context) {
	// 从 activeQ 中获取需要调度的 pod 数据
	podInfo := sched.NextPod()
	pod := podInfo.Pod

	// 为 pod 选择最优的 node 节点
	scheduleResult, assumedPodInfo, err := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
	if err != nil {
		// 如何没有找到节点，则执行失败方法.
		sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, err, scheduleResult.reason, scheduleResult.nominatingInfo, start)
		return
	}

	go func() {
		// 像 apiserver 发起 pod -> node 绑定
		status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
		if !status.IsSuccess() {
			sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
		}
	}()
}
```

`NextPod` 底层引用了 `MakeNextPodFunc` 方法, 其内部从 `PriorityQueue` 队列中获取 pod 对象.

```go
func MakeNextPodFunc(queue SchedulingQueue) func() *framework.QueuedPodInfo {
	return func() *framework.QueuedPodInfo {
		podInfo, err := queue.Pop()
		if err == nil {
			return podInfo
		}
		return nil
	}
}

func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
	p.lock.Lock()
	defer p.lock.Unlock()

	for p.activeQ.Len() == 0 {
		// 没有数据则陷入条件变量的等待接口
		p.cond.Wait()
	}
	obj, err := p.activeQ.Pop()
	if err != nil {
		return nil, err
	}
	pInfo := obj.(*framework.QueuedPodInfo)
	pInfo.Attempts++  // 每次 pop 后都增加下 attempts 次数.
	return pInfo, nil
}
```

## 单实例单协程模型的 schduler 调度器

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301261858892.png)

需要关注的是整个 kubernetes scheduler 调度器只有一个协程处理主调度循环 `scheduleOne`, 虽然 kubernetes scheduler 可以启动多个实例, 但启动时需要 leaderelection 选举, 只有 leader 才可以处理调度, 其他节点作为 follower 等待 leader 失效. 也就是说整个 k8s 集群调度核心的并发度为 1 个. 

云原生社区中有人使用 kubemark 模拟 2000 个节点的规模来压测 kube-scheduler 处理性能及时延, 测试结果是 30s 内完成 15000 个 pod 调度任务. 虽然 kube-scheduler 是单并发模型, 但由于预选和优选都属于计算型任务非阻塞IO, 又有 `percentageOfNodesToScore` 参数优化, 最重要的是创建 pod 的操作通常不会太高并发. 这几点下来单并发模型的 scheduler 也还可以接受的.

### 为什么 scheduler 不支持并发 ?

按照当前 scheudler 调度器的设计原理, 使用预选和优选算法选出最合适的节点, 并发场景下无法保证安全, 比如, 选出的最优节点在并发下会被多个 pod 绑定.

### 使用自定义调度器进行并发调度 ?

k8s 默认的调度器为 `default-scheduler`, 而使用相同调度器只能单并发处理调度. 但是可以使用自定义实现调度器的方案, 在创建 pod 时指定不同的调度器算法 `pod.Spec.schedulerName = xiaorui.cc`, 这样可以使不同调度器的 kube-schedueler 并行调度起来, 各自按照调度算法来调度, 运行互不影响.

当然大多数公司没这个必要.

## schedulingCycle 核心调度周期的实现

`schedulingCycle()` 该方法主要为 pod 选出最优的 node 节点. 先通过预选过程过滤出符合 pod 要求的节点集合, 再通过插件对这些节点进行打分, 使用分值最高的 node 为 pod 调度节点.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301251629071.png)

scheduler 内置各个阶段的各种插件, 预选和优选阶段就是遍历回调插件求出结果.

调度周期 `schedulingCycle` 内关键方法是 `schedulePod`, 其简化流程如下.

1. 先调用 `findNodesThatFitPod` 过滤出符合要求的预选节点.
2. 调用 `prioritizeNodes` 为预选出来的节点进行打分 score.
3. 最后调用 `selectHost` 选择最合适的 node 节点.

```go
func (sched *Scheduler) schedulingCycle(
	...
) (ScheduleResult, *framework.QueuedPodInfo, error) {

	pod := podInfo.Pod

	// 选择节点
	scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
	...

	// 在缓存 cache 中更新状态
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	...
}

func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	// 更新快照
	if err := sched.Cache.UpdateSnapshot(sched.nodeInfoSnapshot); err != nil {
		return result, err
	}

	// 进行预选筛选
	feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}

	// 预选下来，无节点可以用, 返回错误
	if len(feasibleNodes) == 0 {
		return result, &framework.FitError{
			Pod:         pod,
			NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
			Diagnosis:   diagnosis,
		}
	}

	// 经过预选就只有一个节点，那么直接返回
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(diagnosis.NodeToStatusMap),
			FeasibleNodes:  1,
		}, nil
	}

	// 进行优选过程, 对预选出来的节点进行打分
	priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}

	// 从优选中选出最高分的节点
	host, err := selectHost(priorityList)

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

### findNodesThatFitPod

`findNodesThatFitPod` 方法用来实现调度器的预选过程, 其内部调用插件的 PreFilter 和 Filter 方法来筛选出符合 pod 要求的 node 节点集合.

```go
func (sched *Scheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) {
	diagnosis := framework.Diagnosis{
		NodeToStatusMap:      make(framework.NodeToStatusMap),
		UnschedulablePlugins: sets.NewString(),
	}

	// 获取所有的 nodes
	allNodes, err := sched.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, diagnosis, err
	}

	// 调用 framework 的 PreFilter 集合里的插件
	preRes, s := fwk.RunPreFilterPlugins(ctx, state, pod)
	if !s.IsSuccess() {
		// 如果在 prefilter 有异常, 则直接跳出.
		if !s.IsUnschedulable() {
			return nil, diagnosis, s.AsError()
		}
		msg := s.Message()
		diagnosis.PreFilterMsg = msg
		return nil, diagnosis, nil
	}

	...

	// 根据 prefilter 拿到的 node names 获取 node info 对象.
	nodes := allNodes
	if !preRes.AllNodes() {
		nodes = make([]*framework.NodeInfo, 0, len(preRes.NodeNames))
		for n := range preRes.NodeNames {
			nInfo, err := sched.nodeInfoSnapshot.NodeInfos().Get(n)
			if err != nil {
				return nil, diagnosis, err
			}
			nodes = append(nodes, nInfo)
		}
	}

	// 运行 framework 的 filter 插件判断 node 是否可以运行新 pod.
	feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, nodes)

	...

	// 调用额外的 extender 调度器来进行预选
	feasibleNodes, err = findNodesThatPassExtenders(sched.Extenders, pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, diagnosis, err
	}
	return feasibleNodes, diagnosis, nil
}
```

#### findNodesThatPassFilters 并发执行 Filter 插件方法

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301252324752.png)

`findNodesThatPassFilters` 方法用来遍历执行 framework 里 Filter 插件集合的 Filter 方法.

为了加快执行效率, 减少预选阶段的时延, framework 内部有个 Parallelizer 并发控制器, 启用 16 个协程并发调用插件的 Filter 方法. 在大集群下 nodes 节点会很多, 为了避免遍历全量的 nodes 执行 Filter 和后续的插件逻辑, 这里通过 `numFeasibleNodesToFind` 方法来减少扫描计算的 nodes 数量.

当成功执行 filter 插件方法的数量超过 numNodesToFind 时, 则执行 context cancel(). 这样 framework 并发协程池监听到 ctx 被关闭后, 则不会继续执行后续的任务.

```go
func (sched *Scheduler) findNodesThatPassFilters(
	ctx context.Context,
	fwk framework.Framework,
	pod *v1.Pod,
	nodes []*framework.NodeInfo) ([]*v1.Node, error) {

	numAllNodes := len(nodes)
	// 计算需要扫描的 nodes 数, 避免了超大集群下 nodes 的计算数量.
	// 当集群的节点数小于 100 时, 则直接使用集群的节点数作为扫描数据量
	// 当大于 100 时, 则使用公式计算 `numAllNodes * (50 - numAllNodes/125) / 100`
	numNodesToFind := sched.numFeasibleNodesToFind(fwk.PercentageOfNodesToScore(), int32(numAllNodes))

	feasibleNodes := make([]*v1.Node, numNodesToFind)

	// 如果 framework 未注册 Filter 插件, 则退出.
	if !fwk.HasFilterPlugins() {
		for i := range feasibleNodes {
			feasibleNodes[i] = nodes[(sched.nextStartNodeIndex+i)%numAllNodes].Node()
		}
		return feasibleNodes, nil
	}

	// framework 内置并发控制器, 并发 16 个协程去请求插件的 Filter 方法.
	errCh := parallelize.NewErrorChannel()
	var statusesLock sync.Mutex

	checkNode := func(i int) {
		// 获取 node info 对象
		nodeInfo := nodes[(sched.nextStartNodeIndex+i)%numAllNodes]

		// 遍历执行 framework 的 Filter 插件的 Filter 方法.
		status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
		if status.Code() == framework.Error {
			// 如果有错误, 直接把错误传到 errCh 管道里.
			errCh.SendErrorWithCancel(status.AsError(), cancel)
			return
		}
		if status.IsSuccess() {
			// 如果成功执行 Filter 插件的数量超过 numNodesToFind, 则执行 cancel().
			// 当 ctx 被 cancel(), framework 的并发协程池不会继续执行后续的任务.
			length := atomic.AddInt32(&feasibleNodesLen, 1)
			if length > numNodesToFind {
				cancel()
				atomic.AddInt32(&feasibleNodesLen, -1)
			} else {
				feasibleNodes[length-1] = nodeInfo.Node()
			}
		}
		...
	}

	// 并发调用 framework 的 Filter 插件的 Filter 方法.
	fwk.Parallelizer().Until(ctx, numAllNodes, checkNode, frameworkruntime.Filter)
	feasibleNodes = feasibleNodes[:feasibleNodesLen]
	if err := errCh.ReceiveError(); err != nil {
		// 当有错误时, 直接返回.
		statusCode = framework.Error
		return feasibleNodes, err
	}

	// 返回可用的 nodes 列表.
	return feasibleNodes, nil
}
```

framework 的并发库通过封装 `workqueue.ParallelizeUntil` 来实现.

`k8s.io/client-go/util/workqueue/parallelizer.go`

#### numFeasibleNodesToFind 计算多少节点参与预选

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202301/202301261120469.png)

🤔 考虑一个问题, 当 k8s 的 node 节点特别多时, 这些节点都要参与预先的调度过程么 ? 比如大集群有 2500 个节点, 注册的插件有 10 个, 那么 筛选 Filter 和 打分 Score 过程需要进行 2500 * 10 * 2 = 50000 次计算, 最后选定一个最高分值的节点来绑定 pod. k8s scheduler 考虑到了这样的性能开销, 所以加入了百分比参数控制参与预选的节点数.

`numFeasibleNodesToFind` 方法根据当前集群的节点数计算出参与预选的节点数量, 把参与 Filter 的节点范围缩小, 无需全面扫描所有的节点, 这样避免 k8s 集群 nodes 太多时, 造成无效的计算资源开销.

`numFeasibleNodesToFind` 策略是这样的, 当集群节点小于 100 时, 集群中的所有节点都参与预选. 而当大于 100 时, 则使用下面的公式计算扫描数. scheudler 的 `percentageOfNodesToScore` 参数默认为 0, 源码中会赋值为 50 %.

```
numAllNodes * (50 - numAllNodes/125) / 100
```

假设当前集群有 `500` 个 nodes 节点, 那么需要执行 Filter 插件方法的 nodee 节点有 `500 * (50 - 500/125) / 100 = 230` 个.

`numFeasibleNodesToFind` 只是表明扫到这个节点数后就结束了, 但如果前面执行插件发生失败时, 自然会加大扫描数.

```go
func (sched *Scheduler) numFeasibleNodesToFind(percentageOfNodesToScore *int32, numAllNodes int32) (numNodes int32) {
	// 当前的集群小于 100 时, 则直接使用集群节点数作为扫描数
	if numAllNodes < minFeasibleNodesToFind {
		return numAllNodes
	}

	// k8s scheduler 的 nodes 百分比默认为 0
	var percentage int32
	if percentageOfNodesToScore != nil {
		percentage = *percentageOfNodesToScore
	} else {
		percentage = sched.percentageOfNodesToScore
	}

	if percentage == 0 {
		percentage = int32(50) - numAllNodes/125
		// 不能小于 5
		if percentage < minFeasibleNodesPercentageToFind {
			percentage = minFeasibleNodesPercentageToFind
		}
	}

	// 需要扫描的节点数
	numNodes = numAllNodes * percentage / 100
	// 如果小于 100, 则使用 100 作为扫描数. 
	if numNodes < minFeasibleNodesToFind {
		return minFeasibleNodesToFind
	}

	return numNodes
}
```

### prioritizeNodes

`prioritizeNodes` 方法为调度器的优选阶段的实现. 其内部会遍历调用 framework 的 PreScore 插件集合里 `PeScore` 方法, 然后再遍历调用 framework 的 Score 插件集合的 `Score` 方法. 经过 Score 打分计算后可以拿到各个 node 的分值.

```go
func prioritizeNodes(
	ctx context.Context,
	extenders []framework.Extender,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) ([]framework.NodePluginScores, error) {
	// 如果 extenders 为空和score 插件为空, 则跳出
	if len(extenders) == 0 && !fwk.HasScorePlugins() {
		result := make([]framework.NodePluginScores, 0, len(nodes))
		for i := range nodes {
			result = append(result, framework.NodePluginScores{
				Name:       nodes[i].Name,
				TotalScore: 1,
			})
		}
		return result, nil
	}

	// 在 framework 的 PreScore 插件集合里, 遍历执行插件的 PreSocre 方法
	preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
	if !preScoreStatus.IsSuccess() {
		// 只有有异常直接退出
		return nil, preScoreStatus.AsError()
	}

	// 在 framework 的 Score 插件集合里, 遍历执行插件的 Socre 方法
	nodesScores, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}

	klogV := klog.V(10)
	if klogV.Enabled() {
		for _, nodeScore := range nodesScores {
			// 打印插件名字和分值 score
			for _, pluginScore := range nodeScore.Scores {
				klogV.InfoS("Plugin scored node for pod", "pod", klog.KObj(pod), "plugin", pluginScore.Name, "node", nodeScore.Name, "score", pluginScore.Score)
			}
		}
	}

	if len(extenders) != 0 && nodes != nil {
		// 当额外 extenders 调度器不为空时, 则需要计算分值.
		...
		...
	}

	return nodesScores, nil
}
```

### selectHost

`selectHost` 是从优选的 nodes 集合里获取分值 socre 最高的 node. 内部还做了一个小优化, 当相近的两个 node 分值相同时, 则通过随机来选择 node, 目的使 k8s node 的负载更趋于均衡.

```go
func selectHost(nodeScores []framework.NodePluginScores) (string, error) {
	// 如果 nodes 为空, 则返回错误
	if len(nodeScores) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}

	// 直接从头到位遍历 nodeScores 数组, 拿到分值 score 最后的 nodeName.
	maxScore := nodeScores[0].TotalScore
	selected := nodeScores[0].Name
	cntOfMaxScore := 1
	for _, ns := range nodeScores[1:] {
		if ns.TotalScore > maxScore {
			// 当前的分值更大, 则进行赋值.
			maxScore = ns.TotalScore
			selected = ns.Name
			cntOfMaxScore = 1
		} else if ns.TotalScore == maxScore {
			// 当两个 node 的 分值相同时, 
			// 使用随机算法来选择当前和上一个 node.
			cntOfMaxScore++
			if rand.Intn(cntOfMaxScore) == 0 {
				selected = ns.Name
			}
		}
	}

	// 返回分值最高的 node
	return selected, nil
}
```

### PriorityQueue 的实现

`PriorityQueue` 用来实现优先级队列, informer 会条件 pod 到 priorityQueue 队列中, `scheduleOne` 会从该队列中 pop 对象. 在创建延迟队列时传入一个 `less` 比较方法, 时间最小的 podInfo 放在 heap 的最顶端.

`flushBackoffQCompleted` 会不断的检查 backoff heap 堆顶的元素是否满足条件, 当满足条件把 pod 对象扔到 activeQ 队里, 并激活条件变量. 这里没有采用监听等待堆顶到期时间的方法，而是每隔一秒去检查堆顶的 podInfo 是否已到期 `isPodBackingoff`.

backoff 时长是依赖 podInfo.Attempts 重试次数的，默认情况下重试 5次 是 5s, 最大不能超过 10s. 主调度方法 `scheduleOne` 每次从队列获取 podInfo 时, 它的 Attempts 字段都会加一.

代码位置: `pkg/scheduler/internal/queue/scheduling_queue.go`

```go
const (
	DefaultPodMaxBackoffDuration time.Duration = 10 * time.Second
)

// 实例化优先级队列，该队列中含有 activeQ, podBackoffQ, unschedulablePods集合.
func NewPriorityQueue() {
	pq.podBackoffQ = heap.NewWithRecorder(
		podInfoKeyFunc, 
		pq.podsCompareBackoffCompleted, 
		...
	)
}

// 添加 pod 对象
func (p *PriorityQueue) Add(pod *v1.Pod) error {
	p.lock.Lock()
	defer p.lock.Unlock()

	pInfo := p.newQueuedPodInfo(pod)
	gated := pInfo.Gated

	// 把对象加到 activeQ 队列里.
	if added, err := p.addToActiveQ(pInfo); !added {
		return err
	}
	// 如果该对象在不可调度集合中存在, 则需要在里面删除.
	if p.unschedulablePods.get(pod) != nil {
		p.unschedulablePods.delete(pod, gated)
	}

	// 从 backoffQ 删除 pod 对象
	if err := p.podBackoffQ.Delete(pInfo); err == nil {
		klog.ErrorS(nil, "Error: pod is already in the podBackoff queue", "pod", klog.KObj(pod))
	}

	// 条件变量通知
	p.cond.Broadcast()

	return nil
}

// 获取 pod info 对象
func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
	p.lock.Lock()
	defer p.lock.Unlock()

	// 如果 activeQ 为空, 陷入等待
	for p.activeQ.Len() == 0 {
		if p.closed {
			return nil, fmt.Errorf(queueClosed)
		}
		p.cond.Wait()
	}

	// 从 activeQ 堆顶 pop 对象
	obj, err := p.activeQ.Pop()
	if err != nil {
		return nil, err
	}

	pInfo := obj.(*framework.QueuedPodInfo)
	// 加一
	pInfo.Attempts++
	p.schedulingCycle++
	return pInfo, nil
}

// heap 的比较方法，确保 deadline 最低的在 heap 顶部.
func (p *PriorityQueue) podsCompareBackoffCompleted(podInfo1, podInfo2 interface{}) bool {
	bo1 := p.getBackoffTime(pInfo1)
	bo2 := p.getBackoffTime(pInfo2)
	return bo1.Before(bo2)
}

func (p *PriorityQueue) getBackoffTime(podInfo *framework.QueuedPodInfo) time.Time {
	duration := p.calculateBackoffDuration(podInfo)
	backoffTime := podInfo.Timestamp.Add(duration)
	return backoffTime
}

// backoff duration 随着重试次数不断叠加，但最大不能超过 maxBackoffDuration.
func (p *PriorityQueue) calculateBackoffDuration(podInfo *framework.QueuedPodInfo) time.Duration {
	duration := p.podInitialBackoffDuration // 1s
	for i := 1; i < podInfo.Attempts; i++ {
		if duration > p.podMaxBackoffDuration-duration {
			return p.podMaxBackoffDuration // 10s
		}
		duration += duration
	}
	return duration
}
```

### 如何处理调度失败的 pod

前面有说 kubernetes scheduler 的 `scheduleOne` 作为主循环处理函数，当没有为 pod 找到合适 node 时，会调用 `FailureHandler` 方法.

`FailureHandler()` 是由 `handleSchedulingFailure()` 方法实现. 该逻辑的实现简单说就是把失败的 pod 扔到 podBackoffQ 队列或者 unschedulablePods 集合里.

```go
func (sched *Scheduler) handleSchedulingFailure(ctx context.Context, fwk framework.Framework, podInfo *framework.QueuedPodInfo, err error, reason string, nominatingInfo *framework.NominatingInfo, start time.Time) {
	podLister := fwk.SharedInformerFactory().Core().V1().Pods().Lister()
	cachedPod, e := podLister.Pods(pod.Namespace).Get(pod.Name)
	if e != nil {
	} else {
		if len(cachedPod.Spec.NodeName) != 0 {
			klog.InfoS("Pod has been assigned to node. Abort adding it back to queue.", "pod", klog.KObj(pod), "node", cachedPod.Spec.NodeName)
		} else {
			podInfo.PodInfo, _ = framework.NewPodInfo(cachedPod.DeepCopy())

			// 重新入队列
			if err := sched.SchedulingQueue.AddUnschedulableIfNotPresent(podInfo, sched.SchedulingQueue.SchedulingCycle()); err != nil {
				klog.ErrorS(err, "Error occurred")
			}
		}
	}

	...
}

func (p *PriorityQueue) AddUnschedulableIfNotPresent(pInfo *framework.QueuedPodInfo, podSchedulingCycle int64) error {
	pod := pInfo.Pod

	// 去重判断
	if _, exists, _ := p.activeQ.Get(pInfo); exists {
		return fmt.Errorf("Pod %v is already present in the active queue", klog.KObj(pod))
	}
	// 去重判断
	if _, exists, _ := p.podBackoffQ.Get(pInfo); exists {
		return fmt.Errorf("Pod %v is already present in the backoff queue", klog.KObj(pod))
	}

	// 把没有调度成功的 podInfo 扔到 backoffQ 队列或者 unschedulablePods 集合中.
	if p.moveRequestCycle >= podSchedulingCycle {
		if err := p.podBackoffQ.Add(pInfo); err != nil {
			return fmt.Errorf("error adding pod %v to the backoff queue: %v", klog.KObj(pod), err)
		}
	} else {
		p.unschedulablePods.addOrUpdate(pInfo)
	}

	return nil
}
```

scheduler queue 在启动时会开启两个常驻的协程. 一个协程来管理 `flushBackoffQCompleted()`，每隔一秒来调用一次. 另一个协程来管理 `flushUnschedulablePodsLeftover`, 每隔三十秒来调用一次.

```go
// Run starts the goroutine to pump from podBackoffQ to activeQ
func (p *PriorityQueue) Run() {
	go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
	go wait.Until(p.flushUnschedulablePodsLeftover, 30*time.Second, p.stop)
}
```

`flushBackoffQCompleted` 从 podBackoffQ 获取 podInfo, 然后扔到 activeQ 里，等待 scheduleOne 来调度处理.

```go
func (p *PriorityQueue) flushBackoffQCompleted() {
	activated := false
	for {
		rawPodInfo := p.podBackoffQ.Peek()
		pInfo := rawPodInfo.(*framework.QueuedPodInfo)
		pod := pInfo.Pod
        
		// 从 podBackoffQ 中获取上次调度失败的 podInfo.
		_, err := p.podBackoffQ.Pop()
		if err != nil {
			klog.ErrorS(err, "Unable to pop pod from backoff queue despite backoff completion", "pod", klog.KObj(pod))
			break
		}
        
		// 然后把这 podInfo 再扔到 activeQ 里，让 scheduleOne 主循环来处理.
		if added, _ := p.addToActiveQ(pInfo); added {
			klog.V(5).InfoS("Pod moved to an internal scheduling queue", "pod", klog.KObj(pod), "event", BackoffComplete, "queue", activeQName)
			activated = true
		}
	}

	// 如果有添加成功的，则激活条件变量.
	if activated {
		p.cond.Broadcast()
	}
}
```

`flushUnschedulablePodsLeftover` 加锁遍历 PodInfoMap, 如果某个 pod 的距离上次的调度时间大于60s, 则扔到两个队列中的一个，否则等待下个 30s 再来处理.

```go
func (p *PriorityQueue) flushUnschedulablePodsLeftover() {
	p.lock.Lock()
	defer p.lock.Unlock()

	var podsToMove []*framework.QueuedPodInfo
	for _, pInfo := range p.unschedulablePods.podInfoMap {
		lastScheduleTime := pInfo.Timestamp
		if currentTime.Sub(lastScheduleTime) > p.podMaxInUnschedulablePodsDuration {
			podsToMove = append(podsToMove, pInfo)
		}
	}

	if len(podsToMove) > 0 {
		// 把 podInfo 扔到 activeQ 和 backoffQ 队列中.
		p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
	}
}

func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent) {
	activated := false
	for _, pInfo := range podInfoList {
		pod := pInfo.Pod
		if p.isPodBackingoff(pInfo) {
			// 如果还需要 backoff 退避, 则扔到 podBackoffQ 队列中进行延迟, 后面由 flushBackoffQCompleted 处理.
			if err := p.podBackoffQ.Add(pInfo); err != nil {
			} else {
				p.unschedulablePods.delete(pod, pInfo.Gated)
			}
		} else {
			// 把 podInfo 扔到 activeQ 里, 等待 scheduleOne 调度处理.
			if added, _ := p.addToActiveQ(pInfo); added {
				activated = true
				p.unschedulablePods.delete(pod, gated)
			}
		}
	}
    ...
}
```

## bindingCycle 实现 pod 和 node 绑定

`bindingCycle` 用来实现 pod 和 node 绑定, 其过程为 `prebind -> bind -> postbind`.

```go
func (sched *Scheduler) bindingCycle(
	... {

	assumedPod := assumedPodInfo.Pod

	// 执行插件的 prebind 逻辑
	if status := fwk.RunPreBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost); !status.IsSuccess() {
		return status
	}

	// 执行 bind 插件逻辑
	if status := sched.bind(ctx, fwk, assumedPod, scheduleResult.SuggestedHost, state); !status.IsSuccess() {
		return status
	}

	// 在 bind 绑定后执行收尾操作
	fwk.RunPostBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)

	return nil
}

func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed *v1.Pod, targetNode string, state *framework.CycleState) (status *framework.Status) {
	defer func() {
		sched.finishBinding(fwk, assumed, targetNode, status)
	}()

	bound, err := sched.extendersBinding(assumed, targetNode)
	if bound {
		return framework.AsStatus(err)
	}
	return fwk.RunBindPlugins(ctx, state, assumed, targetNode)
}

func (sched *Scheduler) extendersBinding(pod *v1.Pod, node string) (bool, error) {
	for _, extender := range sched.Extenders {
		if !extender.IsBinder() || !extender.IsInterested(pod) {
			continue
		}
		return true, extender.Bind(&v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: pod.Namespace, Name: pod.Name, UID: pod.UID},
			Target:     v1.ObjectReference{Kind: "Node", Name: node},
		})
	}
	return false, nil
}
```

extender 默认就只有 `DefaultBinder` 插件, 该插件的 bind 逻辑是通过 clientset 对 pod 进行 node 绑定..

代码位置: `pkg/scheduler/framework/plugins/defaultbinder/default_binder.go`

```go
func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) *framework.Status {
	binding := &v1.Binding{
		ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
		Target:     v1.ObjectReference{Kind: "Node", Name: nodeName},
	}
	err := b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
	if err != nil {
		return framework.AsStatus(err)
	}
	return nil
}
```

## 总结

kubernetes scheduler 从 informer 监听新资源变动, 当有新 pod 创建时, scheduler 需要为其分配绑定一个 node 节点. 在调度 Pod 时要经过两个阶段, 即 调度周期 和 绑定周期. 

调度周期分为预选阶段和优选阶段.

- 预选 (Predicates) 是通过插件的 Filter 方法过滤出符合要求的 node 节点
- 优选 (Priorities) 则对预选出来的 node 节点进行打分排序

绑定周期默认只有一个默认插件, 仅给 apiserver 发起 node 跟 pod 绑定的请求.