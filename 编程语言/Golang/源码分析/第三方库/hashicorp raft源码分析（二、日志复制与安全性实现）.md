# hashicorp raft源码分析（二、日志复制与安全性实现）

> 本文基于 hashicorp/raft `v1.7.3` 版本进行源码分析
>
> API手册：https://pkg.go.dev/github.com/hashicorp/raft
>
> 源码地址：[hashicorp/raft](https://github.com/hashicorp/raft)
>
> raft论文中文解读：https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md
>
> 在阅读文章前需要有一定的 raft 基础, 不然直接看源码会一头雾水.
>
> 上一篇文章：[hashicorp raft源码分析（一、项目介绍与Leder选举实现）](https://github.com/longpi1/Reading-notes/blob/main/%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80/Golang/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%AC%AC%E4%B8%89%E6%96%B9%E5%BA%93/hashicorp%20raft%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%E3%80%81%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%E4%B8%8ELeder%E9%80%89%E4%B8%BE%E5%AE%9E%E7%8E%B0%EF%BC%89.md)

## 一、日志复制（Log Replication）

本文按照下面流程分析 raft 日志复制的实现原理.

1. 调用上层 Apply 接口写数据.
2. leader 向 follower 同步日志.
3. follower 接收日志.
4. leader 确认提交日志, 并且应用到状态机.
5. follower 确认提交日志.

![image.png](https://s2.loli.net/2025/05/10/sKTCthOB3Xbji5y.png)

### Apply 方法应用日志，写入applyCh通道中

`Apply` 是 hashicorp raft 提供的给上层写数据的入口, 当使用 hashicorp/raft 构建分布式系统时, 作为 leader 节点承担了写操作, 这里写就是调用 api 里的 Apply 方法.

`Apply` 入参的 cmd 为业务需要写的数据, 只支持 `[]byte`, 如是 struct 对象则需要序列化为 `[]byte`, timeout 为写超时, 这里的写超时只是把 logFuture 插入 applyCh 的超时时间, 而不是推到 follower 的时间.

`Apply` 其内部流程是先实例化一个定时器, 然后把业务数据构建成 logFuture 对象, 然后推到 applyCh 队列. applyCh 缓冲队列的大小跟 raft 的并发吞吐有关系的, hashicorp raft 里 applyCh 默认长度为 64.

代码位置: `github.com/hashicorp/raft/api.go`

```go
// 写日志
func (r *Raft) Apply(cmd []byte, timeout time.Duration) ApplyFuture {
	return r.ApplyLog(Log{Data: cmd}, timeout)
}

// ApplyLog 直接接收一个 Log 结构并执行 Apply 操作,最终写入applyCh通道中。
func (r *Raft) ApplyLog(log Log, timeout time.Duration) ApplyFuture {
        // 定义一个定时器通道，用于实现超时机制
        var timer <-chan time.Time
        // 如果调用者设置了超时时间（timeout > 0），则创建一个定时器
        // 当超过指定时间后，timer 通道会接收到一个时间值，表示超时
        if timeout > 0 {
                timer = time.After(timeout)
        }

        // 创建一个日志的 Future 对象，用于异步跟踪日志应用的执行状态
        // 此时日志的索引（Index）和任期（Term）尚未设置，因此为空
        logFuture := &logFuture{
                log: Log{
                        Type:       LogCommand,        // 日志类型设置为命令日志，表示这是一个用户提交的命令
                        Data:       log.Data,          // 从传入的 Log 中提取数据字段
                        Extensions: log.Extensions,    // 从传入的 Log 中提取扩展字段
                },
        }
        logFuture.init()

        select {
        case <-timer:
                // 如果定时器通道接收到值，说明超时时间已到
                return errorFuture{ErrEnqueueTimeout}
        case <-r.shutdownCh:
                // 如果 Raft 实例的关闭通道被触发，说明 Raft 节点正在关闭
                return errorFuture{ErrRaftShutdown}
        case r.applyCh <- logFuture:
                // 如果成功将 logFuture 发送到 Raft 的应用通道（applyCh）
                // 表示日志应用请求已被接受，返回 logFuture 以供调用者跟踪执行结果
                return logFuture
        }
}
```



### 监听 applyCh 并调度通知日志

`leaderLoop` 会监听 applyCh 管道, 该管道的数据是由 hashicorp/raft api 层的 Apply 方法推入, leaderLoop 在收到 apply 日志后, 调用 `dispatchLogs` 来给 `replication` 调度通知日志.

代码位置: `github.com/hashicorp/raft/raft.go`

```go
func (r *Raft) leaderLoop() {
	for r.getState() == Leader {

		select {
		case ...:

		case newLog := <-r.applyCh:
			// ...

			// 日志的组提交, 所谓的组提交就是日志按批次提交, 这是 raft 工程上优化.
			ready := []*logFuture{newLog}
		GROUP_COMMIT_LOOP:
			// 尝试凑齐 MaxAppendEntries 数量的日志
			for i := 0; i < r.config().MaxAppendEntries; i++ {
				select {
				case newLog := <-r.applyCh:
					ready = append(ready, newLog)
				default:
					// applyCh 为空, 中断循环.
					break GROUP_COMMIT_LOOP
				}
			}

			// ...

			// 派发日志, 批量发.
			r.dispatchLogs(ready)
		case ...:

		}
	}
}
```

dispatchLogs 是 Raft 协议中 Leader 节点用于处理日志分发的核心方法

1. **日志持久化**：
   - 将日志写入本地磁盘，确保日志的持久化。如果写入失败，Leader 节点会降级为 Follower 节点，并通知调用者操作失败。
2. **状态更新**：
   - commitment.match 来计算各个 server 的 matchIndex, 计算出 commit 提交索引.
   - 更新 Leader 节点的匹配索引（`match index`），表示本地节点已成功存储日志。
   - 更新 Leader 节点的最后日志索引和任期信息。
3. **触发日志复制**：
   - 异步通知所有 Follower 节点的复制器，触发日志复制流程，确保日志被同步到集群中的其他节点。

```go
// dispatchLogs 是 Raft 协议中 Leader 节点用于处理日志分发的核心方法，
// 它的主要功能是将一批待应用的日志写入本地磁盘，标记这些日志为“inflight”（inflight），
//  如果日志成功写入本地磁盘，更新 Leader 节点的匹配索引（match index），
// 更新 Leader 节点的最后日志索引和任期信息，表示最新的日志状态
// 并开始将这些日志复制到 Follower 节点。
func (r *Raft) dispatchLogs(applyLogs []*logFuture) {
        // 记录方法开始执行的时间，用于后续性能指标的统计
        now := time.Now()
        // 使用 defer 延迟执行性能指标的统计，计算方法执行的总耗时
        defer metrics.MeasureSince([]string{"raft", "leader", "dispatchLog"}, now)

        // 获取当前 Leader 的任期（term），用于标记新日志的任期信息
        term := r.getCurrentTerm()
        // 获取当前日志的最后索引（lastIndex），用于为新日志分配连续的索引号
        lastIndex := r.getLastIndex()

        n := len(applyLogs)        // 获取待分发的日志数量
        logs := make([]*Log, n)        // 创建一个日志切片，用于存储待写入磁盘的日志条目
        // 设置性能指标：记录当前分发的日志数量
        metrics.SetGauge([]string{"raft", "leader", "dispatchNumLogs"}, float32(n))

        // 遍历待分发的日志，为每条日志设置索引、任期和时间戳，并标记为“inflight”
        for idx, applyLog := range applyLogs {
                applyLog.dispatch = now                // 设置日志的调度时间戳，表示日志开始被分发的时间
                lastIndex++                // 为日志分配新的索引号，递增 lastIndex
                applyLog.log.Index = lastIndex                // 设置日志的索引号
                applyLog.log.Term = term                // 设置日志的任期号为当前 Leader 的任期
                applyLog.log.AppendedAt = now               // 设置日志的追加时间戳，表示日志被追加到日志存储的时间
                logs[idx] = &applyLog.log                // 将日志条目存储到 logs 切片中，准备写入磁盘
                r.leaderState.inflight.PushBack(applyLog)                // 将日志标记为“飞行中”，表示该日志正在被处理（尚未被所有节点确认）
        }

        // 将日志条目写入本地磁盘，确保日志持久化
        if err := r.logs.StoreLogs(logs); err != nil {
                // 如果写入失败，记录错误日志
                r.logger.Error("failed to commit logs", "error", err)
                // 遍历所有待分发的日志，通知调用者写入失败
                for _, applyLog := range applyLogs {
                        applyLog.respond(err)
                }
                // 如果日志写入失败，Leader 节点主动降级为 Follower 节点，避免继续处理请求
                r.setState(Follower)
                return
        }
        // 如果日志成功写入本地磁盘，更新 Leader 节点的匹配索引（match index），
        // 表示本地节点已经成功存储了这些日志
        r.leaderState.commitment.match(r.localID, lastIndex)

        // 更新 Leader 节点的最后日志索引和任期信息，表示最新的日志状态
        r.setLastLog(lastIndex, term)

        // 通知所有 Follower 节点的复制器（replicator），触发日志复制
        // 遍历 Leader 节点维护的每个 Follower 节点的复制状态
        for _, f := range r.leaderState.replState {
                // 异步通知复制器的触发通道（triggerCh），启动日志复制流程
                asyncNotifyCh(f.triggerCh)
        }
}
```



### replicate 同步日志



[![img](https://camo.githubusercontent.com/5be1209b9e876958cd1317c9507e77d9c8d7595d4e3f89cfca72d3b2a38ce7c8/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323230313630313335322e706e67)](https://camo.githubusercontent.com/5be1209b9e876958cd1317c9507e77d9c8d7595d4e3f89cfca72d3b2a38ce7c8/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323230313630313335322e706e67)

当节点确认成为 leader 时, 会为每个 follower 启动 replication 对象, 并启动两个协程 replicate 和 heartbeat.

`replicate` 其内部监听 triggerCh 有无发生通知时, 当有日志需要同步给 follower 调用 `replicateTo`. 另外 replicate 每次还会创建一个随机 50ms - 100ms 的定时器, 当定时器触发时, 也会尝试同步日志, 主要用来同步 commitIndex 提交索引.

```go
// replicate 是一个长期运行的 goroutine，负责将日志条目（log entries）复制到 **单个 follower 节点**。
// 它实现了 Raft 的 **日志复制（Log Replication）** 机制，确保 follower 与 Leader 的日志保持一致。
func (r *Raft) replicate(s *followerReplication) {
	stopHeartbeat := make(chan struct{})

	defer close(stopHeartbeat)

	// 启动 **独立的心跳发送 goroutine**（heartbeat）
	// 心跳用于维持 Leader 身份、避免选举超时，但不携带日志数据
	r.goFunc(func() {
		// 调用 r.heartbeat 函数，定期发送 AppendEntries RPC（空日志）
		r.heartbeat(s, stopHeartbeat)
	})

	// RPC 模式（标准复制模式）
	// 在此模式下，Leader **逐条确认（Stop-and-Wait）** follower 的日志复制结果
RPC:
	// 控制变量：标记是否应该停止复制循环
	shouldStop := false
	for !shouldStop {
		select {
		// **情况 1：收到停止信号（优雅退出）**
		case maxIndex := <-s.stopCh:
			// ...

		// **情况 2：延迟错误处理（触发一次强制同步）**
		case deferErr := <-s.triggerDeferErrorCh:
			// ...

		// **情况 3：收到主动触发信号（立即同步日志），监听 triggerCh 有无发生通知**
		case <-s.triggerCh:
			// 获取最新日志索引
			lastLogIdx, _ := r.getLastLog()
			// 调用 replicateTo 同步日志到最新位置
			shouldStop = r.replicateTo(s, lastLogIdx)

		// **情况 4：定时强制同步（CommitTimeout 机制）**
		// 这 **不是传统心跳**，而是确保 follower 及时得知 Leader 的 **已提交索引（commit index）**
		// 见 https://github.com/hashicorp/raft/issues/282
		// 场景：当 Raft 日志自然流动停止时（空闲期），强制同步一次 commitIndex
		case <-randomTimeout(r.config().CommitTimeout):
			// 获取最新日志索引
			lastLogIdx, _ := r.getLastLog()
			// 同步日志到最新索引
			shouldStop = r.replicateTo(s, lastLogIdx)
		}

		// **性能优化判断：是否切换到 Pipeline 模式？**
		// 如果：
		// 1. 当前复制 **未出错（!shouldStop）**
		// 2. **允许 Pipeline 模式（s.allowPipeline=true）**
		// 则尝试 **切换到高性能的 Pipeline 模式**
		if !shouldStop && s.allowPipeline {
			// 跳转到 PIPELINE 标签处，进入流式复制逻辑
			goto PIPELINE
		}
	}
	// 如果循环退出（shouldStop=true），直接结束 replicate 函数
	return

	// PIPELINE 模式（流式复制，高性能）
	// 在此模式下，Leader **连续发送日志、不等待确认**（类似 TCP 滑动窗口）
PIPELINE:
	s.allowPipeline = false

	// **进入 pipelineReplicate 函数**：
	// 它会：
	// 1. 持续发送 AppendEntries RPC **不等待响应**
	// 2. 通过 TCP 连接批量流式发送日志，提高吞吐量
	// 注意：此模式 **无法优雅处理错误**，一旦出错就回退到 RPC 模式
	if err := r.pipelineReplicate(s); err != nil {
		// **特殊情况：follower 不支持 Pipeline（如旧版本 Raft）**
		if err != ErrPipelineReplicationNotSupported {
			// 加锁读取 follower 的节点 ID（peer）
			s.peerLock.RLock()
			peer := s.peer
			s.peerLock.RUnlock()
			// 记录错误日志：Pipeline 模式失败，fallback 到 RPC 模式
			r.logger.Error("failed to start pipeline replication to", "peer", peer, "error", err)
		}
	}

	// 无论成功与否，Pipeline 失败后都回到 RPC 模式
	goto RPC
}

```

[![img](https://camo.githubusercontent.com/963d7130465f8f26613813bef426b30b1c71651fbb4a3e7be9bfa4b81c6fad78/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323230313534343038392e706e67)](https://camo.githubusercontent.com/963d7130465f8f26613813bef426b30b1c71651fbb4a3e7be9bfa4b81c6fad78/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323230313534343038392e706e67)

replicateTo 用来真正的把日志数据同步给 follower.

1. 首先调用 `setupAppendEntries` 装载请求同步的日志, 这里需要装载上一条日志及增量日志.
2. 然后使用 transport 给 follower 发送请求, 之后更新状态.
3. 如果装载日志时, 发现 log index 不存在, 则需要发送快照文件.
4. 在发完快照文件后, 需要判断是否继续发送快照点之后的增量日志, 如含有增量则 goto 切到 1.

[![img](https://camo.githubusercontent.com/7e9108c0009535750a76a347a04f5a6040c3d5ec33c0a9addbbee08fde5f1599/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323230313535333431302e706e67)](https://camo.githubusercontent.com/7e9108c0009535750a76a347a04f5a6040c3d5ec33c0a9addbbee08fde5f1599/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323230313535333431302e706e67)

#### replicateTo

```go
// replicateTo 是 replicate() 的辅助函数，用于将日志复制到指定的最后索引位置。
// 如果跟随者的日志落后，我们会小心地让它们保持最新。
func (r *Raft) replicateTo(s *followerReplication, lastIndex uint64) (shouldStop bool) {
        // 创建基本的请求和响应结构体
        var req AppendEntriesRequest
        var resp AppendEntriesResponse
        var start time.Time
        var peer Server

START:
        // 防止在错误发生时进行过多的重试，通过退避机制控制重试频率
        if s.failures > 0 {
                select {
                case <-time.After(backoff(failureWait, s.failures, maxFailureScale)): // 使用退避算法等待一段时间
                case <-r.shutdownCh: // 如果 Raft 实例正在关闭，则退出
                }
        }

        // 获取当前跟随者的信息（加锁以确保线程安全）
        s.peerLock.RLock()
        peer = s.peer
        s.peerLock.RUnlock()

        // 设置 AppendEntries 请求的内容，包括日志条目等
        // 如果日志未找到（ErrLogNotFound），则跳转到发送快照的逻辑
        if err := r.setupAppendEntries(s, &req, atomic.LoadUint64(&s.nextIndex), lastIndex); err == ErrLogNotFound {
                goto SEND_SNAP
        } else if err != nil { // 如果发生其他错误，直接返回
                return
        }

        // 执行 RPC 调用，向跟随者发送 AppendEntries 请求
        start = time.Now()
        if err := r.trans.AppendEntries(peer.ID, peer.Address, &req, &resp); err != nil {
                // 如果 RPC 调用失败，记录错误日志并增加失败计数
                r.logger.Error("failed to appendEntries to", "peer", peer, "error", err)
                s.failures++
                return
        }
        // 记录统计信息，例如日志条目数量和调用耗时（用于监控和调试）
        appendStats(string(peer.ID), start, float32(len(req.Entries)), r.noLegacyTelemetry)

        // 检查跟随者的任期是否比当前领导者的任期更新，如果是，则说明当前领导者已过时
        if resp.Term > req.Term {
                r.handleStaleTerm(s) // 处理过时任期的情况（通常会触发重新选举）
                return true // 返回 true 表示停止复制
        }

        // 更新最后一次联系时间（用于心跳检测等）
        s.setLastContact()

        // 根据 RPC 调用的响应结果更新状态
        if resp.Success {
                // 如果跟随者成功接收并应用了日志条目
                // 更新复制状态，例如下一个要发送的日志索引
                updateLastAppended(s, &req)

                // 清空失败计数，并允许流水线复制（提高性能）
                s.failures = 0
                s.allowPipeline = true
        } else {
                // 如果跟随者拒绝了日志条目（可能是因为日志不匹配）
                // 调整下一个要发送的日志索引（回退到更早的日志位置）
                atomic.StoreUint64(&s.nextIndex, max(min(s.nextIndex-1, resp.LastLog+1), 1))
                if resp.NoRetryBackoff {
                        // 如果跟随者明确表示无需退避（例如日志匹配但未应用），清空失败计数
                        s.failures = 0
                } else {
                        // 否则增加失败计数，可能触发退避
                        s.failures++
                }
                // 记录警告日志，提示正在尝试发送更早的日志
                r.logger.Warn("appendEntries rejected, sending older logs", "peer", peer, "next", atomic.LoadUint64(&s.nextIndex))
        }

CHECK_MORE:
        // 在循环中检查停止信号，以确保在以下情况下及时退出：
        // 1. 被要求停止复制（例如领导者下台）
        // 2. Raft 实例正在关闭
        // 即使在尽力复制到指定索引后，也应避免向即将离开集群的落后节点发送过多日志
        select {
        case <-s.stopCh: // 如果收到停止信号，返回 true 表示停止复制
                return true
        default:
        }

        // 检查是否还有更多日志需要复制
        // 如果下一个要发送的日志索引小于等于目标索引，则继续循环
        if atomic.LoadUint64(&s.nextIndex) <= lastIndex {
                goto START
        }
        return // 如果没有更多日志需要复制，返回 false 表示正常完成

        // SEND_SNAP 用于处理日志未找到的情况，通常是因为跟随者日志落后太多，
        // 需要发送快照来代替日志条目
SEND_SNAP:
        // 尝试发送最新的快照给跟随者
        if stop, err := r.sendLatestSnapshot(s); stop {
                return true // 如果发送快照过程中要求停止，返回 true
        } else if err != nil {
                // 如果发送快照失败，记录错误日志
                r.logger.Error("failed to send snapshot to", "peer", peer, "error", err)
                return
        }

        // 检查是否还有更多日志需要复制（发送快照后可能需要继续发送日志）
        goto CHECK_MORE
}
```



#### setupAppendEntries

`setupAppendEntries` 方法会把日志数据和其他元数据装载到 `AppendEntriesRequest` 对象里.

下面是 `AppendEntriesRequest` 的数据结构.

```go
type AppendEntriesRequest struct {
	// rpc proto 和 leader 信息.
	RPCHeader

	// 当前 leader 的日志索引值.
	Term uint64

	// 上一条 log 值的 index 和 term, follower 会利用这两个值校验缺失和冲突.
	PrevLogEntry uint64
	PrevLogTerm  uint64

	// 同步给 follower 的增量的日志
	Entries []*Log

	// 已经在 leader 提交的日志索引值
	LeaderCommitIndex uint64
}
```

`setupAppendEntries` 用来构建 `AppendEntriesRequest` 对象, 这里不仅当前节点的最新 log 信息, 还有 follower nextIndex 的上一条 log 日志数据, 还有新增的 log 日志数据.

```go
// setupAppendEntries is used to setup an append entries request.
func (r *Raft) setupAppendEntries(s *followerReplication, req *AppendEntriesRequest, nextIndex, lastIndex uint64) error {
	req.RPCHeader = r.getRPCHeader()
	// 赋值当前的 term 任期号
	req.Term = s.currentTerm
	// 赋值 leader 信息 
	req.Leader = r.trans.EncodePeer(r.localID, r.localAddr)
	// 赋值 commit index 提交索引值
	req.LeaderCommitIndex = r.getCommitIndex()

	// 获取 nextIndex 之前的 log term 和 index.
	if err := r.setPreviousLog(req, nextIndex); err != nil {
		// 错误则跳出, 如果 ErrLogNotFound 错误, 走发送快照逻辑
		return err
	}
	// 获取 nextIndex 到 lastIndex 之间的增量数据, 最大超过 MaxAppendEntries 个.
	if err := r.setNewLogs(req, nextIndex, lastIndex); err != nil {
		return err
	}
	return nil
}
```

`setPreviousLog` 用来获取 follower 的 nextIndex 的上一条数据, 如果在快照临界点, 则使用快照记录的 index 和 term, 否则其他情况调用 LogStore 存储的 GetLog 获取上一条日志.

**需要注意一下, 如果上一条数据的 index 在 logStore 不存在, 那么就需要返回错误, 后面走发送快照逻辑了.**

```go
func (r *Raft) setPreviousLog(req *AppendEntriesRequest, nextIndex uint64) error {
	// 获取快照文件中最大日志的 index 和 term.
	lastSnapIdx, lastSnapTerm := r.getLastSnapshot()

	// 如果 nextIndex 等于 1, 那么 prev 必然为 0.
	if nextIndex == 1 {
		req.PrevLogEntry = 0
		req.PrevLogTerm = 0

	} else if (nextIndex - 1) == lastSnapIdx {
		// 如果 -1 等于 快照数据的最大 index, 则 prev 使用快照的值.
		req.PrevLogEntry = lastSnapIdx
		req.PrevLogTerm = lastSnapTerm

	} else {
		var l Log
		// 从 LogStore 存储获取上一条日志数据.
		if err := r.logs.GetLog(nextIndex-1, &l); err != nil {
			// 如果日志不存在, 说明是在 snapshot 快照文件中.
			return err
		}

		// 进行赋值.
		req.PrevLogEntry = l.Index
		req.PrevLogTerm = l.Term
	}
	return nil
}
```

`setNewLogs` 用来获取 nextIndex 到 lastIndex 之间的增量数据, 为避免一次传递太多的数据, 这里限定单次不能超过 MaxAppendEntries 条日志.

```go
// setNewLogs is used to setup the logs which should be appended for a request.
func (r *Raft) setNewLogs(req *AppendEntriesRequest, nextIndex, lastIndex uint64) error {
	maxAppendEntries := r.config().MaxAppendEntries
	req.Entries = make([]*Log, 0, maxAppendEntries)

	// 单词批量发送不能超过 maxAppendEntries 条的日志.
	// 如果增量的数据不到 maxAppendEntries, 那么就发送 nextIndex > lastIndex 之间的数据.
	maxIndex := min(nextIndex+uint64(maxAppendEntries)-1, lastIndex)
	for i := nextIndex; i <= maxIndex; i++ {
		oldLog := new(Log)
		if err := r.logs.GetLog(i, oldLog); err != nil {
			return err
		}
		req.Entries = append(req.Entries, oldLog)
	}
	return nil
}
```

#### updateLastAppended

`updateLastAppended` 用来更新记录 follower 的 nextIndex 值, 另外还会调用 `commitment.match` 改变 commit 记录, 并通知让状态机应用.

每个 follower 在同步完数据后, 都需要调用一次 `updateLastAppended`, 不仅更新 follower nextIndex, 更重要的是更新 commitIndex 提交索引值, **`commitment.match` 内部检测到 commit 发生变动时, 向 commitCh 提交通知, 最后由 leaderLoop 检测到 commit 通知, 并调用状态机 fsm 应用.**

在本地提交后, 当下次 replicate 同步数据时, 自然会携带更新后的 commitIndex, 在 follower 收到且经过判断对比后, 把数据更新自身的状态机里.

```
func updateLastAppended(s *followerReplication, req *AppendEntriesRequest) {
	// Mark any inflight logs as committed
	if logs := req.Entries; len(logs) > 0 {
		last := logs[len(logs)-1]
		atomic.StoreUint64(&s.nextIndex, last.Index+1)
		s.commitment.match(s.peer.ID, last.Index)
	}

	// Notify still leader
	s.notifyAll(true)
}
```



🔥 重点:

在 leader 里找到绝大数 follower 都满足的 Index 作为 commitIndex 进行提交, 先本地提交, 随着下次 replicate 同步日志时, 通知其他 follower 也提交日志到本地.

#### 计算并提交 commitIndex



[![img](https://camo.githubusercontent.com/b8efb4338cc5b8bacdd4a629dbca60819434fcaad58679ddd9be1f3c9648ac96/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323137313734323239302e706e67)](https://camo.githubusercontent.com/b8efb4338cc5b8bacdd4a629dbca60819434fcaad58679ddd9be1f3c9648ac96/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f3230323330323137313734323239302e706e67)

`match` 通过计算各个 server 的 matchIndex 计算出 commitIndex. commitIndex 可以理解为法定的提交索引值. 对所有 server 的 matchIndex 进行排序, 然后使用 `matched[(len(matched)-1)/2]` 值作为 commitIndex. 这样比 commitIndex 小的 log index 会被推到 commitCh 管道里. 后面由 leaderLoop 进行消费, 然后调用 fsm 状态机进行应用日志.

```go
func (c *commitment) match(server ServerID, matchIndex uint64) {
	c.Lock()
	defer c.Unlock()
	// 如果传入的 server 已投票, 另外 index 大于上一个记录的 index, 则更新 matchIndex.
	if prev, hasVote := c.matchIndexes[server]; hasVote && matchIndex > prev {
		c.matchIndexes[server] = matchIndex

		// 重新计算
		c.recalculate()
	}
}

func (c *commitment) recalculate() {
	// 需要重计算, 但还未初始化 matchIndex 数据时, 直接退出.
	if len(c.matchIndexes) == 0 {
		return
	}

	// 构建一个容器存放各个 server 的 index.
	matched := make([]uint64, 0, len(c.matchIndexes))
	for _, idx := range c.matchIndexes {
		matched = append(matched, idx)
	}

	// 对整数切片进行排序, 从小到大正序排序
	sort.Sort(uint64Slice(matched))

	// 找到 quorum match index 点. 比如 [1 2 3 4 5], 那么 2 为 quorumMatchIndex 法定判断点.
	quorumMatchIndex := matched[(len(matched)-1)/2]

	// 如果法定判断点大于当前的提交点, 并且法定点大于 first index, 则更新 commitIndex 和通知 commitCh.
	if quorumMatchIndex > c.commitIndex && quorumMatchIndex >= c.startIndex {
		c.commitIndex = quorumMatchIndex

		// 给 commitCh 发送通知, 该 chan 由 leaderLoop 来监听处理.
		asyncNotifyCh(c.commitCh)
	}
}
```



### raft transport 网络层

hashicorp transport 层是使用 msgpack rpc 实现的, 其实现原理没什么可说的.

```go
func (n *NetworkTransport) AppendEntries(id ServerID, target ServerAddress, args *AppendEntriesRequest, resp *AppendEntriesResponse) error {
	return n.genericRPC(id, target, rpcAppendEntries, args, resp)
}

// genericRPC 为通用的 rpc 请求方法
func (n *NetworkTransport) genericRPC(id ServerID, target ServerAddress, rpcType uint8, args interface{}, resp interface{}) error {
	// 获取连接对象
	conn, err := n.getConnFromAddressProvider(id, target)
	if err != nil {
		return err
	}

	// 配置超时
	if n.timeout > 0 {
		conn.conn.SetDeadline(time.Now().Add(n.timeout))
	}

	// 封装请求报文, 发送请求
	if err = sendRPC(conn, rpcType, args); err != nil {
		return err
	}

	// decode 数据到 resp 结构对象中
	canReturn, err := decodeResponse(conn, resp)
	if canReturn {
		n.returnConn(conn)
	}
	return err
}
```

msgpack rpc 的协议报文格式如下:

[![img](https://camo.githubusercontent.com/80b72338e79667fbac7153d5243207cec88836c6ca85020c3dbdd30e254b403e/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f53616d706c65253230466c6f77636861727425323054656d706c6174652532302d322d2e6a7067)](https://camo.githubusercontent.com/80b72338e79667fbac7153d5243207cec88836c6ca85020c3dbdd30e254b403e/68747470733a2f2f7869616f7275692d63632e6f73732d636e2d68616e677a686f752e616c6979756e63732e636f6d2f696d616765732f3230323330322f53616d706c65253230466c6f77636861727425323054656d706c6174652532302d322d2e6a7067)

### follower 处理 appendEntries 日志同步

`appendEntries()` 是用来处理来自 leader 发起的 appendEntries 请求. 其内部首先判断请求的日志是否可以用, 能用则保存日志到本地**, 然后调用 `processLogs` 来通知 fsm 状态机应用日志.**

如果请求的上条日志跟本实例最新日志不一致, 则返回失败. 而 leader 会根据 follower 返回结果, 获取 follower 最新的 log term 及 index, 然后再同步给 follower 缺失的日志. 另外当 follower 发现冲突日志时, 也会以 leader 的日志为准来覆盖修复产生冲突的日志.

简单说 `appendEntries()` 同步日志是 leader 和 follower 不断调整位置再同步数据的过程.

```go
// appendEntries 在收到AppendEntries RPC调用时被触发。
// 这个函数必须只在Raft的主线程（事件循环）中调用，以避免并发问题。
func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest) {
	// 使用 metrics 记录 appendEntries RPC 处理的时间。
	defer metrics.MeasureSince([]string{"raft", "rpc", "appendEntries"}, time.Now())

	// 初始化 AppendEntries 响应结构体。
	// 默认设置为失败，并包含当前节点的Term和最后一个日志条目的索引。
	resp := &AppendEntriesResponse{
		RPCHeader:      r.getRPCHeader(),   // 获取标准的RPC头部信息
		Term:           r.getCurrentTerm(), // 响应中包含当前节点的Term
		LastLog:        r.getLastIndex(),   // 响应中包含当前节点的最后一个日志索引
		Success:        false,              // 初始设置为失败
		NoRetryBackoff: false,              // 初始设置为允许重试退避
	}
	var rpcErr error // 用于存储处理过程中可能发生的错误

	// 使用 defer 确保在函数退出前发送响应。
	defer func() {
		rpc.Respond(resp, rpcErr)
	}()

	// 规则 1: 如果请求的Term小于当前节点的Term，则忽略该请求。
	// Leader的Term比Follower旧，说明该Leader已失效。
	if a.Term < r.getCurrentTerm() {
		return // 直接返回，不处理旧Term的AppendEntries
	}

	// 规则 2: 如果请求的Term大于当前节点的Term，或者当前节点不是Follower且不是正在进行领导权转移的Candidate，则更新Term，并转换为Follower状态。
	// 这是Raft的核心规则：看到更高的Term总是意味着过时，必须回退到Follower状态。
	if a.Term > r.getCurrentTerm() || (r.getState() != Follower && !r.candidateFromLeadershipTransfer.Load()) {
		r.setState(Follower)
		r.setCurrentTerm(a.Term)
		resp.Term = a.Term // 更新响应中的Term为新的当前Term
	}

	// 记录Leader的地址和ID。
	if len(a.Addr) > 0 {
		r.setLeader(r.trans.DecodePeer(a.Addr), ServerID(a.ID))
	} else {
		r.setLeader(r.trans.DecodePeer(a.Leader), ServerID(a.ID))
	}

	// 规则 3: 验证前一个日志条目的匹配性（Log Consistency Check）。
	// Leader在AppendEntries请求中包含新条目紧前一个条目的索引(PrevLogEntry)和Term(PrevLogTerm)。
	// Follower必须检查自己日志中对应索引的条目Term是否与Leader一致。
	if a.PrevLogEntry > 0 { // PrevLogEntry == 0 表示这是第一个日志条目，不需要检查前一个
		lastIdx, lastTerm := r.getLastEntry() // 获取当前节点的最后一个日志条目索引和Term

		var prevLogTerm uint64 // 用于存储当前节点 PrevLogEntry 索引处的日志条目的Term
		if a.PrevLogEntry == lastIdx {
			// 如果 Leader 的 PrevLogEntry 恰好是当前节点的最后一个日志条目
			prevLogTerm = lastTerm // 直接使用最后一个日志条目的Term
		} else {
			// 如果 Leader 的 PrevLogEntry 不是当前节点的最后一个日志条目，需要从日志存储中获取
			var prevLog Log
			// 尝试获取 PrevLogEntry 索引处的日志条目
			if err := r.logs.GetLog(a.PrevLogEntry, &prevLog); err != nil {
				// 这种情况下，Leader应该回退并发送更早的日志，所以设置 NoRetryBackoff = true
				resp.NoRetryBackoff = true
				return
			}
			prevLogTerm = prevLog.Term // 获取到日志条目，记录其Term
		}

		// 如果请求体中上次 term 跟当前 term 不一致, 则直接写失败.
		if a.PrevLogTerm != prevLogTerm {
			// Term 不匹配时，Leader 需要回退并发送更早的日志，所以设置 NoRetryBackoff = true
			resp.NoRetryBackoff = true
			return
		}
	}

	// 规则 4: 处理新的日志条目。
	// 如果请求中包含新的日志条目 (a.Entries)
	if len(a.Entries) > 0 {
		start := time.Now() // 记录开始处理日志条目的时间

		// 删除任何冲突的条目，并跳过任何重复的条目。
		lastLogIdx, _ := r.getLastLog() // 获取当前节点的最后一个日志索引 (可能与 getLastEntry 不同，取决于实现细节，这里用于比较)
		var newEntries []*Log           // 用于存放真正需要追加的新条目

		// 遍历 Leader 发送的日志条目
		for i, entry := range a.Entries {
			// 如果当前 Leader 条目的索引大于当前节点的最后一个日志索引，
			// 说明从这里开始的所有条目都是 Leader 新增的，可以直接追加。
			if entry.Index > lastLogIdx {
				newEntries = a.Entries[i:] // 将剩余的条目标记为需要追加的新条目
				break                      // 跳出循环，后续只处理 newEntries
			}

			// 如果 Leader 条目的索引不大于当前节点的最后一个日志索引，
			// 说明当前节点可能已经有了这个索引的条目，需要检查是否冲突。
			var storeEntry Log
			// 尝试从存储中获取当前索引的日志条目
			if err := r.logs.GetLog(entry.Index, &storeEntry); err != nil {
				return
			}

			// 对比 Leader 条目的Term和当前节点对应索引条目的Term
			if entry.Term != storeEntry.Term {
				// 删除从冲突索引到最后一个索引的日志范围
				if err := r.logs.DeleteRange(entry.Index, lastLogIdx); err != nil {
					// 删除日志失					r.logger.Error("failed to clear log suffix", "error", err)
					return
				}
				// 如果被删除的范围包含最新的配置变更日志条目，需要回退最新的配置信息
				if entry.Index <= r.configurations.latestIndex {
					// 将最新的配置设置为已提交的配置，索引也回退到已提交的索引
					r.setLatestConfiguration(r.configurations.committed, r.configurations.committedIndex)
				}
				// 从当前冲突的条目开始，Leader 的所有条目都视为新的，需要追加。
				newEntries = a.Entries[i:]
				break // 跳出循环，后续只处理 newEntries
			}
			// 如果索引小于等于 lastLogIdx 且 Term 匹配，说明这个条目是重复的，已经被 Follower 拥有且一致。
			// 继续循环检查下一个 Leader 条目。
		}

		// 如果有需要追加的新条目 (newEntries 列表不为空)
		if n := len(newEntries); n > 0 {
			// 将新条目追加到日志存储中。
			if err := r.logs.StoreLogs(newEntries); err != nil {
				return
			}

			// 处理任何新的配置变更日志条目。 需要在日志条目追加到存储后处理配置变更。
			for _, newEntry := range newEntries {
				// 对于每个新追加的条目，检查它是否是配置变更条目，并进行处理。
				if err := r.processConfigurationLogEntry(newEntry); err != nil {
					rpcErr = err // 记录RPC错误
					return
				}
			}

			// 更新当前节点的最后一个日志索引和Term，基于实际追加的最后一个条目。
			last := newEntries[n-1]             // 获取追加的最后一个条目
			r.setLastLog(last.Index, last.Term) // 更新节点的 lastLog 状态
		}
	}

	// 规则 5: 更新当前节点的提交索引 (Commit Index)。
	// Leader 在 AppendEntries 请求中包含自己的提交索引 (LeaderCommitIndex)。
	// Follower 必须将自己的提交索引更新为 min(Leader 的提交索引, 自己最后一个日志条目的索引)。
	if a.LeaderCommitIndex > 0 && a.LeaderCommitIndex > r.getCommitIndex() {
		start := time.Now() // 记录开始处理提交的时间
		// 计算新的提交索引：取 Leader 的提交索引和当前节点最后一个日志索引的最小值。
		idx := min(a.LeaderCommitIndex, r.getLastIndex())
		// 更新当前节点的提交索引。
		r.setCommitIndex(idx)

		// 如果最新的配置变更日志条目索引小于等于新的提交索引，则表示该配置变更已提交。
		if r.configurations.latestIndex <= idx {
			// 将最新的配置设置为已提交的配置，并更新已提交的索引。
			r.setCommittedConfiguration(r.configurations.latest, r.configurations.latestIndex)
		}

		// 重点把 commitIndex 之前的日志提交到状态机 FSM 进行应用日志.。
		// 从旧的提交索引开始（或0）处理到新的提交索引 idx。
		r.processLogs(idx, nil) // nil 表示应用到默认的状态机 (或这里没有特定的应用函数)
	}

	// 如果执行到这里没有返回错误或因为旧Term/日志不匹配而提前返回，说明 AppendEntries 成功。
	resp.Success = true
	// 记录最后一次与Leader成功通信的时间。这用于重置选举超时定时器。
	r.setLastContact()
}
```



### 状态机 FSM 应用日志

不管是 Leader 和 Follower 都会调用状态机 FSM 来应用日志. 其流程是先调用 `processLogs` 来打包批量日志, 然后将日志推到 `fsmMutateCh` 管道里, 最后由 `runFSM` 协程来监听该管道, 并把日志应用到状态机里面.

```go
// processLogs 用于应用所有尚未应用的已提交日志条目，直到给定的索引上限。
// 该方法可以被领导者（Leader）和跟随者（Follower）调用。
// 1. 跟随者通过 AppendEntries 方法调用此函数，一次处理 n 条日志条目，且总是传递 futures=nil。
// 2. 领导者在日志条目被提交时调用此函数，并传递来自任何正在处理中的日志的 futures。
func (r *Raft) processLogs(index uint64, futures map[uint64]*logFuture) {
	// 获取当前已经应用的最后一个日志条目的索引
	lastApplied := r.getLastApplied()

	// 如果传入的索引小于或等于已应用的最后一个索引，说明这些日志已经被应用，跳过处理
	if index <= lastApplied {
		// 记录警告日志，表示正在跳过已应用的旧日志
		r.logger.Warn("skipping application of old log", "index", index)
		return
	}

	// 定义 applyBatch 函数，用于将一批日志条目提交给状态机（FSM）进行处理
	applyBatch := func(batch []*commitTuple) {
		select {
		case r.fsmMutateCh <- batch:
			// 将批次日志发送到状态机的通道中，由状态机负责实际应用这些日志
		case <-r.shutdownCh:
			// 如果 Raft 实例正在关闭，则遍历批次中的每个日志条目
			for _, cl := range batch {
				// 如果该日志条目有关联的 future（即异步操作的回调），则通知其 Raft 已关闭
				if cl.future != nil {
					cl.future.respond(ErrRaftShutdown)
				}
			}
		}
	}

	// 获取配置中的 MaxAppendEntries 参数，表示一次最多可以处理的日志条目数量
	maxAppendEntries := r.config().MaxAppendEntries

	// 创建一个批次容器，用于存储待提交给状态机的日志条目
	batch := make([]*commitTuple, 0, maxAppendEntries)

	// 遍历从 lastApplied+1 到 index 的所有日志条目，逐一处理
	for idx := lastApplied + 1; idx <= index; idx++ {
		var preparedLog *commitTuple
		// 检查是否存在与当前索引对应的 future（即领导者传递的正在处理的日志条目）
		future, futureOk := futures[idx]
		if futureOk {
			// 如果存在 future，则从 future 中提取日志条目并进行预处理
			// prepareLog 方法会将日志条目封装为 commitTuple 结构，供状态机处理
			preparedLog = r.prepareLog(&future.log, future)
		} else {
			// 如果没有 future，则从日志存储（log store）中获取对应索引的日志条目
			l := new(Log)
			if err := r.logs.GetLog(idx, l); err != nil {
				// 如果获取日志失败，记录错误日志并触发 panic，因为这通常表示系统状态不一致
				r.logger.Error("failed to get log", "index", idx, "error", err)
				panic(err)
			}
			// 对从日志存储中获取的日志条目进行预处理，注意这里没有 future
			preparedLog = r.prepareLog(l, nil)
		}

		// 根据预处理结果进行不同的处理
		switch {
		case preparedLog != nil:
			// 如果日志条目已经成功预处理（即 preparedLog 不为空），则将其加入批次容器
			// 该日志条目将被发送到状态机线程进行处理，状态机会负责调用 future 的回调
			batch = append(batch, preparedLog)

			// 如果当前批次的大小达到或超过 maxAppendEntries，则提交该批次
			if len(batch) >= maxAppendEntries {
				applyBatch(batch)
				// 提交后清空批次容器，并重新分配容量为 maxAppendEntries 的新容器
				batch = make([]*commitTuple, 0, maxAppendEntries)
			}

		case futureOk:
			// 如果存在 future 但 preparedLog 为空，说明该日志条目无需应用到状态机
			// 直接调用 future 的回调函数通知操作完成（通常用于某些特殊类型的日志，如配置变更）
			future.respond(nil)
		}
	}

	// 如果循环结束后批次容器中仍有未提交的日志条目，则提交剩余的批次
	if len(batch) != 0 {
		applyBatch(batch)
	}

	// 更新 lastApplied 索引，表示所有日志条目已应用到给定的 index
	r.setLastApplied(index)
}
```

#### runFSM

**主要作用 (Main Purpose):**

`runFSM` 是一个长期运行的 goroutine（协程），它 **专门负责将已提交的日志条目应用到用户提供的有限状态机 (FSM)**。它还负责处理 FSM 的快照创建和恢复操作。
其核心设计目的是将 FSM 的操作（可能是耗时的 I/O 操作或复杂计算）与 Raft 核心的共识逻辑 **异步隔离** 开来。这样做可以防止 FSM 的潜在阻塞影响 Raft 内部的及时性和性能，例如心跳、选举、日志复制等关键操作。

`runFSM` 方法的核心逻辑是一个无限循环，通过 `select` 监听多个通道，处理不同类型的请求。以下是其主要流程的分解：

1. **初始化和准备**

   - 定义 `lastIndex` 和 `lastTerm`，用于跟踪状态机已应用的最后一个日志条目的索引和任期。
   - 检查状态机是否支持批处理（`BatchingFSM` 接口）和配置存储（`ConfigurationStore` 接口），以决定后续处理方式。

2. **定义核心处理函数**
   方法内部定义了几个关键的处理函数，用于处理不同的请求类型：

   - `applySingle`

     ：处理单个日志条目。

     - 判断日志类型（`LogCommand` 或 `LogConfiguration`），分别调用状态机的 `Apply` 或 `StoreConfiguration` 方法。
     - 更新 `lastIndex` 和 `lastTerm`。
     - 如果日志条目有关联的 `future`，则通过 `future` 返回响应。

   - `applyBatch`

     ：处理一批日志条目。

     - 如果状态机支持批处理（`BatchingFSM`），则过滤出需要发送的日志条目（仅 `LogCommand` 和 `LogConfiguration` 类型），批量调用 `ApplyBatch`。
     - 如果状态机不支持批处理，则逐个调用 `applySingle`。
     - 更新 `lastIndex` 和 `lastTerm`。
     - 为每个日志条目关联的 `future` 返回响应。

   - `restore`

     ：从快照恢复状态机。

     - 打开指定的快照文件，读取元数据和内容。
     - 调用状态机的恢复方法（`Restore`），将快照数据加载到状态机。
     - 更新 `lastIndex` 和 `lastTerm`。
     - 通过 `future` 返回操作结果。

   - `snapshot`

     ：生成状态机快照。

     - 检查是否有新的日志需要快照（如果 `lastIndex` 为 0，则返回错误）。
     - 调用状态机的 `Snapshot` 方法生成快照。
     - 通过 `future` 返回快照对象和操作结果。

3. **主循环监听通道**
   主循环通过 `select` 监听以下通道，处理不同的请求：

   - `r.fsmMutateCh`

     ：处理日志应用或状态机恢复请求。

     - 如果收到的是 `[]*commitTuple`，则调用 `applyBatch` 批量应用日志。
     - 如果收到的是 `restoreFuture`，则调用 `restore` 从快照恢复状态机。

   - `r.fsmSnapshotCh`

     ：处理快照生成请求。

     - 调用 `snapshot` 生成状态机快照。

   - `r.shutdownCh`

     ：监听 Raft 关闭信号。

     - 如果收到关闭信号，则退出 goroutine。

```go
func (r *Raft) runFSM() {
	var lastIndex, lastTerm uint64

	batchingFSM, batchingEnabled := r.fsm.(BatchingFSM)
	configStore, configStoreEnabled := r.fsm.(ConfigurationStore)
	// 处理单个日志条目。
	applySingle := func(req *commitTuple) {
		var resp interface{}
		...

		switch req.log.Type {
		case LogCommand:
			// 调用 fsm 接口进行提交.
			resp = r.fsm.Apply(req.log)

		case LogConfiguration:
			// 更新配置
			configStore.StoreConfiguration(req.log.Index, DecodeConfiguration(req.log.Data))
		}

		// 更新 index 和 term
		lastIndex = req.log.Index
		lastTerm = req.log.Term
	}
// 处理一批日志条目。
	applyBatch := func(reqs []*commitTuple) {
		// 如果用户实现了 BatchingFSM 接口, 则说明允许批量应用日志.
		// 没实现, 则说明没开启该功能.
		if !batchingEnabled {
			for _, ct := range reqs {
				// 只用单次接口调用状态机应用日志
				applySingle(ct)
			}
			return
		}

		var lastBatchIndex, lastBatchTerm uint64
		sendLogs := make([]*Log, 0, len(reqs))
		for _, req := range reqs {
			// 批量打包
		}

		var responses []interface{}
		if len(sendLogs) > 0 {
			start := time.Now()
			// 调用 fsm 批量应用接口
			responses = batchingFSM.ApplyBatch(sendLogs)
			// ...
		}

		// 更新 index.
		lastIndex = lastBatchIndex
		lastTerm = lastBatchTerm

		for _, req := range reqs {
			// 返回通知
			if req.future != nil {
				req.future.response = resp
				req.future.respond(nil)
			}
			// ...
		}
	}
// 从快照恢复状态机
	restore := func(req *restoreFuture) {
		// 根据 request id 获取已保存本地的快照文件
		meta, source, err := r.snapshots.Open(req.ID)
		if err != nil {
			return
		}
		defer source.Close()

		// 尝试恢复快照数据
		if err := fsmRestoreAndMeasure(snapLogger, r.fsm, source, meta.Size); err != nil {
			return
		}

		// Update the last index and term
		// 更新当前最近的日志的 index 和 term.
		lastIndex = meta.Index
		lastTerm = meta.Term
		req.respond(nil)
	}
	// 生成状态机快照。
	snapshot := func(req *reqSnapshotFuture) {
		// ...
	}
	for {
		select {
		case ptr := <-r.fsmMutateCh:
			switch req := ptr.(type) {
			case []*commitTuple:
				// 进行状态机提交
				applyBatch(req)

			case *restoreFuture:
				// 恢复 snapshot 快照文件
				restore(req)

			default:
				// 其他情况直接 panic
				panic(fmt.Errorf("bad type passed to fsmMutateCh: %#v", ptr))
			}

		case req := <-r.fsmSnapshotCh:
			snapshot(req)
		}
	}
}
```



## 二、总结

上述过程简单来说就是, 上层写日志, leader 同步日志, follower 接收日志, leader 确认提交日志, follower 跟着提交日志.



## 三、问题

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





## 四、参考链接

1.[Hashicorp Raft实现和API分析](https://www.cnblogs.com/aganippe/p/16292050.html)

2.[hashicorp raft源码学习](https://qiankunli.github.io/2020/05/17/hashicorp_raft.html)

3.[源码分析 hashicorp raft replication 日志复制的实现原理](https://github.com/rfyiamcool/notes/blob/main/hashicorp_raft_replication_code.md#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-hashicorp-raft-replication-%E6%97%A5%E5%BF%97%E5%A4%8D%E5%88%B6%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

