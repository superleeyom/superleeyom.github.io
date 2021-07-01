---
title: Spring Cloud 服务注册与发现（Eureka和Consul）
date: 2018-03-15 21:31:33
comments: true
tags:
  - Eureka
  - Consul
categories:
  - 编程
---

Spring Cloud 是一个基于 Spring Boot 实现的云应用开发工具，它为基于JVM的云应用开发中涉及的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。它下面有很多的子项目，比如：Spring Cloud Config、Spring Cloud Netflix、Spring Cloud Bus、Spring Cloud for Cloud Foundry、Spring Cloud Open Service Broker、Spring Cloud Cluster、Spring Cloud Consul等等，本篇主要简单使用 Spring Cloud 的子项目 Spring Cloud Consul 和 Spring Cloud Netflix 模块 Eureka 实现简单的服务注册与发现。

<!-- more -->

# Spring Cloud 版本简介

去 Spring Cloud 的官网瞅了一眼，Spring Cloud 目前最新的版本是 Finchley.M7 。由于 Spring Cloud 下面的子项目太多，他的命名方式采用：版本名+版本号。另外其中需要注意一点的就是，由于 Spring Cloud 是基于 Spring Boot ，所以不同版本的 Spring Cloud 需要搭配不同版本的 Spring Boot，否则可能会出现不兼容的情况。下面是[官网](http://projects.spring.io/spring-cloud/)的一些建议：

- Finchley 版本基于 Spring Boot 2.0.x，在 Spring Boot 1.5.x 存在不兼容。
- Dalston and Edgware 版本基于 Spring Boot 1.5.x ，在 Spring Boot 2.0.x 存在不兼容。
- Camden 版本基于 Spring Boot 1.4.x，但是在 Spring Boot 1.5.x 测试通过。
- Brixton 版本基于 Spring Boot 1.3.x，但是在 Spring Boot 1.4.x 测试通过，在 Spring Boot 1.2.x 无法编译通过。
- Angel 版本基于 Spring Boot 1.2.x，在 Spring Boot 1.3.x. 无法编译通过。

那我实验的 Spring Cloud 版本是 Edgware.SR2，基于 Spring Boot 1.5.9.RELEASE ，所以选择不同版本的 Spring Cloud 一定要注意 Spring Boot 的版本，一般推荐带 SR 的版本，意思是稳定版本。

# Eureka 和 Consul 简介

Eureka 是 Spring Cloud Netflix 的一个服务治理模块，除了Eureka，Spring Cloud Netflix 还有 Zuul、Ribbon、Feign、Hystrix、Hystrix Dashboard、Turbine等组件，而 Spring Cloud Netflix 是 Spring Cloud 的一个子项目，喜欢看美剧的应该都知道 Netflix 公司吧？它是一家互联网流媒体播放商,是美国视频巨头，像《纸牌屋》、《黑镜》、《毒枭》这些美剧就来自 Netflix 公司。Spring Cloud Consul 同样的也是 Spring Cloud 的一个子项目，Spring Cloud Consul 是一个服务发现与配置工具，二者都可以用于服务发现和注册。

# Spring Cloud Eureka

## 创建服务注册中心

在 Dubbo 服务治理体系中，是将服务注册到 Zookeeper 中，那 Eureka 在 Spring Cloud 的服务治理体系中就可以担任服务注册中心的任务，当然他本身也是可以担任客户端（服务）的角色。
- 首先先创建一个 Spring Boot 项目，名称为 `eureka-server`，用于创建服务注册中心。
- 在 pom.xml 文件中引入依赖，这里 Spring Boot 的版本是 `1.5.9.RELEASE`，搭配的 Spring Cloud 的版本是 `Edgware.SR2`， `spring-cloud-starter-eureka-server` 是创建服务注册中心的核心依赖，完整的 [pom.xml](https://github.com/wangleeyom/spring-cloud-learning/blob/master/spring-cloud-eureka/eureka-server/pom.xml)。
  ```xml
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>1.5.9.RELEASE</version>
      <relativePath/>
  </parent>
  <dependencies>
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter</artifactId>
      </dependency>

      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-test</artifactId>
          <scope>test</scope>
      </dependency>

      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-eureka-server</artifactId>
      </dependency>
  </dependencies>

  <dependencyManagement>
      <dependencies>
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-dependencies</artifactId>
              <version>Edgware.SR2</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
  </dependencyManagement>    
  ```
- 配置服务中心的端口和 IP，由于 Eureka 服务注册中心，默认自己做为客户端来注册自己，需要禁止掉，对应的 application.properties。
  ```properties
  spring.application.name=eureka-server
  server.port=8080

  eureka.instance.hostname=localhost
  # 禁止注册中心将自己做为客户端来注册自己
  eureka.client.register-with-eureka=false
  eureka.client.fetch-registry=false  
  ```
- 在 Spring Boot 的启动类 `EurekaServerApplication.java` 开启注解：`@EnableEurekaServer`，标识该服务为注册中心服务。
  ```java
  @SpringBootApplication
  @EnableEurekaServer
  public class EurekaServerApplication {
  	public static void main(String[] args) {
  		SpringApplication.run(EurekaServerApplication.class, args);
  	}
  }  
  ```
- 启动项目，访问 [http://localhost:8080/](http://localhost:8080/)，就可以看到 Eureka 服务注册中心管理界面：
  ![mark](http://image.leeyom.top/blog/180317/I6D34e77Ge.png)
- 这样基于 Eureka 的一个简单的服务注册中心就搭建好了，下面我们创建一个服务并将其注册到服务注册中心。

## 创建服务提供者

- 创建一个基于 Spring Boot 的项目 `eureka-client`，它充当服务提供者，引入核心依赖 `spring-cloud-starter-eureka`，完整的 [pom.xml](https://github.com/wangleeyom/spring-cloud-learning/blob/master/spring-cloud-eureka/eureka-client/pom.xml)。
  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-eureka</artifactId>
  </dependency>  
  ```
- 在 `application.properties` 配置该服务端口号，以及要注册的服务中心的 url。
  ```properties
  #应用名称
  spring.application.name=eureka-client
  #应用端口
  server.port=9090
  #Eureka注册中心的url
  eureka.client.serviceUrl.defaultZone=http://localhost:8080/eureka/  
  ```
- 编写一个 Controller类，并在 Controller 中注入 `DiscoveryClient` 实例，用于获取所有服务名称并返回，`DiscoveryClient` 是 Spring Cloud 在服务发现这一层做的抽象接口，用于获取服务实例和服务清单，另外需要注意一点的就是 `DiscoveryClient` 的类路径是 `org.springframework.cloud.client.discovery.DiscoveryClient`，不是 `com.netflix.discovery.DiscoveryClient`，很容易搞混。
  ```java
  @RestController
  public class DiscoverServiceController {

      @Autowired
      private DiscoveryClient discoveryClient;

      @GetMapping("/getAllServerInstance")
      public String getAllServerInstance() {
          return discoveryClient.getServices().toString();
      }
  }  
  ```
- 在项目的启动类 `EurekaClientApplication` 中开启注解 `@EnableDiscoveryClient`，意味着该服务将会纳入服务治理体系当中，会被注册到服务中心。
  ```java
  @EnableDiscoveryClient
  @SpringBootApplication
  public class EurekaClientApplication {

      public static void main(String[] args) {
          SpringApplication.run(EurekaClientApplication.class, args);
      }
  }
  ```
- 启动项目，访问 [http://localhost:9090/getAllServerInstance](http://localhost:9090/getAllServerInstance)，将返回我们的刚创建的这个服务名称：`[eureka-client]`，既然能返回服务实例名称，那就也意味这我们的服务已经在服务注册中心注册成功了，访问 [http://localhost:8080/](http://localhost:8080/) ，显示服务注册成功。
  ![mark](http://image.leeyom.top/blog/180317/DeHEL4KjC9.png)
- 接下来就将服务治理体系由 Eureka 切换到 Consul。

# Spring Cloud Consul

## 安装 Consul

Consul 是一个服务发现与配置工具，它可以用于服务发现，健康检查，Key/Value 存储，多数据中心等等。使用 Consul 首先先进行安装，安装 Consul 有[两种方式](https://www.consul.io/docs/install/index.html)：

1. 使用预编译的二进制文件
2. 从源代码安装

那我这里直接就采用第一种方式，安装步骤如下：
- 首先[下载](https://www.consul.io/downloads.html)二进制文件，我使用的平台是 MacOS。
- 解压，然后终端执行 `sudo scp consul /usr/local/bin/`，将该二进制文件拷贝到 `/usr/local/bin`目录下。
- 终端执行 `consul` 看命令是否生效，若出现如下内容，说明 `Consul` 安装成功。
  ```
  leeyomdeMacBook-Pro:bin lidong$ consul
  usage: consul [--version] [--help] <command> [<args>]

  Available commands are:
      agent          Runs a Consul agent
      configtest     Validate config file
      event          Fire a new event
      exec           Executes a command on Consul nodes
      force-leave    Forces a member of the cluster to enter the "left" state
      info           Provides debugging information for operators
      join           Tell Consul agent to join cluster
      keygen         Generates a new encryption key
      keyring        Manages gossip layer encryption keys
      kv             Interact with the key-value store
      leave          Gracefully leaves the Consul cluster and shuts down
      lock           Execute a command holding a lock
      maint          Controls node or service maintenance mode
      members        Lists the members of a Consul cluster
      monitor        Stream logs from a Consul agent
      operator       Provides cluster-level tools for Consul operators
      reload         Triggers the agent to reload configuration files
      rtt            Estimates network round trip time between nodes
      snapshot       Saves, restores and inspects snapshots of Consul server state
      version        Prints the Consul version
      watch          Watch for changes in Consul  
  ```

## 创建服务提供者

安装好 `Consul` 后，接下来我们创建一个服务提供者 `consul-client`，然后并测试服务的注册情况。

- 首先，先引入核心依赖：`spring-cloud-starter-consul-discovery`，完整的 [pom.xml](https://github.com/wangleeyom/spring-cloud-learning/blob/master/spring-cloud-consul/consul-client/pom.xml)。
  ```xml
  <dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-consul-discovery</artifactId>
  </dependency>
  ```
- 在 application.properties 中配置注册中心端口和IP。
  ```properties
  #应用名称
  spring.application.name=consul-client
  #Consul注册中心IP
  spring.cloud.consul.host=localhost
  #Consul注册中心端口
  spring.cloud.consul.port=8500  
  ```
- 测试的 Controller 跟 `eureka-clinet` 的 Controller 一样，依旧是返回所有的服务清单，这里就不重复了。
- 在 Spring Boot 的启动类 `ConsulClientApplication` 类中开启注解 `@EnableDiscoveryClient`，将服务注册到 Consul。
  ```java
  @EnableDiscoveryClient
  @SpringBootApplication
  public class ConsulClientApplication {

      public static void main(String[] args) {
          SpringApplication.run(ConsulClientApplication.class, args);
      }
  }  
  ```
- 使用命令 `consul agent -dev` 启动 Consul 的开发模式，会打印如下的内容：
  ```
  ==> Starting Consul agent...
  ==> Consul agent running!
             Version: 'v1.0.6'
             Node ID: 'a2ed19a2-474a-a355-08c0-514162e233b2'
           Node name: 'lmpc0039'
          Datacenter: 'dc1' (Segment: '<all>')
              Server: true (Bootstrap: false)
         Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, DNS: 8600)
        Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
             Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

  ==> Log data will now stream in as it occurs:

  2018/03/17 16:12:42 [DEBUG] Using random ID "a2ed19a2-474a-a355-08c0-514162e233b2" as node ID
  2018/03/17 16:12:42 [INFO] raft: Initial configuration (index=1):
  [{Suffrage:Voter ID:a2ed19a2-474a-a355-08c0-514162e233b2 Address:127.0.0.1:8300}]
  2018/03/17 16:12:42 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
  2018/03/17 16:12:42 [INFO] serf: EventMemberJoin: lmpc0039.dc1 127.0.0.1
  2018/03/17 16:12:42 [INFO] serf: EventMemberJoin: lmpc0039 127.0.0.1
  2018/03/17 16:12:42 [INFO] consul: Adding LAN server lmpc0039 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
  2018/03/17 16:12:42 [INFO] consul: Handled member-join event for server "lmpc0039.dc1" in area "wan"
  2018/03/17 16:12:42 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
  2018/03/17 16:12:42 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
  2018/03/17 16:12:42 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
  2018/03/17 16:12:42 [INFO] agent: started state syncer
  2018/03/17 16:12:42 [WARN] raft: Heartbeat timeout from "" reached, starting election
  2018/03/17 16:12:42 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
  2018/03/17 16:12:42 [DEBUG] raft: Votes needed: 1
  2018/03/17 16:12:42 [DEBUG] raft: Vote granted from a2ed19a2-474a-a355-08c0-514162e233b2 in term 2. Tally: 1
  2018/03/17 16:12:42 [INFO] raft: Election won. Tally: 1
  2018/03/17 16:12:42 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
  2018/03/17 16:12:42 [INFO] consul: cluster leadership acquired
  2018/03/17 16:12:42 [INFO] consul: New leader elected: lmpc0039
  2018/03/17 16:12:42 [DEBUG] consul: Skipping self join check for "lmpc0039" since the cluster is too small
  2018/03/17 16:12:42 [INFO] consul: member 'lmpc0039' joined, marking health alive
  2018/03/17 16:12:42 [DEBUG] Skipping remote check "serfHealth" since it is managed automatically
  2018/03/17 16:12:42 [INFO] agent: Synced node info
  2018/03/17 16:12:42 [DEBUG] agent: Node info in sync
  2018/03/17 16:12:43 [DEBUG] Skipping remote check "serfHealth" since it is managed automatically
  2018/03/17 16:12:43 [DEBUG] agent: Node info in sync  
  ```
- 启动项目，访问：`http://localhost:8080/getAllServerInstance`，返回 `[consul, consul-client]`，访问 `http://localhost:8500/ui/`，在 Consul 的控制台就可以看到我们注册的服务。
  ![mark](http://image.leeyom.top/blog/180317/li0ddJj0jg.png)
- 参考：
  - [Consul 简介、安装、常用命令的使用](http://blog.csdn.net/u010046908/article/details/61916389)
  - [Spring Cloud构建微服务架构：服务注册与发现（Eureka、Consul）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-1/)
  - Spring Cloud 官网：http://projects.spring.io/spring-cloud/
  - Consul 官网：[https://www.consul.io/](https://www.consul.io/)
  - Spring Cloud 中文网：https://springcloud.cc/
  - Spring Cloud Dalston 版中文文档：https://springcloud.cc/spring-cloud-dalston.html
