---
title: Linux折腾笔记
date: 2017-11-12 22:44:22
comments: true
tags:
- linux
categories:
- 资源
---

由于主要是做服务端开发，总会少不了和linux打交道，鉴于我老容易健忘的习惯，我决定还是专门用一篇文章来记录自己在使用linux的一些笔记，主要是哪天记不起来了，可以再翻出来看看，免得再花时间在网上找教程，主要使用的linux版本为centOS，Ubuntu这两个版本，如果有不对的地方，还希望大家指出。

<!-- more -->

## 安装MongoDB

- `MongoDB`是目前比较热门的非关系型数据库，传统的关系型数据库由：数据库（DataBase）、表（Table）、记录（Record）组成，而`MongoDB`则由：数据库（DataBase）、集合（Collection）、文档对象（Document）三个层次组成，但是这里的集合没有行和列的概念。
- 安装环境：
  - CentOS7
  - MongoDB版本为：3.6.3
  - Version: `RHEL 7 Linux 64-bit X64`
- 下载源码包，并解压，重名为`mongodb`，我一般喜欢把软件都装在`/usr/local/develop-tools/`。
  ```
  # 下载源码包
  wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.6.3.tgz
  # 解压
  tar zxvf mongodb-linux-x86_64-rhel70-3.6.3.tgz
  # 删除解压后的源码包
  rm -f mongodb-linux-x86_64-rhel70-3.6.3.tgz
  # 重命名文件夹
  mv mongodb-linux-x86_64-rhel70-3.6.3 mongodb
  ```
- 增加系统变量：
  ```
  # 编辑环境变量配置文件
  vim ~/.bashrc
  # 加入如下配置
  export MONGODB_HOME=/usr/local/develop-tools/mongodb
  export PATH=$MONGODB_HOME/bin:$PATH
  # 刷新配置文件，使配置文件生效
  source ~/.bashrc
  ```
- 测试是否安装成功，使用命令`mongod -v`，若出现如下内容，便说明安装成功：
  ```
  2018-03-05T22:01:49.391+0800 D NETWORK  [main] fd limit hard:4096 soft:1024 max conn: 819
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] MongoDB starting : pid=14555 port=27017 dbpath=/data/db 64-bit host=centos-linux.shared
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] db version v3.6.3
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] git version: 9586e557d54ef70f9ca4b43c26892cd55257e1a5
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] allocator: tcmalloc
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] modules: none
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] build environment:
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten]     distmod: rhel70
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten]     distarch: x86_64
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten]     target_arch: x86_64
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] options: { systemLog: { verbosity: 1 } }
  2018-03-05T22:01:49.396+0800 D -        [initandlisten] User Assertion: 29:Data directory /data/db not found. src/mongo/db/service_context_d.cpp 98
  2018-03-05T22:01:49.396+0800 I STORAGE  [initandlisten] exception in initAndListen: NonExistentPath: Data directory /data/db not found., terminating
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] now exiting
  2018-03-05T22:01:49.396+0800 I CONTROL  [initandlisten] shutting down with code:100    
  ```
- 创建数据库、日志存放目录：
  ```
  # 创建数据库目录
  mkdir -p /usr/local/develop-tools/mongodb/data
  # 创建日志目录
  mkdir -p /usr/local/develop-tools/mongodb/log
  # 创建日志文件
  touch /usr/local/develop-tools/mongodb/log/mongodb.log
  ```
- 创建全局配置文件：
  ```
  # 编辑配置文件
  vim /etc/mongodb.conf
  # 写入：数据库目录地址、端口号、日志文件路径等配置信息
  dbpath=/usr/local/develop-tools/mongodb/data
  logpath=/usr/local/develop-tools/mongodb/log/mongodb.log
  logappend=true
  port=27017
  fork=true  
  ```
- 先确认是否之前已经启动过`mongodb`，使用命令：`ps -ef | grep mongo`，如果已经启动该任务，先kill掉该进程。
- 通过配置文件启动`mongodb`，执行命令：`mongod -f /etc/mongodb.conf`，如出现如下信息，说明服务启动成功：
  ```
  about to fork child process, waiting until server is ready for connections.
  forked process: 17247
  child process started successfully, parent exiting
  ```
- 进入MongoDB后台shell：`cd /usr/local/develop-tools/mongodb/bin/ && ./mongo`
- 创建一个超级管理员账号：
  ```
  # 切换到管理员模式
  use admin
  # 创建超级管理员
  db.createUser(
     {
       user: "root",
       pwd: "root",
       roles: [ { role: "root", db: "admin" } ]
     }
   )  
  ```
