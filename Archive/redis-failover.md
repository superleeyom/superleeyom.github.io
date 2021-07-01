---
title: Redis Cluster 故障转移机制浅析
date: 2018-05-09 18:17:55
tags:
- redis
categories:
- 编程
---
我们知道在单机模式下，单机模式就是指 Redis 主节点以单个节点的形式存在，这个主节点可读可写，从节点只读，通常单机模式为“1主 N 备”的结构，单机模式下要实现故障转移需要“哨兵” Sentinel 辅助，由一个或者多个 Sentinel 实例组成的系统可以监视 Redis 主节点及其从节点，实现故障转移。而 Redis Cluster 集群自身实现了高可用，并不需要借助哨兵，这完全是一个自治的过程，接下来就它的故障转移机制做一个简单的浅析。

<!-- more -->

# 故障发现

故障发现通常包含两个环节：主观下线（pfail）和客观下线（fail）。

## 主观下线

简单理解为：指某个节点认为另一个节点不可用，即下线状态，这个状态并不是最终的故障判定，只能代表一个节点的意见，可能存在误判情况。深入一点理解可以这么说：集群中每个节点都会定期向其他节点发送 ping 消息，接收节点回复 pong 消息作为响应。如果在 `cluster-node-timeout` 时间内通信一直失败，则发送节点会认为接收节点存在故障，把接收节点标记为主观下线（pfail）状态。具体的流程如下：

1. 节点 a 发送 ping 消息给节点 b，如果通信正常将接收到 pong 消息，节点 a 更新最近一次与节点 b 的通信时间。
2. 如果节点 a 与节点 b 通信出现问题则断开连接，下次会进行重连。如 果一直通信失败，则节点 a 记录的与节点 b 最后通信时间将无法更新。
3. 节点 a 内的定时任务检测到与节点 b 最后通信时间超高 `cluster-node-timeout` 时，更新本地对节点 b 的状态为主观下线（pfail）。

当 `cluster-note-timeout` 时间内某节点无法与另一 个节点顺利完成 ping 消息通信时，则将该节点标记为主观下线状态。主观下线的伪代码如下：

```java
// 定时任务,默认每秒执行10次
def clusterCron():
  // ... 忽略其他代码
  for(node in server.cluster.nodes):
    // 忽略自身节点比较
    if(node.flags == CLUSTER_NODE_MYSELF):
      continue;
    // 系统当前时间
    long now = mstime();
    // 自身节点最后一次与该节点PING通信的时间差
    long delay = now - node.ping_sent;
    // 如果通信时间差超过 cluster_node_timeout，将该节点标记为PFAIL（主观下线）
    if (delay > server.cluster_node_timeout) :
      node.flags = CLUSTER_NODE_PFAIL;
```
但是主观下线并不能判定一个节点是否真正的出现故障，比方说，节点 6379 与 6385 通信中断，导致 6379 判断 6385 为主观下线状态，但是 6380 与 6385 节点之间通信正常，这种情况不能判定节点 6385 发生故障。那怎样才能真正的判定一个节点是否故障呢？那这就要求多个节点协作完成故障发现了，这个过程叫客观下线。

## 客观下线

当某个节点判断另外一个节点主观下线后，相应的节点状态会跟随消息在集群内传播。ping/pong 消息的消息体会携带集群 1/10 的其他节点状态数据， 当接受节点发现消息体中含有主观下线的节点状态时，会在本地找到故障节点的 ClusterNode 结构，保存到下线报告链表中。通过 Gossip 消息传播，集群内节点不断收集到故障节点的下线报告。当 **半数以上持有槽的主节点** 都标记某个节点是主观下线时，就会触发客观下线。大致的流程如下：

1. 当消息体内含有其他节点的 pfail 状态会判断发送节点的状态，如果发送节点是主节点则对报告的 pfail 状态处理，如果发送节点是从节点则忽略。
2. 找到 pfail 对应的节点结构，更新 clusterNode 内部下线报告链表。
3. 根据更新后的下线报告链表告尝试进行客观下线。

那对于上面的流程还要细分两个点，就是 **维护下线报告链表** 和 **尝试客观下线**。

### 维护下线报告链表

每个节点 ClusterNode 结构中都会存在一个下线链表结构，保存了其他主节点针对当前节点的下线报告，当接收到 pfail 状态时，会维护对应节点的下线上报链表，伪代码如下：

```java
def clusterNodeAddFailureReport(clusterNode failNode, clusterNode senderNode) :
  // 获取故障节点的下线报告链表
  list report_list = failNode.fail_reports;
  // 查找发送节点的下线报告是否存在
  for(clusterNodeFailReport report : report_list):
    // 存在发送节点的下线报告上报
    if(senderNode == report.node):
      // 更新下线报告时间
      report.time = now();
      return 0;
    // 如果下线报告不存在,插入新的下线报告
    report_list.add(new clusterNodeFailReport(senderNode,now()));
  return 1;
```

每个下线报告都存在有效期，有效期为 `cluster-node-time*2`，每次在尝试触发客观下线时，都会检测下线报告是否过期，对于过期的下线报告将被删除。如果在 `cluster-node-time*2` 的时间内该下线报告没有得到更新则过期并删除，伪代码如下：

```java
def clusterNodeCleanupFailureReports(clusterNode node) :
  list report_list = node.fail_reports;
  long maxtime = server.cluster_node_timeout * 2;
  long now = now();
  for(clusterNodeFailReport report : report_list):
    // 如果最后上报过期时间大于cluster_node_timeout * 2则删除
    if(now - report.time > maxtime):
      report_list.del(report);
```
这样做的目的是防止故障误报，例如节点 A 在上一小时报告节点 B 主观下线，但是之后又恢复正常。现在又有其他节点上报节点 B 主观下线，根据实际情况之前的属于误报不能被使用。

