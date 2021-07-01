---
title: Redis Cluster 集群搭建和部署
date: 2018-04-28 15:13:35
tags:
- redis
categories:
- 编程
---

Redis Cluster 是 Redis 的分布式解决方案，在 3.0 版本正式推出，有效地解决了 Redis 分布式方面的需求。当遇到单机内存、并发、流量等瓶颈时，可以采用 Cluster 架构方案达到负载均衡的目的。之前，Redis 分布式方案一般客户端分区和代理，但是这些方案都有部署复杂度、性能损耗、无法高可用等等缺陷。Redis Cluster 它非常优雅地解决了 Redis 集群方面的问题，因此理解应用好 Redis Cluster 将极大地解放我们使用分布式 Redis 的工作量，同时它也是学习分布式存储的绝佳案例。

<!--more-->

Redis 集群一般由多个节点组成，节点数量至少为 6 个才能保证组成完整高可用的集群。作为举例，本文将搭建一个 6（3主3从）个节点的集群，主备关系如下图所示，其中 M 代码 Master 节点，S 代表 Slave 节点，A-M 和 A-S 为一对主备节点，麻雀虽小，五脏俱全。先手动搭建，然后再尝试用 Redis 集群管理工具 redis-trib 搭建，这样就能更加理解 Redis Cluster 的[工作原理](http://www.leeyom.top/2018/04/28/dis-storage/)。

<img src="http://image.leeyom.top/20180427152481315320882.png" title="示意图" />

# 准备节点

建议为集群内所有节点统一目录，一般划分三个目录：conf、 data、log，分别存放配置、数据和日志相关文件。把 6 个节点配置统一放在 conf 目录下，在解压文件夹 redis-4.0.9 中有一个 Redis 配置文件 redis.conf，其中一些默认的配置项需要修改（配置项较多，本文仅为举例，修改一些必要的配置）。以下仅以 6379 端口为例进行配置，6380，6381 等端口配置操作类似。配置文件命名规则 `redis-{port}.conf`，将修改后的配置文件分别放入 conf 文件夹下面，核心要修改的配置项如下：

| 配置项名称 | 解释 |
| :-: | :-: |
| `port 6379` | 指定 Reidis 监听端口 |
| `bind 127.0.0.1` | 默认绑定主机地址，本文只有一台机器，故采用默认配置 |
| `logfile "" `| 日志名称，本文设置为：`logfile "/usr/local/redis-cluster/log/{port}.log"` |
| `dbfilename dump.rdb` | 本地数据库持久化文件名，本文设置为：`dbfilename dump-{port}.rdb` |
| `dir ./` | 本地数据库持久化文件存放路径，本文设置为：`dir /usr/local/redis-cluster/data/` |
| `pidfile /var/run/redis_6379.pid` | Redis 守护进程 pid，本文设置为：`pidfile /var/run/redis_{port}.pid` |
| `cluster-enabled yes` | 将 Redis 配置以集群模式启动 |
| `cluster-config-file node-6379.conf` | 集群模式下自动生成的集群配置文件，采用 `node-{port}.conf` 格式定义 |
| `cluster-node-timeout 15000` | 节点超时时间，单位毫秒 |

[准备](http://www.leeyom.top/2018/04/25/redis-sentinel/#%E5%87%86%E5%A4%87)好后，启动所有的节点，一个一个启动有点麻烦，创建一个启动脚本和停止脚本，批量拉起和停止 redis 服务，脚本内容如下：

- 启动脚本，start.sh

  ```
  redis-server /usr/local/redis-cluster/conf/redis-6379.conf &
  redis-server /usr/local/redis-cluster/conf/redis-6380.conf &
  redis-server /usr/local/redis-cluster/conf/redis-6381.conf &
  redis-server /usr/local/redis-cluster/conf/redis-6382.conf &
  redis-server /usr/local/redis-cluster/conf/redis-6383.conf &
  redis-server /usr/local/redis-cluster/conf/redis-6384.conf &
  ```

- 停止脚本，stop.sh

  ```
  ps -ef | grep redis-server | grep -v grep | awk '{print $2}' | xargs kill -9
  ```

执行启动脚本 start.sh ，查看 redis-server 进程如下：

```
[root@centos-redis-cluster redis-cluster]# ps -ef|grep redis-server
root     32181     1  0 17:50 ?        00:00:00 redis-server 127.0.0.1:6379 [cluster]
root     32182     1  0 17:50 ?        00:00:00 redis-server 127.0.0.1:6382 [cluster]
root     32183     1  0 17:50 ?        00:00:00 redis-server 127.0.0.1:6380 [cluster]
root     32184     1  0 17:50 ?        00:00:00 redis-server 127.0.0.1:6383 [cluster]
root     32185     1  0 17:50 ?        00:00:00 redis-server 127.0.0.1:6384 [cluster]
root     32186     1  0 17:50 ?        00:00:00 redis-server 127.0.0.1:6381 [cluster]
```

如节点 6379 首次启动后生成集群配置 nodes-6379.conf 如下：

```
541c76599b2e872b91babdabc9a6bd22f6e8b16d :0@0 myself,master - 0 0 0 connected
rs currentEpoch 0 lastVoteEpoch 0
```
文件内容记录了集群初始状态，这里最重要的是节点 ID，它是一个 40 位 16 进制字符串，用于唯一标识集群内一个节点，之后很多集群操作都要借助于节点 ID 来完成。需要注意是，节点 ID 不同于运行 ID 。节点 ID 在集群初始化时只创建一次，节点重启时会加载集群配置文件进行重用，而 Redis 的运行 ID 每次重启都会变化。

在节点 6380 执行 `cluster nodes` 命令获取集群节点状态：

```
[root@centos-redis-cluster data]# redis-cli -h 127.0.0.1 -p 6380
127.0.0.1:6380> cluster nodes
4bc590060dc0c987e84dbb7e155da37df74a4026 :6380@16380 myself,master - 0 0 0 connected
```
但是目前它只能识别自己，下面通过节点握手让 6 个节点彼此建立联系从而组成一个集群。

# 节点握手

节点握手是指一批运行在集群模式下的节点通过 Gossip 协议彼此通信，达到感知对方的过程。节点握手是集群彼此通信的第一步，由客户端发起命令：`cluster meet {ip} {port}`，比如这里 6379 和 6380 两个节点进行握手通信：

- 节点 6379 本地创建 6380 节点信息对象，并发送 meet 消息。
- 节点 6380 接受到 meet 消息后，保存 6379 节点信息并回复 pong 消息。
- 之后节点 6379 和 6380 彼此定期通过 ping/pong 消息进行正常的节点通信。

集群内的每个 Redis 实例监听两个 TCP 端口，6379（默认）用于服务客户端查询，16379（默认服务端口 + 10000）用于集群内部通信，所有的 Redis 节点彼此互联（PING-PONG 机制），节点间通信使用轻量的二进制协议，减少带宽占用。

下面分别执行 meet 命令让其他节点加入到集群中：

```
127.0.0.1:6379> cluster meet 127.0.0.1 6380
127.0.0.1:6379> cluster meet 127.0.0.1 6381
127.0.0.1:6379> cluster meet 127.0.0.1 6382
127.0.0.1:6379> cluster meet 127.0.0.1 6383
127.0.0.1:6379> cluster meet 127.0.0.1 6384
```

最后执行 `cluster nodes` 命令确认 6 个节点都彼此感知并组成集群：

```
127.0.0.1:6379> cluster nodes
fb07612cf5be9ffec1f7b16d2533995547dbf1a6 127.0.0.1:6382@16382 master - 0 1524799644954 0 connected
541c76599b2e872b91babdabc9a6bd22f6e8b16d 127.0.0.1:6379@16379 myself,master - 0 1524799642000 2 connected
17cb468a7805ed5f286f44c5ffdecef645beb2e2 127.0.0.1:6384@16384 master - 0 1524799643000 5 connected
224f02fb1a4a232fa0190a89569ccf924e60ce1f 127.0.0.1:6383@16383 master - 0 1524799641926 4 connected
90b5c3ae81411306931d1bd5a4363b3476ce7c59 127.0.0.1:6381@16381 master - 0 1524799643000 3 connected
4bc590060dc0c987e84dbb7e155da37df74a4026 127.0.0.1:6380@16380 master - 0 1524799643950 1 connected
```

节点握手之后，集群还不能进行工作，为啥？因为所有的节点还没有分配槽（slot），对吧？执行 `cluster info`：

```
127.0.0.1:6379> cluster info
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:0
cluster_current_epoch:5
cluster_my_epoch:2
cluster_stats_messages_ping_sent:196
cluster_stats_messages_pong_sent:218
cluster_stats_messages_meet_sent:5
cluster_stats_messages_sent:419
cluster_stats_messages_ping_received:218
cluster_stats_messages_pong_received:201
cluster_stats_messages_received:419
```

被分配的槽（cluster_slots_assigned）是 0，所以接下来就是给这些节点分配槽（slot）。

# 分配槽

Redis 集群使用的是 hash solt 算法，会把所有的数据映射到 16384 个槽中，每个 key 会映射为一个固定的槽，只有当节点分配了槽，才能响应和这些槽关联的键命令。通过：`cluster addslots {start..end}` 命令为节点分配槽。因为示例的这个集群是 3 主 3 从，那只需要将 16384 个 slot 平均分配给 6379、6380、6381 三个主节点，由于使用的是 cluster addslots 的命令，由于不支持批量添加操作，所以需要使用 shell 脚本进行添加，创建脚本 addslot.sh 并执行：

```
redis-cli -h 127.0.0.1 -p 6379 cluster addslots {0..5461}
redis-cli -h 127.0.0.1 -p 6380 cluster addslots {5462..10922}
redis-cli -h 127.0.0.1 -p 6381 cluster addslots {10923..16383}
```

执行cluster info 查看集群状态，如下所示：

```
127.0.0.1:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:5
cluster_my_epoch:2
cluster_stats_messages_ping_sent:3601
cluster_stats_messages_pong_sent:3692
cluster_stats_messages_meet_sent:5
cluster_stats_messages_sent:7298
cluster_stats_messages_ping_received:3692
cluster_stats_messages_pong_received:3606
cluster_stats_messages_received:7298
```

当前集群状态是 OK，hash slot 均已分配完毕，集群进入在线状态。剩下的三个节点（6382、6383、6384）充当从节点，负责主节点出现故障时，可以自动进行故障转移。**注意在集群模式下 slaveof 添加从节点操作不再支持**，应该使用 `cluster replicate {nodeId}` 命令让一个节点成为从节点。其中命令执行必须在对应的从节点上执行，nodeId 是要复制主节点的节点 ID，通过 `cluster nodes` 命令可以查看所以节点的 nodeId，前面的 40 位 16 进制字符串就是对应节点的 nodeId :

```
127.0.0.1:6379> cluster nodes
fb07612cf5be9ffec1f7b16d2533995547dbf1a6 127.0.0.1:6382@16382 master - 0 1524811513984 0 connected
541c76599b2e872b91babdabc9a6bd22f6e8b16d 127.0.0.1:6379@16379 myself,master - 0 1524811511000 2 connected 0-5461
17cb468a7805ed5f286f44c5ffdecef645beb2e2 127.0.0.1:6384@16384 master - 0 1524811514991 5 connected
224f02fb1a4a232fa0190a89569ccf924e60ce1f 127.0.0.1:6383@16383 master - 0 1524811512000 4 connected
90b5c3ae81411306931d1bd5a4363b3476ce7c59 127.0.0.1:6381@16381 master - 0 1524811513000 3 connected 10923-16383
4bc590060dc0c987e84dbb7e155da37df74a4026 127.0.0.1:6380@16380 master - 0 1524811512975 1 connected 5462-10922
```

开始进行复制：

```
127.0.0.1:6382> cluster replicate 541c76599b2e872b91babdabc9a6bd22f6e8b16d
127.0.0.1:6383> cluster replicate 4bc590060dc0c987e84dbb7e155da37df74a4026
127.0.0.1:6384> cluster replicate 90b5c3ae81411306931d1bd5a4363b3476ce7c59
```
通过` cluster nodes` 命令查看集群状态和复制关系，分别是 3个 master 和 3 个 slave：

```
127.0.0.1:6379> cluster nodes
fb07612cf5be9ffec1f7b16d2533995547dbf1a6 127.0.0.1:6382@16382 slave 541c76599b2e872b91babdabc9a6bd22f6e8b16d 0 1524811962000 2 connected
541c76599b2e872b91babdabc9a6bd22f6e8b16d 127.0.0.1:6379@16379 myself,master - 0 1524811962000 2 connected 0-5461
17cb468a7805ed5f286f44c5ffdecef645beb2e2 127.0.0.1:6384@16384 slave 90b5c3ae81411306931d1bd5a4363b3476ce7c59 0 1524811964596 5 connected
224f02fb1a4a232fa0190a89569ccf924e60ce1f 127.0.0.1:6383@16383 slave 4bc590060dc0c987e84dbb7e155da37df74a4026 0 1524811962581 4 connected
90b5c3ae81411306931d1bd5a4363b3476ce7c59 127.0.0.1:6381@16381 master - 0 1524811963588 3 connected 10923-16383
4bc590060dc0c987e84dbb7e155da37df74a4026 127.0.0.1:6380@16380 master - 0 1524811963000 1 connected 5462-10922
```

目前为止，依照 Redis 协议手动建立一个集群。它由 6 个节点构成， 3 个主节点负责处理槽和相关数据，3 个从节点负责故障转移。手动搭建集群便于理解集群建立的流程和细节，但是要是节点数比较多的话，手动这样搭建，岂不是要累死，因此 Redis 官方提供了 `redis-trib.rb` 工具方便我们快速搭建集群，下面就来体验下这个工具的好处吧。

# redis-trib 搭建集群

## Ruby 环境准备

- 安装 Ruby ，[下载](http://www.ruby-lang.org/en/downloads/)并解压缩，然后执行以下操作：

  ```
  $ ./configure
  $ make
  $ sudo make install
  ```
  检测是否安装成功，输入 `ruby -v`，打印版本号，说明安装成功。

- 安装 rubygem redis 依赖：

  ```
  $ wget http://rubygems.org/downloads/redis-4.0.1.gem
  $ gem install -l redis-4.0.1.gem
  $ gem list --check redis gem
  ```
  中间在执行 `gem install -l redis-4.0.1.gem` 这一步报错报错：

  ```
  ERROR:  Loading command: install (LoadError)
        cannot load such file -- zlib
  ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke_with_build_args' for nil:NilClass
  ```
  意思是缺少 zlib 依赖，需要安装 zlib 库，那就接下来安装 zlib：

  ```
  yum install zlib-devel
  ```
  集成 zlib 库到 ruby 环境：

  ```
  $ cd /root/ruby-2.5.1/ext/zlib
  $ ruby extconf.rb
  # 在操作下一步之前需要修改 Makefile 文件中的 zlib.o: $(top_srcdir)/include/ruby.h，将 $(top_srcdir) 修改为 ../.. 如下
  # zlib.o: ../../include/ruby.h
  # 这一步如果不修改，make 时会爆出另外一个错误
  # make:*** No rule to make target `/include/ruby.h', needed by `zlib.o'.  Stop
  $ make && make install  
  ```

  成功之后，再次运行 `gem install -l redis-4.0.1.gem`，出现以下界面：

  ```
  [root@centos-redis-cluster ~]# gem install -l redis-4.0.1.gem
  Successfully installed redis-4.0.1
  Parsing documentation for redis-4.0.1
  Installing ri documentation for redis-4.0.1
  Done installing documentation for redis after 0 seconds
  1 gem installed  
  ```
  成功安装 rubygem redis 依赖。

- 安装 redis-trib.rb：

  ```
  sudo cp /{redis_home}/src/redis-trib.rb /usr/local/bin
  ```
  确认 redis-trib.rb 是否安装成功：

  ```
  [root@centos-redis-cluster src]# redis-trib.rb
  Usage: redis-trib <command> <options> <arguments ...>

    create          host1:port1 ... hostN:portN
                    --replicas <arg>
    check           host:port
    info            host:port
    fix             host:port
                    --timeout <arg>
    reshard         host:port
                    --from <arg>
                    --to <arg>
                    --slots <arg>
                    --yes
                    --timeout <arg>
                    --pipeline <arg>
    rebalance       host:port
                    --weight <arg>
                    --auto-weights
                    --use-empty-masters
                    --timeout <arg>
                    --simulate
                    --pipeline <arg>
                    --threshold <arg>
    add-node        new_host:new_port existing_host:existing_port
                    --slave
                    --master-id <arg>
    del-node        host:port node_id
    // 省略
    ...
  ```
  从 redis-trib.rb 的提示信息可以看出，它提供了集群创建、检查、修复、均衡等命令行工具。

## 准备节点

  跟之前内容一样准备好节点配置并用 start.sh 脚本批量启动：

  ```
  redis-server /usr/local/redis-trib/conf/redis-6481.conf &
  redis-server /usr/local/redis-trib/conf/redis-6482.conf &
  redis-server /usr/local/redis-trib/conf/redis-6483.conf &
  redis-server /usr/local/redis-trib/conf/redis-6484.conf &
  redis-server /usr/local/redis-trib/conf/redis-6485.conf &
  redis-server /usr/local/redis-trib/conf/redis-6486.conf &  
  ```

## 创建集群

  启动好 6 个节点之后，使用 `redis-trib.rb create` 命令完成节点握手和槽分配过程，命令如下：

  ```
  redis-trib.rb create --replicas 1 127.0.0.1:6481 127.0.0.1:6482 127.0.0.1:6483 127.0.0.1:6484 127.0.0.1:6485 127.0.0.1:6486
  ```

  replicas 参数指定集群中每个主节点配备几个从节点，这里设置为 1。
  中间会让你输入 yes 同意主从节点角色分配的计划，redis-trib.rb 开始执行节点握手和槽分配操作，输出如下：

  ```
  >>> Creating cluster
  >>> Performing hash slots allocation on 6 nodes...
  Using 3 masters:
  127.0.0.1:6481
  127.0.0.1:6482
  127.0.0.1:6483
  Adding replica 127.0.0.1:6485 to 127.0.0.1:6481
  Adding replica 127.0.0.1:6486 to 127.0.0.1:6482
  Adding replica 127.0.0.1:6484 to 127.0.0.1:6483
  >>> Trying to optimize slaves allocation for anti-affinity
  [WARNING] Some slaves are in the same host as their master
  M: 01286db488f5d17d349b902f8cc0a847fcd9ccc1 127.0.0.1:6481
     slots:0-5460 (5461 slots) master
  M: f995f54df2a20510625a5f0643441d284a17d168 127.0.0.1:6482
     slots:5461-10922 (5462 slots) master
  M: e8c2e4bd5388cb2559577f552b2b5aff56c471c0 127.0.0.1:6483
     slots:10923-16383 (5461 slots) master
  S: 8332a7745d8dd5dc9dd294f5aae07166f25a8314 127.0.0.1:6484
     replicates e8c2e4bd5388cb2559577f552b2b5aff56c471c0
  S: ebcb250ba7b69ad5d9346367b63cb4a706e5b217 127.0.0.1:6485
     replicates 01286db488f5d17d349b902f8cc0a847fcd9ccc1
  S: 55df5e160bcea284d27f0d6c193535567858765d 127.0.0.1:6486
     replicates f995f54df2a20510625a5f0643441d284a17d168
  Can I set the above configuration? (type 'yes' to accept): yes
  >>> Nodes configuration updated
  >>> Assign a different config epoch to each node
  >>> Sending CLUSTER MEET messages to join the cluster
  Waiting for the cluster to join.......
  >>> Performing Cluster Check (using node 127.0.0.1:6481)
  M: 01286db488f5d17d349b902f8cc0a847fcd9ccc1 127.0.0.1:6481
     slots:0-5460 (5461 slots) master
     1 additional replica(s)
  M: e8c2e4bd5388cb2559577f552b2b5aff56c471c0 127.0.0.1:6483
     slots:10923-16383 (5461 slots) master
     1 additional replica(s)
  S: ebcb250ba7b69ad5d9346367b63cb4a706e5b217 127.0.0.1:6485
     slots: (0 slots) slave
     replicates 01286db488f5d17d349b902f8cc0a847fcd9ccc1
  S: 8332a7745d8dd5dc9dd294f5aae07166f25a8314 127.0.0.1:6484
     slots: (0 slots) slave
     replicates e8c2e4bd5388cb2559577f552b2b5aff56c471c0
  M: f995f54df2a20510625a5f0643441d284a17d168 127.0.0.1:6482
     slots:5461-10922 (5462 slots) master
     1 additional replica(s)
  S: 55df5e160bcea284d27f0d6c193535567858765d 127.0.0.1:6486
     slots: (0 slots) slave
     replicates f995f54df2a20510625a5f0643441d284a17d168
  [OK] All nodes agree about slots configuration.
  >>> Check for open slots...
  >>> Check slots coverage...
  [OK] All 16384 slots covered.  
  ```

  16384 个槽全部被分配，集群创建成功。这里需要 **注意给 redis-trib.rb 的节点地址必须是不包含任何槽/数据的节点**，否则会拒绝创建集群。

## 检查集群的完整性

接下来就要检查下是否所有的槽都分配到存活的主节点上，只要 16384 个槽中有一个没有分配给节点则表示集群不完整。可以使用 `redis-trib.rb check` 命令检测。check 命令只需要给出集群中任意一个节 点地址就可以完成整个集群的检查工作，命令如下：

```
$ redis-trib.rb check 127.0.0.1:6481
```
最后输出提示信息，提示所有的槽都分配到主节点上去：

```
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered
```
总得来说，使用 redis-trib 极大的们简化集群创建、检查、槽迁移和均衡等常见运维操作。

# 资料

- [gem install redis报错解决办法](https://blog.csdn.net/feinifi/article/details/78251486)
- [基于 Redis 的分布式缓存实现方案及可靠性加固策略](http://gitbook.cn/gitchat/activity/5a9b8abbf055ac6f65966638)
- [示例涉及的配置文件](https://github.com/wangleeyom/code-grocery-shop/tree/master/%E6%90%AD%E5%BB%BA%20Redis%20%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
