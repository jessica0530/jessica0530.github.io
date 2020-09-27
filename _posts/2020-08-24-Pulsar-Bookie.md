---
layout: post
title: Pulsar Bookie介绍
categories: [bookie]
description: Pulsar Bookie
keywords: bookie
---

# Pulsar存储和Kafka对比

存储服务的分层的架构 和以Segment为中心的存储 是Apache Pulsar（使用Apache BookKeeper）的两个关键设计理念。 为Pulsar提供了许多重要的好处：

- 无限制的主题分区存储
- 即时扩展，无需数据迁移
  - 无缝Broker故障恢复
  - 无缝集群扩展
  - 无缝的存储（Bookie）故障恢复
- 独立的可扩展性

![kafka与pulsar的存储](/images/posts/kafka与pulsar的存储.png)

存储上结构上的差异

|      | Kafka                                                        | Pulsar                                                       |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|      | 一个paritition在一个broker上,partition目录下有多个segment,保存数据文件 以及索引文件,000000000.index, 0000000000.log | 以bookie做为存储,同一个paritition下的数据可以分成 多个 ledger 再分成多个fragment 存储再 bookkeeper上的 不同Bookie中,举例:"ledgers" : [ {     "ledgerId" : 6516,     "entries" : 9611,     "size" : 1231464423,     "offloaded" : false   }, |
|      | Kafka要查一下                                                | 每份Ledger的累积字节大小不相同 ，主要是由于它是根据我们配置的时间进行滚动生成(roll over), 默认最小是10分钟 ，最大是4小时，如果吞吐过大，每份Ledger中统计的字节可达10GB以上，只能更改pulsar broker中的配置，重启后才能生效 |
|      |                                                              |                                                              |
|      |                                                              |                                                              |
|      |                                                              |                                                              |

![Bookie](/images/posts/Bookie.png)

# Bookie读写隔离

![Bookie读写隔离](/images/posts/Bookie读写隔离.PNG)

Apache BookKeeper 提供的三个核心特性：I/O 分离、并行复制和容易理解的一致性模型。它们能够很好地满足我们对于持久化、多副本和一致性的要求。

Ledgers和Fragments是在Zookeeper中维护和跟踪的逻辑结构。物理上数据不存储在Ledgers和Fragments对应的文件中。

- 单个BookKeeper Broker 写入流程如下：
- 将写请求记入 WAL【类似于数据库的 Journal 文件】， 主要是日志记录；该日志可用于Bookie故障恢复时恢复尚未写入Entry Log文件的数据。

> 一般工程实践上建议把 WAL 和数据存储文件分别存储到两种存储盘上，如把 WAL 存入一个 SSD 盘，而数据文件存入另一个 SSD 或者 SATA 盘。

1. 将数据写入内存缓存中；
2. 写缓存写满后，进行数据排序并进行 Flush 操作，排序时将同一个 Ledger 的数据聚合后以时间先后进行排序，以便数据读取时快速顺序读取；
3. 将 <(LedgerID, EntryID), EntryLogID> 写入 RocksDB。

> LedgerID 相当于 kafka 的 ParitionID，EntryID 即是 Log Message 的逻辑 ID，EntryLogId 就是 Log消息在 Pulsar Fragment文件的物理 Offset。 这里把这个映射关系存储 RocksDB 只是为了加快写入速度，其自身并不是 Pulsar Bookie 的关键组件





- 其读取流程如下：

  1.从写缓存读取数据【因为写缓存有最新的数据】；

  2.如果写缓存不命中，则从读缓存读取数据；

  3.如果读缓存不命中，则根据 RocksDB 存储的映射关系查找消息对应的物理存储位置，然后从磁盘上读取数据；

  4.把从磁盘读取的数据回填到读缓存中；

  5.把数据返回给 Broker