### 尝试客观下线

集群中的节点每次接收到其他节点的 pfail 状态，都会尝试触发客观下线，流程如下：

- 首先统计有效的下线报告数量，如果小于集群内持有槽的主节点总数的一半则退出。
- 当下线报告大于槽主节点数量一半时，标记对应故障节点为客观下线状态。
- 向集群广播一条 fail 消息，通知所有的节点将故障节点标记为客观下线，fail 消息的消息体只包含故障节点的 ID。
  - 通知集群内所有的节点标记故障节点为客观下线状态并立刻生效。
  - 通知故障节点的从节点触发故障转移流程。

整个流程的伪代码如下：

```java
def markNodeAsFailingIfNeeded(clusterNode failNode) {
  // 获取集群持有槽的节点数量
  int slotNodeSize = getSlotNodeSize();
  // 主观下线节点数必须超过槽节点数量的一半
  int needed_quorum = (slotNodeSize / 2) + 1;
  // 统计failNode节点有效的下线报告数量（不包括当前节点）
  int failures = clusterNodeFailureReportsCount(failNode);
  // 如果当前节点是主节点，将当前节点计累加到failures
  if (nodeIsMaster(myself)):
    failures++;
    // 下线报告数量不足槽节点的一半退出
  if (failures < needed_quorum):
    return;
  // 将改节点标记为客观下线状态(fail)
  failNode.flags = REDIS_NODE_FAIL;
  // 更新客观下线的时间
  failNode.fail_time = mstime();
  // 如果当前节点为主节点,向集群广播对应节点的fail消息
  if (nodeIsMaster(myself))
    clusterSendFail(failNode);
}
```

# 故障恢复

故障节点变为客观下线后，如果下线节点是持有槽的主节点则需要在它 的从节点中选出一个替换它，从而保证集群的高可用。下线主节点的所有从 节点承担故障恢复的义务，当从节点通过内部定时任务发现自身复制的主节 点进入客观下线时，将会触发故障恢复流程：
- **资格检查**：
  - 每个从节点都要检查最后与主节点断线时间，判断是否有资格替换故障的主节点。如果从节点与主节点断线时间超过 `cluster-node-time*cluster-slave-validity-factor`，则当前从节点不具备故障转移资格。
  - 参数 `cluster-slave-validity-factor`用于从节点的有效因子，默认为 10。
- **准备选举时间**:
  - 当从节点符合故障转移资格后，更新触发故障选举的时间，只有到达该时间后才能执行后续流程。
  - 复制偏移量越大说明从节点延迟越低，那么它应该具有更高的优先级来替换故障主节点。
  - 使用之上的优先级排名，更新选举触发时间，触发时间越低的从从节点，优先发起选举操作。
    ```java
    def updateFailoverTime():
      // 默认触发选举时间：发现客观下线后一秒内执行。
      server.cluster.failover_auth_time = now() + 500 + random() % 500;
      // 获取当前从节点排名，偏移量越大，排名越靠前，数值越小
      int rank = clusterGetSlaveRank();
      long added_delay = rank * 1000;
      // 使用added_delay时间累加到failover_auth_time中
      server.cluster.failover_auth_time += added_delay;
    ```
- **发起选举**：
  - 当从节点定时任务检测到达故障选举时间（failover_auth_time）到达后，发起选举流程如下：
    - 更新配置纪元
      - 配置纪元是一个只增不减的整数，每个主节点自身维护一个配置纪元 （clusterNode.configEpoch）标示当前主节点的版本，所有主节点的配置纪元都不相等，从节点会复制主节点的配置纪元。
    - 广播选举消息
      - 在集群内广播选举消息（FAILOVER_AUTH_REQUEST），并记录已发送过消息的状态，保证该从节点在一个配置纪元内只能发起一次选举。
      - 请求所有收到这条消息并且具有投票权的主节点给自己投票。
- **选举投票**：
  - 只有持有槽的主节点才会处理故障选举消息，因为每个持有槽的节点在一个配置纪元内都有唯一的一张选票，当接到第一个请求投票的从节点消息时回复 `FAILOVER_AUTH_ACK` 消息作为投票，之后相同配置纪元内其他从节点的选举消息将忽略。
  - 如集群内有 N 个持有槽的主节 点代表有 N 张选票。由于在每个配置纪元内持有槽的主节点只能投票给一个从节点，因此只能有一个从节点获得 N/2+1 的选票，保证能够找出唯一的从节点。
  - 当从节点收集到 N/2+1 个持有槽的主节点投票时，从节点可以执行替换故障节点操作。
  - 当然也有存在投票作废的情况：
    - 每个配置纪元代表了一次选举周期，如果在开始投票之后的 `cluster-node-timeout*2` 时间内从节点没有获取足够数量的投票，则本次选举作废。从节点对配置纪元自增并发起下一轮投票，直到选举成功为止。
  - 还需要知道一个点：
    - 故障主节点也算在投票数内，假设集群内节点规模是3主3从，其中有 2 个主节点部署在一台机器上，当这台机器宕机时，由于从节点无法收集到 3/2+1 个主节点选票将导致故障转移失败。
    - 因此部署集群时所有主节点最少需要部署在3台物理机上才能避免单点问题。
- **替换主节点**：
  - 当从节点收集到足够的选票之后，触发替换主节点操作：
    - 当前从节点取消复制变为主节点。
    - 执行 clusterDelSlot 操作撤销故障主节点负责的槽，并执行 clusterAddSlot 把这些槽委派给自己。
    - 向集群广播自己的 pong 消息，通知集群内所有的节点当前从节点变为主节点并接管了故障主节点的槽信息。
