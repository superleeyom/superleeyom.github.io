---
title: Docker初探
date: 2017-12-14 21:30:03
comments: true
tags:
- Docker
categories:
- 编程
---

Docker是一个开源的引擎，可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、bare metal、OpenStack 集群和其他的基础应用平台。最近准备着手搭建一套微服务框架，其中准备用Docker进行容器化部署，所以就补习下Docker方面的知识。


<!-- more -->

# 什么是Docker

[Docker](https://github.com/moby/moby)目前在 github 上差不多四万六千多个 star，可见其火爆程度。那 Docker 到底是什么呢？Docker 是由 go 语言编写的，基于 Linux 内核，对进程进行封装隔离，属于属于操作系统层面的虚拟化技术。Docker 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。使得 Docker 技术比虚拟机技术更为轻便、快捷。

# 为什么要使用Docker

当我们使用微服务架构后，我们将原本一个系统，按照业务拆分成多个子系统，而这多个子系统，都是部署在独立的环境中，互相隔离。在没有Docker出现之前，我们是通过虚拟机的方式部署多个子系统，那样是非常的消耗计算机资源。而有了Docker，同样相同配置的计算机，我们可以部署更多的应用。并且Docker启动的速度非常的快，基本是秒级和毫秒级别的，而虚拟机启动的速度，大家想想就知道了。

# 基本概念

Docker有三个比较重要的概念，分别是：

1. 容器（Container）
2. 镜像（Image）
3. 仓库（Repository）

## 镜像
> 我们都知道，操作系统分为内核和用户空间。对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系 统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu 16.04 最小系统的 root 文件 系统。
> Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文 件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像 不包含任何动态数据，其内容在构建之后也不会被改变。

我所理解的Docker镜像，就是一个小型的操作系统，只是这个操作系统没有我们日常使用的系统比如Windows这么庞大，他提供了应用运行的最小环境。

## 容器
> 镜像（ Image ）和容器（ Container ）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
> 容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的命名空间。因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。也因为这种隔离的特性，很多人初学 Docker 时常常会混淆容器和虚拟机。

## 仓库
> 镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。
> 一个 Docker Registry 中可以包含多个仓库（Repository），每个仓库可以包含多个标签，每个标签对应一个镜像。
> 通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 <仓库名>:<标签> 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 latest 作为默认标签。

# 安装Docker

由于我使用的机器是mac，所以就以mac为例。

从[Docker官网](https://download.docker.com/mac/stable/Docker.dmg)官网下载对应的dmg文件，然后打开将其拖到`Application`文件夹，中间会输入用户名密码，这样就安装完成了，貌似有点简单噢。

<p align="center">
    <image src="http://image.leeyom.top/20171214151326247025319.jpg"/>
</p>

Docker这个LOGO还是蛮可爱的，一个海豚背上驮着一个集装箱。接下来点击Docker，运行Docker。运行终端，查看Docker是否启动，终端分别执行`docker --version`、`docker-compose --version`、`docker-machine --version`命令，若打印如下的内容，代表Docker是安装成功的。

<p align="center">
    <image src="http://image.leeyom.top/20171214151326378432015.jpg"/>
</p>

尝试创建一个Nginx服务器，终端输入命令：

```
docker run -d -p 80:80 --name webserver nginx
```
浏览器访问：[http://localhost/](http://localhost/)，页面出现`Welcome to nginx!`，说明Docker在mac上是安装成功的。如果镜像拉取非常非常慢，请先尝试切换镜像源：[镜像加速](#mirrorSpeed)。

要停止 Nginx 服务器并删除执行下面的命令：

```
docker stop webserver
docker rm webserver
```

# 使用镜像

##  <span id="mirrorSpeed">镜像加速</span>

由于国内的网络问题，拉取镜像是非常慢，则需要配置国内的镜像加速，这里使用的是Docker官方提供的中国的镜像地址：`https://registry.docker-cn.com/`。mac系统，在任务栏点击 Docker for mac 应用图标 -> Perferences... -> Daemon -> Registry mirrors。在列表中填写加速器地址即可。修改完成之后，点击 Apply & Restart 按钮，Docker 就会重启并应用配置的镜像地址了。验证是否启用了该镜像地址，终端输入：`docker info`，若看到如下内容，说明是配置成功的。

<p align="center">
    <image src="http://image.leeyom.top/20171214151326501334897.png"/>
</p>

## 获取镜像

获取镜像的命令是：`docker pull`，其命令格式为：

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

- Docker Registry 地址：地址格式一般是 <域名/IP>:[:端口号]。默认的地址是Docker Hub。
- 仓库名：即<用户名>/<软件名>，对于Docker Hub，如果不给出用户名，则默认是`library`，也就是官方镜像。

举个栗子：

```
$ docker pull ubuntu:16.04
16.04: Pulling from library/ubuntu
bf5d46315322: Pull complete
9f13e0ac480c: Pull complete
e8988b5b3097: Pull complete
40af181810e7: Pull complete
e6f7c7e5c03e: Pull complete
Digest: sha256:147913621d9cdea08853f6ba9116c2e27a3ceffecf3b492983ae97c3d643fbbe
Status: Downloaded newer image for ubuntu:16.04
```

上面的命令没有给出Docker镜像仓库的地址，因此会从Docker Hub获取镜像。镜像名称为`ubuntu:16.04`，因此将会获取官方镜像 library/ubuntu 仓库中标签为 16.04 的镜像。

运行镜像里面的容器的命令是：`docker run`，比如在上面刚下载的ubuntu:16.04这个镜像里启动bash进行交互操作，执行如下的命令：

```
$ docker run -it --rm \
ubuntu:16.04 \
bash
```

## 列出镜像

使用命令`docker image ls`。

<p align="center">
    <image src="http://image.leeyom.top/20171214151326684081409.png"/>
</p>

列表包含了仓库名、标签、镜像ID、创建时间、占用空间。

其他列出镜像的指令：

- `docker image ls -a`：列出所有的镜像，包括中间镜像。
- `docker image ls nginx`：列出部分镜像，这里是列出nginx相关的镜像。

## 定制镜像

定制镜像有个重要的脚本：`Dockerfile`，这个脚本主要用来构建、定制镜像。下面用`Dockerfile`脚本来定制一个nginx镜像。

1. 创建一个目录：`mynginx`
2. 进入到该目录下面，在该目录下面创建一个文件名为`Dockerfile`的文件，终端输入命令：`touch Dockerfile`
3. 打开`Dockerfile`文件，添加如下的脚本：
   ```
   FROM nginx
   RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
   ```
    - `FROM` 用于指定基础镜像。
    - `RUN` 用于执行命令。
4. 开始构建镜像，在`Dockerfile`文件所在目录，也就是文件夹`mynginx`下面，终端执行：`docker build -t nginx:v2 .`，后面有个点，代表上下文路径。
    - 镜像构建命令：`docker build [选项] <上下文路径/URL/->`
    - 这里我们指定镜像名称为`nginx:v2`，通过命令：`docker run -d -p 80:80 --name webserver nginx:v2`即可运行该镜像，访问http://localhost/，发现nginx默认启动的页面已经被修改。

5. Dockerfile指令
    - COPY：复制文件。
    - ADD：更高级的复制指令，如果`源路径`为一个`tar`压缩文件的话，压缩格式为`gzip`，`bzip2`以及`xz`的情况下，`ADD`指令将会自动解压缩这个压缩文件到`目标路径`去。在 COPY 和 ADD 指令中选择的时候，可以遵循这样的原则，所有的文件复制均使用 COPY 指令，仅在需要自动解压缩的场合使用 ADD 。
    - CMD：容器启动命令，指定容器启动程序及参数。
    - ENTRYPOINT：入口点


