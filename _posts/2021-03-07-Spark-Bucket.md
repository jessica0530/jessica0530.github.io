---
layout: post
title: Spark Bucket
categories: [spark]
description: Spark Bucket
keywords: spark
---



引入Bucket 就是为了优化SortMergeJoin的shuffle 和 sort, Bucket 的思想是 pre-(shuffle+sort)

指定join keys 为buckets 字段，insert 的时候按照bucket 字段进行shuffle和sort,以避免在join时候再来shuffle



## Spark Bucketing 和HIve Bucketing 兼容

|      | Spark                                                        | HIve                                                         |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|      | 一组文件表示一个bucket，文件内有序,组成一个bucket 的所有文件之间无序 | 一个文件表示一个bucket，每个文件全局有序                     |
|      | insert的时候无需shuffle,每个task 可以写至多bucket个数个文件  | Insert 的时候需要shuffle,shuffle 后的task 个数和bucket个数一致，每个task 写一个文件 |
|      | 采用Murmur3Hash                                              | 采用HiveHash                                                 |
|      | join的时候走bucket join 无需shuffle和sort                    | join的时候走bucket join 无需shuffle,但需要对每个bucket对应的多个文件进行排序 |



为了兼容

1.增加hivehash方式

2.允许spark 创建 hive bucketed table

   2.1在表的properties 里增加一个属性 hive.compatible,表示是否与hive兼容,对于provider=parquet 或者orc,默认是 false, 对于 provider =hive （表示由hive 创建的表），hive.compatible 始终为true

  2.2 如果hive.compatible=true，将bucket 信息写到hive 的metadata里面，HiveExternalCatalog#createDataSourceTable



3.确保Spark 写 Hive bucketed table 符合 hive bucketing 语义

INSERT 的时候要按照 bucket keys 进行shuffle,

writingCommand 增加 requiredDistribution 和 requiredOrdering ,表示写数据时需要满足一定的数据分布和排序方式

如果是 hive.compatible requiredDistribution 就返回 hivehash 的HashClusterdDistribution

4.确保 Spark 读Hive bucketed table 可以应用 query planner optimizations

HiveTableScanExec 和 FileSourceScanExec 

定义了 outputPartitioning 和 outputOrdering 两个方法, 表示该算子承诺他的输出应该满足的数据分布和排序分布,默认是UnknownPartitioning和Nil

因此,需要给ScanExec 重写这两个方法,bucketSpec 存在并且bucketingEnabled开启,则其outputPartitioning 返回hivehash 的HashPartitioning,如果bucketSpec 中指定了排序列,则outputOrdering返回 SortOrder

这样,在应用EnsureRequirements 规则之后,SortMergeJoinExec 的Children满足其requiredChildDistribution 和 requiredChildOrdering,就不会增加 Exchange和Sort节点



5 允许写空文件

hive里面,如果某个bucket没有数据,也会创建一个对应的空文件,来保证会有bucket个数文件,而spark 的task 在写文件的时候,只有存在数据的bucket才会写文件,这样会导致bucket数与文件数不一致,不符合hive bucketing语义,因此要允许没有数据的bucket也要创建一个空文件。

6.确保一个bucket对应一个partition

用来读取 hive bucketed table 时保证一个bucket文件对应RDD的一个partition

每个bucket 文件构造一个inputSplit,每个inputSplit最终会是一个 partition



## Bucket Join

两张 bucket表,bucket个数成倍数关系,可以走bucket join

1. 按照bucket 更小的处理,即 3：6 -> 3:3 ,对于bucket 为6的表,把bucket_Id % min_bucket_num 相同的划分到同一个partition里面,这样只生成 3个filepartition,一个 filepartition 有2个文件,这需要一个 sort
2. 按照bucket更大的处理,即3：6-> 6:6 ，对于bucket 为3 的表,他只有3个filepartition,"复制"一份就成了6个filepartition，这样就可以跟bucket=6的表join，只适用于 inner join (outer join时 join不上的数据会重复出现,可以尝试通过过滤掉那些join不上, 并且partition id不等于 bucket id 的数据的方式来解决)

## Bucket Join 支持超集

针对 join的场景

两张bucket 表 bucket key 都是(a), 对于下面这张查询:

```
select * from x join y in x.a = y.a and x.b = y.b
```

join key (a,b) 是 bucket key(a) 的超集,可以bucket join,不用 shuffle

实现思路,把shuffle key 由 join key 调整为 bucket key



SortMergeJoinExec 调整 SortMergeJoin 的leftKeys 和 rightKeys 为bucket key

JoinUtil 判断SortMergeJoinExec 的left 和 right节点的outputPartition 是否满足ClusteredDistribution(leftKeys), 如果满足,说明join key 是bucket key的超集,把join key 调整为 bucket key



## Bucket Join 兼容历史分区

修改 bucket表的bucket数之后,再读取历史分区,由于历史分区的文件数和bucket数修改之后不一致,读取会出错,所以需要兼容历史分区

实现思路是,在表里增加一个属性 bucket.begin.partition.datexxx 指定一个日期,在该日期之前认为是非bucket表,在该日期之后(包括该日期当天)认为是bucket表





