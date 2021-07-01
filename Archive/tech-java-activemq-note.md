---
title: Java消息中间件ActiveMQ学习笔记
date: 2017-07-23 13:39:20
comments: true
tags:
- ActiveMQ
categories:
- 编程
---

消息队列有两个基本的的用法，一个是作为服务内部的缓冲区，防流量高峰，也能达到异步处理的作用，一个是用于分布式系统，各个不同的进程(同一台机器或者不同机器)通过消息队列进行通信，最近闲着没事，就关于消息中间件ActiveMQ做了个简单的学习。

<!-- more -->

## 什么是ActiveMQ？

* 术语定义
 * 维基百科：https://zh.wikipedia.org/wiki/Apache_ActiveMQ
  > Apache ActiveMQ是Apache软件基金会所研发的开放源代码消息中间件；由于ActiveMQ是一个纯Java程序，因此只需要操作系统支持Java虚拟机，ActiveMQ便可运行。
 * 百度百科：https://baike.baidu.com/item/ActiveMQ/7889688?fr=aladdin
  > ActiveMQ 是Apache出品，最流行的，能力强劲的开源消息总线。ActiveMQ 是一个完全支持JMS1.1和J2EE 1.4规范的 JMS Provider实现，尽管JMS规范出台已经是很久的事情了，但是JMS在当今的J2EE应用中间仍然扮演着特殊的地位。
<!-- more -->
* 同类技术对比
 * ![](/img/15007971416993.jpg)
* 学习前提
 * 要有 Java 基础
 * 要有 Java Web 基础
 * 要有 Spring 基础

## 为什么使用ActiveMQ(消息中间件)？
我们都知道，应用程序之间的调用都是通过暴露接口形式的相互调用，但是如果业务越来越复杂，接口会越来越多，各个应用程序之间相互耦合，后期管理起来会非常的麻烦，如下图所示：
![](/img/15007985608005.jpg)
这时候如果通过消息中间件的方法的话，只要在需要的时候把消息发送到消息中间件就可以，这时候`消息中间件`就成了嫁接各个系统之间的桥梁，如下图所示：
![](/img/15007986805155.jpg)
那么从上面两张图片我们可以总结出：`通过消息中间件可以解耦服务调用！`

## 了解JMS规范
* 术语定义
 * JMS即 Java 消息服务（Java Message Service）应用程序接口，是一个 Java 平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。Java 消息服务是一个与具体平台无关的 API，绝大多数MOM提供商都对JMS提供支持。
* 相关概念
 * 提供者：实现JMS规范的消息中间件服务器。
 * 客户端：发送或者接受消息的客户端。
 * 生产者/发布者：创建并发送消息的客户端。
 * 消费者/订阅者：接受并处理消息的客户端。
 * 消息：应用程序之间传递的数据内容。
 * 消息模式：在客户端之间传递消息的方式，JMS中定义了主题和队列两种模式。
* JMS消息模式
 * **队列模型**：
   * 客户端包括生产者和消费者。
   * 队列中的消息只能被一个消费者消费。
   * 消费者可以随时的消费队列中的消息。
   * 队列模型示意图![队列模型示意图](/img/15007999134263.jpg)
  * **主题模型**：
    * 客户端包括发布者和订阅者。
    * 主题中的消息被所有的订阅者消费。
    * 消费者不能消费订阅之前就发送到主题中的消息。
    * 主题模型示意图![](/img/15008002971212.jpg)
* JMS编码接口
 * `ConnectionFactory`用于创建连接到消息中间件的连接工厂。
 * `Destination`指消息发布和接收的地点，包括主题和队列。
 * `Connection`代表了应用程序和消息服务器之间的通信链路。
 * `Session`代表一个单线程的上下文，用于发送和接收消息。
 * `MessageConsumer`由会话创建，用于接收发送到目标的消息。
 * `MessageProducer`由会话创建，用于发送消息到目标。
 * `Message`是消费者和生产者之间传送的消息对象，消息头，一组消息属性，一个消息体。
 * 编码接口关系示意图![](/img/15008010925844.jpg)

## 安装ActiveMQ
### 安装环境
- win10/OSX 10.11.6
- JDK 1.8 (**注意一点，ActiveMQ 5.15的版本需要JDK 的最低版本为1.8，否则将安装失败！！！**)
- ActiveMQ 5.15

