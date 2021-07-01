---
title: 在centOS上安装nginx教程
date: 2016-12-08 21:56:13
comments: true
categories:
- 资源
tags:
- nginx
---

是一个使用c语言开发的高性能的http服务器及反向代理服务器。Nginx是一款高性能的http 服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器。由俄罗斯的程序设计师Igor Sysoev所开发，官方测试nginx能够支支撑5万并发链接，并且cpu、内存等资源消耗却非常低，运行非常稳定。最近项目中用到了反向代理服务器 nginx，没用过这玩意儿，自己就尝试着在虚拟机中安装nginx，下面就把整个安装过程中遇到的问题以及安装过程记录一下。

<!-- more -->

## 应用场景

1. http服务器。Nginx是一个http服务可以独立提供http服务。可以做网页静态服务器。
2. 虚拟主机。可以实现在一台服务器虚拟出多个网站。例如个人网站使用的虚拟主机。
3. 反向代理，负载均衡。当网站的访问量达到一定程度后，单台服务器不能满足用户的请求时，需要用多台服务器集群可以使用nginx做反向代理。并且多台服务器可以平均分担负载，不会因为某台服务器负载高宕机而某台服务器闲置的情况。



## 环境

* 虚拟机安装的是centOS 6.8
* nginx-1.8.0.tar.gz
* 本机是mac OSX 10.11

## 下载

进入 http://nginx.org/en/download.html 可进行下载

## 安装nginx依赖的包

### gcc

因为nginx是用c语言开发的，所以我们需要用gcc对我们下载下来的源码进行编译，在安装之前先查看本机是否已经安装了gcc，终端输入**gcc -v**，如果显示gcc版本号，说明本机是已经安装好gcc的，就没必要安装了，如果没有显示gcc对应的版本号，在本机联网的情况下，终端输入指令：**yum install gcc-c++**，即可安装。

### PCRE

PCRE(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。nginx的http模块使用pcre来解析正则表达式，所以需要在linux上安装pcre库。安装指令：**yum install -y pcre pcre-devel**

### zlib

zlib库提供了很多种压缩和解压缩的方式，nginx使用zlib对http包的内容进行gzip，所以需要在linux上安装zlib库。安装指令：**yum install -y zlib zlib-devel**

### openssl

OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库。安装指令：**yum install -y openssl openssl-devel**

## 安装步骤

1. 用iTerm2 ssh 远程连接终端，输入 **ssh -p 22 root@10.211.55.6**

2. 将nginx源码上传到centOS

 ![2016120810700QQ20161208-0@2x.png](http://image.leeyom.top/2016120810700QQ20161208-0@2x.png)

 > 假如我们是root用户登录，那么上传的文件所在的文件夹在“/root“目录下，如果是普通用户的话，上传的文件夹是“/home/用户名/“下面。

3. 将压缩包解压缩，执行命令：**tar -zxvf nginx-1.8.0.tar.gz**

4. 进行configure配置，终端输入：

 ```
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi
 ```
 > <span style="color:red">注意：上边将临时文件目录指定为/var/temp/nginx，需要在/var下创建temp及nginx目录，否则就回报错</span> 	

5. 执行make指令

6. 执行make install

## nginx启动与停止

1. 启动：进入到/usr/local/nginx/sbin，终端输入 ./nginx 就可以启动。浏览器访问 输入linux 主机 ip出现如下图，说明配置安装成功。

 ![2016120852810QQ20161208-1@2x.png](http://image.leeyom.top/2016120852810QQ20161208-1@2x.png)

 中间有遇到过一个小问题，就是在centOS上本机可以访问，但是我在mac机器上却无法访问，后来上网寻找到解决办法，是centOS防火墙的原因，执行如下的命令，即可解决.

 ![2016120855490QQ20161208-2@2x.png](http://image.leeyom.top/2016120855490QQ20161208-2@2x.png)

2. 关闭nginx：在sbin目录下执行命令：**./nginx -s stop**

3. 刷新配置：在sbin目录下执行命令：**./nginx -s reload**

## Niginx的配置

在`/usr/local/nginx/conf/`的`nginx.conf`文件便是nginx的配置文件

![2016121586546nginxConf.png](http://image.leeyom.top/2016121586546nginxConf.png)

## 使用nginx配置虚拟机

### 通过端口区分虚拟机

在`nginx.conf`添加一个server节点，如下所示：

```
server {  
        listen   81;  
        server_name  localhost;  
        #charset koi8-r;  
        #access_log  logs/host.access.log  main;  
        location / {  
        root   html81;  
        index  index.html index.htm;  
        }  
       }
```


添加后刷新配置文件，进入到sbin目录，执行`./nginx -s reload`，配置文件就会生效。
浏览器访问`http://ip地址:81`，就可以访问到81端口的资源内容。

### 通过域名区分虚拟主机

#### 通过域名如何访问web服务器

1. 原理

 ![2016121520476dns.png](http://image.leeyom.top/2016121520476dns.png)

 借助软件做个测试，修改本机的host，模拟不同的域名指向同一个端口，mac平台修改host软件有[iHosts](https://h.ihosts.toolinbox.net/cn/)，win平台下面有[switchHosts](http://www.appinn.com/switchhosts/)

 ![2016121556448QQ20161215-205921@2x.png](http://image.leeyom.top/2016121556448QQ20161215-205921@2x.png)

 这样在浏览器输入`test.taotao.com`、`test2.taotao.com`、`test3.taotao.com`，都会访问linux 主机80端口。

2. 配置基于域名的虚拟主机

 ```
 server {  
 		listen       80;  
 		server_name  域名;  

 		#charset koi8-r;  

 		#access_log  logs/host.access.log  main;  

 		location / {  
 			root   html-test3;  
 			index  index.html index.htm;  
 			}  
 		}
 ```

 修改后要重新刷新`ngin.conf`配置文件。

## 总结

到此，教程就结束啦，整个过程还算比较顺利，也没遇到啥棘手的问题，特此记录，方便日后查看。

