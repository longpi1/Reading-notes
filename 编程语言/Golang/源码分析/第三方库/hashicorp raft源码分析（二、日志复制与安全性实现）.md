# hashicorp raft源码分析（二、日志复制与安全性实现）

## 一、日志复制（Log Replication）

### leader 复制日志

![img](https://img2022.cnblogs.com/blog/2794988/202205/2794988-20220522153708740-687414905.png)

1. 在`runLeader()`函数中，调用`startStopReplication()`函数，执行日志复制功能
2. 启动一个新协程，调用`replicate()`函数，执行日志复制相关的功能
3. 在`replicate()`函数中，调用`replicateTo()`函数，执行步骤4，如果开启了流水线复制模式，执行步骤5
4. 在`replicateTo()`函数中，进行日志复制和日志一致性检测，如果日志复制成功，则设置`s.allowPipeline = true`，开启流水线复制模式
5. 调用`pipelineReplicate()`函数，采用更搞笑的流水线方式进行日志复制

6. 目标是在环境正常的情况下，提升日志复制性能，如果在日志复制过程中出错了，就进入 RPC 复制模式，继续调用 replicateTo() 函数，进行日志复制。

### follower 接收日志

![img](https://img2022.cnblogs.com/blog/2794988/202205/2794988-20220522155156888-154465698.png)

1. 在`runFollower()`函数中，调用`processRPC()`函数，处理接收到的RPC消息
2. 在`processRPC()`函数中，调用`appendEntries()`函数，处理接收到的日志复制RPC请求
3. `appendEntries()`函数，是跟随者处理日志的核心函数。在步骤3.1中，比较日志一致性；在步骤3.2中，将新日志项存放到本地；在步骤3.3中，根据领导者最新提交的日志项索引值，来计算当前需要被应用的日志项，并应用到本地状态机



## 二、安全性（Safety）









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

