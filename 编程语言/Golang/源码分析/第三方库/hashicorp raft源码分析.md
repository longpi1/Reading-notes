## hashicorp raft源码分析

> API手册：https://pkg.go.dev/github.com/hashicorp/raft
>
> 源码地址：[hashicorp/raft](https://github.com/hashicorp/raft)
>
> raft论文解读：https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md

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
raft
    fuzzy    // 用来模拟测试 启动一个集群， 启动多个raftNode 节点。
    raft.go
    state.go    // 节点状态相关的数据结构和函数
        type RaftState uint32
        type raftState struct 
    commands.go // RPC 消息相关的数据结构
        type AppendEntriesRequest struct    // 日志复制 RPC 的请求消息
        type AppendEntriesResponse struct
        type RequestVoteRequest struct
        type RequestVoteResponse struct
        type InstallSnapshotRequest struct
        type InstallSnapshotResponse struct
    log.go      // 日志对应的数据结构和函数接口
        type Log struct
        type LogStore interface

github.com/hashicorp/raft/
├── api.go          # 对外暴露的核心 API（Raft 接口定义）
├── config.go       # Raft 节点配置（超时、选举、日志等参数）
├── file_snapshot.go # 基于文件的快照（Snapshot）实现
├── fsm.go          # 有限状态机（Finite State Machine）接口定义
├── future.go       # 异步操作的 Future 模式实现（请求结果回调）
├── log.go          # Raft 日志结构（LogEntry、日志压缩等）
├── log_store.go    # 日志存储接口（BoltDB、内存存储、文件存储实现）
├── net_transport.go # 网络层抽象（TCP、TLS、Inmem 传输实现）
├── observer.go     # 事件监听器（观察者模式，监控集群状态变化）
├── raft.go         # 【核心文件】Raft 状态机、主逻辑（选举、日志复制核心循环）
├── raft_test.go    # 单元测试、集成测试（非常重要，覆盖所有场景）
├── snapshot.go     # 快照管理（压缩日志、持久化状态）
├── stable_store.go # 持久化存储接口（稳定存储 Raft 日志、任期等）
├── transport.go    # 网络传输接口定义（RPC 调用，消息传递）
└── util.go         # 辅助函数（随机数、时间戳、Debug 工具）
```

## 三、功能实现



## 四、问题





## 五、参考链接



