---
layout: post
title: Spark Shuffle
categories: [spark]
description: Spark Shuffle
keywords: spark
---



1.sortShufflewriter 适用于什么场景,什么时候只对partition id排序,什么时候同时对partiition id 与key排序

2.什么时候选择BypassShuffleWriter?ByPassShuffleWriter的优势是什么,劣势是什么,是否需要排序,如何排序

3.什么时候选择UnsafeShuffleWriter? 是partition id还是key排序？ Unsafe 是指什么？ 与offheap 什么关系



4. 什么时候进行Spill,如何保证及时Spill而不OOM
5. Map 与 array 分别对应什么场景
6. Map side combine 与partial aggregate 的区别是什么,各自适用场景是什么

## Shuffle产生

一个Job会被DAGScheduler划分成不同的Stage

DAGScheduler 根据ShuffleDependency来划分Stage

ShuffleDependency会导致shuffle



Shuffle/narrow deps

Narrow deps (父RDD分区对应一个子RDD分区), map.filter,union,join with inputs co-partitioned

Shuffle deps(父RDD分区对应多个子RDD分区) groupbykey, join with inputs not co-partitioned



## Shuffle 开销

Data partition: invoke very expensive data sorting works

Data ser/deser:transfer through network or across process

Data compression: reduce IO bandwidth

DISK IO

## Shuffle history

0.8: hash shuffle,  consolidation 机制

1.1: 加入sort shuffle 机制,但默认 任然使用 hash shuffle

1.2: 默认使用sort shuffle

1.4: 引入tungsten-sort shuffle

1.6: 将 tungsten-sort shuffle 与sort shuffle 合并,由Spark 自动决定采用哪一种方式

2.0: hash shuffle机制被删除



## ShuffleManager

SparkEnv#create

Spark.shuffle.manager

Sort && tungsten-sort

## ShuffleWriter

ShuffleWriter: ShuffleManager#getWriter

ShuffleHandle: ShuffleManager#registerShuffle



## ShuffleHandle

BypassMergeSortShuffleHandle 

触发条件:

   don't need to do map-side aggregation (map端没有聚合操作)

   numPartitions <= bypassMergeThreshold(处理数据分区数小于参数。。。bypassMergeThreshold)

SerializedShuffleHandle - UnsafeShuffleWriter

 supportsRelocationOfSerializedObjects

 aggregator

numPartitions <= 2^24

BaseShuffleHandle - SortShuffleWriter



BypassMergeSortShuffleWriter

优势:

Sort Shuffle 仅对于大量ReduceTask 环境下map端有聚合或者需要排序的任务来说会非常高效

对于ReduceTask数量不多且不需要聚合和排序HashBasedShuffle反而更高效

BypassMergeSortShuffleWrite 可以认为是改进版的HashBasedShuffle

劣势

产生较多的临时文件,不适合有较多分区的任务

不进行map端聚合和排序



## SortShuffleWriter  —— ExternalSorter

最通用的ShuffleWriter

支持map端排序,合并

根据是否需要聚合存放位置不同

shouldCombine: PartitionedAppendOnlyMap

两倍容量数组来存储数据,自己实现了hash表

根据key.hashcode得出key的位置

采用了二次探测算法避免哈希冲突

在写入的时候会根据aggregator来合并数据

!shouldCombine:PartititonedPairBuffer

两倍容量数组来存储数据

key和value依次存放在数组中

估计占用内存大小:SizeTracker#afterUpdate

SizeEstimator.estimate

bytesPerUpdate:(latest.size - previous.size).toDouble/(latest.numUpdates - previous.numUpdates)



### maybespill

当前collection 内存使用 > 5M (默认Threshold)，尝试扩大2倍,并申请内存配额

Threshold 在原来的基础上增大相应的申请到的配额

Collection 内存使用仍然大于Threshold,则进行真正的Spill

### Spill

需要ordering 或者aggregator,则先按照partition排序,再按照 partition内key排序,否则只按照partition排序

将排序好的内存中的数据写到tmp目录中

最终调用writePartitionedFile 将内存和磁盘tmp目录中的文件写到一个文件上,返回mapinfo

根据mapinfo 创建索引文件

## UnsafeShuffleWriter

UnsafeShuffleWriter的实现,也就是Tungsten-Sort

Unsafe,即使用了Java的Unsafe API来操作数据

TungstenProject

Databricks公司提出的对Spark优化内存和Cpu使用的计划,其目的在于榨干硬件,让Spark充分利用申请到的资源

UnsafeShuffleWriter是对普通sort的一种优化

排序的不是内容本身,而是内容序列化后字节数组的指针（元数据）

没有序列化和反序列化的过程

内存的消耗降低,相应的也会减少gc的开销

触发条件:

对象序列化方式支持Relocation,目前只有Kryo序列化支持

任务中没有 aggregation或者key排序

Reduce分区小于2^24个（24位存储分区ID）

优点:

直接在serialized binary data 上sort 而不是java objects，j减少了memory的开销和GC的overhead

提供cache-efficient sorter,使用一个8 bytes的指针,把排序转化成了一个指针数组的排序

spill的mergr过程也无需反序列化即可完成

|             | UNsafeShuffleWriter          | SortShuffleWriter               |
| ----------- | ---------------------------- | ------------------------------- |
| 排序方式    | 只是partition级别的排序      | 先partition排序,相同分区key有序 |
| Aggregation | 没有反序列化,没有aggregation | 支持aggregation                 |

