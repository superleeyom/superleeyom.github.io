---
title: RocketMQ实战与原理解析笔记
date: 2020-06-08 22:28:23
tags:
- 读书笔记
categories:
- 笔记
---

简单来说，消息队列就是基础数据结构课程里“先进先出”的一种数据结构，但是如果要消除单点故障，保证消息传输的可靠性，并且还能应对大流量的冲击，对消息队列的要求就很高了。现在互联网“微架构”模式兴起，原有大型集中式的IT服务因为各种弊端，通常被分拆成细粒度的多个“微服务”，这些微服务可以在一个局域网内，也可能跨机房部署。一方面对服务之间松耦合的要求越来越高，另一方面，服务之间的联系却越来越紧密，对通信质量的要求也越来越高。分布式消息队列可以提供应用解耦、流量消峰、消息分发等功能，已经成为大型互联网服务架构里标配的中间件。

<!--more-->

- **2.1 RocketMQ各部分角色介绍：**
	- 现实生活中的邮政系统要正常运行，离不开下面这四个角色，一是发信者，二是收信者，三是负责暂存、传输的邮局，四是负责协调各个地方邮局的管理机构。对应到RocketMQ中，这四个角色就是Producer、Consumer、Broker和NameServer。
- **2.4 常用管理命令：**
	- 订阅组在提高系统的高可用性和吞吐量方面扮演着重要的角色，比如用Clustering模式消费一个Topic里的消息内容时，可以启动多个消费者并行消费，每个消费者只消费Topic里消息的一部分，以此提高消费速度，这个时候就是通过订阅组来指明哪些消费者是同一组，同一组的消费者共同消费同一个Topic里的内容。
- **3.1 不同类型的消费者：**
	- DefaultMQPushConsumer需要设置三个参数：一是这个Consumer的GroupName，二是NameServer的地址和端口号，三是Topic的名称，下面将分别进行详细介绍
	- RocketMQ支持两种消息模式：Clustering和Broadcasting。❑ 在Clustering模式下，同一个ConsumerGroup（GroupName相同）里的每个Consumer只消费所订阅消息的一部分内容，同一个ConsumerGroup里所有的Consumer消费的内容合起来才是所订阅Topic内容的整体，从而达到负载均衡的目的。❑ 在Broadcasting模式下，同一个ConsumerGroup里的每个Consumer都能消费到所订阅Topic的全部消息，也就是一个消息会被多次分发，被多个Consumer消费。
	- Topic名称用来标识消息类型，需要提前创建。如果不需要消费某个Topic下的所有消息，可以通过指定消息的Tag进行消息过滤，比如：Consumer.subscribe（"TopicTest", "tag1 || tag2 || tag3"），表示这个Consumer要消费“TopicTest”下带有tag1或tag2或tag3的消息（Tag是在发送消息时设置的标签）。在填写Tag参数的位置，用null或者“*”表示要消费这个Topic的所有消息。
	- Push方式是Server端接收到消息后，主动把消息推送给Client端，实时性高。对于一个提供队列服务的Server来说，用Push方式主动推送有很多弊端：首先是加大Server端的工作量，进而影响Server的性能；其次，Client的处理能力各不相同，Client的状态不受Server控制，如果Client不能及时处理Server推送过来的消息，会造成各种潜在问题。
	- Pull方式是Client端循环地从Server端拉取消息，主动权在Client手里，自己拉取到一定量消息后，处理妥当了再接着取。Pull方式的问题是循环拉取消息的间隔不好设定，间隔太短就处在一个“忙等”的状态，浪费资源；每个Pull的时间间隔太长，Server端有消息到来时，有可能没有被及时处理。
	- “长轮询”方式通过Client端和Server端的配合，达到既拥有Pull的优点，又能达到保证实时性的目的。
	- “长轮询”的核心是，Broker端HOLD住客户端过来的请求一小段时间，在这个时间内有新消息到达，就利用现有的连接立刻返回消息给Consumer。“长轮询”的主动权还是掌握在Consumer手中，Broker即使有大量消息积压，也不会主动推送给Consumer。
	- Consumer分为Push和Pull两种方式，对于PullConsumer来说，使用者主动权很高，可以根据实际需要暂停、停止、启动消费过程。需要注意的是Offset的保存，要在程序的异常处理部分增加把Offset写入磁盘方面的处理，记准了每个Message Queue的Offset，才能保证消息消费的准确性
	- PushConsumer在启动的时候，会做各种配置检查，然后连接NameServer获取Topic信息，启动时如果遇到异常，比如无法连接NameServer，程序仍然可以正常启动不报错（日志里有WARN信息）。在单机环境下可以测试这种情况，启动DefaultMQPushConsumer时故意把NameServer地址填错，程序仍然可以正常启动，但是不会收到消息。
	- 如果需要在DefaultMQPushConsumer启动的时候，及时暴露配置问题，该如何操作呢？可以在Consumer.start()语句后调用：Consumer.fetchSubscribeMessageQueues("TopicName")，这时如果配置信息写得不准确，或者当前服务不可用，这个语句会报MQClientException异常。
