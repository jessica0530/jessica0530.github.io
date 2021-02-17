---
layout: post
title: Flink 对接Hive
categories: [flink]
description: flink 对接Hive
keywords: flink
---

# 流数据写入Hive的实现原理





## StreamingFileWriter



每个 subtask 都往不同的Bucket去写数据,每个subtask写Bucket同一时间会维持3种文件



In-progress Files 表示正在写的文件



Pending Files 表示文件已经写完了但是还没有提交



Finished Files 表示文件已经写完并且也已经提交了



## StreamingFileCommitter

在StreamingFileWriter 后执行,用来提交分区,对于非分区就不需要了



当StreamingFileWriter 的一个分区数据准备好后,StreamingFileWriter就会向 StreamingFileCommitter发一个 Commit Message，Commit Message 告诉告诉StreamingFileCommitter那些数据已经准备好了,然后进行提交的触发Commit Trigger