- 创建一个数据库：`use hello_mongodb`
- 创建普通用户，并指定权限：
  ```
  db.createUser(
      {
          user: "leeyom",
          pwd: "root",
          roles: [
              { role: "dbAdmin", db: "hello_mongodb" },
              { role: "readWrite", db: "hello_mongodb" }
          ]
      }
  )  
  ```
- 开启防火墙端口：
  ```
  iptables -A INPUT -p tcp -m tcp --dport 27017 -j ACCEPT
  service iptables save
  service iptables restart  
  ```
- 开启远程访问：
  ```
  # 编辑全局配置文件
  vim /etc/mongodb.conf
  # 开启远程访问
  bind_ip = 0.0.0.0
  ```
- 开启用户验证：
  ```
  # 编辑全局配置文件
  vim /etc/mongodb.conf
  # 新增开启用户验证
  auth=true
  ```
  以上两个开启后，先重启下电脑，使配置生效，然后重新启动服务，下次进入mongod命令行，如果有授权用户格式为：`mongo 127.0.0.1:27017/admin -u 用户名 -p 用户密码`
- 如何关闭`MongoDB`服务：
  ```
  # 用超级管理员身份进入mongod命令行
  mongo 127.0.0.1:27017/admin -u root -p root
  # 切换到管理员模式
  use admin
  # 关闭mongodb服务
  db.shutdownServer()
  ```