- **3.2 不同类型的生产者：**
	- 延迟消息的使用方法是在创建Message对象时，调用setDelayTimeLevel（int level）方法设置延迟时间，然后再把这个消息发送出去。目前延迟的时间不支持任意设置，仅支持预设值的时间长度（1s/5s/10s/30s/1m/2m/3m/4m/5m/6m/7m/8m/9m/10m/20m/30m/1h/2h）。比如setDelayTimeLevel(3)表示延迟10s。
- **3.3 如何存储队列位置信息：**
	- Offset是指某个Topic下的一条消息在某个Message Queue里的位置，通过Offset的值可以定位到这条消息，或者指示Consumer从这条消息开始向后继续处理。
	- 默认是CLUSTERING模式，也就是同一个Consumer group里的多个消费者每人消费一部分，各自收到的消息内容不一样。这种情况下，由Broker端存储和控制Offset的值，使用RemoteBrokerOffsetStore结构。
	- 在DefaultMQPushConsumer里的BROADCASTING模式下，每个Consumer都收到这个Topic的全部消息，各个Consumer间相互没有干扰，RocketMQ使用LocalFileOffsetStore，把Offset存到本地。
- **4.1 NameServer的功能：**
	- NameServer是整个消息队列中的状态服务器，集群的各个组件通过它来了解全局的信息。同时，各个角色的机器都要定期向NameServer上报自己的状态，超时不上报的话，NameServer会认为某个机器出故障不可用了，其他的组件会把这个机器从可用列表里移除。
- **5.2 消息存储结构：**
	- RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog, ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。
- **5.5 同步复制和异步复制：**
	- 通常情况下，应该把Master和Save配置成ASYNC_FLUSH的刷盘方式，主从之间配置成SYNC_MASTER的复制方式，这样即使有一台机器出故障，仍然能保证数据不丢，是个不错的选择。
- **6.1 顺序消息：**
	- 要保证全局顺序消息，需要先把Topic的读写队列数设置为一，然后Producer和Consumer的并发设置也要是一。简单来说，为了保证整个Topic的全局消息有序，只能消除所有的并发处理，各部分都设置成单线程处理。这时高并发、高吞吐量的功能完全用不上了。
	- 要保证部分消息有序，需要发送端和消费端配合处理。在发送端，要做到把同一业务ID的消息发送到同一个Message Queue；在消费过程中，要做到从同一个MessageQueue读取的消息不被并发处理，这样才能达到部分有序。
- **6.2 消息重复问题：**
	- 解决消息重复有两种方法：第一种方法是保证消费逻辑的幂等性（多次调用和一次调用效果相同）；另一种方法是维护一个已消费消息的记录，消费前查询这个消息是否被消费过。这两种方法都需要使用者自己实现。
- **6.3 动态增减机器：**
	- NameServer是RocketMQ集群的协调者，集群的各个组件是通过NameServer获取各种属性和地址信息的。主要功能包括两部分：一个各个Broker定期上报自己的状态信息到NameServer；另一个是各个客户端，包括Producer、Consumer，以及命令行工具，通过NameServer获取最新的状态信息。
- **6.4 各种故障对消息的影响：**
	- 1）多Master，每个Master带有Slave；2）主从之间设置成SYNC_MASTER；3）Producer用同步方式写；4）刷盘策略设置成SYNC_FLUSH。就可以消除单点依赖，即使某台机器出现极端故障也不会丢消息。
- **7.1 在Broker端进行消息过滤：**
	- Tag标签是一个普通字符串，是在创建Message的时候添加的，一个Message只能有一个Tag。使用Tag方式过滤非常高效，Broker端可以在ConsumeQueue中做这种过滤，只从CommitLog里读取过滤后被命中的消息。
	- SQL表达式方式的过滤需要Broker先读出消息里的属性内容，然后做SQL计算，增大磁盘压力，没有Tag方式高效。
- **7.2 提高Consumer处理能力：**
	- 注意总的Consumer数量不要超过Topic下Read Queue数量，超过的Consumer实例接收不到消息。
- **7.3 Consumer的负载均衡：**
	- 在RocketMQ中，负载均衡或者消息分配是在Consumer端代码中完成的，Consumer从Broker处获得全局信息，然后自己做负载均衡，只处理分给自己的那部分消息。
- **7.4 提高Producer的发送速度：**
	- 比如日志收集类应用，可以采用Oneway方式发送，Oneway方式只发送请求不等待应答，即将数据写入客户端的Socket缓冲区就返回，不等待对方返回结果，用这种方式发送消息的耗时可以缩短到微秒级。
	- 在Linux操作系统层级进行调优，推荐使用EXT4文件系统，IO调度算法使用deadline算法。
- **11.1 整体流程：**
	- 根据消费消息方式的不同，OffsetStore的类型也不同。如果是BROADCASTING模式，使用的是LocalFileOffsetStore, Offset存到本地；如果是CLUSTERING模式，使用的是RemoteBrokerOffsetStore, Offset存到Broker机器上。