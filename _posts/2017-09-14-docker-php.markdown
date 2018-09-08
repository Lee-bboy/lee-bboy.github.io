---
layout:     post
title:      "docker 构建基于centos的php72-fpm镜像"
subtitle:   ""
date:       2017-09-14 15:00:00
author:     "Lee"
header-img: ""
tags:
    - docker
---

> 发现基于ubuntu的php-fpm构建镜像是主流的，应该也可以用于生产环境。

docker的标准用法是每个docker容器只提供一个服务。
所以应该是php-fpm单独一个容器，nginx单独一个容器。本文的思路也是这样的，坚决不搞docker全家桶。
===========================================
这是原始的,名字：amazeeio/centos:7，该镜像内含了一些原作者自己的写命令，用于后续的构建，可惜不是通过Dockerfile构建的，不知具体内容，很遗憾。所以，如果用的话，只能依赖于原作者这个镜像。
这是新的，直接可用的一个基于centos的php-fpm镜像。
amazeeio/centos7-php
原作者的docker仓库地址。用于不打算自己改，直接用原作者的。
https://hub.docker.com/r/amazeeio/centos7-php/
假设想自己构建镜像，则，
```
git clone https://github.com/amazeeio/docker-centos7-php
cd docker-centos7-php/
git checkout 7.2
docker build -t 自己的centos7php镜像名 .
```
=========================================
我现在假设，直接用它的。
```
docker pull nginx:1.12
docker pull amazeeio/centos7-php:7.2
docker network create -d bridge php-net
```
创建两个容器
```
docker run -d --network php-net  --name c_fpm -p  9000:9000 -v /var/www/html:/usr/share/nginx/html amazeeio/centos7-php:7.2
docker run -d --network php-net  --name c_nginx    -p 80:80 -v /var/www/html:/usr/share/nginx/html nginx:1.12
```
确认一下两个容器始终打开
```
dcoker ps -a
```
复制nginx容器的配置文件到docker宿主机。
```
docker cp c_nginx:/etc/nginx/conf.d/default.conf ./default.conf
```
vi default.conf

```
###
location ~ \.php$ {
  fastcgi_pass c_fpm:9000;
  fastcgi_index index.php;
  fastcgi_param SCRIPT_FILENAME /usr/share/nginx/html$fastcgi_script_name;
  fastcgi_param SCRIPT_NAME $fastcgi_script_name;
  include fastcgi_params;
}
```
###
把上面这段话加入到 nginx配置。
```
docker cp  ./default.conf  c_nginx:/etc/nginx/conf.d/default.conf
docker stop c_nginx
docker start c_nginx
```

vi /var/www/html/1.php
```
<?php
phpinfo();
```
在最外面的宿主机，浏览器
http://192.168.99.100/1.php
显示 PHP Version 7.2.0RC2.
不喜欢那个RC2，就自己构建吧！
只需在创建两个容器那里，替换镜像名为你自己构建的镜像名即可。
