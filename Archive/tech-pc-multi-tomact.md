---
title: 一台机子上面启动多个tomact的问题
date: 2016-10-27 21:57:15
comments: true
categories:
- 编程
tags:
- tomact
---

之前一直在myeclipse上进行开发，但是myeclipse10非常的卡顿，实在受不了，就换IDEA 进行开发。但是换到IDEA后，把项目部署后，也能够跑起来，但是却发现无论如何都无法调取到接口。后来在终端用命令行启动tomact，只要把其中一个项目启动的时候，再启动第二个tomact的时候，总是启动后，又马上关闭了，有人说是在mac系统下权限的问题，但是改了后依旧无果。后来冷静的想了下，http端口改了，会不会两个tomact有其他的端口冲突呢？果不其然，上网查了下，确实有这个问题，所以特意记录下。

<!--more-->

如果需要在一台机子上启动多个Tomcat服务器，在默认设置下肯定会发生端口冲突。为解决这个问题，只需修改conf子目录中的server.xml文件即可。共需修改三处：

- 修改http访问端口（默认为8080端口）:

	```xml
	<Connector port=”8080” protocol=”HTTP/1.1″
	connectionTimeout=”20000″
	redirectPort=”8443″ URIEncoding=”gb2312″/>
	```
- 修改Shutdown端口（默认为8005端口）:

	```xml
	<Server port=”8005” shutdown=”SHUTDOWN”>
	```
- 修改JVM启动端口（默认为8009端口）:

	```xml
	<Connector port=”8009” protocol=”AJP/1.3″ redirectPort=”8443″ />
	```
