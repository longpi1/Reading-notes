# 分布式一致性算法-Paxos、Raft、ZAB、Gossip

### **为什么需要一致性**

1. 数据不能存在单个节点（主机）上，否则可能出现单点故障。
2. 多个节点（主机）需要保证具有相同的数据。
3. 一致性算法就是为了解决上面两个问题。

### **一致性算法的定义**

> *一致性*就是数据保持*一致*，在分布式系统中，可以理解为多个节点中数据的值是*一致*的。

### **一致性的分类**

- 强一致性

- - 说明：保证系统改变提交以后立即改变集群的状态。

  - 模型：

  - - Paxos
    - Raft（muti-paxos）
    - ZAB（muti-paxos）

- 弱一致性

- - 说明：也叫最终一致性，系统不保证改变提交以后立即改变集群的状态，但是随着时间的推移最终状态是一致的。

  - 模型：

  - - DNS系统
    - Gossip协议

### **一致性算法实现举例**

- Google的Chubby分布式锁服务，采用了Paxos算法
- etcd分布式键值数据库，采用了Raft算法
- ZooKeeper分布式应用协调服务，Chubby的开源实现，采用ZAB算法

### **Paxos算法**

- **概念介绍**

1. Proposal提案，即分布式系统的修改请求，可以表示为**[提案编号N，提案内容value]**
2. Client用户，类似社会民众，负责提出建议
3. Propser议员，类似基层人大代表，负责帮Client上交提案
4. Acceptor投票者，类似全国人大代表，负责为提案投票，**不同意比自己以前接收过的提案编号要小的提案，其他提案都同意**，例如A以前给N号提案表决过，那么再收到小于等于N号的提案时就直接拒绝了
5. Learner提案接受者，类似记录被通过提案的记录员，负责记录提案

- Basic Paxos算法
- **步骤**

1. Propser准备一个N号提案
2. Propser询问Acceptor中的多数派是否接收过N号的提案，如果都没有进入下一步，否则本提案不被考虑
3. Acceptor开始表决，Acceptor**无条件同意**从未接收过的N号提案，达到多数派同意后，进入下一步
4. Learner记录提案

