---
title: Redis Sentinel 的安装与部署
date: 2018-04-20 10:18:17
tags:
- Sentinel
categories:
- 编程
---

Redis 的主从复制可以支撑住大并发量的读写操作，但是主从复制也会带来如下的的问题，一旦主节点出现故障，需要手动将一个从节点晋升为主节点，同时需要修改应用方的主节点地址，还需要命令其他从节点去复制新的主节点，整个过程都需要人工干预，无论对于 Redis 的应用方还是运维方都带来了很大的不便。Redis Sentinel 正是用于解决这些问题，当主节点出现故障时，Redis Sentinel 能自动完成故障发现和故障转移， 并通知应用方，从而实现真正的高可用。

<!--more-->

# 高可用性

Redis Sentinel 是一个分布式架构，其中包含若干个 Sentinel（哨兵） 节点和 Redis 数据节点，每个 Sentinel 节点会对数据节点和其余Sentinel 节点进行监控，当它发现节点不可达时，会对节点做下线标识。如果被标识的是主节点，它还会和其他 Sentinel 节点进行“协商”，当大多数 Sentinel 节点都认为主节点不可达时，它们会选举出一个 Sentinel 节点来完成自动故障转移的工作，同时会将这个变化实时通知给Redis 应用方。整个过程完全是自动的，不需要人工来介入，所以这套方案很有效地解决了 Redis 的高可用问题。

下面以1个主节点、2个从节点、3个 Sentinel 节点组成的 Redis Sentinel 为例子进行说明，拓扑结构如图所示：
<img src="http://image.leeyom.top/20180425152462467352740.png" title="Redis Sentinel 拓扑图" />

整个故障转移的处理逻辑有下面4个步骤：

1. 主节点出现故障，此时两个从节点与主节点失去连接，主从复制失败。
2. 每个 Sentinel 节点通过定期监控发现主节点出现了故障。
3. 多个 Sentinel 节点对主节点的故障达成一致，选举出 sentinel-3 节点作为领导者负责故障转移。
4. Sentinel 领导者节点执行了故障转移，具体的细节如下：
    - 如果主节点无法正常启动，需要选出一个从节点（slave-1），对其执行 `slaveof no one` 命令使其成为新的主节点（new-master）。
    - 另一个从节点（slave-2）去复制新的主节点（new-master），执行 `slaveof new-master` 命令。
    - 原来的从节点（slave-1）成为新的主节点后，更新应用方的主节点信息，重新启动应用方（client）。
    - 待原来的主节点（old-master）恢复后，让它去复制新的主节点（new-master），`slaveof new-master`。

  故障转移后的拓扑图如下：
  <img src="http://image.leeyom.top/20180425152462854435905.png" title="故障转移后拓扑图" />

Sentinel 节点至少三个且奇数个，3个以上是通过增加 Sentinel 节点的个数提高对于故障判定的准确性，因为 Sentinel 领导者选举需要至少一半加 1 （N/2+1）个节点，奇数个节点可以在满足该条件的基础上节省一个节点。多个 Sentinel 节点可以防止误判，即使个别 Sentinel 节点不可用，整个 Sentinel 节点集合依然是健壮的。Sentinel 节点其实是一个独立的 Redis 节点，只是他们并不存放数据，只支持部分命令，这里需要注意一下。

# 安装与部署

下面将以 3 个 Sentinel 节点、1 个主节点、2 个从节点组成一个 Redis Sentinel，并进行安装和部署，拓扑图如下：
<img src="http://image.leeyom.top/20180425152462948012671.png" title="Redis Sentinel 安装示例拓扑图" />

## 部署 Redis 节点

### 准备

```
$ wget http://download.redis.io/releases/redis-4.0.9.tar.gz
$ tar xzf redis-4.0.9.tar.gz
$ cd redis-4.0.9
$ make
```

- 下载 Redis 指定版本的源码压缩包到当前目录，推荐安装目录：`/usr/local/`。
- 进入redis目录。
- 编译（编译之前确保操作系统已经安装 gcc）。
- 安装。
  - 进入 src 目录，将 `redis-benchmark`、`redis-cli`、`redis-check-aof`、`redis-check-rdb`、`redis-sentinel`、`redis-server` 执行文件拷贝到 `/usr/local/bin/` 目录下。
  - 借助 chmod 指令修改文件权限：`chmod 777 redis-*`，否则执行对应的命令会出现 `permission denied` 权限不足。

