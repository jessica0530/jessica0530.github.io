---
layout: post
title: Flink FLIP
categories: [flink]
description: flink flip
keywords: flink
---

# Flink FLIP

## Flip-1 

当任务在执行过程中失败时，Flink当前会重置整个执行图，并从上一个完成的检查点触发完全重新执行。这比仅重新执行失败的任务要昂贵

对于许多流作业，此行为并不重要，因为许多任务与其前任（上游）或后继（下游）具有所有依赖关系（keyBy，事件时间）。在那种情况下，只要一项任务没有交付输入或接受输出，操作员通常就无法取得进展。完全重新启动仅意味着这些任务还会重新计算其状态，而不是处于空闲和等待状态。

更细粒度的恢复可以帮助减少恢复时需要转移的状态量。如果只有1/100个操作员需要恢复其状态，则一个操作员具有到检查点的持久性存储的全部带宽，而不是与恢复其状态的其他操作员共享该带宽。

对于某些流作业，完全重新启动的代价是不必要的。特别是对于尴尬的并行作业（没有keyBy（）或redistribute（）操作），其他并行子任务/分区可以继续运行，并且流程序整体上将取得进展。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-1+%3A+Fine+Grained+Recovery+from+Task+Failures

### 解决方案-在中间结果处限制流水线连接的组件

为了进一步减少需要重新启动的任务数量，我们可以使用某些类型的数据流交换。在运行时中，它们被称为“中间结果类型”，因为在运算符之间交换的每个数据流都表示一个中间结果。

#### 缓存中间结果

这种类型的数据流会缓存自最新检查点以来的所有元素，如果数据超出内存容量，则可能会将其溢出到磁盘。

当下游操作员从该检查点重新启动时，它可以简单地重新读取该数据流，而无需生产操作员重新启动。适用于批处理（有界）和流传输（无界）操作。当不使用检查点（批量）时，它需要缓存所有数据。

#### 纯内存缓存中间结果

与缓存中间结果类似，但是一旦超过内存缓冲容量，就丢弃发送的数据。充当恢复的“尽力而为”助手，当检查点足够频繁以将数据保存在内存中的检查点之间时，它将限制恢复。另一方面，它绝对是免费的，它只使用了内存，否则将永远不会使用。

#### 阻止中间结果

这仅适用于有限的中间结果（批处理作业）。这意味着消耗操作员仅在产生整个有界结果之后才开始。这限制了批处理作业中下游的取消/重新启动。

ExecutionVertex：表示ExecutionJobVertex的其中一个并发子任务，输入是ExecutionEdge，输出是IntermediateResultPartition![failover-region](/Users/jessica/ideaproject-github/jessica0530.github.io/_posts/failover-region.png)



## FLIP-5

和 LatencyMarker sub-task  发送并发度 *并发度 是一个问题

https://cwiki.apache.org/confluence/display/FLINK/FLIP-5%3A+Only+send+data+to+each+taskmanager+once+for+broadcasts



## FLIP-6 

Flink 在各个资源管理 上的Deployment

https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077



## FLIP-8

Union STATE?

https://cwiki.apache.org/confluence/display/FLINK/FLIP-8%3A+Rescalable+Non-Partitioned+State

## FLIP-12

Async IO -维表

https://cwiki.apache.org/confluence/display/FLINK/FLIP-8%3A+Rescalable+Non-Partitioned+State



## FLIP-13

SideOutput 打标签 输出  Window数据延迟输出

https://cwiki.apache.org/confluence/display/FLINK/FLIP-13+Side+Outputs+in+Flink

## FLIP-18

代码生成,用户提高排序性能

https://cwiki.apache.org/confluence/display/FLINK/FLIP-18%3A+Code+Generation+for+improving+sorting+performance



## FLIP-19

BLOB System

https://cwiki.apache.org/confluence/display/FLINK/FLIP-19%3A+Improved+BLOB+storage+architecture

## FLIP-21

对象 Copy

https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=71012982



## FLIP-25

TTLState

https://cwiki.apache.org/confluence/display/FLINK/FLIP-25%3A+Support+User+State+TTL+Natively

## Flip-27

Source 接口重构

https://cwiki.apache.org/confluence/display/FLINK/FLIP-27%3A+Refactor+Source+Interface



## FLIP-30

CATALOG API

https://cwiki.apache.org/confluence/display/FLINK/FLIP-30%3A+Unified+Catalog+APIs

## FLIP-36

交互式编程里面 增加 CacheTable

一般而言，应用程序可能包含一个或多个作业，并且他们可能希望与其他人共享数据。在Flink中，同一应用程序中的作业是独立的，彼此之间不共享任何内容。如果Flink应用程序涉及几个连续的步骤，则每个步骤（作为一个独立的作业）都必须将其中间结果写入外部接收器，以便后续步骤（作业）可以将其结果用作源。

尽管从功能上来说可行，但此编程范例仍存在一些缺点：

1. 为了共享结果，必须提供接收器。
2. 由于中间结果上存在大量IO，因此复杂的应用程序效率低下。
3. 使用编程API的用户的用户体验会减弱（SQL用户不会成为受害者，因为临时表是由框架创建的）

事实证明，交互式编程支持对于批处理方案中Flink上的用户体验至关重要。以下代码给出了一个示例：

https://cwiki.apache.org/confluence/display/FLINK/FLIP-36%3A+Support+Interactive+Programming+in+Flink