![img](https://pic3.zhimg.com/80/v2-992940ee3313553711a2a9f9422c2f16_1440w.jpg)

​                                                                                                     **Basic Paxos算法**

- - 节点故障

  - - 若Proposer故障，没关系，再从集群中选出Proposer即可
    - 若Acceptor故障，表决时能达到多数派也没问题

  - 潜在问题-**活锁**

  - - 假设系统有多个Proposer，他们不断向Acceptor发出提案，还没等到上一个提案达到多数派下一个提案又来了，就会导致Acceptor放弃当前提案转向处理下一个提案，于是所有提案都别想通过了。

- **Multi Paxos算法**

- - 根据Basic Paxos的改进：整个系统**只有一个**Proposer，称之为Leader。
  - 步骤

1. 若集群中没有Leader，则在集群中选出一个节点并声明它为**第M任Leader**。
2. 集群的Acceptor只表决**最新的Leader**发出的最新的提案
3. 其他步骤和Basic Paxos相同

![img](https://pic2.zhimg.com/80/v2-79a5a32f39fcb379a6dd47f53be3a8a5_1440w.jpg)                                                                          

​                                                                                                                          **Multi Paxos算法** 

- - 算法优化
    Multi Paxos角色过多，对于计算机集群而言，可以将Proposer、Acceptor和Learner三者身份**集中在一个节点上**，此时只需要从集群中选出Proposer，其他节点都是Acceptor和Learner。

### **Raft算法**

Raft 算法是斯坦福大学 2017 年提出，论文名称《In Search of an Understandable Consensus Algorithm》，望文生义，该算法的目的是易于理解。Raft 这一名字来源于"Reliable, Replicated, Redundant, And Fault-Tolerant"（“可靠、可复制、可冗余、可容错”）的首字母缩写。

Raft 使用了分治思想把算法流程分为三个子问题：选举（Leader election）、日志复制（Log replication）、安全性（Safety）三个子问题。

- **Leader 选举**：当前 leader 跪了或集群初始化的情况下，新 leader 被选举出来。
- **日志复制**：leader 必须能够从客户端接收请求，然后将它们复制给其他机器，强制它们与自己的数据一致。
- **安全性**：如何保证上述选举和日志复制的安全，使系统满足最终一致性。

#### （1）概念介绍

在 Raft 中，节点被分为 Leader Follower Cabdidate 三种角色：

- **Leader**：处理与客户端的交互和与 follower 的日志复制等，一般只有一个 Leader；
- **Follower**：被动学习 Leader 的日志同步，同时也会在 leader 超时后转变为 Candidate 参与竞选；
- **Candidate**：在竞选期间参与竞选；

![img](https://static001.geekbang.org/infoq/b1/b1a40bab271b924a959abbaac49f8182.png)



**Term**：是有连续单调递增的编号，每个 term 开始于选举阶段，一般由选举阶段和领导阶段组成。



![img](https://static001.geekbang.org/infoq/ec/ec2cdafb4206d759b8bafd342c107737.png)



**随机超时时间**：Follower 节点每次收到 Leader 的心跳请求后，会设置一个随机的，区间位于[150ms, 300ms)的超时时间。如果超过超时时间，还没有收到 Leader 的下一条请求，则认为 Leader 过期/故障了。

**心跳续命**：Leader 在当选期间，会以一定时间间隔向其他节点发送心跳请求，以维护自己的 Leader 地位。

#### （2）协议流程

- 步骤：Raft算法将一致性问题分解为两个的子问题，**Leader选举**和**状态复制**

- - Leader选举

1. 每个Follower都持有一个**定时器**

![img](https://pic2.zhimg.com/80/v2-54ff345bdee35c3b90efef4e525aea69_1440w.jpg)

2.当定时器时间到了而集群中仍然没有Leader，Follower将声明自己是Candidate并参与Leader选举，同时**将消息发给其他节点来争取他们的投票**，若其他节点长时间没有响应Candidate将重新发送选举信息

![img](https://pic2.zhimg.com/80/v2-ddf8c68b3e1e57594a08f032482e8251_1440w.jpg)

3. 集群中其他节点将给Candidate投票

![img](https://pic2.zhimg.com/80/v2-b0d887514aa22bf81a9ebdd2f03cc2dd_1440w.jpg)

4. 获得多数派支持的Candidate将成为**第M任Leader**（M任是最新的任期）

![img](https://pic4.zhimg.com/80/v2-2bf8a06c823ad2fb38e1be20fe60b0df_1440w.jpg)

5. 在任期内的Leader会**不断发送心跳**给其他节点证明自己还活着，其他节点受到心跳以后就清空自己的计时器并回复Leader的心跳。这个机制保证其他节点不会在Leader任期内参加Leader选举。

![img](https://pic1.zhimg.com/80/v2-bd510806788699634cab9500f6c2edfc_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-5beebe7894d879cf1ea54f460467649e_1440w.jpg)

6. 当Leader节点出现故障而导致Leader失联，没有接收到心跳的Follower节点将准备成为Candidate进入下一轮Leader选举

7. 若出现两个Candidate同时选举并获得了相同的票数，那么这两个Candidate将**随机推迟一段时间**后再向其他节点发出投票请求，这保证了再次发送投票请求以后不冲突

![img](https://pic3.zhimg.com/80/v2-70d1b9ad1fdb058b04ff7d5a521435be_1440w.jpg)

- - 状态复制

1. Leader负责接收来自Client的提案请求**（红色提案表示未确认）**

![img](https://pic1.zhimg.com/80/v2-a27ccc3437ed92669fa479f142176614_1440w.jpg)

2. 提案内容将包含在Leader发出的**下一个心跳中**

![img](https://pic3.zhimg.com/80/v2-461b76641286d57534288d6b3be4dbf6_1440w.jpg)

3. Follower接收到心跳以后回复Leader的心跳

![img](https://pic2.zhimg.com/80/v2-f5e771562f2eda04c8db30d1d8462e39_1440w.jpg)

4. Leader接收到多数派Follower的回复以后**确认提案**并写入自己的存储空间中并**回复Client**

![img](https://pic4.zhimg.com/80/v2-3e614177fe59302ec933b629f55be2f7_1440w.jpg)

5. Leader**通知Follower节点确认提案**并写入自己的存储空间，随后所有的节点都拥有相同的数据

![img](https://pic2.zhimg.com/80/v2-dd0d388855310707dd85b4ed6c8c5f6d_1440w.jpg)

6. 若集群中出现网络异常，导致集群被分割，将出现多个Leader

![img](https://pic2.zhimg.com/80/v2-ad1fa60698389bc0efbe3413b2549729_1440w.jpg)

7. 被分割出的非多数派集群将无法达到共识，即**脑裂**，如图中的A、B节点将无法确认提案

![img](https://pic1.zhimg.com/80/v2-093a8f1ea3c1ea389281036f7ee9c320_1440w.jpg)

![img](https://pic2.zhimg.com/80/v2-e54f332371974d6124ba494fc68f6fd5_1440w.jpg)

8. 当集群再次连通时，将**只听从最新任期Leader**的指挥，旧Leader将退化为Follower，如图中B节点的Leader（任期1）需要听从D节点的Leader（任期2）的指挥，此时集群重新达到一致性状态

![img](https://pic2.zhimg.com/80/v2-c86b7d545b291d7d07d4b47e53a334cd_1440w.jpg)

![img](https://pic2.zhimg.com/80/v2-477365ef270e6e967fee60e64886cd6d_1440w.jpg)

### **ZAB算法**

- 说明：ZAB也是对Multi Paxos算法的改进，大部分和raft相同
- 和raft算法的主要区别：

1. 对于Leader的任期，raft叫做term，而ZAB叫做epoch
2. 在状态复制的过程中，raft的心跳从Leader向Follower发送，而ZAB则相反。

### **Gossip算法**

- 说明：Gossip算法每个节点都是对等的，即没有角色之分。Gossip算法中的每个节点都会将数据改动告诉其他节点**（类似传八卦）**。有话说得好："最多通过六个人你就能认识全世界任何一个陌生人"，因此数据改动的消息很快就会传遍整个集群。
- 步骤：

1. 集群启动，如下图所示（这里设置集群有20个节点）

![img](https://pic3.zhimg.com/80/v2-b246affb04dd4a8d8ada2309be2aa03e_1440w.jpg)

2. 某节点收到数据改动，并将改动传播给其他4个节点，传播路径表示为较粗的4条线

![img](https://pic4.zhimg.com/80/v2-4d20a1caeb116d252573364af1684063_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-1984160f78595b28e0da899365cde26e_1440w.jpg)

3. 收到数据改动的节点重复上面的过程直到所有的节点都被感染

## **参考文献**

[Raft](https://link.zhihu.com/?target=http%3A//thesecretlivesofdata.com/raft/)

[https://github.com/flopezluis/gossip-simulator](https://link.zhihu.com/?target=https%3A//github.com/flopezluis/gossip-simulator)

[https://www.youtube.com/channel/UCr](https://link.zhihu.com/?target=https%3A//www.youtube.com/channel/UCrTVwxlwmn2CJINfuaiLB1Q)

b站视频：https://www.bilibili.com/video/BV1TW411M7Fx/?spm_id_from=333.788.recommend_more_video.-1&vd_source=34718180774b041b23050c8689cdbaf2

进阶学习可查看：[Raft论文导读与etcd解读](https://hardcore.feishu.cn/docs/doccnMRVFcMWn1zsEYBrbsDf8De#)