### 主节点

- 将 `redis.conf` 配置文件拷贝一份到 `/etc/redis/` 目录下，并重命名为：`redis-6379.conf`。
  ```
  $ cp redis.conf /etc/redis/redis-6379.conf
  ```
- 配置 `redis-6379.conf`：
  ```
  # 端口
  port 6379
  # 后台运行
  daemonize yes
  # 日志名称
  logfile "6379.log"
  # 持久化文件
  dbfilename "dump-6379.rdb"
  # 工作目录，日志和数据持久化默认存放位置
  dir "/opt/soft/redis/data/"
  ```
- 启动主节点：`redis-server /etc/redis/redis-6379.conf`。
- 确认是否启动，ping 命令检测一下，确认 Redis 节点是否启动了。
  ```
  $ redis-cli -h 127.0.0.1 -p 6379 ping
  PONG
  ```

### 从节点

两个从节点的配置是完全一样的，下面以一个从节点为例子进行说明， 和主节点的配置不一样的是添加了 slaveof 配置，例如 `redis-6380.conf`：

```
port 6380
daemonize yes
logfile "6380.log"
dbfilename "dump-6380.rdb"
dir "/opt/soft/redis/data/"
slaveof 127.0.0.1 6379
```

启动两个从节点：

```
$ redis-server /etc/redis/redis-6380.conf
$ redis-server /etc/redis/redis-6381.conf
```

验证：
```
$ redis-cli -h 127.0.0.1 -p 6380 ping
PONG
$ redis-cli -h 127.0.0.1 -p 6381 ping
PONG
```

### 确认主从关系

接下来便是确认主从关系，首先站在主节点的角度，执行 `redis-cli -h 127.0.0.1 -p 6379 info replication`，返回的信息如下：

```
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=8971552,lag=1
slave1:ip=127.0.0.1,port=6381,state=online,offset=8971552,lag=0
master_replid:24598661a0fd73cae32cb59c017b1056c6433272
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:8971818
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:7923243
repl_backlog_histlen:1048576
```
意思是，该节点为主节点，他有两个从节点，分别是 127.0.0.1:6380 和 127.0.0.1:6381。

站在从节点的角度，执行：`redis-cli -h 127.0.0.1 -p 6380 info replication`，该节点为从节点，它的主节点是 127.0.0.1：6379。
```
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:9356524
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:24598661a0fd73cae32cb59c017b1056c6433272
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:9356524
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:8307949
repl_backlog_histlen:1048576
```
到此我们的一主两从的结构搭建好了，下面就是部署 Sentinel 节点。

## 部署 Sentinel 节点

3 个 Sentinel 节点的部署方法是完全一致的（端口不同），下面以 sentinel-1 节点的部署为例子进行说明。

- 将 `sentinel.conf` 复制一份到 `/etc/redis/`，并重命名为 `redis-sentinel-26379.conf`。
- 配置 Sentinel 节点：
  ```
  port 26379
  daemonize yes
  logfile "26379.log"
  dir /opt/soft/redis/data
  sentinel monitor mymaster 127.0.0.1 6379 2
  sentinel down-after-milliseconds mymaster 30000
  sentinel parallel-syncs mymaster 1
  sentinel failover-timeout mymaster 180000
  ```
  解读下这些配置项：
    - `port 26379`：Sentinel 节点的默认端口是 26379。
    - `sentinel monitor mymaster 127.0.0.1 6379 2`：代表 sentinel-1 节点监控 127.0.0.1 63792 这个主节点，该主节点的别名为 mymaster，2 代表判断主节点失败至少需要 2 个 Sentinel 节点同意。
    - `sentinel down-after-milliseconds mymaster 30000`：每个 Sentinel 节点都要通过定期发送 ping 命令来判断 Redis 数据节点和其余 Sentinel 节点是否可达，如果超过了 down-after-milliseconds 配置的时间且没有有效的回复，则判定节点不可达，30000（单位为毫秒）就是超时时间，这个配置是对节点失败判定的重要依据。down-after-milliseconds 越大，代表 Sentinel 节点对于节点不可达的条件越宽松，反之越严格。
    - `sentinel parallel-syncs mymaster 1`：用来限制在一次故障转移之后，每次向新的主节点发起复制操作的从节点个数。如果这个参数配置的比较大，那么多个从节点会向新的主节点同时发起复制操作，尽管复制操作通常不会阻塞主节点，但是同时向主节点发起复制，必然会对主节点所在的机器造成一定的网络和磁盘 IO 开销。例如：parallel- syncs = 3 从节点会同时发起复制（并行），parallel-syncs = 1 时从节点会轮询发起复制（顺序），这个是有区别的。
    - `sentinel failover-timeout mymaster 180000`：故障转移超时时间，从节点复制新的主节点超过了 failover-timeout（不包含复制时间）， 则故障转移失败。
    - `sentinel auth-pass`：如果 Sentinel 监控的主节点配置了密码，sentinel auth-pass 配置通过添加主节点的密码，防止 Sentinel 节点对主节点无法监控。
