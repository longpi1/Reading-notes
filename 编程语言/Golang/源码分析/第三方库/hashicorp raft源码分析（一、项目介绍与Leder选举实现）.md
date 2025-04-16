## hashicorp raft源码分析（一、项目介绍与Leder选举实现）

> 本文基于 hashicorp/raft `v1.7.3` 版本进行源码分析
>
> API手册：https://pkg.go.dev/github.com/hashicorp/raft
>
> 源码地址：[hashicorp/raft](https://github.com/hashicorp/raft)
>
> raft论文中文解读：https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md
>
> 在阅读文章前需要有一定的 raft 基础, 不然直接看源码会一头雾水.

## 一、项目背景：什么是 Raft？

在聊代码结构前，先简单回顾 Raft 算法的基本思想：

- **Raft** 是一种**分布式一致性算法**（Consensus Algorithm），用于在分布式系统（如集群、多节点存储系统）中保证**数据一致性**和**高可用性**。

- 相比 Paxos 更

  易理解、易实现

  ，Raft 将一致性问题分解为：

  1. **Leader 选举（Leader Election）**：集群中选出一个 Leader 节点负责日志复制。
  2. **日志复制（Log Replication）**：Leader 接收客户端写请求，记录到自己的日志，并**同步**到 Follower 节点。
  3. **安全性（Safety）**：确保**所有节点最终状态一致**，防止脑裂、日志冲突。

Hashicorp 的 Raft 是 Raft 论文（[In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)）的**Go语言实现**，被广泛应用在：

- Hashicorp 的 **Consul**、**Vault**、**Nomad**
- 类似 **etcd**（另一个 Raft 实现）



## 二、项目结构

```go
github.com/hashicorp/raft/ 相关核心目录与代码
├── fuzzy    // 用来模拟测试 启动一个集群， 启动多个raftNode 节点。
├── bench    // 包含基准测试，用于评估 Raft 的性能。
├── raft.go         // 【核心文件】Raft 状态机、主逻辑（选举、日志复制核心循环）
├── api.go          // 对外暴露的核心 API（Raft 接口定义）
├── config.go       // Raft 节点配置（超时、选举、日志等参数）
├── file_snapshot.go // 基于文件的快照（Snapshot）实现
├── fsm.go          // 有限状态机（Finite State Machine）接口定义
├── future.go       // 异步操作的 Future 模式实现（请求结果回调）
├── log.go          // Raft 日志结构（LogEntry、日志压缩等）
├── log_store.go    // 日志存储接口（BoltDB、内存存储、文件存储实现）
├── net_transport.go // 网络层抽象（TCP、TLS、Inmem 传输实现）
├── observer.go     // 事件监听器（观察者模式，监控集群状态变化）
├── raft_test.go    // 单元测试、集成测试（非常重要，覆盖所有场景）
├── snapshot.go     // 快照管理（压缩日志、持久化状态）
├── stable_store.go // 持久化存储接口（稳定存储 Raft 日志、任期等）
├── transport.go    // 网络传输接口定义（RPC 调用，消息传递）
└── util.go         // 辅助函数（随机数、时间戳、Debug 工具）
```

**关键子目录/文件说明：**

- `raft.go`:**核心**！实现了Raft算法的主要逻辑，包括：
  - 节点状态切换（成为候选人、领导者、跟随者）
  - 心跳机制（Leader发送心跳维持领导权）
  - 选举定时器（触发选举）
  - 日志复制（Leader向Follower同步日志）
  - 状态机应用（`FSM.Apply()`）
- `log.go` & `log_store.go`: Raft日志的存储与管理：
  - `Log`结构体（Index、Term、Command）
  - `LogStore`接口（持久化日志的`Append()`、`Get()`等）
  - 默认内存实现（`inmem_store.go`）和BoltDB实现（`raftboltdb/`）用于**持久化**
- `fsm.go`:有限状态机接口，使用者需实现：
  - `Apply(logEntry []byte) interface{}`（应用日志到状态机）
  - `Snapshot() (FSMSnapshot, error)`（创建快照）
  - `Restore(snapshot io.ReadCloser) error`（从快照恢复）
- `snapshot.go` & `file_snapshot.go`: 快照的创建、传输、恢复逻辑：
  - `SnapshotSink`（写快照到文件）
  - `SnapshotStore`接口（支持文件系统、内存快照）
- `transport.go` & `net_transport.go`:网络通信层
  - `Transport`接口（`AppendEntries()`、`RequestVote()`等RPC）
  - 基于TCP的默认实现（`NetTransport`），封装了`encoding/gob`序列化
- `future.go`: 异步操作机制：
  - 提交日志(`raft.Apply()`)时返回`Future`对象
  - 可阻塞等待操作结果（成功/失败）

**整体结构特点：**

1. **接口与实现分离**：如`LogStore`、`SnapshotStore`、`Transport`等关键组件均为接口，方便替换存储引擎（内存/BoltDB）或网络层（TCP、自定义）
2. 分层清晰：
   - 最底层：`StableStore`（任期、投票持久化）、`LogStore`（日志存储）
   - 中间层：`Raft`状态机（选举、日志复制）
   - 上层：`FSM`（业务状态机，由用户实现）
3. 大量使用Go的`goroutine`+`channel`：
   - 后台心跳/选举定时器
   - 处理RPC调用
   - 日志异步复制



## 三、Leader选举



在raft算法中，典型的领导者选举在本质上是节点状态的变更。具体到raft源码中，领导者选举的入口函数就是`run()`，在raft.go中以一个单独的协程运行，来实现节点状态的变更

在下面的实现代码中，可以看到`Follower`、`Candidate`和`Leader`**3 种节点状态都有分别对应的功能函数**

```go
func (r *Raft) run() {
	for {
		select {
		case <-r.shutdownCh:
			r.setLeader("")
			return
		default:
		}
 
		switch r.getState() {
		case Follower:
			r.runFollower()
		case Candidate:
			r.runCandidate()
		case Leader:
			r.runLeader()
		}
	}
}
```

#### 1.1首先，在初始状态下，集群中所有的节点都处于跟随者状态，函数`runFollower()`运行，大致的执行步骤为：

```go
// runFollower 在 Follower 状态下运行主循环。
func (r *Raft) runFollower() {
    didWarn := false                                                  // 是否已经发出警告（没有已知节点或不在配置中）
    leaderAddr, leaderID := r.LeaderWithID()                          // 获取当前已知的 leader 地址和 ID
    r.logger.Info("entering follower state", "follower", r, "leader-address", leaderAddr, "leader-id", leaderID) // 记录进入 Follower 状态的日志
    metrics.IncrCounter([]string{"raft", "state", "follower"}, 1)     // 增加 Follower 状态的计数器

    // 生成一个随机的心跳超时定时器。使用随机超时是为了防止所有 follower 同时超时并转换为 candidate。
    heartbeatTimer := randomTimeout(r.config().HeartbeatTimeout)

    // 循环，直到节点不再是 Follower 状态
    for r.getState() == Follower {
       r.mainThreadSaturation.sleeping() // 表示主线程当前处于空闲状态

       // 使用 select 语句监听多个 channel
       select {
       case rpc := <-r.rpcCh:
          // 收到 RPC 请求
          r.mainThreadSaturation.working() // 表示主线程开始工作
          r.processRPC(rpc)                // 处理 RPC 请求

       case c := <-r.configurationChangeCh:
          r.mainThreadSaturation.working()
          // 收到配置变更请求
          // 因为不是 leader，拒绝所有操作
          c.respond(ErrNotLeader) // 回复 "不是 Leader" 错误

       case a := <-r.applyCh:
          r.mainThreadSaturation.working()
          // 收到日志应用请求
          // 因为不是 leader，拒绝所有操作
          a.respond(ErrNotLeader) // 回复 "不是 Leader" 错误
   
           
        // .....忽略非核心代码......


       case <-heartbeatTimer:
          r.mainThreadSaturation.working()
          // 心跳超时
          // 重新启动心跳定时器
          hbTimeout := r.config().HeartbeatTimeout  // 获取配置的心跳超时时间
          heartbeatTimer = randomTimeout(hbTimeout) // 生成新的随机超时定时器,randomTimeout 防止多个 Follower 同时超时发起选举，造成网络风暴（split vote）。

          // 检查最近是否与 leader 有过成功联系
          lastContact := r.LastContact() // 获取最后一次联系的时间
          if time.Since(lastContact) < hbTimeout {
             // 如果最后一次联系时间小于心跳超时时间，说明最近收到过 leader 的心跳，继续循环
             continue
          }

          // 心跳失败！转换为 Candidate 状态
          lastLeaderAddr, lastLeaderID := r.LeaderWithID() // 获取最后已知的 leader 地址和 ID
          r.setLeader("", "")                              // 清空 leader 信息

          if r.configurations.latestIndex == 0 {
             // 如果最新的配置索引为 0，表示还没有任何已知的配置
             if !didWarn {
                r.logger.Warn("no known peers, aborting election") // 记录警告日志：没有已知的节点，中止选举
                didWarn = true                                     // 设置警告标志，避免重复警告
             }
          } else if r.configurations.latestIndex == r.configurations.committedIndex &&
             !hasVote(r.configurations.latest, r.localID) {
             // 如果最新的配置索引等于已提交的配置索引，并且当前节点在最新的配置中没有投票权
             if !didWarn {
                r.logger.Warn("not part of stable configuration, aborting election") // 记录警告日志：不在稳定的配置中，中止选举
                didWarn = true                                                       // 设置警告标志，避免重复警告
             }
          } else {
             // 满足选举条件
             metrics.IncrCounter([]string{"raft", "transition", "heartbeat_timeout"}, 1) // 增加心跳超时的计数器
             if hasVote(r.configurations.latest, r.localID) {
                // 如果当前节点在最新的配置中有投票权
                r.logger.Warn("heartbeat timeout reached, starting election", "last-leader-addr", lastLeaderAddr, "last-leader-id", lastLeaderID) // 记录警告日志：心跳超时，开始选举
                r.setState(Candidate)                                                                                                             // 设置节点状态为 Candidate
                return                                                                                                                            // 退出 runFollower 函数，开始选举流程
             } else if !didWarn {
                // 当前节点没有投票权，并且之前没有发出过警告
                r.logger.Warn("heartbeat timeout reached, not part of a stable configuration or a non-voter, not triggering a leader election")
                didWarn = true
             }
          }

       case <-r.shutdownCh:
          // 收到关闭信号
          return // 退出 runFollower 函数
       }
    }
}


// hasVote 检查给定的服务器 ID 在给定的配置中是否拥有投票权。
//
// 参数:
//   configuration: Configuration 类型，表示集群的配置。
//   id: ServerID 类型，表示要检查的服务器的 ID。
//
// 返回值:
//   bool 类型，如果服务器拥有投票权则返回 true，否则返回 false。
func hasVote(configuration Configuration, id ServerID) bool {
        // 遍历配置中的所有服务器
        for _, server := range configuration.Servers {
                // 检查当前遍历到的服务器的 ID 是否与要查找的 ID 匹配
                if server.ID == id {
                        // 如果 ID 匹配，则检查该服务器的投票权 (Suffrage) 是否为 Voter (投票者)
                        return server.Suffrage == Voter // 如果是 Voter，则返回 true
                }
        }
        // 如果遍历完所有服务器都没有找到匹配的 ID，或者找到的服务器不是 Voter，则返回 false
        return false
}
```

#### 1.2当节点推举自己为候选人之后，执行`runCandidate()`函数

```go
// 在 Raft 算法中，当 Follower 心跳超时且满足选举条件时，会进入 Candidate 状态，尝试成为 Leader。
func (r *Raft) runCandidate() {
    // 1. 更新当前任期（Term），Candidate 总是在新的任期中发起选举
    term := r.getCurrentTerm() + 1

    // 记录日志：节点进入 Candidate 状态，并附带当前节点信息和任期号
    r.logger.Info("entering candidate state", "node", r, "term", term)

    // 增加监控指标：记录节点进入 Candidate 状态的次数
    metrics.IncrCounter([]string{"raft", "state", "candidate"}, 1)

    // 2. 发起选举前的准备：决定是先进行预投票（Pre-Vote）还是直接正式投票（RequestVote）
    var voteCh <-chan *voteResult       // 正式投票结果通道
    var prevoteCh <-chan *preVoteResult // 预投票结果通道

    // 检查是否启用预投票机制（pre-vote），并且当前 Candidate 状态不是由领导权转移（Leadership Transfer）触发的
    // 领导权转移场景下，Raft 协议规定跳过预投票，直接请求投票（避免额外的延迟）
    if !r.preVoteDisabled && !r.candidateFromLeadershipTransfer.Load() {
       // 发起预投票（Pre-Vote）：询问其他节点是否同意在新任期中投票给自己
       // 预投票不会真正增加任期号，只是试探能不能赢得选举，避免无效的 Term 增长
       prevoteCh = r.preElectSelf()
    } else {
       // 直接发起正式投票（RequestVote）：向其他节点请求在新任期中投票
       voteCh = r.electSelf()
    }

    // 3. 确保领导权转移标志（candidateFromLeadershipTransfer）在函数退出时重置
    // 这个标志如果为 true，会在 RequestVote RPC 中设置 LeadershipTransfer=true，
    // 使得其他节点即使已有 Leader 也会投票（这是领导权转移的特殊逻辑）。
    // 必须重置，否则可能被滥用（节点一直伪造领导权转移请求强行拉票）。
    defer func() { r.candidateFromLeadershipTransfer.Store(false) }()

    // 4. 设置选举超时时间（ElectionTimeout），随机化防止多个 Candidate 同时发起选举
    electionTimeout := r.config().ElectionTimeout
    electionTimer := randomTimeout(electionTimeout)

    // 5. 初始化投票计数器
    preVoteGrantedVotes := 0      // 预投票中获得的赞同票数
    preVoteRefusedVotes := 0      // 预投票中获得的反对票数
    grantedVotes := 0             // 正式投票中获得的赞同票数
    votesNeeded := r.quorumSize() // 计算当选所需的最小票数（多数派，N/2 + 1）

    // 打印调试日志：显示当前任期需要的最少票数
    r.logger.Debug("calculated votes needed", "needed", votesNeeded, "term", term)

    // 6. 主循环：持续处理事件，直到状态不再是 Candidate
    for r.getState() == Candidate {
       // 标记主线程当前处于休眠（等待事件）状态，用于监控线程忙碌程度
       r.mainThreadSaturation.sleeping()

       // 通过 select 监听多个通道，处理不同事件
       select {
       // 7.1 处理来自其他节点的 RPC 请求（可能是投票响应、日志复制、心跳等）
       case rpc := <-r.rpcCh:
          r.mainThreadSaturation.working() // 标记线程从休眠中醒来，开始工作
          r.processRPC(rpc)                // 处理 RPC 请求（如 AppendEntries、RequestVote 等）

       // 7.2 处理预投票（Pre-Vote）结果
       case preVote := <-prevoteCh:
          r.mainThreadSaturation.working()
          // 打印调试日志：显示哪个节点给了预投票结果
          r.logger.Debug("pre-vote received", "from", preVote.voterID, "term", preVote.Term, "tally", preVoteGrantedVotes)

          // 7.2.1 如果预投票返回的任期比我们的大，说明有更新的 Leader 存在，放弃选举，退回 Follower
          if preVote.Term > term {
             r.logger.Debug("pre-vote denied: found newer term, falling back to follower", "term", preVote.Term)
             r.setState(Follower)           // 修改状态为 Follower
             r.setCurrentTerm(preVote.Term) // 更新本地任期
             return                         // 退出 Candidate 循环
          }

          // 7.2.2 统计预投票结果：Granted=true 表示对方愿意投票
          if preVote.Granted {
             preVoteGrantedVotes++ // 赞同票数 +1
             r.logger.Debug("pre-vote granted", "from", preVote.voterID, "term", preVote.Term, "tally", preVoteGrantedVotes)
          } else {
             preVoteRefusedVotes++ // 反对票数 +1
             r.logger.Debug("pre-vote denied", "from", preVote.voterID, "term", preVote.Term, "tally", preVoteGrantedVotes)
          }

          // 7.2.3 检查预投票是否成功（获得多数派赞同票）
          if preVoteGrantedVotes >= votesNeeded {
             // 预投票通过！进入正式选举阶段
             r.logger.Info("pre-vote successful, starting election", "term", preVote.Term,
                "tally", preVoteGrantedVotes, "refused", preVoteRefusedVotes, "votesNeeded", votesNeeded)
             // 重置计数器，启动正式投票计时器
             preVoteGrantedVotes = 0
             preVoteRefusedVotes = 0
             electionTimer = randomTimeout(electionTimeout)
             // 关闭预投票通道，切换到正式投票逻辑
             prevoteCh = nil
             voteCh = r.electSelf() // 发起正式投票（RequestVote RPC）
          }

          // 7.2.4 如果预投票被拒绝（多数派不同意），等待超时后再重试
          if preVoteRefusedVotes >= votesNeeded {
             r.logger.Info("pre-vote campaign failed, waiting for election timeout", "term", preVote.Term,
                "tally", preVoteGrantedVotes, "refused", preVoteRefusedVotes, "votesNeeded", votesNeeded)
             // 这里不会立即退出，而是等待 electionTimer 超时后重试
          }

       // 7.3 处理正式投票（RequestVote）结果
       case vote := <-voteCh:
          r.mainThreadSaturation.working()
          // 7.3.1 如果对方返回更高的任期，说明已有新 Leader，Candidate 放弃竞选，退回 Follower
          if vote.Term > r.getCurrentTerm() {
             r.logger.Debug("newer term discovered, fallback to follower", "term", vote.Term)
             r.setState(Follower)
             r.setCurrentTerm(vote.Term)
             return
          }

          // 7.3.2 统计正式投票结果
          if vote.Granted {
             grantedVotes++ // 赞成票 +1
             r.logger.Debug("vote granted", "from", vote.voterID, "term", vote.Term, "tally", grantedVotes)
          }

          // 7.3.3 检查是否赢得选举（获得多数派投票）
          if grantedVotes >= votesNeeded {
             // 赢得选举！成为 Leader
             r.logger.Info("election won", "term", vote.Term, "tally", grantedVotes)
             r.setState(Leader)                  // 状态切换为 Leader
             r.setLeader(r.localAddr, r.localID) // 设置自己为 Leader
             return                              // 退出 Candidate 循环，进入 Leader 逻辑
          }

       // 7.4 以下 case 都是拒绝非 Leader 请求，因为 Candidate 还不是 Leader
       case c := <-r.configurationChangeCh: // 集群配置变更请求
          r.mainThreadSaturation.working()
          c.respond(ErrNotLeader) // 返回错误：当前不是 Leader

           
        // .....忽略非核心代码......


       case <-electionTimer:
           // 选举超时
          r.mainThreadSaturation.working()
          // Election failed! Restart the election. We simply return,
          // which will kick us back into runCandidate
          r.logger.Warn("Election timeout reached, restarting election")
          return

       case <-r.shutdownCh:
          return
       }
    }
}
```



#### 1.3当节点当选为候选人之后，函数`runLeader()`执行，大致的执行步骤如下：

```go
// runLeader 运行 Raft 节点处于 Leader（领导者）状态时的主逻辑。
// 它首先进行 Leader 状态的初始化设置，然后进入 leaderLoop 热循环，持续处理集群管理工作。
func (r *Raft) runLeader() {
   // 1. 日志记录：节点进入 Leader 状态，并附带当前 Leader 实例信息
    r.logger.Info("entering leader state", "leader", r)
	// 2. 增加监控指标：统计进入 Leader 状态的次数
    metrics.IncrCounter([]string{"raft", "state", "leader"}, 1)

    // 3. 通知外部监听者：当前节点已成为 Leader
    // 通过 leaderCh 管道通知状态变更，overrideNotifyBool 会确保只有 true 值能被推送（防止重复通知）
    overrideNotifyBool(r.leaderCh, true)

    // 4. 获取配置中的通知通道（NotifyCh），用于向外部应用发送 Leader 变化通知
    // 这个 NotifyCh 是由用户创建并传入 Raft 配置的，用于监听 Leader 切换事件
    notify := r.config().NotifyCh

    // 5. 如果配置了 NotifyCh，立即推送 Leader=true 信号
    if notify != nil {
            select {
            case notify <- true: // 尝试推送通知
            case <-r.shutdownCh: // 如果收到关闭信号，确保推送通知（但不阻塞）
                    select {
                    case notify <- true: // 再次尝试推送
                    default: // 无法推送时忽略（防止阻塞 shutdown）
                    }
            }
    }

    // 6. 初始化 Leader 状态结构体（leaderState），包含：
    //   - commitCh：用于通知日志提交事件
    //   - inflight：待提交的日志队列
    //   - replState：每个 Follower 的复制状态机
    //   - notify：待验证请求的回调队列
    r.setupLeaderState()

    // 7. 启动后台协程，定期汇报日志存储的年龄（最老日志的时间戳），用于监控日志堆积情况
    stopCh := make(chan struct{})
    go emitLogStoreMetrics(r.logs, []string{"raft", "leader"}, oldestLogGaugeInterval, stopCh)

    // 8. 定义 defer 清理逻辑，当 Leader 降级（step down）时执行，停止相关任务，通知降级
    defer func() {
           
        // .....忽略非核心代码......
    }()

    // 9. 启动日志复制机制：为每个 Follower 创建复制协程（replication routine）
    // 这些协程会定期向 Follower 发送 AppendEntries RPC，保持日志同步
    r.startStopReplication()

    // 10. 立即追加一条空日志（Noop Log）到 Raft 日志流中
    // 作用：
    //   1. 快速推进 commitIndex，即使没有客户端请求
    //   2. 避免未提交的配置日志（Config Log）无限制增长（历史 Bug 修复）
    // 以前这里会追加配置日志，但现在限制：最多只能有一个未提交的配置日志
    noop := &logFuture{log: Log{Type: LogNoop}} // 构造空日志条目
    r.dispatchLogs([]*logFuture{noop})           // 分发日志，触发复制流程

    // 11. 进入 Leader 主循环（leaderLoop）：持续处理日志复制、心跳、成员管理等核心逻辑
    // 直到因选举超时、收到更高 Term RPC 等原因降级（step down）
    r.leaderLoop()
}





// leaderLoop 是 Leader 状态下的核心循环。
// 它在 Leader 所有初始化设置完成后被调用，负责处理日志复制、心跳、成员管理等核心逻辑。
func (r *Raft) leaderLoop() {
	// 1. stepDown 标志：用于判断是否有正在进行的日志操作会导致 Leader 降级
	// 典型场景：Leader 收到移除自己（RemovePeer）的日志请求，此时不能并行处理其他日志，
	// 否则提交判断只依赖单个节点（自己），复制目标也不明确（未知的 Peer 集合）。
	stepDown := false

	// 2. 初始化 Leader Lease（租约）计时器，用于检查 Leader 是否仍然有效
	lease := time.After(r.config().LeaderLeaseTimeout)

	// 3. Leader 主循环：持续运行，直到状态不再是 Leader
	for r.getState() == Leader {
		// 标记主线程当前处于休眠（等待事件）状态，用于监控线程忙碌程度
		r.mainThreadSaturation.sleeping()
		select {
		// 4.1 处理来自其他节点的 RPC 请求（可能是投票、日志复制、心跳响应等）
		case rpc := <-r.rpcCh:
			r.mainThreadSaturation.working() // 标记线程从休眠中醒来，开始工作
			r.processRPC(rpc)                // 处理 RPC 请求（如 AppendEntriesResponse、RequestVote 等）

		// 4.2 收到降级信号（通常是配置变更导致 Leader 需要下台）
		case <-r.leaderState.stepDown:
			r.mainThreadSaturation.working()
			// 立即降级为 Follower，退出 Leader 循环
			r.setState(Follower)

		// 4.3 处理领导权转移（Leadership Transfer）请求：手动切换 Leader
		case future := <-r.leadershipTransferCh:
        // .....忽略非核心代码......



		// 4.4 处理日志提交事件（commitCh）：有新日志被多数派确认
		case <-r.leaderState.commitCh:
			r.mainThreadSaturation.working()
			// 4.4.1 更新本地 commitIndex（已提交日志的最大索引）
			oldCommitIndex := r.getCommitIndex()
			commitIndex := r.leaderState.commitment.getCommitIndex()
			r.setCommitIndex(commitIndex)

			// 4.4.2 检查新提交的配置日志（Configuration Entry）
			if r.configurations.latestIndex > oldCommitIndex &&
				r.configurations.latestIndex <= commitIndex {
				// 新配置已提交，更新 committedConfiguration
				r.setCommittedConfiguration(r.configurations.latest, r.configurations.latestIndex)
				// 如果 Leader 发现自己被移除了（不再是 Voter），标记 stepDown
				if !hasVote(r.configurations.committed, r.localID) {
					stepDown = true
				}
			}

			// 4.4.3 批量处理已提交的日志（Group Commit）
			start := time.Now()
			var groupReady []*list.Element              // 待处理的已提交日志队列
			groupFutures := make(map[uint64]*logFuture) // 日志索引 -> logFuture 映射
			var lastIdxInGroup uint64                   // 最后一个日志的索引

			// 遍历 inflight（待提交日志队列），找出所有已达到 commitIndex 的日志
			for e := r.leaderState.inflight.Front(); e != nil; e = e.Next() {
				commitLog := e.Value.(*logFuture)
				idx := commitLog.log.Index
				if idx > commitIndex { // 超过 commitIndex 的日志跳过
					break
				}
				// 统计日志提交延迟（从分发到提交的时间）
				metrics.MeasureSince([]string{"raft", "commitTime"}, commitLog.dispatch)
				// 加入待处理队列
				groupReady = append(groupReady, e)
				groupFutures[idx] = commitLog
				lastIdxInGroup = idx
			}

			// 4.4.4 批量应用已提交日志到状态机（FSM）
			if len(groupReady) != 0 {
				r.processLogs(lastIdxInGroup, groupFutures)

				// 清理已处理的日志，防止重复提交
				for _, e := range groupReady {
					r.leaderState.inflight.Remove(e)
				}
			}

			// 统计日志入队耗时（FSM 应用延迟）
			metrics.MeasureSince([]string{"raft", "fsm", "enqueue"}, start)
			// 统计本次提交的日志数量
			metrics.SetGauge([]string{"raft", "commitNumLogs"}, float32(len(groupReady)))

			// 4.4.5 如果 Leader 发现自己被移除了（stepDown=true）
			if stepDown {
				if r.config().ShutdownOnRemove {
					// 配置要求移除时关闭节点，优雅退出
					r.logger.Info("removed ourself, shutting down")
					r.Shutdown()
				} else {
					// 否则降级为 Follower
					r.logger.Info("removed ourself, transitioning to follower")
					r.setState(Follower)
				}
			}

		// 4.5 处理成员验证请求（verifyCh）：检查 Leader 是否仍然有效
		case v := <-r.verifyCh:
			r.mainThreadSaturation.working()
			if v.quorumSize == 0 {
				// 初始状态：启动验证流程（向 Follower 发送验证 RPC）
				r.verifyLeader(v)
			} else if v.votes < v.quorumSize {
				// 验证失败：说明已有新 Leader，当前 Leader 降级
				// Early return, means there must be a new leader
				r.logger.Warn("new leader elected, stepping down")
				r.setState(Follower)
				delete(r.leaderState.notify, v)
				for _, repl := range r.leaderState.replState {
					repl.cleanNotify(v)
				}
				v.respond(ErrNotLeader)

			} else {
				// Quorum of members agree, we are still leader
				delete(r.leaderState.notify, v)
				for _, repl := range r.leaderState.replState {
					repl.cleanNotify(v)
				}
				v.respond(nil)
			}
           
            
        // .....忽略非核心代码......
		case <-r.shutdownCh:
			return
		}
	}
}

```






### 2.日志复制（Log Replication）

### leader 复制日志

![img](https://img2022.cnblogs.com/blog/2794988/202205/2794988-20220522153708740-687414905.png)

1. 在`runLeader()`函数中，调用`startStopReplication()`函数，执行日志复制功能
2. 启动一个新协程，调用`replicate()`函数，执行日志复制相关的功能
3. 在`replicate()`函数中，调用`replicateTo()`函数，执行步骤4，如果开启了流水线复制模式，执行步骤5
4. 在`replicateTo()`函数中，进行日志复制和日志一致性检测，如果日志复制成功，则设置`s.allowPipeline = true`，开启流水线复制模式
5. 调用`pipelineReplicate()`函数，采用更搞笑的流水线方式进行日志复制

1. 目标是在环境正常的情况下，提升日志复制性能，如果在日志复制过程中出错了，就进入 RPC 复制模式，继续调用 replicateTo() 函数，进行日志复制。

### follower 接收日志

![img](https://img2022.cnblogs.com/blog/2794988/202205/2794988-20220522155156888-154465698.png)

1. 在`runFollower()`函数中，调用`processRPC()`函数，处理接收到的RPC消息
2. 在`processRPC()`函数中，调用`appendEntries()`函数，处理接收到的日志复制RPC请求
3. `appendEntries()`函数，是跟随者处理日志的核心函数。在步骤3.1中，比较日志一致性；在步骤3.2中，将新日志项存放到本地；在步骤3.3中，根据领导者最新提交的日志项索引值，来计算当前需要被应用的日志项，并应用到本地状态机



### 3.安全性（Safety）









## 四、问题

- **问题 1：Raft 如何处理网络分区？**
  - 解答：Raft 通过领导人选举和多数派机制来处理网络分区。
    - **分区形成：** 当网络分区发生时，集群可能分裂成多个部分，每个部分都无法与多数节点通信。
    - **选举：** 在每个分区中，如果 Follower 节点在选举超时时间内没有收到 Leader 的心跳，它们会发起选举。
    - **多数派：** 只有包含多数节点的分区才能选出新的 Leader。少数派分区中的节点无法赢得选举，因为它们无法获得多数票。
    - **旧 Leader：** 如果旧 Leader 位于少数派分区，它会因为无法与多数节点通信而退位成 Follower。
    - **数据一致性：** Raft 保证，即使在网络分区的情况下，也只有一个分区能够提交新的日志条目，从而保证数据一致性。
    - **分区恢复：** 当网络分区恢复后，少数派分区中的节点会重新加入集群，并从新 Leader 那里同步最新的日志。
- **问题 2：Raft 如何保证数据一致性？**
  - 解答：Raft 通过以下机制保证数据一致性：
    - **强领导者：** 只有 Leader 才能接受客户端请求并生成新的日志条目。
    - **日志复制：** Leader 将日志条目复制到所有 Follower。只有当多数 Follower 确认收到日志条目后，Leader 才会提交该条目。
    - **仅追加日志：** 日志条目只能追加到日志末尾，不能修改或删除。
    - **选举限制：** 只有拥有最新日志的节点才能成为 Leader。
    - **提交限制：** 只有 Leader 才能推进 `commitIndex`，且 `commitIndex` 只会单调递增。
    - **状态机：** 所有节点按照相同的顺序应用已提交的日志条目到状态机，保证状态机状态一致。
- **问题 3：Raft 中的 `commitIndex` 和 `lastApplied` 有什么区别？**
  - 解答：
    - **`commitIndex`：** 表示已知已提交的最高日志条目的索引。这意味着索引小于或等于 `commitIndex` 的所有日志条目都已安全地复制到多数节点，可以应用到状态机。
    - **`lastApplied`：** 表示已应用到状态机的最高日志条目的索引。每个节点独立维护自己的 `lastApplied`。
    - **关系：** `lastApplied` 通常小于或等于 `commitIndex`。当 `lastApplied` 小于 `commitIndex` 时，表示节点正在将已提交的日志条目应用到状态机。当两者相等时，表示状态机是最新的。
- **问题 4：Raft 如何处理客户端请求的幂等性？**
  - 解答：Raft 本身不直接处理客户端请求的幂等性。幂等性通常需要在客户端或应用层实现。一种常见的做法是：
    - **客户端生成唯一 ID：** 客户端为每个请求生成一个唯一的 ID（例如 UUID）。
    - **服务器跟踪 ID：** 服务器跟踪已处理的请求 ID。如果收到具有相同 ID 的重复请求，服务器可以直接返回之前的结果，而无需重新执行操作。
    - **状态机：** 在应用层，状态机可以记录已执行的请求 ID，以避免重复执行。







## 五、参考链接

1.[Hashicorp Raft实现和API分析](https://www.cnblogs.com/aganippe/p/16292050.html)

2.[hashicorp raft源码学习](https://qiankunli.github.io/2020/05/17/hashicorp_raft.html)

3.[源码分析 hashicorp raft replication 日志复制的实现原理](https://github.com/rfyiamcool/notes/blob/main/hashicorp_raft_replication_code.md#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-hashicorp-raft-replication-%E6%97%A5%E5%BF%97%E5%A4%8D%E5%88%B6%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

