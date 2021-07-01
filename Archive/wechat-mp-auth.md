---
title: 微信公众号前后端分离授权方案
date: 2018-09-16 14:29:11
tags:
- 微信公众号
categories:
- 编程
---

最近在重构公司的微信公众号项目，需要将其重构现在流行的微服务架构，使其前后端分离，分开独立部署，其中需要解决的一个需求就是基于公众号端的登录方案。前端采用 vue 框架，用户在微信客户端中访问第三方网页，公众号可以通过微信网页授权机制，来获取用户基本信息，进而实现业务逻辑，中间就涉及到了前后端分离情况下的微信授权，微信授权涉及到签名和 token 机制，都需要后端与前端协调好，下面就针对此问题的实践方案做一个总结。

<!-- more -->

## 步骤

下面是一个简单的流程图：

<img src="http://image.leeyom.top/blog/wechat-mp-oauth.jpg" title="微信前后端分离授权方案" />

步骤如下：
1. 微信公众号的菜单 url，微信有提供接口，可以通过后台接口生成，生成的 url 如下：
    ```
    https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx55dffe0ea4c7c68a&
    redirect_uri=https%3A%2F%2Fdev.leeyom.top%2Fhome.html%3Fts%3D1536829668153&
    response_type=code&scope=snsapi_userinfo&state=5&connect_redirect=1#wechat_redirect
    ```
    - `redirect_uri`：回调页面，也就是我们自己的客户端页面，这里我们统一的跳转到 home.html 页面。
    - `scope`：应用授权作用域，snsapi_base （不弹出授权页面，直接跳转，只能获取用户openid），snsapi_userinfo （弹出授权页面，可通过 openid 拿到昵称、性别、所在地。并且，即使在未关注的情况下，只要用户授权，也能获取其信息，如果已关注，那就是静默授权，也就是说不会弹出授权框）。
    - `state`：重定向后会带上 state 参数，这个参数后端跟前端约定好，比如 1 跳转到订单页面，2 跳转到注册页面等等，前端根据此参数进行路由。
2. 微信回调到指定的页面后，会在 url 的后面携带 code 参数，前端通过这个 code 码调用后端接口，去换取用户的信息（openid、昵称、地址等等）和 token 值，这个 code 码是一次性的，使用一次后就失效，token 值主要是用来保证后端接口安全，采用 [jwt](https://github.com/jwtk/jjwt) 处理，前端后期发起的所有请求，header 里都要带上此 token 值，。
3. 拿到用户的信息（openid），判断用户是否绑定，如果已经绑定，根据 state，跳转到指定的页面界面，若未绑定，则跳转到绑定界面。
4. 绑定成功，后端返回新的用户信息（userInfo）和 token 值，前端同时更新用户信息和 token 值到 localStorage 里，若绑定失败，则提示用户重新进入，之所以会绑定失败，可能是 token 失效，那就需要用户重新进入，去拿用户的 openid。

## 其他

1. 会存在两次跳转，用户点击菜单后，首先会跳转到微信的服务器，然后再跳转到我们的业务页面。
2. state 需要前后端严格约定好，否则就会出现页面路由出错。
3. 微信公众号每次新加了菜单，都需要后端手动调用创建菜单的接口。
4. 由于微信内置浏览器不支持 session 和 cookie，也就是说，每次请求，jsessionid 都是不一样，那么后端 `request.getSession().getAttribute("xxx")` 是无法获取 session 中的值。

## 参考
- [Vue微信公众号开发踩坑记录](https://segmentfault.com/a/1190000010753247)
- [微信网页授权](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140842)
- [优化单点登录流程的好东西：JWT 介绍](http://www.youmeek.com/jwt/)
- [全能微信Java开发工具包](https://github.com/Wechat-Group/weixin-java-tools)