- 启动 Sentinel 节点，一般有两种方式，两种本质上都一样，依次启动3个 Sentinel 节点：
  - 使用 redis-sentinel 命令：`redis-sentinel /etc/redis/redis-sentinel-26379.conf`
  - 使用 redis-server 命令加 --sentinel 参数：`redis-server /etc/redis/redis-sentinel-26379.conf --sentinel`
- Sentinel 节点本质上是一个特殊的 Redis 节点，所以也可以通过 info 命令来查询它的相关信息，从下面 info 的 Sentinel 片段来看，Sentinel节点找到了主节点 127.0.0.1：6379，发现了它的两个从节点，同时发现 Redis Sentinel一共 有3个 Sentinel 节点。
  ```
  $ redis-cli -h 127.0.0.1 -p 26379 info Sentinel
  # Sentinel
  sentinel_masters:1
  sentinel_tilt:0
  sentinel_running_scripts:0
  sentinel_scripts_queue_length:0
  sentinel_simulate_failure_flags:0
  master0:name=mymaster,status=ok,address=127.0.0.1:6379,slaves=2,sentinels=3
  ```
- 这个时候在打开 `redis-sentinel-26379.conf` 配置文件，发现 Sentinel 节点会在启动后，会给自己分配一个 sentinel myid 值，同时会将他所感知到的一些节点信息保存到他指定的配置文件中去。
  ```
  sentinel myid 44e9a5ae2c0e4d1fc7e05aed409c259666fc3243
  # Generated by CONFIG REWRITE
  sentinel known-slave mymaster 127.0.0.1 6381
  sentinel known-slave mymaster 127.0.0.1 6380
  sentinel known-sentinel mymaster 127.0.0.1 26381 4d0f2e2d8434d05a785be7ea690e53ef02e76f80
  sentinel known-sentinel mymaster 127.0.0.1 26380 9479fc9afe3a7a21da443f858866d58b10d069ee
  sentinel current-epoch 0
  ```
  那我在执行到这一步的时候，遇到了一个问题，就是我初次启动 sentinel-1、sentinel-2、sentinel-3 这三个 Sentinel 节点，发现他们生成的 sentinel myid 值居然是一样的，然后通过 `redis-cli -h 127.0.0.1 -p 26379 info Sentinel` 命令查询相关信息的时候，sentinels=1，那肯定是不对的，明明有三个 Sentinel 节点，怎么只有一个呢？难道是因为他们 sentinel myid 的值一样造成的？然后我将 sentinel-2、sentinel-3 两个节点的 sentinel myid 删掉，再重新启动，发现他们两个节点重新生成的 myid 终于不一样了， sentinels=3 显示正常了。关于这个问题，实在有点摸不着头脑。

这样一个简单的 Redis Sentinel 已经搭建起来了，整体上还是比较容易的，最终的整体的拓扑图如下：
<img src="http://image.leeyom.top/20180425152463904923751.png" title="最终拓扑图" />

# 部署技巧

- Redis Sentinel 可以同时监控多个主节点，配置方法也比较简单，只需要指定多个 masterName 来区分不同的主节点即可，例如下面的配置监控 master-business-1（10.211.55.4:6379）和 master-business-2（10.211.55.5:6379）两个主节点：
  ```
  sentinel monitor master-business-1 10.211.55.4 6379 2
  sentinel down-after-milliseconds master-business-1 60000
  sentinel failover-timeout master-business-1 180000
  sentinel parallel-syncs master-business-1 1

  sentinel monitor master-business-2 10.211.55.5 6380 2
  sentinel down-after-milliseconds master-business-2 10000
  sentinel failover-timeout master-business-2 180000
  sentinel parallel-syncs master-business-2 1
  ```
