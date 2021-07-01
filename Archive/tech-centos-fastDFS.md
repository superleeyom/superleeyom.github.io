---
title: centOS上搭建FastDFS图片服务器教程
date: 2016-12-22 22:27:06
comments: true
categories:
- 资源
tags:
- FastDFS
---

FastDFS是用c语言编写的一款开源的分布式文件系统。FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

<!-- more -->

![20161223801021.png](http://og1m51u2s.bkt.clouddn.com/20161223801021.png)

## 文件上传流程

![20161223932372.png](http://og1m51u2s.bkt.clouddn.com/20161223932372.png)

## 文件下载流程

![20161223635383.png](http://og1m51u2s.bkt.clouddn.com/20161223635383.png)## 上传文件的文件名

客户端上传文件后存储服务器将文件ID返回给客户端，此文件ID用于以后访问该文件的索引信息。文件索引信息包括：组名，虚拟磁盘路径，数据两级目录，文件名。

![20161223506684.png](http://og1m51u2s.bkt.clouddn.com/20161223506684.png)

* 组名：文件上传后所在的storage组名称，在文件上传成功后有storage服务器返回，需要客户端自行保存。
* 虚拟磁盘路径：storage配置的虚拟路径，与磁盘选项store_path*对应。如果配置了store_path0则是M00，如果配置了store_path1则是M01，以此类推。
* 数据两级目录：storage服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。* 文件名：与文件上传时不同。是由存储服务器根据特定信息生成，文件名包含：源存储服务器IP地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。

## FastDFS搭建

![20161223488205.png](http://og1m51u2s.bkt.clouddn.com/20161223488205.png)

可以使用一台虚拟机来模拟，只有一个Tracker、一个Storage服务。配置nginx访问图片。

### 搭建步骤

1. 第一步：把fastDFS源码上传到linux系统。
2. 第二步：安装FastDFS之前，先安装libevent工具包。命令：`yum -y install libevent`。
3. 第三步：安装libfastcommonV1.0.7工具包。
 * 将libfastcommonV1.0.7源码解压缩
 * `./make.sh`
 * `./make.sh install`
 * 把`/usr/lib64/libfastcommon.so`文件向`/usr/lib/`下复制一份

4. 第四步：安装Tracker服务。
 * fastDFS源码解压缩
 * `./make.sh`
 * `./make.sh install`
 * 把`/root/FastDFS/conf`目录下的所有的配置文件都复制到`/etc/fdfs`下。
 * 配置`tracker`服务。修改`/etc/fdfs/tracker.conf`文件。配置`tracker`日志文件路径。
 ![20161223411366.png](http://og1m51u2s.bkt.clouddn.com/20161223411366.png)

 * 启动tracker，执行命令：`/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf`，重启命令：`/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart`。

5. 第五步：安装storage服务。
  * 如果storage是按照在其他的服务器上面，第四步的1~4需要重新执行。
  * 配置storage服务。修改`/etc/fdfs/storage.conf`文件。
 ![20161223669197.png](http://og1m51u2s.bkt.clouddn.com/20161223669197.png)

 ![201612234318.png](http://og1m51u2s.bkt.clouddn.com/201612234318.png)

 ![20161223979359.png](http://og1m51u2s.bkt.clouddn.com/20161223979359.png)

 * 启动storage服务。`/usr/bin/fdfs_storaged /etc/fdfs/storage.conf`

6. 第六步：测试服务。修改客户端配置文件`/etc/fdfs/client.conf`。

 ![201612231068010.png](http://og1m51u2s.bkt.clouddn.com/201612231068010.png)

 ![201612235555411.png](http://og1m51u2s.bkt.clouddn.com/201612235555411.png)

 执行命令：`/usr/bin/fdfs_test /etc/fdfs/client.conf upload anti-steal.jpg`

7. 第七步：搭建nginx提供http服务

 使用官方提供的nginx插件，`fastdfs-nginx-module_v1.16.tar.gz`，添加该插件后，nginx则需要重新编译。

 * 将源码包上传到root根目录，然后解压缩。
 * 修改/root/fastdfs-nginx-module/src/config文件，把其中的local去掉。	![2017011282674nginxModule.png](http://og1m51u2s.bkt.clouddn.com/2017011282674nginxModule.png)

 * 对nginx重新config，进入到`/root/nginx-1.8.0/`，在终端执行如下代码：

 ![2017011223778QQ20170112-225927@2x.png](http://og1m51u2s.bkt.clouddn.com/2017011223778QQ20170112-225927@2x.png)

 > 不晓得是markdown格式不对还是怎么，这贴上去老是打乱整个文章格式，无奈只能截图。

 * `make`
 * `make install`
 * 把/root/fastdfs-nginx-module/src/mod_fastdfs.conf文件复制到/etc/fdfs目录下。并编辑该文件，修改如下几个地方：

 ![2017011266710nginxModule1.png](http://og1m51u2s.bkt.clouddn.com/2017011266710nginxModule1.png)

 ![2017011232614nginxModule2.png](http://og1m51u2s.bkt.clouddn.com/2017011232614nginxModule2.png)

 ![2017011279200nginxModule3.png](http://og1m51u2s.bkt.clouddn.com/2017011279200nginxModule3.png)

 ![2017011257498nginxModule4.png](http://og1m51u2s.bkt.clouddn.com/2017011257498nginxModule4.png)

 * 修改`/usr/local/nginx/conf/nginx.conf`nginx配置文件，修改server节点，如下所示

 ```
 server {
        listen       80;
        server_name  localhost;

        location /group1/M00/{
                ngx_fastdfs_module;
        }
}
 ```

 * 将libfdfsclient.so拷贝至/usr/lib下，执行命令：`cp /usr/lib64/libfdfsclient.so /usr/lib/`

 * 启动nginx,启动tracker，启动storage。

8. 第八步：测试

`cd /etc/fdfs/`,然后执行上传命令：`/usr/bin/fdfs_test /etc/fdfs/client.conf upload anti-steal.jpg`，假如出现如下的图，说明是上传成功的。

![2017011214771QQ20170112-225034@2x.png](http://og1m51u2s.bkt.clouddn.com/2017011214771QQ20170112-225034@2x.png)

然后再浏览器访问生成的链接，假如能访问到我们刚上传的图片，就说明完整搭建好FastDFS。中间我一直上传不了，后来才发现自己没有启动tracker和storage，这里需要注意一下。

![2017011235411QQ20170112-225350@2x.png](http://og1m51u2s.bkt.clouddn.com/2017011235411QQ20170112-225350@2x.png)

## 文件下载

链接: https://pan.baidu.com/s/1mipA8QG 密码: yd6r
