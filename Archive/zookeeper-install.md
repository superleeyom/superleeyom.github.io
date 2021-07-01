---
title: zookeeper安装教程
date: 2017-11-08 21:26:06
comments: true
tags:
- zookeeper
categories:
- 资源
---

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。本篇记录了在Ubuntu系统上安装ZooKeeper的教程。


<!-- more -->

# 系统需求

## 支持平台

| 系统 | 开发环境 | 生产环境 |
| :-: | :-: | :-: |
| Linux | 支持 | 支持 |
| Solaris | 支持 | 支持 |
| FreeBSD | 支持 | 支持 |
| Windows | 支持 | 不支持 |
| MacOS | 支持 | 不支持 |


## 运行环境

ZooKeeper运行在java平台，需要**JRE 1.6或者以上**的版本。对于集群模式下的ZooKeeper部署，3个ZooKeeper服务进程是建议的最小进程数量。

# 下载

- 下载地址：[http://apache.forsale.plus/zookeeper/](http://apache.forsale.plus/zookeeper/)。
- 我下载的版本是`zookeeper-3.4.10`，这个版本比较稳定。

# 安装方式

ZooKeeper有三种安装方式，分别是单机模式、集群模式、伪集群模式。

## 单机模式

将下载的`zookeeper-3.4.10.tar.gz`压缩包解压到`/home/leeyom/develop-tools/`，执行命令：

- `sudo tar -zxvf zookeeper-3.4.10.tar.gz -C /home/leeyom/develop-tools/`

在`/etc/profile`文件中加入ZooKeeper的环境变量设置，具体的内容如下：

```
# ZooKeeper
export ZOOKEEPER_HOME=/home/leeyom/develop-tools/zookeeper-3.4.10
export PATH=$PATH:$ZOOKEEPER_HOME/bin:$ZOOKEEPER_HOME/conf
```

ZooKeeper服务器包含在单个的jar文件中，安装此服务需要用户创建一个配置文档，对其进行设置。进入ZooKeeper配置文件目录`/home/leeyom/develop-tools/zookeeper-3.4.10/conf`，该目录下面有一个参考的配置文件`zoo_sample.cfg`，可供参考。在conf目录下创建我们自己的配置文件`zoo.cfg`，内容如下：

```
tickTime=2000
dataDir=/home/leeyom/develop-tools/zookeeper-3.4.10/zookeeper-data
dataLogDir=/home/leeyom/develop-tools/zookeeper-3.4.10/zookeeper-logs
clientPort=2181
```

目录`/home/leeyom/develop-tools/zookeeper-3.4.10/zookeeper-data/`和`/home/leeyom/develop-tools/zookeeper-3.4.10/zookeeper-logs/`默认是没有创建的，需要我们自己**手动创建此目录**，下面是每个参数的含义：

- **tickTime**：基本的时间单元，以毫秒为单位，用他来指示心跳。
- **dataDir**：存储内存中数据库快照的位置，如果不设置参数，更新事务日志将被存储到默认位置。
- **clientPort**：监听客户端连接的端口。
- **dataLogDir** : 保存zookeeper日志路径，当此配置不存在时默认路径与dataDir一致。

单机模式下需要注意：**在这种配置方式下，是没有ZooKeeper副本，如果zookeeper服务器出现故障，zookeeper服务将会停止**！。

## 集群模式

为了获得可靠的zookeeper服务，我们应该在一个集群上部署zookeeper。只要集群上的大多数zookeeper服务启动了，那么总的zookeeper服务便是可用的。另外，最好使用奇数台服务器。如果zookeeper拥有5台服务器，那么在最多2台服务器出现故障后，整个服务还可以正常使用。

具体的安装其实跟单机模式下基本上差不多，不同之处在于每台机器上的`conf/zoo.cfg`配置文件的参数设置不同，可以参考其中一台的机器的配置：

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=10
syncLimit=5
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
maxClientCnxns=60
```

在这个配置文件中，出现几个新的参数，其含义如下：

- **initLimit**：此配置表示允许follower连接并同步到leader的初始化时间，它以tickTime的倍数来表示。当超过设置倍数的tickTime时间，则连接失败。
- **syncLimit**：Leader服务器与follower服务器之间信息同步允许的最大时间间隔，如果超过次间隔，默认follower服务器与leader服务器之间断开链接。
- **maxClientCnxns**：限制连接到zookeeper服务器客户端的数量。
- **server.id=host:port:port**：表示了不同的zookeeper服务器的自身标识，作为集群的一部分，每一台服务器应该知道其他服务器的信息。用户可以从`server.id=host:port:port`中读取到相关信息。在服务器的data(dataDir参数所指定的目录)下创建一个文件名为myid的文件，这个文件的内容只有一行，指定的是自身的id值。比如，服务器“1”应该在myid文件中写入“1”。这个id必须在集群环境中服务器标识中是唯一的，且大小在1～255之间。这一样配置中，zoo1代表第一台服务器的IP地址。第一个端口号（port）是从follower连接到leader机器的端口，第二个端口是用来进行leader选举时所用的端口。所以，在集群配置过程中有三个非常重要的端口：**clientPort：2181、port:2888、port:3888**。

## 伪集群模式

伪集群模式就是在单机环境下模拟集群的Zookeeper服务。

在zookeeper集群配置文件中，clientPort参数用来设置客户端连接zookeeper服务器的端口。server.1=IP1:2888:3888中，IP1指的是Zookeeper服务器的IP地址，2888为组成zookeeper服务器之间的通信端口，3888为用来选举leader的端口。由于伪集群模式中，我们使用的是同一台服务器，也就是说，需要在单台机器上运行多个zookeeper实例，所以我们必须要保证多个zookeeper实例的配置文件的client端口不能冲突。

> server.A=B:C:D：其中A是一个数字，代表第几号服务器，B是服务器的ip地址，C表示服务器与群集中的“领导者”交换信息的端口；当领导者失效后，D表示用来执行选举时服务器相互通信的端口。所以说伪集群的模式下，每个zookeeper实例需要保证clientPort、C、D三个端口都要不同，否则zookeeper服务将启动报错。

首先先将`zookeeper-3.4.10.tar.gz`分别解压到server1，server2，server3目录下，执行如下的命令：

```
sudo tar -zxvf zookeeper-3.4.10.tar.gz /home/leeyom/develop-tools/zookeeper_cluster/server1
sudo tar -zxvf zookeeper-3.4.10.tar.gz /home/leeyom/develop-tools/zookeeper_cluster/server2
sudo tar -zxvf zookeeper-3.4.10.tar.gz /home/leeyom/develop-tools/zookeeper_cluster/server3
```

- 在`server1`、`server2`、`server3`目录下先创建`data`和`dataLog`目录，然后在`server1/data/`目录下创建文件myid文件，并写入“1”，同样在`server2/data/`，目录下创建文件myid，并写入“2”，server3进行同样的操作。
- 然后分别在`server1/conf/`、`server2/conf/`、`server3/conf/`目录下创建`zoo.cfg`配置文件，三个配置文件如下：

```
# Server 1
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/leeyom/develop-tools/zookeeper_cluster/server1/data
dataLogDir=/home/leeyom/develop-tools/zookeeper_cluster/server1/dataLog
clientPort=2181
server.1= 127.0.0.1:2888:3888
server.2= 127.0.0.1:2889:3889
server.3= 127.0.0.1:2890:3890
maxClientCnxns=60
```

```
# Server 2
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/leeyom/develop-tools/zookeeper_cluster/server2/data
dataLogDir=/home/leeyom/develop-tools/zookeeper_cluster/server2/dataLog
clientPort=2182
server.1= 127.0.0.1:2888:3888
server.2= 127.0.0.1:2889:3889
server.3= 127.0.0.1:2890:3890
maxClientCnxns=60
```

```
# Server 3
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/leeyom/develop-tools/zookeeper_cluster/server3/data
dataLogDir=/home/leeyom/develop-tools/zookeeper_cluster/server3/dataLog
clientPort=2183
server.1= 127.0.0.1:2888:3888
server.2= 127.0.0.1:2889:3889
server.3= 127.0.0.1:2890:3890
maxClientCnxns=60
```

以上便是伪集群模式下的配置。

# 启动服务

1. 单机模式：
    - 进入到zookeeper的安装目录下的`bin`目录下。
    - 执行命令：`./zkServer.sh start`
    - 若出现如下的内容，不要以为就启动成功了：
      ```
      ZooKeeper JMX enabled by default
      Using config: /home/leeyom/develop-tools/zookeeper-3.4.10/bin/../conf/zoo.cfg
      Starting zookeeper ... STARTED
      ```
      继续执行：`./zkServer.sh status`，查看启动状态，若出现如下内容，才说明zookeeper服务才是真的启动成功，否则都是启动失败：
      ```
      ZooKeeper JMX enabled by default
      Using config: /home/leeyom/develop-tools/zookeeper-3.4.10/bin/../conf zoo.cfg
      Mode: standalone
      ```
2. 集群模式下需要用户在每台 ZooKeeper 机器上运行单机模式下的命令，这里不再赘述。

3. 在集群伪分布模式下，按先后顺序依次启动`Server1`、`Server2`、`Server3`。这里拿`Server1`示例。
  - 进入到`server1/bin`目录下。
  - 执行`./zkServer.sh start`命令。
  - 出现`Starting zookeeper ...STARTED`提示，说明服务启动成功。

4. 其他命令，需要进入到`zookeeper-3.4.10/bin`目录下，然后执行如下命令：
  - 查看ZK服务状态: `./zkServer.sh status`
  - 停止ZK服务：`./zkServer.sh stop`
  - 重启ZK服务：`./zkServer.sh restart`
  - 客户端连接：`./zkCli.sh`
  - 客户端退出：`quit`

# 遇到的问题

因为我实验的环境是ubuntu桌面版的linux系统，我是用普通账户登录的，我之前是将zookeeper安装到`/usr/local/develop-tools`文件夹下，然后执行命令：`sudo bash zkServer.sh start`启动服务。虽然说终端打印了：

```
ZooKeeper JMX enabled by default
Using config: /usr/local/develop-tools/zookeeper-3.4.10/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```
我便以为服务启动了，其实是错误的，此时服务并没有启动，由于权限的问题，当我启动服务的时候，zookeeper在`/usr/local`目录下没有读写权限，所以导致服务启动失败。具体的解决办法是：

- 方式一：`cd /usr/local/`目录下，执行chmod命令增加权限，然后再次启动。
    - `chmod a+xwr zookeeper-3.4.10/`
- 方式二：将zookeeper安装到`/home/leeyom/`目录下面，这样zookeeper不会出现读写权限的问题。

所以我们在启动zookeeper服务的时候，最好要用`./zkServer.sh status`命令检查下zookeeper服务状态，确保zookeeper服务启动。


