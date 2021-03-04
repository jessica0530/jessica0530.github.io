---
layout: post
title: Hudi Metadata
categories: [hudi]
description: Hudi Metadata
keywords: hudi
---
分布 读写 交互 保证读写一致性

## Timeline

file.timestamp.deltacommit.request

2021022695006.commit



## Action 

Action 只有一些特定类型的表有 

## State

request

inflight （正在写入）

completed



## DeltaCommit



获取checkpoint时间



需不需要rollback(有没有 compaction的情况)

整个commit 结束要不要做 compact

Compact 结束后做 parquet版本的整理 



一个文件 512 兆左右

base path/partition/fileGroup

Deltacommit 无限增长的情况,要不要缩减



Temp 下做 marker.file

Log 文件一般是 append

Compaction.parquert 和log 进行 merge

find touched files(maker file 做 rollback的时候使用)，可以少访问文件做merge操作

锁和租约？ 等待这个作业是否过时

租约的恢复？？？



Log：  没有并发控制,不支持并发写



Filesystem view 是否实时更新

每个作业都有自己的filesystem view

Rollback ()

Mysql dump （最多往下消费一小时,自己做的）

## Compaction 异步

1.compact first log file append了一个新的数据,compact 时间线之后， 所有数据都过滤掉,保证数据完整性

2.commit -> compaction

Compaction 基于timeline 可以保证和commit同时执行

Read real-time

1. Get lastestInstant

2. listing files(文件数量很多的情况下,就很耗时)

   社区的方案 hudi 里面维护所有的list 可以快速过滤,维护在hdfs

## MetaTable Update

## Delta commit

## MetaTable Sync

Dataset timeline instants corresponding metaTable's

Write (数据提交完后 做 meta同步)

Read 社区 说不保证 先 sink 再read 只能 read MetaTable

但是 MetaTable 和DataSet不能保证是一致的

所以在某些 场景下 是不能保证一致的