- 重新启动服务：`mongod -f /etc/mongodb.conf`
- 若客户端GUI连接不到`mongodb`，首先先查看是否开启防火墙端口，其次看开启远程访问，一般是这两个问题引起的。
- 参考文章：[MongoDB 安装和配置](https://github.com/judasn/Linux-Tutorial/blob/master/markdown-file/MongoDB-Install-And-Settings.md)
- 开始使用！

## 安装FastDFS

参见文章：[centOS上搭建FastDFS图片服务器教程](http://www.leeyom.top/2016/12/22/tech-centos-fastDFS/)

## 安装Nginx

参见文章：[在centOS上安装nginx教程](http://www.leeyom.top/2016/12/08/tech-centOS-nginx/)

## 安装RabbitMQ

- 系统环境为cetos7。
- 首先先安装`EPEL`源，执行命令：
  - `yum -y install epel-release`
- 由于`RabbitMQ`是由`erlang`语言开发的，所以还需要安装`erlang`依赖环境，命令如下：
  - `yum install erlang -y`
- 接下来就可以安装`RabbitMQ`，执行如下命令：
  - `yum install rabbitmq-server -y`
- 安装成功后，启动服务：
  - 先看下自己的主机名，执行命令：`hostname`，我的主机名是：`centos-linux.shared`。
  - 先修改一下 host 文件：`vim /etc/hosts`，添加一行：`127.0.0.1 centos-linux.shared`。
  - 启动：`service rabbitmq-server start`。
  - 停止：`service rabbitmq-server stop`。
  - 状态：`service rabbitmq-server status`。
  - 重启：`service rabbitmq-server restart`。
  - 自启：`chkconfig rabbitmq-server on`。
- 配置：
  - 进入指定目录：`cd /etc/rabbitmq`。
  - 编辑配置文件，开启用户远程访问：`vim rabbitmq.config`。
    - 将`%% {loopback_users, []},`（注意后面有一个逗号）更改为：`{loopback_users, []}`。
  - 开启 Web 界面管理：`rabbitmq-plugins enable rabbitmq_management`。
    - `epel`源安装的话，则是这样运行：`cd /usr/lib/rabbitmq/bin;./rabbitmq-plugins enable rabbitmq_management`。
  - 重启 `RabbitMQ` 服务：`service rabbitmq-server restart`。
  - 开放防火墙端口：
    - `sudo iptables -I INPUT -p tcp -m tcp --dport 15672 -j ACCEPT`
    - `sudo iptables -I INPUT -p tcp -m tcp --dport 5672 -j ACCEPT`
    - `sudo service iptables save`
    - `sudo service iptables restart`
- 浏览器访问：`http://ip地址:15672` 默认管理员账号：guest 默认管理员密码：guest，出现如下图，恭喜，安装成功！
  - ![20180209151818165842106.png](http://image.leeyom.top/20180209151818165842106.png)

## 卸载CentOS7自带的jdk

CentOS7自带的是openjdk，而我们平常开发用的jdk是sunjdk，所以需要卸载掉它自带的jdk，整个过程如下：

- 使用命令：`rpm -qa|grep java`，查询系统自带的jdk，查询的结果如下：
  ```
  [root@centos-linux ~]# rpm -qa|grep java
  java-1.7.0-openjdk-headless-1.7.0.91-2.6.2.3.el7.x86_64
  javapackages-tools-3.4.1-11.el7.noarch
  java-1.7.0-openjdk-1.7.0.91-2.6.2.3.el7.x86_64
  tzdata-java-2015g-1.el7.noarch
  python-javapackages-3.4.1-11.el7.noarch  
  ```
- 使用命令：`rpm -e --nodeps xxx`，xxx代表系统自带的jdk名，这个命令删除系统自带的jdk，这里需要注意的就是，卸载的是带`java`、`openjdk`关键字的包，其他的比方`tzdata-java-2015g-1.el7.noarch`是不能删除的，下面是完整的删除命令：
  ```
  rpm -e --nodeps java-1.7.0-openjdk-headless-1.7.0.91-2.6.2.3.el7.x86_64  
  rpm -e --nodeps java-1.7.0-openjdk-1.7.0.91-2.6.2.3.el7.x86_64
  ```
- 执行`java -version`命令，查看是否卸载成功，若找不到这个命令，说明是卸载成功了，否则卸载失败。

## 安装redis
- linux版本为centOS 6.7。
- 下载地址：[http://redis.io/download](http://redis.io/download)，下载最新版本。
- 利用FTP上传工具，将源码包上传到`/usr/local/`目录。
- 将`redis-4.0.6.tar.gz`源码包解压，解压成功后，删除源码包，如果权限不够，需要加上`sudo`权限。
  ```
  $ tar xzf redis-4.0.6.tar.gz
  $ rm -f redis-4.0.6.tar.gz
  ```
- 进入到redis源码解压后的目录，然后开始编译。
  ```
  $ cd redis-4.0.6/
  $ make
  ```
- 编译完成后，目录下会出现编译后的redis服务程序 `redis-server`,还有用于测试的客户端程序 `redis-cli`,两个程序位于安装目录 src 目录下。
- 进入到src目录下面，启动redis服务，注意此时redis还不能后台运行，按`ctrl+c`就会结束服务。
  ```
  $ cd src
  $ ./redis-server
  ```
  ![启动redis服务](http://image.leeyom.top/blog/180124/cmb3jcC32F.png)
- 默认情况，Redis不是在后台运行，我们需要把redis放在后台运行，编辑redis的配置文件`redis.conf`，该配置文件路径在 `redis-4.0.6/` 目录下。
  ```
  $ vim redis.conf
  将daemonize的值改为yes
  ```
- 默认情况下，Redis默认的连接密码为null，但是为了安全我们需要设置一个密码。
  ```
  $ vim redis.conf
  将requirepass前面的注释去掉，然后设置密码，比如设置为root
  ```
- 默认情况下 redis 服务只能本机访问，将 `redis.conf` 配置文件中的 `bind 127.0.0.1` 注释掉，不注释掉的话，只能本机访问，其他的IP将无法访问 reids。
  ```
  # IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
  # JUST COMMENT THE FOLLOWING LINE.
  # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  # bind 127.0.0.1
  ```
- 启动redis后台服务。
  ```
  $ /usr/local/redis-4.0.6/src/redis-server /usr/local/redis-4.0.6/redis.conf
  ```
  打印如下内容，启动成功：
  ```
  $ oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
  $ Redis version=4.0.6, bits=64, commit=00000000, modified=0, pid=25640, just started
  $ Configuration loaded
  ```
  最好检测下redis进程是否已经启动，若出现redis-server等关键字，说明服务启动成功。
  ```
  $ ps -ef | grep redis
  ```
- 设置 redis 开机启动：
  首先将 `redis-4.0.6/utils` 目录下的 `redis_init_script` 脚本拷贝到 `/etc/init.d` 下修改名字为 redis ：
  ```
  $ cp redis_init_script /etc/init.d/redis
  ```
  然后编辑刚刚拷贝的文件：
  ```
  $ vim /etc/init.d/redis
  ```
  将一段注释加到文件的头部：
  ![20180419152412426675410.png](http://image.leeyom.top/20180419152412426675410.png)
  同时还要注意 redis 客户端和服务端的路径问题，改成你自己的即可：
  ![20180419152412482277272.png](http://image.leeyom.top/20180419152412482277272.png)
  接着拷贝 `redis.conf` 文件到 `/etc/redis` 目录下，并重命名为 `6379.conf`:
  ```
  $ mkdir /etc/redis
  $ cp redis.conf /etc/redis/6379.conf
  ```
  设置读写权限并设置开机自启：
  ```
  $ chmod +x /etc/init.d/redis
  $ chkconfig redis on
  ```
  这样，以后便可以开机自启动 redis 服务了。
- 停止Redis服务。
  如果有启动过redis客户端，执行如下的命令：
  ```
  $ /usr/local/redis-4.0.6/src/redis-cli shutdown
  ```
  或者直接暴力一点：
  ```
  $ pkill redis-server
  ```
- 这个就是整个redis安装的过程，以及需要注意的地方。

## 安装搜狗输入法
- linux版本为Ubuntu 16.04。
- 去[http://pinyin.sogou.com/linux/](http://pinyin.sogou.com/linux/)下载安装包。
- 添加源：`sudo add-apt-repository ppa:fcitx-team/nightly`
- 然后更新：`sudo apt-get update`
- 开始安装`fcitx`
  - 执行命令：`sudo apt-get install fcitx`
  - 出现错误，执行：`sudo apt-get -f install`
  - 然后再次执行：`sudo apt-get install fcitx`
- 安装fcitx的配置工具：`sudo apt-get install fcitx-config-gtk`
- 安装fcitx的table-all包：`sudo apt-get install fcitx-table-all`
- 安装im-switch工具：`sudo apt-get install im-switch`
- 进入到搜狗输入法安装包所在文件夹，执行：`sudo dpkg -i sougoupingyin_2.1.0.deb`
- 最后注销，重启，系统设置-->语言支持，将键盘输入法系统设置为fcitx。
- 搜索出fcitx配置，将sogou输入法设为默认即可。

## Tomcat端口占用问题

1. linux环境下解决tomcat 8080端口被占用的方法：
  - 先查看是否有tomcat在运行，执行命令：`ps -ef |grep tomcat `，若出现如下的内容，说明tomcat正在运行
    ```
    [root@1 ocalhost bin] # ps -ef |grep tomcat
    root 51379 10 Nov29 pts/0 OO:03: 03
    /usr/java/jdk1.8.0_121/bin/java-Djava.uti]; 1 oggi ng. Config. Fi le=/opt/tomcat /conf/ logging.
    Properties -Djava. Uti1. Logging. Manager=org. Apache. Ju1 i. Clas sLoaderLogManager-djdk. T1 s.
    EphemeralDHKeySi ze=2048-Djava. Protoco1. Handler. Pkgs=org. Apache. Catal in a.
    Webresources-classpath /opt/tomcat /bin/bootstrap.jar: /opt/tomcat /bin/tomcat-ju1 i.jar.-Dcata] ina.
    base=/opt/tomcat-Dcata lina.home=/opt/tomcat-Djava.iO.tmpdi r=/opt/ton cat/temp org. Apache. Cata
    lina. Startup. Bootstrap start root 71250118237 0 11: 39 pts/0 OO: OO:00 grep--color=autotomcat    
    ```
  - 输入命令：`netstat -anp|grep 8080`，找到这个端口对应的进程(PID)
    ```
    [root@loca1 host bin] # netstat -anp|grep 8080
    tcp6 0 0:::8080 LISTEN 51379/java    
    ```
  - 杀死该进程：`kill -9 51379`

2. windows下解决tomcat 8080端口被占用的方法：
  - 输入命令：`netstat -aon|findstr "8080"`，找到了这个端口对应的进程(PID)，如下所示：
    ```
    C: \Users\lwang> netstat -aon|findstr "8080"

    TCP O. O. O. O:8080 O. O. O. O:0 LISTENING 12956

    TCP  [::]:8080  [::]:0 LISTENING 12956    
    ```
  - 查看什么应用占用了该端口，输入命令：`tasklist|findstr "12956"`，如下所示：
    ```
    C: \Users\lwang> tasklist|findstr "12956"

    javaw. Exe 12956 Console 286,212 K  
    ```
  - 杀死该进程，输入命令：`taskkill /pid 12956 /f`，如下图所示：
    ```
    C: \Users\lwang> taskkill /pid 12956 /f
    成功：已终止 PID 为 12956 的进程。    
    ```

## 主题美化

ubuntu自带的主题感觉很丑，所以准备换个扁平化的主题-`Flatabulous`。安装步骤如下：

- 安装Unity 图形化管理工具-`Unity Tweak Tool`，安装命令：
    - `sudo apt-get install unity-tweak-tool`。
- 安装`Flatabulous`主题，并按顺序执行如下的命令：
    - `sudo add-apt-repository ppa:noobslab/themes`
    - `sudo apt-get update`
    - `sudo apt-get install flatabulous-theme`
- 安装该主题配套的图标，按顺序执行如下的命令：
    - `sudo add-apt-repository ppa:noobslab/icons`
    - `sudo apt-get update`
    - `sudo apt-get install ultra-flat-icons`

安装完成后，打开`unity-tweak-tool`，修改主题和图标：

- 点击`Theme`，选择`Flatabulous`。
- 点击`Icons`，选择`Ultra-flat`。
- 最后重启系统。

最终效果图如下：

<p align="center"><img src="http://image.leeyom.top/20171112151049907266113.png" width="80%" height="80%"></p>

## 安装JDK

基本步骤如下：

- linux版本为Ubuntu 16.04。
- [下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)。
- 安装版本为：`jdk-8u151-linux-x64.tar.gz`。
- 我习惯将软件安装到`/usr/local/develop-tools/`目录，方便管理，如果没有`develop-tools`目录，先创建该目录。
- 进入到源码包所在的文件夹，将源码包解压到`/usr/local/develop-tools/`目录下。
    - `sudo tar -zxvf jdk-8u151-linux-x64.tar.gz -C /usr/local/develop-tools/`
- 编辑全局环境变量：
    - 编辑配置文件：`sudo vim /etc/profile`
    - 在该文件的最尾巴，添加下面内容：
      ```xml
      # JDK
      JAVA_HOME=/usr/local/develop-tools/jdk1.8.0_151
      JRE_HOME=$JAVA_HOME/jre
      PATH=$PATH:$JAVA_HOME/bin
      CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
      export JAVA_HOME
      export JRE_HOME
      export PATH
      export CLASSPATH  
      ```
    - 执行命令，刷新该配置：`source /etc/profile`
    - 检查JDK是否生效：`java -version`

## 安装zookeeper

详情见：[zookeeper安装教程](http://leeyom.top/2017/11/08/zookeeper-install/)。

## VMware安装ubuntu问题

### VMware tools 无法安装问题

使用VMware Workstation安装Ubuntu 16.04，`安装VMware Tools`选项按钮为灰色，如果不安装`VMware Tools`，则会出现以下几个问题：

- 物理机无法向直接向虚拟机复制和粘贴。
- 虚拟机的分辨率无法进行自动适配。

解决方法：

1. 先关闭虚拟机。
2. 点击`编辑虚拟机设置`，添加`CD/DVD`驱动器。
3. 接下来，驱动器介质，选择`使用ISO映像(M)`。
4. 选择ISO映像地址，这个ISO映像就是`VMware Workstation`安装根目录下面的那个`linux.iso`，别选错了！
5. 点击确定后，然后重启Ubuntu虚拟机，这个时候，会在左侧的任务栏看到一个CD/DVD，点击打开，将`VMwareTools-9.9.2-2496486.tar.gz`源码包拷贝到桌面，并解压。
6. 进入到解压后的文件夹内，执行`sudo ./vmware-install.pl`，安装VMware Tools。
7. 中间会有一些询问项，一路回车即可。
8. 安装完成后，重启系统即可。

### Ubuntu登陆后花屏

**虚拟机 --> 设置 --> 显示器 -->加速3D图形加速前面的勾去掉就可以了**，降低显卡的的负担。

### 显示器分辨率问题

ubuntu登录后报错：`could not apply the stored configuration for monitors`，意思是无法将存储的配置应用于当前的显示器。解决办法就是：

- 终端执行`sudo rm -f ~/.config/monitors.xml `，移除掉之前的显示器配置文件即可，之后会自动生成当前分辨率分配置文件。

## 安装SSH

- linux版本为Ubuntu 16.04。

为了在windows平台使用`SecureCRT`和`Xshell`等终端工具连接ubuntu等linux服务器，linux服务器需要安装`SSH`，执行如下的命令：

- `sudo apt-get install openssh-server`

确认ssh server是否启动，执行如下命令：

- `ps -e | grep ssh`

如果只有`ssh-agent`，那`ssh-server`还没有启动，需要执行`/etc/init.d/ssh start`，如果看到`sshd`那说明`ssh-server`已经启动了。

`ssh-server`配置文件位于`/etc/ssh/sshd_config`，在这里可以定义SSH的服务端口，默认端口是`22`，你可以自己定义成其他端口号，如222。然后重启SSH服务：

- `sudo /etc/init.d/ssh resart`
