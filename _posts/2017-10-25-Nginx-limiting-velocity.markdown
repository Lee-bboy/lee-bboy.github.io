---
layout:     post
title:      "Nginx 服务器速率限制之妙用"
subtitle:   ""
date:       2017-10-25 12:00:00
author:     "Lee"
header-img: ""
tags:
    - Nginx
---

> Nginx服务器有一个非常有用的限速功能，但是它却常常被错误配置。  
> 这个功能用来限制用户在某此时间段内请求的的HTTP请求数,此请求应该是 GET 或POST 来发出的请求   
> <br/>
>  这个限速功能常常被应用于网络安全方面。比如减慢暴力密码破解的攻击，爬虫对网页的抓取，防止DDOS攻击等。通过它来限制和过滤为为真实用户的标准数值，它会把来源URL等信息写到系统日志中。更确切地说，这个功能常用于提供极少量的应用服务器，用户访问量不多，但却常常瘫痪的问题。

**Nginx限速是怎样工作的**

Nginx限速使用 Leaky（唝水桶）算法，比喻为水桶顶部倒水，底部漏水，如果倒入水的速率超过漏水的速度，则水桶漏出。在电信网络和分组交换网络中，带宽有限的情况下该算法使用场景较多。

就请求处理而言，水代表客户端的请求，存水的桶按先进先出（FIFO）调度算法处理的队列。漏出的水表示退出缓冲区等服务器处理，而溢出表示请被丢弃且不再提供服务。

**配置基本的速率限制**

速率限制主要有2个主要指令，limit_req_zone和limit_req。如下代码：

```javascript
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/sm;
server {

    location /login/ {

        limit_req zone=mylimit;

        proxy_pass http://my_upstream;

    }

}
```

limit_req_zone指令定义了速度限制的参数，同时在出现的上下文中启用速率限制。（在本例中是针对于 /login/ URI的所有请求）

limit_requ_zone 指令通常定义在HTTP块中，这样可以用于多个上下文。它包含3个参数：

**Key** - 定义应用限制的请求特征。 在这个例子中，它是Nginx变量$binary_remote_addr ，它保存着客户端IP地址的二进制表示。 这意味着我们将每个唯一的IP地址限制为由第三个参数定义的请求速率（我们使用这个变量，因为它比客户端IP地址的字符串表示$remote_addr占用更少的空间）。

**Zone** - 定义用于存储每个IP地址状态的共享内存区域以及访问请求受限URL的频率。 将信息保存在共享内存中意味着它可以在Nginx工作进程之间共享。

定义有两个部分： zone= keyword标识的区域名称和冒号后面的大小。 大约16,000个IP地址的状态信息需要1兆字节，所以我们的区域可以存储大约160,000个地址。 如果Nginx需要添加一个新条目时，存储空间将被耗尽，它将删除最旧的条目。

如果释放的空间不足以容纳新记录，则Nginx返回状态码503(Temporarily Unavailable) 。 此外，为了防止内存耗尽，每当Nginx创建一个新条目时，最多可以删除两个在前60秒内没有使用的条目。

**Rate** - 设置最大请求率。 在这个例子中，速率不能超过每秒10个请求。 Nginx实际上以毫秒粒度跟踪请求，所以这个限制对应于每100毫秒1个请求。 由于我们不允许爆发，这意味着如果请求在前一个允许的时间之后小于100毫秒时被拒绝。

limit_req_zone指令为速率限制和共享内存区域设置参数，但实际上并不限制请求速率。

因此，您需要通过在其中包含limit_req指令来将限制应用于特定location或server块。 在这个例子中，我们是对/login/的URI速率限制请求。

因此，现在每个唯一的IP地址被限制，/login/每秒10个请求 - 或者更确切地说，在前一个100毫秒内不能请求该URL。

### 处理并发

如果我们在100毫秒内得到两个请求会怎么样？ 对于第二个请求，Nginx将状态码503返回给客户端。 这可能不是我们想要的，因为应用程序本质上是突发性的。

相反，我们想要缓冲任何多余的请求并及时提供服务。 这是我们使用burst参数limit_req ，在这个更新的配置：

```javascript
location /login/ {

  limit_req zone=mylimit burst=20;

  proxy_pass http://my_upstream;

}
```

burst参数定义了客户端可以超过区域指定的速率（使用我们的示例mylimit区域，速率限制为每秒10个请求，或每100毫秒1个）可以产生多少个请求。

在前一个请求到达100毫秒后的请求被放入一个队列中，这里我们将队列大小设置为20。

这意味着如果21个请求同时从一个给定的IP地址到达，Nginx立即将第一个请求转发到上游服务器组，并将剩下的20个放入队列中。 然后，它每100毫秒转发一个排队的请求，并且只有当传入的请求使排队请求的数量超过20时才返回503给客户端

###无延迟队列###

具有burst的配置会导致流量畅通，但不是很实用，因为它可能会使您的网站显得很慢。



在我们的例子中，队列中的第20个数据包等待2秒钟被转发，此时对其的响应可能对客户端不再有用。 要解决这种情况，请将nodelay参数与burst参数一起添加：


```javascript
location /login/ {

  limit_req zone=mylimit burst=20 nodelay;

  proxy_pass http://my_upstream;

}
```