## FLIP-41

统一 Binary Format 对于Keyed State

https://cwiki.apache.org/confluence/display/FLINK/FLIP-41%3A+Unify+Binary+format+for+Keyed+State



## FLIP-43

Flink为用户功能提供状态抽象，以确保对流进行容错处理。用户可以使用非分区状态和分区状态。

分区状态接口提供对不同类型的状态的访问，这些状态的范围都限于当前输入元素的键。这种类型的状态仅在通过stream.keyBy（）创建的键控流内部可用。

当前，所有此状态都在Flink内部，并用于在故障情况下提供处理保证（例如，一次处理）。从外部访问状态的唯一方法是通过Queryable状态，但这仅限于只读操作，一次只能操作一个键。

状态处理器API提供了强大的功能，可以使用Flink的批处理DataSet API 读取，写入和修改保存点。

这对于以下用途很有用：

- 分析状态以获得有趣的模式

- 通过检查状态差异来对作业进行故障排除或审核

- 新应用程序的引导状态

- 修改保存点，例如：

- - 改变最大并行度
  - 进行重大的模式更改
  - 纠正无效状态

https://cwiki.apache.org/confluence/display/FLINK/FLIP-43%3A+State+Processor+API

## FLIP-44

Local Agg  感觉就是 代码生成的时候不加 isStateBackend

https://cwiki.apache.org/confluence/display/FLINK/FLIP-44%3A+Support+Local+Aggregation+in+Flink



## FLIP-45

当前，在我们发布的版本[1]中，主要有两种完成工作的方法：停止和取消，它们之间的区别如下：

- 在取消调用时，作业中的操作员会立即收到cancel（）方法调用，以尽快将其取消。如果取消调用后操作员没有停止，Flink将开始定期中断线程，直到其停止。
- “停止”调用是停止正在运行的流作业的一种更合适的方法。停止仅适用于使用实现StoppableFunction接口的源的作业。当用户请求停止工作时，所有源都将收到stop（）方法调用。作业将一直运行，直到所有源均正确关闭为止。这使作业可以完成对所有飞行数据的处理。

但是，对于具有保留检查点的有状态运算符，stop调用将不会使用任何检查点，因此，在恢复作业时，需要通过源倒带从最新的检查点恢复作业，这导致等待处理所有运行中数据毫无意义（所有操作都需要再次处理）。换句话说，在这种情况下，停止和取消之间没有真正的区别，因此在概念上存在歧义。

另一方面，在[FLIP-34](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=103090212)之后的最新主分支中 ，作业停止总是伴随着一个保存点，这具有以下问题：

- 从用户角度来看，这是意外的行为更改，旧的stop命令将在没有保存点配置的作业上失败。
- 这会减慢作业停止过程的速度，并且在争用资源时可能会阻止启动新作业。

本文档旨在增强作业停止的语义，增加正常停止（不带保存点）的支持，以及在启用保留检查点时防止不必要的源倒带。为了实现这些目标，我们将首先讨论作业停止和取消之间以及检查点和保存点之间的概念差异，类似于数据库系统中的概念。然后，我们将描述如何增强作业停止语义以及如何实现它。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-45%3A+Reinforce+Job+Stop+Semantic

## FLIP-47

我觉得 除了 增量 和触发方式 确实没啥区别

| **Y** | Yes, the feature is supported    |
| ----- | -------------------------------- |
| **N** | No, the feature is not supported |
| **M** | Supported but not in all cases   |

|             | user-controlled | incremental | self-contained | side-effects | recovery | rescaling | unified format |
| ----------- | --------------- | ----------- | -------------- | ------------ | -------- | --------- | -------------- |
| savepoints  | **Y**           | **N**       | **Y**          | **Y**        | **Y**    | **Y**     | **Y**          |
| checkpoints | **M**           | **Y**       | **Y**          | **Y**        | **Y**    | **M**     | **N**          |



尽管创建保存点和检查点时会考虑到不同的语义和假设，但是在此过程中添加的一些功能使这些行变得模糊，并使这些假设在所有情况下均不成立，并给用户带来了潜在的警告。另外，这两个功能的某些语义没有明确定义，对用户造成负面影响。

该提案： 

1. 讨论检查点和保存点之间的关系， 
2. 尝试在统一的视角下修复其语义，基于该视角，检查点和保存点都可以视为状态快照，并且
3. 基于上述内容，它提出了一些附加措施，旨在减少用户用脚射击自己的风险。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-47%3A+Checkpoints+vs.+Savepoints



## FLIP-48

Intermediate Result 我还没搞懂是啥

https://cwiki.apache.org/confluence/display/FLINK/FLIP-48%3A+Pluggable+Intermediate+Result+Storage



## FLIP-49

内部 on-heap /off-heap的配置

https://cwiki.apache.org/confluence/display/FLINK/FLIP-49%3A+Unified+Memory+Configuration+for+TaskExecutors



## Flip-50

HeapKeyedState 可以 spill 到磁盘, 问题是 如果是 排序 或者是 查询的话 怎么办

https://cwiki.apache.org/confluence/display/FLINK/FLIP-50%3A+Spill-able+Heap+Keyed+State+Backend

## FLIP-53

资源 声明配置调度

https://cwiki.apache.org/confluence/display/FLINK/FLIP-53%3A+Fine+Grained+Operator+Resource+Management