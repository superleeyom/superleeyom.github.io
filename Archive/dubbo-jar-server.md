---
title: Dubbo 服务启动方式浅析
date: 2018-03-07 21:33:09
comments: true
tags:
  - dubbo
categories:
  - 编程
---

Dubbo是阿里巴巴公司开源的一个高性能优秀的服务框架，使得应用可通过高性能的RPC实现服务的输出和输入功能，可以和Spring框架无缝集成。最近这段时间，参与了公司一个项目的重构，最主要的就是使用Dubbo将其拆分成微服务架构。本篇中不细说如何使用Dubbo进行rpc服务的调用，主要是浅析下基于Dubbo的微服务的打包以及启动方式。

<!-- more -->

Dubbo服务的打包方式主要有两种，一种是打成jar，另外一种是打成war。但是通常来说，一般是将服务打成jar包，然后controller通过rpc调用服务。

Dubbo服务的启动方式主要有三种：

- web容器运行，比方说Tomcat、JBoss，那他对应的服务打包方式自然就是war包。
- 自建Main方法，初始化spring配置文件，基于spring容器。
- Dubbo自带容器，Spring Container。

这三种启动方式的区别主要如下：

## web容器

通过web容器运行Dubbo服务应该是最简单的，只要把服务打成war包，丢到tomcat就完事，但是这样做会有额外的资源开销，比如需要为web容器分配端口，内存等等，这种方式不太推荐。

## 自建main方法

自建main方法，他所对应的打包方式是jar，运行jar包都需要一个入口（main方法），首先看自建main方法的：

```java
public class Main {
	private static boolean running = true;

	@SuppressWarnings("resource")
	public static void main(String[] args) throws Exception {
		final ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
				new String[] { "classpath:spring-context.xml" });
		Runtime.getRuntime().addShutdownHook(new Thread() {
			public void run() {
				context.stop();
				System.err.println("===================== service has stop!!!========================");
				synchronized (Main.class) {
					running = false;
					Main.class.notify();
				}
			}
		});

		context.start();
		System.err.println("===================== service has start!!!========================");
		synchronized (Main.class) {
			while (running) {
				try {
					Main.class.wait();
				} catch (Throwable e) {
				}
			}
		}
	}
}
```
他的原理是读取spring配置文件，装载spring容器，这种方法虽说也是可行的，但是他有个缺点就是，无法使用Dubbo自带的一些高级特性，比方说Dubbo的优雅停机。此方式适合在开发阶段使用。

## Dubbo自带容器

其实Dubbo是自带容器的，服务容器是一个 standalone 的启动程序，他不需要tomcat等web容器的支持就可以启动，服务容器只是一个简单的 Main 方法，并加载一个简单的 Spring 容器，用于暴露服务。

Dubbo 内嵌了三种容器，分别是：

- `Spring Container`：这个就是我们常用的spring容器
- `Jetty Container`：内嵌 Jetty web容器
- `Log4j Container`：自动配置 log4j 容器

首先先来看下完整的pom文件：[pom.xml](https://gist.github.com/superleeyom/2e8b96520ee93b7ac52d823475905bed)

因为Dubbo默认加载的是`META-INF/spring` 目录下的所有 Spring 配置，那所以我们需要在项目打包的时候将spring配置文件复制到`/META-INF/spring`目录下，我这里的spring的配置文件是`spring-context.xml`，具体的实现代码块：
```xml
<resource>
  <targetPath>${project.build.directory}/classes/META-INF/spring</targetPath>
  <directory>src/main/resources</directory>
  <filtering>true</filtering>
  <includes>
    <include>spring-context.xml</include>
  </includes>
</resource>
```
其次需要借助maven的插件`maven-shade-plugin`，将我们所有的依赖打成一个jar包，同时指定该jar包的入口，也就是Dubbo指定的入口：`com.alibaba.dubbo.container.Main`，所以就有如下的片段：
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>2.4.1</version>
  <configuration>
    <!-- 是否生成缩减的pom文件，默认不配置是true -->
    <createDependencyReducedPom>false</createDependencyReducedPom>
  </configuration>
  <executions>
    <execution>
      <phase>package</phase>
      <goals>
        <goal>shade</goal>
      </goals>
      <configuration>
        <transformers>
          <transformer
            implementation="org.apache.maven.plugins.shade.resource.
            ManifestResourceTransformer">
            <!-- 启动类 -->
            <mainClass>com.alibaba.dubbo.container.Main</mainClass>
          </transformer>
          <transformer
            implementation="org.apache.maven.plugins.shade.resource.
            AppendingTransformer">
            <resource>META-INF/spring.handlers</resource>
          </transformer>
          <transformer
            implementation="org.apache.maven.plugins.shade.resource.
            AppendingTransformer">
            <resource>META-INF/spring.schemas</resource>
          </transformer>
        </transformers>
      </configuration>
    </execution>
  </executions>
</plugin>
```
执行 eclipse 右击你要打包的项目，执行 `Run as` --> `maven install`，最终target目录下会生成两个jar包，他们的名字是：

- `xxx.jar`
- `original-xxx.jar`

带original前缀的是原始的jar，这个jar没有将其他的依赖打进去，而xxx.jar则是完整的jar包，也是我们最终要部署运行的jar包。最后将打包好的项目使用 `java -jar xxx.jar &`运行服务，若打印`Dubbo service server started!`意味这服务是启动成功了。

另外也可以使用`maven-dependency-plugin`和`maven-jar-plugin`两个插件来进行打包，这两个插件的应用场景如下：

- `maven-dependency-plugin`：用于复制依赖的jar包到指定的文件夹里，这里我是将所有涉及到的依赖jar复制到lib目录下。
- `maven-jar-plugin`：打成jar时，设定manifest的参数，比如指定运行的Main class，还有依赖的jar包，加入classpath中。

对应的pom文件：[pom.xml](https://gist.github.com/superleeyom/ce850ee54123334634345adf0b23cae8)
编译打包成功后，会生成一个jar包和lib目录，lib目录里面放了该服务涉及到的依赖jar包，jar包部署的时候，要将lib目录以及打包好的jar包放到同级目录，否则jar包服务是无法启动。

所以综上，我们更推荐使用Dubbo官方推荐的服务容器去运行服务，一个我们不需要自己创建多余的Main方法，其次可实现优雅关机（ShutdownHook），何乐而不为不是吗？


