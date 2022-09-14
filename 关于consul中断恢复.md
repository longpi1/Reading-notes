# 关于consul中断恢复

根据您的[部署配置](https://www.consul.io/docs/internals/consensus.html#deployment_table)，集群不可用可能只需要一个服务器故障。恢复需要操作员进行干预，但过程很简单。

该数据中心概述了从由于集群中的大多数服务器节点丢失而导致的 Consul 中断中恢复的过程。有几种类型的中断，具体取决于服务器节点的数量和故障服务器节点的数量。我们将概述如何从以下情况中恢复：

- 单个服务器集群故障。这是当你有一个单独的 Consul 服务器并且它失败的时候。
- 多服务器集群中的服务器发生故障。这是当一台服务器发生故障，但 Consul 集群有三台或更多台服务器时。
- 多服务器集群中的多台服务器出现故障。这在由三台或更多服务器组成的集群中多台服务器发生故障时。这种情况可能是最严重的，因为它可能导致数据丢失。

## [»](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#failure-of-a-single-server-cluster)单服务器集群故障

如果您只有一个服务器并且它发生故障，只需重新启动它。单个服务器配置需要 [`-bootstrap`](https://www.consul.io/docs/agent/options.html#_bootstrap)or [`-bootstrap-expect=1`](https://www.consul.io/docs/agent/options.html#_bootstrap_expect) 标志。

```shell-session
$ consul agent -bootstrap-expect=1
```

如果服务器无法恢复，则需要使用 [部署指南](https://learn.hashicorp.com/tutorials/consul/deployment-guide)启动新服务器。

在单个服务器集群中出现不可恢复的服务器故障并且没有备份过程的情况下，数据丢失是不可避免的，因为数据没有复制到任何其他服务器。这就是为什么 **从不**推荐单服务器部署的原因。

[当代理执行反熵](https://www.consul.io/docs/internals/anti-entropy.html)时，新服务器上线时，任何向代理注册的服务都将重新填充 。

### [»](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#failure-of-a-single-server-cluster-after-upgrading-raft-protocol)单服务器集群故障：升级Raft协议后

将服务器[`-raft-protocol`](https://www.consul.io/docs/agent/options.html#_raft_protocol)从版本升级`2`到后`3`，服务器运行的 Raft 协议版本 3 将不再允许添加运行旧版本的服务器。

在单服务器集群场景中，重启服务器可能会导致服务器无法选举自己为领导者，从而导致集群无法使用。

要从中恢复，请转到[`-data-dir`](https://nomadproject.io/docs/configuration/index.html#data_dir)失败的 Consul 服务器。在数据目录里面会有一个`raft/`子目录。在目录中创建一个`raft/peers.json`文件`raft/`。

对于 Raft 协议版本 3 及更高版本，这应该被格式化为一个 JSON 数组，其中包含节点 ID、地址：端口和 Consul 服务器的选举权信息，如下所示：

```json
[
  {
    "id": "<node-id>",
    "address": "<node-ip>:8300",
    "non_voter": false
  }
]
```

确保你用正确的值替换你的 Consul 服务器`node-id`。`node-ip`

最后，重启你的 Consul 服务器。

## [»](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#failure-of-a-server-in-a-multi-server-cluster)多服务器集群中的服务器故障

如果故障服务器是可恢复的，最好的选择是将其重新联机并使用相同的 IP 地址重新加入集群。这将使集群恢复到完全健康的状态。同样，如果您需要重建一个新的 Consul 服务器，以替换故障节点，您可能希望立即执行此操作。请注意，重建的服务器需要与故障服务器具有相同的 IP 地址。同样，一旦此服务器在线并重新加入，集群将恢复到完全健康的状态。

```shell-session
$ consul agent -bootstrap-expect=3 -bind=192.172.2.4 -auto-rejoin=192.172.2.3
```

这两种策略都可能需要很长时间来重新启动或重建故障服务器。如果这是不切实际的，或者如果无法构建具有相同 IP 的新服务器，则需要删除发生故障的服务器。[`consul force-leave`](https://www.consul.io/commands/force-leave)通常，如果故障服务器仍然是集群的成员，您可以发出命令来移除故障服务器。

```shell-session
$ consul force-leave <node.name.consul>
```

如果[`consul force-leave`](https://www.consul.io/commands/force-leave)无法删除服务器，您有两种方法可以删除它，具体取决于您的 Consul 版本：

- 在 Consul 0.7 及更高版本中，[`consul operator`](https://www.consul.io/docs/commands/operator/raft.html)如果集群有领导者，您可以使用该命令即时删除陈旧的对等服务器，而无需停机。
- 在 0.7 之前的 Consul 版本中，您可以使用`raft/peers.json`所有剩余服务器上的恢复文件手动删除过时的对等服务器。有关此过程的详细信息，请参阅以下[部分](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#manual-recovery-using-peers-json)。此过程需要 Consul 停机才能完成。

在 Consul 0.7 及更高版本中，您可以使用以下`consul operator` 命令检查 Raft 配置：

```shell-session
$ consul operator raft list-peers

Node     ID              Address         State     Voter RaftProtocol
alice    10.0.1.8:8300   10.0.1.8:8300   follower  true  3
bob      10.0.1.6:8300   10.0.1.6:8300   leader    true  3
carol    10.0.1.7:8300   10.0.1.7:8300   follower  true  3
```

## [»](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#failure-of-multiple-servers-in-a-multi-server-cluster)多服务器集群中的多台服务器故障

如果多台服务器丢失，导致仲裁丢失和完全中断，则可以使用集群中剩余服务器上的数据进行部分恢复。在这种情况下可能会丢失数据，因为丢失了多个服务器，因此有关已提交内容的信息可能不完整。恢复过程会隐式提交所有未提交的 Raft 日志条目，因此也可以提交失败前未提交的数据。

有关恢复过程的详细信息，请参阅下面有关使用 peers.json 进行手动恢复的部分。您只需在 `raft/peers.json`恢复文件中包含剩余的服务器。`raft/peers.json`一旦剩余的服务器都以相同的配置重新启动，集群应该能够选举领导者 。

您稍后介绍的任何新服务器都可以是全新的，具有完全干净的数据目录，并使用 Consul 的`join`命令加入。

```shell-session
$ consul agent -join=192.172.2.3
```

在极端情况下，应该可以通过将其自身作为 `raft/peers.json`恢复文件中唯一的对等方启动该单个服务器来仅使用一个剩余的服务器进行恢复。

在 Consul 0.7 之前，它并不总是可以从某些类型的中断中恢复，`raft/peers.json`因为这是在任何 Raft 日志条目被回放之前被摄取的。在 Consul 0.7 及更高版本中，`raft/peers.json` 恢复文件是最终文件，并在摄取后拍摄快照，因此您可以保证从恢复的配置开始。这确实会隐式提交所有 Raft 日志条目，因此应该只用于从中断中恢复，但它应该允许从存在一些可用集群数据的任何情况中恢复。

### [»](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#manual-recovery-using-peers-json)使用 peers.json 手动恢复

首先，停止所有剩余的服务器。您可以尝试优雅的休假，但在大多数情况下都行不通。如果请假退出时出错，请不要担心。集群处于不健康状态，因此这是意料之中的。

在 Consul 0.7 及更高版本中，该`peers.json`文件默认不再存在，仅在执行恢复时使用。在 Consul 启动并摄取该文件后，该文件将被删除。Consul 0.7 还使用一个新的、自动创建`raft/peers.info`的文件来避免`raft/peers.json`在升级后第一次启动时摄取该文件。请务必留`raft/peers.info`在原地以进行正常操作。

`raft/peers.json`用于恢复可能会导致未提交的 Raft 日志条目被隐式提交，因此只能在没有其他选项可用于恢复丢失的服务器的中断后使用。确保您没有任何自动化流程来定期放置对等文件。

下一步是转到[`-data-dir`](https://nomadproject.io/docs/configuration/index.html#data_dir)每个 Consul 服务器的。在该目录中，将有一个`raft/`子目录。我们需要创建一个`raft/peers.json`文件。该文件的格式取决于服务器为其 [Raft 协议](https://www.consul.io/docs/agent/options.html#_raft_protocol)版本配置的内容。

对于 Raft 协议版本 2 和更早版本，这应该被格式化为一个 JSON 数组，其中包含集群中每个 Consul 服务器的地址和端口，如下所示：

```json
["10.1.0.1:8300", "10.1.0.2:8300", "10.1.0.3:8300"]
```

对于 Raft 协议版本 3 及更高版本，这应格式化为 JSON 数组，其中包含集群中每个 Consul 服务器的节点 ID、地址：端口和选举权信息，如下所示：

```json
[
  {
    "id": "adf4238a-882b-9ddc-4a9d-5b6758e4159e",
    "address": "10.1.0.1:8300",
    "non_voter": false
  },
  {
    "id": "8b6dda82-3103-11e7-93ae-92361f002671",
    "address": "10.1.0.2:8300",
    "non_voter": false
  },
  {
    "id": "97e17742-3103-11e7-93ae-92361f002671",
    "address": "10.1.0.3:8300",
    "non_voter": false
  }
]
```

- [`id`](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#id) `(string: <required>)`- 指定服务器的[节点 ID](https://www.consul.io/docs/agent/options.html#_node_id)。如果它是自动生成的，则可以在服务器启动时的日志中找到它，也可以`node-id`在服务器数据目录的文件中找到它。
- [`address`](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#address) `(string: <required>)`- 指定服务器的 IP 和端口。该端口是用于集群通信的服务器的 RPC 端口。
- [`non_voter`](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#non_voter) `(bool: <false>)`- 控制服务器是否为非投票者，在一些高级 [自动驾驶](https://learn.hashicorp.com/tutorials/consul/autopilot-datacenter-operations) 配置中使用。如果省略，它将默认为 false，这对于大多数集群来说是典型的。

只需为所有服务器创建条目。您必须确认您未在此处包含的服务器确实发生了故障，并且以后不会重新加入集群。确保此文件在所有剩余的服务器节点中相同。

此时，您可以重新启动所有剩余的服务器。在 Consul 0.7 及更高版本中，您将看到它们摄取恢复文件：

```log
...
[INFO] consul: found peers.json file, recovering Raft configuration
...
[INFO] consul.fsm: snapshot created in 12.484µs
[INFO] snapshot: Creating new snapshot at /tmp/peers/raft/snapshots/2-5-1471383560779.tmp
[INFO] consul: deleted peers.json file after successful recovery
[INFO] raft: Restored from snapshot 2-5-1471383560779
[INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:10.212.15.121:8300 Address:10.212.15.121:8300}]
...
```

如果任何服务器设法执行优雅离开，您可能需要使用以下 [`join`](https://www.consul.io/commands/join)命令让它们重新加入集群：

```shell-session
$ consul join <Node Address>

Successfully joined cluster by contacting 1 nodes.
```

应该注意的是，任何现有成员都可以用来重新加入集群，因为 gossip 协议将负责发现服务器节点。

此时，集群应该再次处于可操作状态。其中一个节点应声明领导权并发出如下日志：

```log
[INFO] consul: cluster leadership acquired
```

在 Consul 0.7 及更高版本中，您可以使用以下`consul operator` 命令检查 Raft 配置：

```shell-session
$ consul operator raft list-peers

Node     ID              Address         State     Voter  RaftProtocol
alice    10.0.1.8:8300   10.0.1.8:8300   follower  true   3
bob      10.0.1.6:8300   10.0.1.6:8300   leader    true   3
carol    10.0.1.7:8300   10.0.1.7:8300   follower  true   3
```

### [»](https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations#kubernetes-specific-recovery-steps)Kubernetes 特定的恢复步骤

文件`peers.json`恢复方法在 Kubernetes 中运行良好，但有一个例外。在远程执行到服务器 pod 并创建`peers.json`文件后，您通常会通过删除其各自的 pod 来重新启动服务器。Kubernetes 然后重新调度 Consul 服务器节点，并创建并启动一个新的 pod。问题是——新的 pod 也有一个新的 IP 地址，这会导致`peers.json`文件中的信息过时。

要解决此问题 - 请遵循以下基本流程：

1. 远程执行到每个服务器 pod 并`peers.json`使用每个服务器 pod 的当前 IP 地址创建一个如上所述的文件。
2. `consul leave`当您在集群内时，通过远程执行在每个服务器 pod 上执行一次。这将导致`consul`进程退出并由 Kubernetes 重新启动。Kubernetes 不会重新调度 pod，因此它会保留相同的 IP 地址。

按照上述步骤，您应该可以恢复您的 Kubernetes 集群。

原文地址：https://learn.hashicorp.com/tutorials/consul/recovery-outage?in=consul/datacenter-operations