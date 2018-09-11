---
layout:     post
title:      "Kafka基于topic的分区设计"
subtitle:   ""
date:       2018-04-19 19:00:00
author:     "Lee"
header-img: ""
tags:
    - Java
---

> 若没有分区，一个topic对应的消息集在分布式集群服务组中，就会分布不均匀，即可能导致某台服务器A记录当前topic的消息集很多，若此topic的消息压力很大的情况下，服务器A就可能导致压力很大，吞吐也容易导致瓶颈。

有了分区后，假设一个topic可能分为10个分区，kafka内部会根据一定的算法把10分区尽可能均匀分布到不同的服务器上，比如：A服务器负责topic的分区1，B服务器负责topic的分区2，在此情况下，Producer发消息时若没指定发送到哪个分区的时候，kafka就会根据一定算法上个消息可能分区1，下个消息可能在分区.


**kafka为什么要在topic里加入分区的概念？**

topic是逻辑的概念，partition是物理的概念，对用户来说是透明的。producer只需要关心消息发往哪个topic，而consumer只关心自己订阅哪个topic，并不关心每条消息存于整个集群的哪个broker。

为了性能考虑，如果topic内的消息只存于一个broker，那这个broker会成为瓶颈，无法做到水平扩展。所以把topic内的数据分布到整个集群就是一个自然而然的设计方式。Partition的引入就是解决水平扩展问题的一个方案。

如同我在Kafka设计解析（一）里所讲，每个partition可以被认为是一个无限长度的数组，新数据顺序追加进这个数组。物理上，每个partition对应于一个文件夹。一个broker上可以存放多个partition。这样，producer可以将数据发送给多个broker上的多个partition，consumer也可以并行从多个broker上的不同paritition上读数据，实现了水平扩展

**如果没有分区,topic中的segment消息写满后,直接给订阅者不是也可以吗**

“segment消息写满后”，consume消费数据并不需要等到segment写满，只要有一条数据被commit，就可以立马被消费。

segment对应一个文件（实现上对应2个文件，一个数据文件，一个索引文件），一个partition对应一个文件夹，一个partition里理论上可以包含任意多个segment。所以partition可以认为是在segment上做了一层包装。这个问题换个角度问可能更好，

**为什么有了partition还需要segment**

如果不引入segment，一个partition直接对应一个文件（应该说两个文件，一个数据文件，一个索引文件），那这个文件会一直增大。同时，在做data purge时，需要把文件的前面部分给删除，不符合kafka对文件的顺序写优化设计方案。引入segment后，每次做data purge，只需要把旧的segment整个文件删除即可，保证了每个segment的顺序写，
