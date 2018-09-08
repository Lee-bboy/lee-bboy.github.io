---
layout:     post
title:      "golang通过json文件生成struct及解析处理过程"
subtitle:   ""
date:       2017-07-19 10:00:00
author:     "Lee"
header-img: ""
tags:
    - golang
---

> 高德地图的返回json结果里面有时候字段对应值是字符串，有时候对应结果是[]一对中括号，代表结果为空，使用golang自带的json解析工具折腾半天，最后一查资料据说是性能最差的，说某东用的是easyjson去解析处理json文件，大概记录一下处理过程及遇到的坑。

1.easyjson的使用

easyjson提供了一个命令行的工具，可以根据.go源码内的struct自动生成一个代理调用类文件，大概使用过程是先

```
go get -u github.com/mailru/easyjson
把它下载回来，然后
```

```
go install  github.com/mailru/easyjson/easyjson
```
自动编译安装到%GOPATH%\bin目录下面，需要生成代理代码需要你的项目里面有与json对应的struct，这个见下面的处理过程。



2.高德地图官方有在线的调试工具，可以根据参数返回json格式的查询结果，拿到结果以后我最初是手敲代码挨个去对应json文件字段的，其实有全自动的方法，就是访问https://mholt.github.io/json-to-go/ 这个网址，把json文件复制进去，自动就生成好了golang的struct，复制到项目里面，运行easyjson的自动生成代码命令生成项目内的代理代码：
```
easyjson -all models.go
```
以上我是把网站里面生成的代码保存为我本机的models.go里面了，自动生成的文件叫做models_easyjson.go，这里面有序列号反序列化的代码。



3.第二步生成的代码去接收高德地图返回的json结果还是有些问题，就是个别字段有值的时候是字符串，没有值的时候是空[]结果的情况，解决办法是把原来的
```
Tel      []interface{}   `json:"tel"`
```
改为

```
Tel      interface{}   `json:"tel"`
```
就好了。
