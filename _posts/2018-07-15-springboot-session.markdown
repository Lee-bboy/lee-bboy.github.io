---
layout:     post
title:      "spring boot 开发微信小程序的session问题"
subtitle:   ""
date:       2018-07-15 16:00:00
author:     "Lee"
header-img: ""
tags:
    - Java
---

> 微信小程序开发 之保湿登录状态既session不改变

在微信小程序开发中，由wx.request()发起的每次请求对于服务端来说都是不同的一次会话，微信小程序不会把session信息带回服务端，即对应服务端不同的session，由于项目中使用session保存用户信息所以导致后续请求相当于未登录的情况。

注意，这里的session不是小程序维护的那个通过wx.login()方法维护的session，而是我们自己的服务端的session。

而网上大部分的技术文章解决方案都是复制粘贴了，且大部分是旧包，或者引入方式问题，所以只能作为参考！
大家可以看下 ： https://www.cnblogs.com/gdutzyh/p/7251432.html 、 https://blog.csdn.net/b376924098/article/details/79314620 参考下！


**注意**
细节暂时不说，可以参考以上2篇文章，我这边主要说明可能由于引入的新包或者不同版本的包的问题：

**通过header读取方式**

1、创建session配置类时大部分写的是在如下配置

![](/img/in-post/post-springboot/session_token.png)

可是我发现引入的包根本没这个方法，所以就参考它的写法，终于可以实现了，以下主要实现的截图：

引入相关包：

![](/img/in-post/post-springboot/session_dependency.png)

配置类和实现源码：

![](/img/in-post/post-springboot/http_session_id.png)

![](/img/in-post/post-springboot/header_XAuthToken.png)


通过header读取sessionId:

![](/img/in-post/post-springboot/headers_result.png)

* 通过截图的配置类里的实现源码，发现让客户端传入的是X-Auth-Token请求头，千万注意！暂时没有支持set的方法，所以也不太建议修改源码！