- Sentinel 节点不应该部署在一台物理“机器”上，若干虚拟机虽然有不同的 IP 地址，但实际上它们都是同一台物理机，同一台物理机意味着如果这台机器有什么硬件故障，所有的虚拟机都会受到影响，为了实现 Sentinel 节点集合真正的高可用，请勿将 Sentinel 节点部署在同一台物理机器上。
- 部署至少三个且奇数个的 Sentinel 节点。
- 如果 Sentinel 节点集合监控的是同一个业务的多个主节点集合，那么只需要一套 Sentinel，让他监控多个主节点，否则的话，就为每个主节点配置一套 Sentinel。

# 从节点高可用

从节点一般可以起到两个作用：
- 第一，当主节点出现故障时，作为主节点的后备“顶”上来实现故障转移，Redis Sentinel 已经实现了该功能的自动化，实现了真正的高可用。
- 第二，扩展主节点的读能力，尤其是在读多写少的场景非常适用。

模型图如下：
<img src="http://image.leeyom.top/20180425152464762421022.png" title="一般读写分离模型" />
但是上述从节点不是高可用的，如果 slave-1 节点出现故障，首先客户端 client-1 将与其失联，其次 Sentinel 节点只会对该节点做主观下线，因为 Redis Sentinel 的故障转移是针对主节点的。所以很多时候，Redis Sentinel 中的从节点仅仅是作为主节点一个热备，不让它参与客户端的读操作，就是为了保证整体高可用性，但实际上这种使用方法还是有一些浪费，尤其是在有很多从节点或者确实需要读写分离的场景，所以如何实现从节点的高可用是非常有必要的。

所以在设计 Redis Sentinel 的从节点高可用时，只要能够实时掌握所有从节点的状态，把所有从节点看做一个资源池（如图下图所示），无论是上线还是下线从节点，客户端都能及时感知到（将其从资源池中添加或者删除），这样从节点的高可用目标就达到了。
<img src="http://image.leeyom.top/20180425152464786244796.png" title="高可用的读写分离" />

# API

Sentinel 节点是一个特殊的 Redis 节点，它有自己专属的 API，常用的 API 如下：

- `sentinel masters`：展示所有被监控的主节点状态以及相关的统计信息。
- `sentinel master <master name>`：展示指定 master name 的主节点状态以及相关的统计信息。
- `sentinel slaves <master name>`：展示指定 master name 的从节点状态以及相关的统计信息。
- `sentinel sentinels <master name>`：展示指定 master name 的 Sentinel 节点集合（不包含当前 Sentinel 节点）。
- `sentinel get-master-addr-by-name <master name>`：返回指定 master name 主节点的 IP 地址和端口。
- `sentinel reset <pattern>`：当前 Sentinel 节点对符合 pattern（通配符风格）主节点的配置进行重置，包含清除主节点的相关状态（例如故障转移），重新发现从节点和 Sentinel 节点。
- `sentinel failover <master name>`：对指定 master name 主节点进行强制故障转移（没有和其他 Sentinel 节点“协商”），当故障转移完成后，其他 Sentinel 节点按照故障转移的结果更新自身配置，这个命令在 Redis Sentinel 的日常运维中非常有用。
- `sentinel ckquorum <master name>`：检测当前可达的 Sentinel 节点总数是否达到 quorum 的个数，如果 quorum = 2，将无法实现故障转移。
- `sentinel flushconfig`：将 Sentinel 节点的配置强制刷到磁盘上。
- `sentinel remove <master name>`：取消当前 Sentinel 节点对于指定 master name 主节点的监控。
- `sentinel monitor <master name> <ip> <port> <quorum>`：监控指定的主节点，跟配置文件中的 sentinel monitor mymaster 127.0.0.1 6379 2 一个道理。
- `sentinel is-master-down-by-addr`：Sentinel 节点之间用来交换对主节点是否下线的判断，根据参数的不同，还可以作为 Sentinel 领导者选举的通信方式。
- `sentinel set <master name>`：动态修改 Sentinel 节点配置选项，跟配置文件中那些配置是一个道理。

# 其他

- [跟示例相关的配置文件](https://github.com/wangleeyom/code-grocery-shop/tree/master/%E6%90%AD%E5%BB%BA%20Redis%20Sentinel%20%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
