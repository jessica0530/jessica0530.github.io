---
layout: post
title: Kafka和Pulsar的区别
categories: [pulsar]
description: Kafka Pulsar区别
keywords: pulsar
---



| Kafka          | Pulsar Bookie                                                |                                                              |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据分片与分布 | 存在本地Broker的磁盘上，被切分成多个分区文件                 | 数据被分成多个ledger, 分布在多个Bookie上                     |
| 数据持久化     | 设置保留时间，特定大小的文件超过保存时间会被定时清理         | 数据在超过配置的保留时间之后被删除，或者设置消息的存活时长(TTL) |
| 写操作         | 可以批量并行写入                                             | 可以通过Topic Partition并行写入，但是Ledger只能被单一Pulsar Broker写入 |
| 读操作         | 主要从partition leader 所在的Broker读出，最新数据可以在文件系统页缓存中读出 | 可以从任何有数据的Bookie 节点读出                            |
| 数据复制       | ISR 集合中的副本将数据写入文件系统，主副本更新High Watermark(高水位) | Pulsar Broker 并发写入Bookie, 得到Qa 数量的响应后，更新 LAC. |
| 复制修复       | 通过增加新的副本来从主副本上拷贝所有数据                     | 通过Bookie 的自动恢复机制来保证复制因子                      |
| 集群扩展       | 增加kafka broker 后需要重新进行rebalance来使用整个集群负载均衡。保证在迁移过程不会占满网络和磁盘 | 增加Pulsar Broker或者Bookie 时不需要做数据重新分布，新的日志分区会自动分配到新加入的Bookie. |
| 存储           | 每个分区中存储日志文件，索引文件                             | 同一个Ledger 有多个Fragment,存储在不同的bookie上，交错存储格式 |
| 持久性         | 只写到文件系统的页缓冲中。可以配置只等待主副本所在Broker 的确认，或者等待ISR 中的所有副本所在的Broker 的确认 | 所有写入都是显式的fsync操作持久化到硬盘上才算写入成功。等待Qa 个Bookie的确认才认为是写入是成功的 |
| I/O 隔离       | 没有物理I/O 隔离，依靠文件系统缓冲                           | WAL (预写日志)存储与Log Entry 存储在不同的磁盘上，有物理I/O隔离 |

从以下几点来对比Pulsar 与Kafka 的优势

- 操作
  - 集群扩容时，即不管是pulsar broker 与 bookie 扩容，都不需要rebalance 数据；
  - Pulsar Broker 故障时恢复时，不存在数据恢复与数据复制；
  - Bookie 可以无限扩容，同时，存在数据损坏时，能迅速修复；
  - Pulsar Broker Server 端对消息进行了去重的功能， 主要是同一Pulsar Broker处理同一SequenceId的消息不会被多次发送到bookie，不需要在客户端进行去重；
  - 写入操作都是在收到ACK之前都先通过fsync刷到硬盘上的，提供了可靠性保证，而Kafka 在ISR 副本写入文件系统缓存 就会返回ACK；
- 复制
  - Kafka 使用的ISR 复制算法，leader 写入成功后，所有的ISR 中的follower 从leader主动拉取数据，leader会维护一个高水位线（HW，High Watermark），即每个分区最新提交的数据记录的偏移量。当Topic replica=3, ack=all, min.insync.replicas=2 时，能够保证故障一台broker 时，不影响数据的可靠性。
  - Pulsar Broker 会并发地把数据记录写入所有存储节点，并在得到超过配置数量的存储节点确认之后，才认为数据已成功提交，保证数据的可靠性。
  - 存储节点也只在数据被显式地调用flush操作刷入磁盘之后才会响应写入请求。Pulsar Broker 也会维护一个日志流的最新提交的数据记录的偏移量，Apache BookKeeper中的LAC（LastAddConfirmed）。LAC也会保存在数据记录中（来节省额外的RPC调用开销）, 并不断复制到别的Bookie存储节点上，同时Pulsar Broker 将该LAC 指针 保存在Zookeeper 中;
- 存储
  - 每个Kafka分区都以若干个文件的形式保存在Broker的磁盘上。它利用文件系统的页缓存和I/O调度机制来得到高性能，从日志文件中读取历史数据时(catch-up reads)，而这些读取历史数据 在文件系统角度通常会表现为大批量的扫描，文件系统会进行大量的预读取到 Page Cache 里，从而挤掉最新的数据而影响写操作和 日志的末尾读(tailing reads)操作。
  - 写入（蓝线）、末尾读（红线）和中间读（紫线）这三种常见的I/O操作都被隔离到了三种物理上不同的I/O子系统中。所有写入都被顺序地追加到磁盘上的日志文件，再批量提交到硬盘上。在写操作持久化到磁盘上之后，它们就会放到一个RocksDB Memtable中，再向客户端发回响应 。Memtable中的数据会被异步刷新到交叉存取的索引数据结构中：Entry 被追加到日志文件中，偏移量(Offsets)则在Ledger的索引文件中根据记录ID(LedgerId, EntryId)索引起来。最新的数据肯定在Memtable中，供末尾读操作使用。中间读会从记录日志文件中获取数据