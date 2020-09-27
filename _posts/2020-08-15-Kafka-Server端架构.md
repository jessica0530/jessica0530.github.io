---
layout: post
title: kafka server端
categories: [kafka]
description: Kafka Server端
keywords: kafka
---

# Kafka Server架构

![kafka-server网络层](/images/posts/kafka-server网络层.png)

## ReActor模式

![kafka-reactor](/images/posts/kafka-reactor.png)

KafkaProducer-Server端

![kafka-producer服务端](/images/posts/kafka-producer服务端.png)

RequestQueue 可以 限速 保护 Kafka集群不被大量请求 压垮

# 日志存储

Log 分为多个LogSegment,LogSegment 对应磁盘上的一个日志文件和一个索引文件,日志文件记录消息,索引文件保存了消息的索引,随着消息的不断写入,日志文件的大小达到一个阈值就创建新的日志文件和索引文件继续写后面的消息

用稀疏索引的方式为日志文件中的部分消息建立索引

![kafka-Logsegment](/images/posts/kafka-Logsegment.png)

LOG 对多个LogSegment对象的顺序组合,为了快速定位 LogSegment，Log使用跳表SkipList对 LogSegment进行管理

![Kafka-skiplist](/images/posts/Kafka-skiplist.png)

# TimingWheel

时间轮  环形队列

# 延迟操作组件

1. 延迟操作：kafka将一些需要等待满足一定条件之后才触发的操作成为延迟操作，并将这些操作定义为一个抽象类DelayedOperation。
2. kafka的延迟组件有DelayedProduce、DelayedFetch、DelayedHeartbeat、DelayedJoin、DelayedCreateTopics，这些都继承于DelayedOperation抽象类，分别用来协助相应的组件对不同的请求完成延迟处理。

Ack=-1的时候 需要所有副本 都 写好数据 再返回结果,这个返回结果的操作 就是个延迟操作

![kafka-延迟组件](/images/posts/kafka-延迟组件.png)

# ZooKeeper中存储的信息

![kafka-zookeeper](/images/posts/kafka-zookeeper.png)