---
title: Tomcat关闭报内存溢出异解决方案
date: 2017-06-20 21:13:20
comments: true
tags:
- Tomcat
categories:
- 编程
---

在项目开发的时候，每次关闭Tomcat控制台都会报内存溢出的异常，因为是warning级别的警告，并且开发阶段也并不影响项目的运行，所以呢也就没有去在意。但是当把项目打包部署到Linux服务器上后，启动Tomcat，然后再停止，又出现内存溢出的情况，然后再启动，发现Tomcat启动再也无法启动，只能重启服务器，才意识到这个问题是比较严重的问题。通过查找一系列的资料，终于把这个问题解决了，所以记录一下这个问题解决过程，给以后可能会遇到这个问题的朋友一个解决方案。

<!--more-->

## 开发环境

* JDK 1.8
* Tomcat 8
* MySQL 5.7
* IDEA 2017.1

## 异常信息
首先来看下我遇到的异常信息是什么：
```java
警告: The web application [LeadermentEnterpriseSystemV2] registered the JDBC driver [com.mysql.jdbc.Driver] but failed to unregister it when the web application was stopped. To prevent a memory leak, the JDBC Driver has been forcibly unregistered.
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-1] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-2] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-3] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-4] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-5] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-6] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-7] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-8] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-9] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [DefaultQuartzScheduler_Worker-10] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:519)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [Timer-0] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 java.util.TimerThread.mainLoop(Timer.java:552)
 java.util.TimerThread.run(Timer.java:505)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread-#0] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:534)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread-#1] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:534)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread-#2] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:534)
六月 20, 2017 2:16:22 下午 org.apache.catalina.loader.WebappClassLoaderBase clearReferencesThreads
警告: The web application [LeadermentEnterpriseSystemV2] appears to have started a thread named [Abandoned connection cleanup thread] but has failed to stop it. This is very likely to create a memory leak. Stack trace of thread:
 java.lang.Object.wait(Native Method)
 java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
 com.mysql.jdbc.AbandonedConnectionCleanupThread.run(AbandonedConnectionCleanupThread.java:43)
六月 20, 2017 2:16:22 下午 org.apache.coyote.AbstractProtocol stop
信息: Stopping ProtocolHandler ["http-nio-8080"]
六月 20, 2017 2:16:22 下午 org.apache.coyote.AbstractProtocol stop
信息: Stopping ProtocolHandler ["ajp-nio-8009"]
六月 20, 2017 2:16:22 下午 org.apache.coyote.AbstractProtocol destroy
信息: Destroying ProtocolHandler ["http-nio-8080"]
六月 20, 2017 2:16:22 下午 org.apache.coyote.AbstractProtocol destroy
信息: Destroying ProtocolHandler ["ajp-nio-8009"]
```

## 异常原因分析及解决方案

我总结了下，出现以上的异常主要有以下几个原因：

1. mysql jdbc 未注销
2. shiro 权限框架会话验证调度器未关闭
3. c3p0连接池链接未关闭

下面来逐一的分析下为什么会出现上面这样的情况：

### mysql jdbc 未注销

在`tomcat 6.0.24`版本之后，加入了一个`memory leak listener`(JreMemoryLeakPreventionListener，有兴趣可详细查去源码), 在tomcat stop、undeployed、reloaded的时候，他会检测当前应用的classloader，查看是否有引用泄露。
tomcat定义了一系列的引用泄露规则：

* threadlocal保持引用
* 线程池保持引用
* 驱动注册

如有引用泄露，则提示错误，例如`his is very likely to create a memory leak`类似这样的错误。对于`The web application [LeadermentEnterpriseSystemV2] registered the JDBC driver [com.mysql.jdbc.Driver] but failed to unregister it when the web application was stopped. To prevent a memory leak, the JDBC Driver has been forcibly unregistered.`这种异常其实就是MySQL的JDBC驱动无法注销的原因所造成的。我在网上查找了很多的解决方案，其中比较权威的stackoverflow给出的解决方案邮两种：

1. 将MySQL的驱动放到Tomcat的lib目录下面，同时移除WEB-INF/lib目录下的MySQL 驱动，但是我实验了下，对于我现在这种情况并不生效。

2. 创建一个 `ServletContextListener`,然后在 contextDestroyed 方法中手动注销。

  这个方案中又分为两步：
  * 2.1 将MySQL JDBC驱动在pom.xml文件中更新成最新的版本(5.1.42)。
  * 2.2 新建 JdbcDriverListener.java 文件，具体内容如下：

  ```java
  public class JdbcDriverListener implements ServletContextListener {

    private static Logger logger = Logger.getLogger(UserAccountsHandler.class);

    @Override
    public void contextInitialized(ServletContextEvent sce) {

    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        // 解决Tomcat mysql 驱动内存泄漏，手动注销JDBC
        Enumeration<Driver> drivers = DriverManager.getDrivers();
        Driver d = null;
        while (drivers.hasMoreElements()) {
            try {
                d = drivers.nextElement();
                DriverManager.deregisterDriver(d);
            } catch (SQLException ex) {
            }
        }
        AbandonedConnectionCleanupThread.shutdown();
    }
}
  ```
不要忘记在web.xml文件中注册该监听器：
```xml
<listener>
        <listener-class>com.leaderment.common.listener.JdbcDriverListener</listener-class>
</listener>
```

这样的话就可以解决MySQL驱动无法注销的问题。

### shiro 权限框架会话验证调度器未关闭

这个问题是应为shiro权限框架里有用到quartz，所以会出现`appears to have started a thread named [DefaultQuartzScheduler_Worker-1] `等异常。我的解决办法就是将shiro的配置文件中的

```xml
<bean id="sessionValidationScheduler" class="org.apache.shiro.session.mgt.quartz.QuartzSessionValidationScheduler">
        <property name="sessionValidationInterval" value="1800000"/>
        <property name="sessionManager" ref="sessionManager"/>
</bean>
```

代码给注释掉，自己重新写了个过滤器，去判定session是否过期，而不采用shiro的会话验证器，所以该问题也就这样被解决。

### c3p0连接池链接未关闭

修改applicationContext.xml文件，添加属性 `destroy-method="close"`，即可解决数据库连接未关闭的问题。

```xml
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">
	<property name="user" value="${jdbc.user}"></property>
	<property name="password" value="${jdbc.password}"></property>
	<property name="jdbcUrl" value="${jdbc.jdbcUrl}"></property>
	<property name="driverClass" value="${jdbc.driverClass}"></property>
	<property name="idleConnectionTestPeriod" value="60"/>
	<property name="maxIdleTime" value="60"/>
	<property name="testConnectionOnCheckin" value="false" />
	<property name="testConnectionOnCheckout" value="true" />
	<property name="preferredTestQuery" value="SELECT 1" />
</bean>
```

## 总结

最终的原因归根结底就是Tomcat的进程无法释放的问题，那么造成这个问题出现的就是上面三个原因，只有通过一步步的排查才能根本上解决问题。这个问题花费了我一两天的时间，虽然有些地方不算是彻底解决，但是至少Tomcat停止的时候，不在报内存溢出的警告了。我的强迫症总算好受一点了。如果有更好的解决方案，可以在评论区留言一起讨论。


