---
layout: post
title: Spark Core
categories: [spark]
description: Spark core
keywords: spark
---



## RDD

什么是RDD

如何生成RDD

RDD内partition个数如何确定 (id: task id, attemped id, block id)

(task id /attempt id 可以在task scheduler)

RDD间依赖关系

## 如何划分 Job/ Stage /Task

Transformation vs Action

按Action 划分 Job

按Shuffle 划分 Stage

Stage 内最后一个RDD的Partition 数决定了该 Stage 内Task个数 

## Shuffle机制演进 (block id 计算映射关系)

BlockManager,Zero copy

Hash shuffle -> Bypassshuffle

Sort shuffle

Unsafe Shuffle

HDFS based shuffle

Reducer-centric shuffle



## 作业调度与Exactly once机制

1.group by 是否都需要shuffle，比如有 bucket时

2.UNsafe shuffle

3.join 时默认Partition个数

4.Spark Core 与Spark SQL

5.Spark SQL 与 RDD的关系



DAGScheduler/TaskScheduler/SchedulerBackend/MapOutputTracker/OutputCommitter

1.DAGScheduler 负责DAG 生成,Stage 划分与提交

2.TaskScheduler 负责Task 调度

3.SchedulerBackend 负责资源申请

4.MapOutputTracker 负责跟踪每次Shuffle write 结果的元信息

5.OutputCommitter 保证Job结果输出时的ExactlyOnce

6.Dynamic Allocation 原理

7.Delay Scheduling 与Data Locality原理