通过nodelay参数，Nginx仍然根据burst参数在队列中分配时隙，并且强加配置的速率限制，但是不排除转发排队的请求。 相反，当请求到达“太快”时，Nginx会立即转发，只要队列中有一个可用的时隙。 它将该插槽标记为“已占用”，并且不会将其释放以供其他请求使用，直到经过适当的时间（在本例中为100毫秒之后）。



假设像以前一样，20个时隙的队列是空的，21个请求同时从给定的IP地址到达。 Nginx立即转发所有21个请求，并将队列中的20个插槽标记为已占用，然后每100毫秒释放1个插槽（如果有25个请求，Nginx会立即转发21个插槽，标记20个插槽，拒绝4个请求状态503 ）。



现在假设在第一组请求之后101毫秒被转发，另外20个请求同时到达。 队列中只有1个插槽被释放，所以Nginx转发1个请求，并拒绝其他19个状态为503的队列。 如果在20个新请求到达之前经过了501毫秒，那么5个空闲空间，所以Nginx立即转发5个请求，拒绝15个请求。



效果相当于每秒10个请求的速率限制。 如果您希望在不限制请求之间的允许间隔的情况下施加速率限制，则nodelay选项非常有用。



注意：对于大多数部署，我们建议将burst和nodelay参数包含到limit_req指令中。



### 高级配置示例



通过将基本速率限制与其他Nginx功能相结合，您可以实现更多细微的流量限制。



白名单



此示例显示如何对不在“白名单”上的任何人的请求施加速率限制。


```javascript
geo $limit {

  default 1; 10.0.0.0/8 0; 192.168.0.0/24 0;

}

map $limit $limit_key {

  0 ""; 1 $binary_remote_addr;

}

limit_req_zone $limit_key zone=req_zone:10m rate=5r/s;

server {

  location / {

    limit_req zone=req_zone burst=10 nodelay;

     # ...

  }

}
```


这个例子使用了geo和map指令。 geo块为白名单中的IP地址分配一个0值到$limit值，其他0 1 。 然后，我们使用地图将这些值转换为一个密钥，以便：



如果$limit是0，$limit_key设置为空字符串。



如果$limit是1，则$limit_key以二进制格式设置为客户端的IP地址。

把两者放在一起，$limit_key被设置为白名单IP地址的空字符串，否则设置为客户端的IP地址。 当limit_req_zone目录（密钥）的第一个参数为空字符串时，限制不适用，因此列入白名单的IP地址（在10.0.0.0/8和192.168.0.0/24子网中）不受限制。 所有其他IP地址每秒限制为5个请求。



limit_req指令将限制应用于/位置,并且允许在配置的限制上突发多达10个分组而没有转发延迟



在一个位置包含多个limit_req指令



您可以在一个位置包含多个limit_req指令。 所有与给定请求匹配的限制都被应用，这意味着使用最严格的限制。 例如，如果多于一个指令施加延迟，则使用最长的延迟。 同样，如果这是任何指令的影响，即使其他指令允许它们通过，请求也会被拒绝。



扩展前面的例子，我们可以对白名单上的IP地址应用速率限制：


```javascript
http {

  # ... limit_req_zone $limit_key zone=req_zone:10m rate=5r/s;]

  limit_req_zone $binary_remote_addr zone=req_zone_wl:10m rate=15r/s;

  server {

   # ... location / {

      limit_req zone=req_zone burst=10 nodelay;

      limit_req zone=req_zone_wl burst=20 nodelay; # ...

    }

  }

}
```
白名单上的IP地址与第一个速率限制（ req_zone ）不匹配，但匹配第二个（ req_zone_wl ），因此每秒限制为15个请求。

不在白名单上的IP地址与两个速率限制相匹配，所以限制性较强的一个适用：每秒5个请求。



### 配置相关功能



记录



默认情况下，Nginx 记录由于速率限制而延迟或丢弃的请求，如下例所示：


```javascript
 2017/06/13 04:20:00 [error] 120315#0: *32086 limiting requests, excess: 1.000 by zone "mylimit", client: 192.168.1.2, server: Nginx.com, request: "GET / HTTP/1.0", host: "Nginx.com"
 ```



日志条目中的字段包括：


limitingrequests - 指示日志条目记录速率限制。

excess - 此请求表示的配置速率每毫秒的请求数。

zone - 定义强加的限制的区域。

client - 发出请求的客户client IP地址。

server - server IP地址或主机名。

request - 客户端request实际HTTP请求。

host - Host HTTP头的值。

默认情况下，Nginx在error级别记录被拒绝的请求，如上例中的[error]所示（它记录延迟的请求在一个较低的级别，所以默认info ）。 要更改日志级别，请使用limit_req_log_level指令。 在这里，我们设置了拒绝的请求来记录warn级别：


```javascript
location /login/ {

  limit_req zone=mylimit burst=20 nodelay;

  limit_req_log_level warn;

  proxy_pass http://my_upstream;

}
```


错误代码发送到客户端



默认情况下，当客户端超出速率限制时，Nginx以状态码503作为响应。



使用limit_req_status指令来设置一个不同的状态码（在这个例子中是444 ）：


```javascript
location /login/ {

  limit_req zone=mylimit burst=20 nodelay;

  limit_req_status 444;

}
```


拒绝所有请求到特定的位置



如果您想要拒绝所有特定URL的请求，而不是限制它们，请为其配置一个块并包含all指令：


```javascript
location /assets/header.php {

 	deny all;

}
```
