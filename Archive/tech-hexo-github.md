---
title: hexo+github搭建个人博客网站
date: 2016-09-25 20:41:19
comments: true
categories:
- 资源
tags:
- hexo
---

之前一直想搭建一个自己的个人博客，用来记录一些自己的个技术笔记，但是因为种种原因被耽搁了，趁周末有时间，终于把博客搭建好了，下面记录一下整个搭建的过程，也给想搭建博客的同学一个参考。

<!-- more -->

## 环境准备

- [node.js](https://nodejs.org/en/):因为 hexo 整个博客框架是基于node.js的，所以必须安装node.js环境。我安装的是最新的版本，v4.5.0，安装过程一路 next 即可。
- [git客户端](https://git-scm.com/downloads/):用来将hexo相关文件提交到github上去，安装过程一路next。

## 安装hexo框架

环境准备好以后，我们便可以安装hexo博客框架。

桌面鼠标右击，选择Git Bash Here ，打开git命令行窗口，如下图

![Git Bash Here](/img/gitbash.png)

输入安装命令，回车

- `npm install -g hexo `

然后在指定的盘新建一个hexo文件夹，进入到新建的这个文件夹内，点击鼠标右键，选择Git Bash Here，输入初始化指令，回车，就会看到生成一系列的文件：

- `hexo init`

安装相关的依赖包，输入下面的指令，回车

- `npm install`

下面来解说一下各个文件夹的作用，指定文件夹的目录如下：

![hexo文件目录](/img/direction.png)

- `_config.yml`：用来配置站点信息，大多数配置都在这里进行。
- `package.json`：管理我们安装的一些插件，我们要删除某些插件的时候，可以在这里进行。
- `scaffolds`：模板文件夹。
- `source`：源文件夹是存放用户资源的地方,他下面有一个_posts，文件夹，我们发表的文件就存放在这个文件夹下。
- `themes`：主题文件夹，Hexo 会根据主题来生成静态页面。

接着在hexo文件夹下执行：

- `hexo g`
- `hexo s`

然后用浏览器访问[http://localhost:4000](http://localhost:4000)，就能看到hexo初始化界面，是不是很激动呢？但是这个界面只有我们自己可以看到，要想别人看到怎么办呢？ok，接下来我们就来完成这一步骤。

## 创建一个github账号

已经有账号的暂时忽略这一步，传送门：[github](https://github.com/)，没有的话就老老实实创建一个吧。

## 创建一个仓库

我们有了账号以后，便可以创建repository，中文即是仓库，如下图所示

![仓库](/img/repository.png)

这里仓库的名字要注意，格式应该是：**你的github账号名字.github.io**，像我的就是 wangleeyom.github.io。为什么要这么设置，我也不太清楚，可能是github约定的吧

## 部署文件到github

用notepad++或者sublime打开 hexo 文件夹下的 `_config.yml` 文件，找到关键字 deploy ，然后修改成如下，我就用我自己的做示例：

```
deploy:
  type: git
  repository: https://github.com/wangleeyom/wangleeyom.github.io.git
  branch: master

```

需要注意的一点：**冒号后面要有一个空格**，比如 type 冒号后面要加一个空格，很多同学部署不上去就是问题出在这里，需要注意，修改完以后，保存即可。

但是这样还不能连接到 github ，我们还需要配置SSH，找到路径`C:\Users\leeyom\.ssh`，如果已经存在SSH Keys ，直接删除`.ssh` 文件夹下的所有的文件，如下图。

![SSH Keys](/img/sshKeys.png)

然后输入以下指令

```
ssh-keygen -t rsa -C "635709492@qq.com"
```

这个邮箱是我们当初注册github的时候填写的邮箱，然后回车，需要回车三次。最后出现如下的结果

![keygen](/img/keygen.png)


>图片引用自http://www.jianshu.com/p/ba76165ca84d

然后再输入指令

```
ssh-agent -s
```
继续输入

```
ssh-add ~/.ssh/id_rsa
```

到这里后，可能会出现如下的错误，我的就出现如下的错误：

![mistake](/img/mistake.png)

>图片引用自http://www.jianshu.com/p/ba76165ca84d

如果你出现了上面的错误，不要着急，输入下面的指令即可以解决，如果没有，则跳过这步。

```
eval `ssh-agent -s`
ssh-add
```

![添加SSH key到Github](/img/keysuccess.png)

>图片引用自http://www.jianshu.com/p/ba76165ca84d

接着拷贝key，执行如下命令：

```
clip < ~/.ssh/id_rsa.pub
```

打开github，点击setting，然后再点击SSH and GPG keys，如下图：

![](/img/sshSetting.png)

接下来

![](/img/SSHkeys2.png)

测试一下，输入如下命令，然后会让你输入yes/no，你输入yes即可。以上ssh就配好了，我们就可以将项目部署到github上。

```
ssh -T git@github.com
```

执行以下命令部署项目到github

```
hexo g
hexo d
```

但是输入hexo d 可能会报 **ERROR Deployer not fount： git** 错误,这是因为没有安装`hexo-deployer-git`这个模块，无法识别该指令，安装该模块即可，输入如下命令：

```
npm install hexo-deployer-git --save
```

可能网速的原因，安装该模块的时候，很慢，需要等待一会儿。我安装的时候一直没动静，以为挂掉了，试了几次都不行。后来出去有事，就执行命令后就不管他了，回来后，就安装好了。
安装好以后输入 hexo d ，会有弹出框，输入github账号密码即可，就可以访问了。浏览器输入 `wangleeyom.github.io` 即可访问，烦人的404没有了，出现的是 hexo 默认界面。

## 备注

假如一直出现如下图的错误，一直无法发布到github上

![](/img/mistake_2.png)

尝试将站点配置_config.yml的deploy部分更改成:

```
deploy:
  type: git
  repository: git://github.com/wangleeyom/wangleeyom.github.io.git
  branch: master

```

也就是把https换成git试试。

## 后记

至此，hexo 的搭建就完成了，是不是很激动呢？别着急，界面是不是很丑？咋发表文章呢？哈哈。。后面有时间的话会讲解如何发布文章，添加tags，分类，更换主题，如何绑定域名，如何接入第三方的服务等一系列的问题，敬请期待。。。

## 参考

* 本文章很大一部分参考的 http://www.jianshu.com/p/ba76165ca84d ，感谢该作者。
* 潘柏信：http://www.jianshu.com/p/465830080ea9
* next主题官网：http://theme-next.iissnan.com/
* hexo 中文网：https://hexo.io/zh-cn/

## 附录

常用的指令

```
hexo g #完整命令为hexo generate,用于生成静态文件
hexo s #完整命令为hexo server,用于启动服务器，主要用来本地预览
hexo d #完整命令为hexo deploy,用于将本地文件发布到github上
hexo n #完整命令为hexo new,用于新建一篇文章
```