### win安装ActiveMQ
1. 下载ActiveMQ安装包，地址：http://activemq.apache.org/activemq-5150-release.html
 ![](/img/15008015863809.jpg)
2. 解压到本地目录，进入`C:\software\apache-activemq-5.15.0\bin\win64`下面，32位系统进入到win32目录下面。![](/img/15008025079668.jpg)
3. 启动服务，鼠标右击`activemq.bat`，以管理员权限运行。这时候会弹出一个终端窗口，表示服务已经启动。关闭终端窗口，服务将会被停止。**PS：如果一直无法启动ActiveMQ服务，请检查是否安装了JDK。**![](/img/15008035551025.jpg)
4. 上面这种启动方式，一旦关闭终端窗口，服务讲会被停止，也可以采用注册服务的方式。鼠标右击以管理员权限运行`InstallService.bat`，然后右击我的电脑，点击管理，进入windows服务列表，启动ActiveMQ服务。这样ActiveMQ服务就会一直在后台运行。![](/img/15008039026918.jpg)
5. 浏览器访问 [http://localhost:8161/](http://localhost:8161/)，如果看到如下页面，表示已经安装成功。![](/img/15008041048386.jpg)

### Linux/mac安装ActiveMQ
1. 下载ActiveMQ安装包，地址：http://activemq.apache.org/activemq-5150-release.html
 ![](/img/15008044271116.jpg)
2. 根据个人习惯，将源码解压到指定的目录，我的解压路径：`/usr/local/apache-activemq-5.15.0`。
3. 打开终端，cd 到`/usr/local/apache-activemq-5.15.0/bin`目录下面，执行`./activemq start` 命令，启动 ActiveMQ服务。如果要停止服务，执行`./activemq stop`命令。
4. 浏览器访问 [http://localhost:8161/](http://localhost:8161/)，如果看到如下页面，表示已经安装成功。![](/img/15008041048386.jpg)

## 演示
### 组织结构

1. 创建一个maven项目，项目名为：jms-test，其中pom.xml文件中引入ActiveMQ的核心包

 ```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.leeyom.jms</groupId>
  <artifactId>jms-test</artifactId>
  <packaging>war</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>jms-test Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.activemq</groupId>
      <artifactId>activemq-all</artifactId>
      <version>5.12.0</version>
    </dependency>
  </dependencies>
  <build>
    <finalName>jms-test</finalName>
  </build>
</project>
```
2. 项目结构

 ```lua
├── jms-test
|    ├── java
|    |   ├──com.leeyom.jms.queue -- 队列模式
|    |   |   ├── AppConsumer.java -- 队列模式消费者
|    |   |   ├── AppProducer.java -- 队列模式生产者
|    |   ├──com.leeyom.jms.topic -- 主题模式
|    |   |   ├── AppConsumer.java -- 主题模式订阅者
|    |   |   ├── AppProducer.java -- 主题模式发布者
```

### 队列模式的消息演示

1. 创建一个生产者的类，`AppProducer.java`，用于发送消息。

 ```java
package com.leeyom.jms.queue;
import org.apache.activemq.ActiveMQConnectionFactory;
import javax.jms.*;
/**
 * description: 生产者
 * author: leeyom
 * date: 2017-07-22 05:42
 * Copyright © 2017 by leeyom
 */
public class AppProducer {
    private static final String url = "tcp://192.168.31.190:61616";
    private static final String queueName = "queue-test";
    public static void main(String[] args) throws JMSException {
        //1. 创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(url);
        //2. 创建连接
        Connection connection = connectionFactory.createConnection();
        //3. 启动连接
        connection.start();
        //4. 创建会话
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        //5. 创建目的
        Destination destination = session.createQueue(queueName);
        //6. 创建一个生产者
        MessageProducer producer = session.createProducer(destination);
        for (int i = 0 ; i < 100 ; i++) {
            //7. 创建一个消息
            TextMessage textMessage = session.createTextMessage("test" + i);
            //8. 发送消息
            producer.send(textMessage);
            System.out.println("发送消息：" + textMessage.getText());
        }
        //9. 关闭连接
        connection.close();
    }
}
```

2. 创建一个消费者的类，`AppConsumer.java`，用于接收生产者发送的消息。

 ```java
package com.leeyom.jms.queue;
import org.apache.activemq.ActiveMQConnectionFactory;
import javax.jms.*;
/**
 * description: 队列模式消费者
 * author: leeyom
 * date: 2017-07-22 05:41
 * Copyright © 2017 by leeyom
 */
public class AppConsumer {
    private static final String url = "tcp://192.168.31.190:61616";
    private static final String queueName = "queue-test";
    public static void main(String[] args) throws JMSException {
        //1. 创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(url);
        //2. 创建连接
        Connection connection = connectionFactory.createConnection();
        //3. 启动连接
        connection.start();
        //4. 创建会话
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        //5. 创建目的
        Destination destination = session.createQueue(queueName);
        //6. 创建一个消费者
        MessageConsumer consumer = session.createConsumer(destination);
        //7. 消费者接收消息
        consumer.setMessageListener(new MessageListener() {
            public void onMessage(Message message) {
                TextMessage textMessage = (TextMessage) message;
                try {
                    System.out.println("接收消息：" + textMessage.getText());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

3. 运行生产者类`AppProducer.java`，然后登陆ActiveMQ管理界面，我们会发现我们创建了100条消息。![](/img/15008072688122.jpg)
4. 运行消费者类`AppConsumer.java`去接收刚刚生产者发送的100条消息。这里我们创建两个消费者实例，通过对比我们可以验证我们之前所说的`队列中的消息只能被一个消费者消费。`那我们创建两个消费者将平分接收那100条消息，每人各接收50条，并且不会出现重复的消息。
 ![](/img/15008076847030.jpg)
![](/img/15008077213162.jpg)


### 主题模式的消息演示
1. 创建一个发布者的类，`AppProducer.java`，用于发布消息。

 ```java
package com.leeyom.jms.topic;
import org.apache.activemq.ActiveMQConnectionFactory;
import javax.jms.*;
/**
 * description: 主题模式发布者
 * author: leeyom
 * date: 2017-07-22 05:42
 * Copyright © 2017 by leeyom
 */
public class AppProducer {
    private static final String url = "tcp://192.168.31.190:61616";
    private static final String topicName = "topic-test";
    public static void main(String[] args) throws JMSException {
        //1. 创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(url);
        //2. 创建连接
        Connection connection = connectionFactory.createConnection();
        //3. 启动连接
        connection.start();
        //4. 创建会话
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        //5. 创建目的
        Destination destination = session.createTopic(topicName);
        //6. 创建一个发布者
        MessageProducer producer = session.createProducer(destination);
        for (int i = 0 ; i < 100 ; i++) {
            //7. 创建一个消息
            TextMessage textMessage = session.createTextMessage("test" + i);
            //8. 发送消息
            producer.send(textMessage);
            System.out.println("发送消息：" + textMessage.getText());
        }
        //9. 关闭连接
        connection.close();
    }
}
```

2. 创建一个订阅者的类，`AppConsumer.java`，用于接收发布者发送的消息。

 ```java
package com.leeyom.jms.topic;
import org.apache.activemq.ActiveMQConnectionFactory;
import javax.jms.*;
/**
 * description: 主题模式订阅者
 * author: leeyom
 * date: 2017-07-22 05:42
 * Copyright © 2017 by leeyom
 */
public class AppConsumer {
    private static final String url = "tcp://192.168.31.190:61616";
    private static final String topicName = "topic-test";
    public static void main(String[] args) throws JMSException {
        //1. 创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(url);
        //2. 创建连接
        Connection connection = connectionFactory.createConnection();
        //3. 启动连接
        connection.start();
        //4. 创建会话
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        //5. 创建目的
        Destination destination = session.createTopic(topicName);
        //6. 创建一个订阅
        MessageConsumer consumer = session.createConsumer(destination);
        //7. 订阅者接收消息
        consumer.setMessageListener(new MessageListener() {
            public void onMessage(Message message) {
                TextMessage textMessage = (TextMessage) message;
                try {
                    System.out.println("接收消息：" + textMessage.getText());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```
4. 运行订阅者类`AppConsumer.java`去接收刚刚发布者发送的100条消息。这里我们创建两个订阅者实例，通过对比我们可以验证我们之前所说的JMS主题模型图，我们创建两个订阅者都接收到了100条的消息。
![](/img/15008093093543.jpg)
![](/img/15008093532999.jpg)

## 参考资料
* [慕课网Java消息中间件教学视频](http://www.imooc.com/learn/856)
* 项目源码地址：[github](https://github.com/wangleeyom/demo-warehouse)，喜欢就点个star吧😁
