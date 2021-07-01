---
title: hexo博客进阶教程
date: 2016-09-29 16:10:15
comments: true
categories:
- 资源
tags:
- hexo
---

前几天写了一篇关于如何用hexo+github来搭建个人的博客网站，最近项目验收完成了，终于可以有时间把接下来的教程完成，ok，废话不多说，咱们直接开始吧！

<!-- more -->

## 添加标签页

标签页需要我们自己去新建，默认是不显示的

- 在终端窗口下，进入到 **hexo站点** 文件夹下，输入如下命令：
 	```
	$ hexo new page tags
	```
- 进入到`hexo/source/tags/`目录下，用markdown编辑器打开index.md文件，主题将自动为这个页面显示标签云，显示如下：
	```
	title: 标签
	date: 2014-12-22 12:39:04
	type: "tags"
	---
	```
- 修改主题配置文件，添加tags到menu中，显示如下：
	```
	menu:
  	home: /
  	archives: /archives
  	tags: /tags
	```

## 添加分类页

大致的过程是跟添加标签页是一个意思，这里就不在阐述，[点我](http://theme-next.iissnan.com/theme-settings.html#categories-page)

> 如果有启用 多说 或者 Disqus 评论，页面也会带有评论。 若需要关闭的话，请添加字段 comments 并将值设置为 false

## 留言板

- 打开主题配置文件，即 `Hexo/themes/next/_config.yml` 在 menu 下添加字段，名称任意，只要自己能区分出来就行，比如我添加的是 board 字段，下面全部是以 board 为例。
	```
	menu:
	  home: /
	  categories: /categories
	  about: /about
	  archives: /archives
	  tags: /tags
	  commonweal: /404.html
	  board: /board
	```
	在 menu_icons 下为留言板设定一个图标，我用的是 book 这个图标，如果想要设定为其他图标，请访问：[Font Awesome Icons](https://link.zhihu.com/?target=http%3A//fontawesome.io/icons/)， 然后把自己喜欢的图标后的关键字填写到 board后面。
	```
	menu_icons:
	  enable: true
	  #KeyMapsToMenuItemKey: NameOfTheIconFromFontAwesome
	  home: home
	  about: user
	  categories: th
	  tags: tags
	  archives: archive
	  commonweal: heartbeat
	  board: book
	```

- 进入 `Hexo/source` ，创建一个 board 文件夹

- 打开刚才创建的文件夹，新建一个 index.md 文件

- 打开创建的 index.md，在开头添加
	```
	title: board
	date: 2016-07-07 21:43:11
	comments: ture
	---
	```
- 打开 `Hexo\themes\next\languages` 文件夹，找到当前使用的语言
在 menu 下添加 borad 字段
	```
	menu:
  		home: 首页
  		archives: 归档
  		categories: 分类
  		tags: 标签
  		about: 关于
  		search: Search
  		commonweal: 公益404
  		board: 留言
	```
- 大双引号效果，只需要在`Hexo/source/border/`下编辑`index.md`，添加
	```
	<blockquote class="blockquote-center">你想写的东西</blockquote>
	```


## 更换主题

hexo 默认的主题显得有些单调，那该如何才能更换hexo的主题呢？在这里推荐一款比较简约的主题**next**

- 首先下载[next](https://github.com/iissnan/hexo-theme-next/releases)主题安装包,下载好以后，解压该压缩文件，重命名为next。
- 然后将解压后的整个文件夹复制到`F:\Hexo\themes`目录下。
- 打开站点配置文件(hexo 根目录下的`_config.yml`文件，后面都统一称为 **站点配置文件** )，找到theme字段，将其值改为next。
- 执行命令`hexo -g` , `hexo -s`,在浏览器输入：`http://localhost:4000` 校验主题是否更改成功,如果显示如下的图所示，说明安装成功。

	![主题更换成功](/img/20160929/hexoTheme.png)

- 主题更换成功以后，就可以更改主题的标题、头像、菜单、侧边栏，具体参考[next主题官方文档](http://theme-next.iissnan.com/getting-started.html)，写的很详细，在这里就不详细叙述了。

## 多说评论

hexo 博客本身是不带有评论系统的，但是为了方便博主与读者之间交流，我就需要接入第三方服务----**多说评论**

- 登录[多说](http://duoshuo.com/)后在首页选择 “我要安装”。
- 创建站点，填写表单。多说域名 这一栏填写的即是你的 `duoshuo_shortname`，如图：
	![多说](/img/20160929/duoshuo.png)
- 创建站点完成后在 站点配置文件 中新增 `duoshuo_shortname` 字段，值设置成上一步中的值。
- 在多说的设置界面，可以自定义文本，默认头像，外观主题，下面我贴出自己的评论框的自定义的css样式，给大家一个参考。
	```css
	#ds-reset .ds-avatar img,
	#ds-recent-visitors .ds-avatar img {
	width: 54px;
	height: 54px;     /*设置图像的长和宽，这里要根据自己的评论框情况更改*/
	border-radius: 27px;     /*设置图像圆角效果,在这里我直接设置了超过width/2的像素，即为圆形了*/
	-webkit-border-radius: 27px;     /*圆角效果：兼容webkit浏览器*/
	-moz-border-radius: 27px;
	box-shadow: inset 0 -1px 0 #3333sf;     /*设置图像阴影效果*/
	-webkit-box-shadow: inset 0 -1px 0 #3333sf;
	-webkit-transition: 0.4s;
	-webkit-transition: -webkit-transform 0.4s ease-out;
	transition: transform 0.4s ease-out;     /*变化时间设置为0.4秒(变化动作即为下面的图像旋转360读）*/
	-moz-transition: -moz-transform 0.4s ease-out;
	}

	#ds-reset .ds-avatar img:hover,
	#ds-recent-visitors .ds-avatar img:hover {

	/*设置鼠标悬浮在头像时的CSS样式*/    box-shadow: 0 0 10px #fff;
	rgba(255, 255, 255, .6), inset 0 0 20px rgba(255, 255, 255, 1);
	-webkit-box-shadow: 0 0 10px #fff;
	rgba(255, 255, 255, .6), inset 0 0 20px rgba(255, 255, 255, 1);
	transform: rotateZ(360deg);     /*图像旋转360度*/
	-webkit-transform: rotateZ(360deg);
	-moz-transform: rotateZ(360deg);
	}

	#ds-thread #ds-reset .ds-textarea-wrapper textarea {

	}

	#ds-recent-visitors .ds-avatar {
	float: left
	}
	/*隐藏多说底部版权*/
	#ds-thread #ds-reset .ds-powered-by {
	display: none;
	}
	```

## 文章统计与站点统计

- 站点统计用的是[不蒜子统计](http://service.ibruce.info/)，传送门（[点我](http://theme-next.iissnan.com/third-party-services.html#analytics-busuanzi)），next主题官网写的比较详细，我这里就不在阐述了哦

	![](/img/20160929/不蒜子统计.png)

- 文章统计，再次召唤传送门([点我](https://notes.wanghao.work/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html#))

## 背景效果

- 把 js 文件 [particle.js](https://github.com/ehlxr/ehlxr.github.io/blob/master/js/src/particle.js) 放在`\themes\next\source\js\src`文件目录下
- 更新`\themes\next\layout\_layout.swig`文件，在末尾（在前面引用会出现找不到的bug）添加以下 js 引入代码：
	```
	<script type="text/javascript" src="/js/src/particle.js"></script>
	```

## 站点搜索

- 安装 `hexo-generator-search`，在站点的根目录下执行以下命令：
	```
	$ npm install hexo-generator-search --save
	```
- 在站点配置文件中，新增以下内容到任意位置
	```
	search:
  		path: search.xml
  		field: post
	```

## 头像旋转

主要是修改 Hexo 目录下 `\themes\next\source\css\\_common\components\sidebar\sidebar-author.styl` 文件，以下是我的配置文件，大家可以参考一下。

```css
.site-author-image {
  display: block;
  margin: 0 auto;
  padding: $site-author-image-padding;
  max-width: $site-author-image-width;
  height: $site-author-image-height;
  border: $site-author-image-border-width solid $site-author-image-border-color;

  /* 头像圆形 */
  border-radius: 80px;
  -webkit-border-radius: 80px;
  -moz-border-radius: 80px;
  box-shadow: inset 0 -1px 0 #333sf;

  /* 设置循环动画 [animation: (play)动画名称 (2s)动画播放时长单位秒或微秒 (ase-out)动画播放的速度曲线为以低速结束
    (1s)等待1秒然后开始动画 (1)动画播放次数(infinite为循环播放) ]*/
  -webkit-animation: play 2s ease-out 1s 1;
  -moz-animation: play 2s ease-out 1s 1;
  animation: play 2s ease-out 1s 1;

  /* 鼠标经过头像旋转360度 */
  -webkit-transition: -webkit-transform 1.5s ease-out;
  -moz-transition: -moz-transform 1.5s ease-out;
  transition: transform 1.5s ease-out;
}

img:hover {
  /* 鼠标经过停止头像旋转
  -webkit-animation-play-state:paused;
  animation-play-state:paused;*/

  /* 鼠标经过头像旋转360度 */
  -webkit-transform: rotateZ(360deg);
  -moz-transform: rotateZ(360deg);
  transform: rotateZ(360deg);
}

/* Z 轴旋转动画 */
@-webkit-keyframes play {
  0% {
    -webkit-transform: rotateZ(0deg);
  }
  100% {
    -webkit-transform: rotateZ(-360deg);
  }
}
@-moz-keyframes play {
  0% {
    -moz-transform: rotateZ(0deg);
  }
  100% {
    -moz-transform: rotateZ(-360deg);
  }
}
@keyframes play {
  0% {
    transform: rotateZ(0deg);
  }
  100% {
    transform: rotateZ(-360deg);
  }
}

.site-author-name {
  margin: $site-author-name-margin;
  text-align: $site-author-name-align;
  color: $site-author-name-color;
  font-weight: $site-author-name-weight;
}

.site-description {
  margin-top: $site-description-margin-top;
  text-align: $site-description-align;
  font-size: $site-description-font-size;
  color: $site-description-color;
}
```

## 总结

ok，基本上大致的配置就这么多吧，还有些东西，比如打赏啊，公益404，友情链接，大家都可以在[next主题官网](http://theme-next.iissnan.com/theme-settings.html)可以看到配置教程，在这里就不多说了，有不懂得同学可以留言，我会第一时间回复你的。ok，睡觉去了，好困。。。
