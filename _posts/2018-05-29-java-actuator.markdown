---
layout:     post
title:      "Java 程序的内存信息（spring boot actuator原理）"
subtitle:   ""
date:       2018-05-29 12:00:00
author:     "Lee"
header-img: ""
tags:
    - Java
---

> 在 Spring boot 框架中引入 actuator 就能实现程序的部分功能和性能，以及运行情况的监控。那么 actuator 的监控原理是什么呢？非 Spring Boot 程序如何实现内存等信息的监控呢？本文告诉你如何实现这些功能。

actuator 的监控指标其实就是 Java 程序的内存等信息，这些都可以使用 Java 的 api 获取到。那么 Java 的内存信息是如何被监控的呢？答案就是 Runtime.getRuntime() 提供的几个方法。

![](/img/in-post/post-springboot/springboot_actuator.png)

关于maxMemory()，freeMemory()和totalMemory()：maxMemory()为JVM的最大可用内存，可通过-Xmx设置，默认值为物理内存的1/4，设置不能高于计算机物理内存；  totalMemory()为当前JVM占用的内存总数，其值相当于当前JVM已使用的内存及freeMemory()的总和，会随着JVM使用内存的增加而增加；  freeMemory()为当前JVM空闲内存，因为JVM只有在需要内存时才占用物理内存使用，所以freeMemory()的值一般情况下都很小，而JVM实际可用内存并不等于freeMemory()，而应该等于maxMemory()-totalMemory()+freeMemory()。
