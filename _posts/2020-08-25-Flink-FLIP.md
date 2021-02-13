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



## FLIP-141 

引入了基于分数的方法来共享插槽内的托管内存。随着引入了也使用托管内存的python运算符，该方法需要扩展。该FLIP提出了一种设计，用于扩展python运算符和其他将来可能使用的托管内存使用案例的插槽内托管内存共享。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-141%3A+Intra-Slot+Managed+Memory+Sharing

## FLIP-142

新的 statebackend接口 ,便于用户理解

状态后端仅定义状态在TM上本地存储的位置和方式，而检查点存储定义在何处以及如何存储检查点以进行恢复。

| 老的                                              | 新的                                                         |
| :------------------------------------------------ | :----------------------------------------------------------- |
| MemoryStateBackend（）                            | HashMapStateBackend（）+ JobManagerCheckpointStorage（）     |
| FsStateBackend（）                                | HashMapStateBackend（）+ FileSystemCheckpointStorage（）     |
| RocksDBStateBackend（new MemoryStateBackend（）） | EmbeddedRocksDBStateBackend（）+ JobManagerCheckpointStorage（） |
| RocksDBStateBackend（new FsStateBackend（））     | EmbeddedRocksDBStateBackend（）+ FileSystemCheckpointStorage（） |
| MemoryStateBackend（“ file：// path”）            | HashMapStateBackend（）+ JobManagerCheckpointStorage（“ file：// path”） |

https://cwiki.apache.org/confluence/display/FLINK/FLIP-142%3A+Disentangle+StateBackends+from+Checkpointing



## FLIP-143 Unified Sink API 

如[FLIP-131中所述](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=158866741)，Flink将不推荐使用DataSet API，而推荐使用DataStream API和Table API。用户应该能够使用DataStream API编写支持有界和无界执行模式的作业。但是，Flink没有提供接收器API以确保在有界和无界场景中的语义都恰好一次，从而阻止了统一。 

因此，我们想引入一个新的统一接收器API，该API可使用户开发一次接收器并在任何地方运行它。特别是Flink允许用户 

1. 选择其他SDK（SQL / Table / DataStream）
2. 根据场景（有界/无界）选择不同的执行模式（批处理/流） 

我们希望这些东西（SDK /执行模式）对接收器API是透明的。

该文档包括三个部分：第一部分描述了统一接收器API应该支持的语义。根据第一部分，第二部分提出了一个新的统一接收器API。在最后一部分中，我们介绍了两个与API相关的开放性问题。

https://issues.apache.org/jira/browse/FLINK-19510

https://cwiki.apache.org/confluence/display/FLINK/FLIP-143%3A+Unified+Sink+API

## FLIP-144  Native Kubernetes HA  for flink

舍弃zookeeper做HA,

高可用性（aka HA）是生产中非常基本的要求。它有助于消除Flink群集的单点故障。对于Flink HA配置，群集中必须有多个JobManager，即活动和备用JobManager。一旦活动的JobManager发生异常故障，其他备用的Manager可以接管领导并从最新的检查点恢复作业。启动多个JobManager将使恢复更快。从纱线应用程序尝试或Kubernetes（又名K8s）部署中受益，可以轻松地连续或同时启动多个JobManager。

目前，Flink已提供Zookeeper HA，并已广泛用于生产环境中。它可以集成在独立集群，Yarn和Kubernetes部署中。但是，由于我们需要管理Zookeeper集群，因此在K8s中使用Zookeeper HA会花费额外的费用。同时，K8s提供了一些公共API来进行领导者选举和配置存储（即ConfigMap ）。我们可以利用这些功能，并使在K8上运行HA配置的Flink群集更加方便。

注意：[K8](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/kubernetes.html)和[独立K8上](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/kubernetes.html)的[独立](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/kubernetes.html)[版本](https://ci.apache.org/projects/flink/flink-docs-master/ops/deployment/native_kubernetes.html)都可以从新引入的***KubernetesHaService中受益\***。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-144%3A+Native+Kubernetes+HA+for+Flink

## FLIP-145 支持SQL窗口化 value 函数

重点看吧

https://issues.apache.org/jira/browse/FLINK-19604

此FLIP的主要目的是改善Flink的近实时（NRT）体验。我们建议支持窗口化表值函数（TVF）语法作为NRT用例的切入点。我们将解释为什么要做出这个决定，以及引入加窗TVF的好处。

通常，我们可以将数据处理分为**实时**（以秒为单位），**近实时**（以分钟为单位）和**批处理**（>小时）以内。Flink是众所周知的流处理系统，擅长实时场景。同时，社区为增强批处理能力付出了很多努力。然而，

我听到一些用户抱怨说，将Flink用于NRT用例很昂贵。据我们所知，NRT是非常普遍的情况。我们以前可能已经忽略了它。这就是为什么我们要对其进行改进，并使Flink成为NRT场景的强大引擎。

目标

为此我们需要做什么？我们调查了许多Flink流作业，发现了以下痛点。

- **学习曲线。**通常，用户使用Windows进行分钟/秒粒度统计。但是，当前在Flink SQL中不容易使用Windows。它仅支持窗口聚合，不支持窗口连接，窗口TopN，重复数据删除窗口。很难级联不同的操作（例如join，agg），用户必须学习如何保留时间属性和某些流特定的功能，例如`TUMBLE_ROWTIME` 。
- **表现。**Flink是本机流引擎，它可以提供低延迟，并降低每个记录状态操作的成本。但是在某些情况下，用户不需要这么低的延迟。如果可以将容忍的延迟换成吞吐率的极大提高，那就太好了。

在行业中，用户通常使用批处理引擎和调度程序来构建NRT管道。我们还调查了很多此类工作，发现大多数工作是15分钟和累积汇总（从0到当前分钟的汇总）。例如，在10:00时的累积UV数表示从00:00到10:00的UV总数。因此，每日报告是一条单调递增的线，如下图所示。Snowflake还提供了累积窗口聚合的[示例](https://docs.snowflake.com/en/sql-reference/functions-analytic.html#cumulative-window-frame-examples)。



https://cwiki.apache.org/confluence/display/FLINK/FLIP-145%3A+Support+SQL+windowing+table-valued+function

## FLIP-146

暂时不看吧

https://cwiki.apache.org/confluence/display/FLINK/FLIP-146%3A+Improve+new+TableSource+and+TableSink+interfaces



## FLIP-147  Support Checkpoints After TaskFinished 

批看完再看吧

https://cwiki.apache.org/confluence/display/FLINK/FLIP-147%3A+Support+Checkpoints+After+Tasks+Finished

## FLIP-148 FLINK 批处理的Sort-Merge Based Blocking

批处理,等看完flink 批处理再说吧

基于散列的阻塞重排和基于排序合并的阻塞重排是现有的分布式数据处理框架广泛采用的两个主要阻塞重排实现。基于哈希的实现将发送到不同的reducer任务的数据同时写入单独的文件中，而基于排序合并的方法将这些数据一起写入单个文件中，并将这些小文件合并为更大的文件。与基于排序合并的方法相比，基于哈希的方法在运行大规模批处理作业时有几个弱点：

1. **稳定性：**对于高度并行（成千上万）批处理作业，当前基于散列的阻塞混洗实现会同时写入太多文件，这给文件系统带来了巨大压力，例如，维护了太多文件元，索引节点或文件描述符耗尽。所有这些都可能是潜在的稳定性问题。基于排序合并的阻止混洗没有问题，因为对于一个结果分区，一次只能写入一个文件。
2. **性能：** 大量小的随机播放文件和随机IO会严重影响随机播放性能，特别是对于HDD（对于ssd，由于预读和缓存，顺序读取也很重要）。对于处理海量数据的批处理作业，由于高度并行性，每个子分区的数据量很少。此外，数据偏斜是子分区文件较小的另一个原因。通过将所有子分区的数据合并到一个文件中，可以实现更多顺序读取。
3. **资源：**对于当前基于散列的实现，每个子分区至少需要一个缓冲区。对于大规模的批量改组，内存消耗可能非常大。例如，如果并行度设置为10000，并且每个结果分区至少需要320M网络内存，并且由于巨大的网络消耗，很难为大规模批处理作业配置网络内存，并且有时由于以下原因不能增加并行度：网络内存不足会导致不良的用户体验。

通过将基于排序合并的方法引入Flink，我们可以提高Flink运行大规模批处理作业的能力。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-148%3A+Introduce+Sort-Merge+Based+Blocking+Shuffle+to+Flink

## FLIP-149 引入 upsert-kafka 

这个 就先不看把,内部已经实现一个版本了

https://cwiki.apache.org/confluence/display/FLINK/FLIP-149%3A+Introduce+the+upsert-kafka+Connector

## FLIP-150 引入 Hybrid Source

这个必看。。



https://cwiki.apache.org/confluence/display/FLINK/FLIP-150%3A+Introduce+Hybrid+Source

## FLIP-151  基于堆的State 的增量快照

**重点看**

当前，使用最广泛的Flink状态后端是基于RocksDB和Heap的。

与RocksDB相比，基于堆的优点如下：

1. 每个检查点序列化一次，而不是每个状态修改一次

2. 1. 这允许“挤压”更新相同的密钥
   2. （但也可能是不利的，因为不会在检查点上分摊序列化）

3. 同步阶段更短（与RocksDB增量比较）

4. 无需分类

5. 无需压实

6. 无IO放大

7. 没有JNI开销

这样可以潜在地提高吞吐量和效率。

但是堆后端的使用受到以下限制：

1. 状态必须适合记忆
2. 缺少增量快照

该FLIP旨在解决后者。

例如，[Pinterest提出](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/FsStateBackend-vs-RocksDBStateBackend-td32480.html)了一个[问题](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/FsStateBackend-vs-RocksDBStateBackend-td32480.html)，其中[提出](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/FsStateBackend-vs-RocksDBStateBackend-td32480.html)了适合内存[的250G状态部署](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/FsStateBackend-vs-RocksDBStateBackend-td32480.html)（2020年1月30日）。**
**

### 解决方案

提议的解决方案包括：

1. 计算增量，包括：

2. 1. 一组更改的密钥（包含在密钥组中的密钥）
   2. 对于每个这样的键，状态更改（例如，附加列表值） 

3. 编写快照（迭代和序列化）

4. 参考先前的快照

5. 恢复：遍历快照并应用差异

6. 清理（压缩）



https://cwiki.apache.org/confluence/display/FLINK/FLIP-151%3A+Incremental+snapshots+for+heap-based+state+backend



Merge-on-read 不行  ？？



## FLIP-152

Hive Query 语法兼容性



应该有需求 要看

FLIP-123实现了与HiveQL兼容的DDL，因此用户可以在HiveQL中处理元数据。该FLIP旨在为查询提供语法兼容性。与FLIP-123类似，此FLIP将改善与Hive的互操作性并减少迁移工作。此外，该FLIP还可以扩展HiveQL以支持流功能。借助此FLIP，可以支持以下典型用例：

1. 用户可以将其批处理Hive作业迁移到Flink，而无需修改SQL脚本。
2. 用户可以编写HiveQL以将流功能与Hive表集成在一起，例如从Kafka到Hive的流数据。
3. 用户可以批量或在流作业中编写HiveQL来处理非Hive表。

对于迁移的用户，我们认为希望他们能够继续编写Hive语法。它不仅使迁移更加容易，而且还帮助他们更快地将Flink用于新的方案，从而提供统一的批处理流体验。

https://cwiki.apache.org/confluence/display/FLINK/FLIP-152%3A+Hive+Query+Syntax+Compatibility

## FLIP-153

Python API

https://cwiki.apache.org/confluence/display/FLINK/FLIP-153%3A+Support+state+access+in+Python+DataStream+API



## FLIP-154 SQL 隐式转换的

![SQL-隐式转换](/Users/jessica/ideaproject-github/jessica0530.github.io/images/posts/SQL-隐式转换.png)

https://cwiki.apache.org/confluence/display/FLINK/FLIP-154%3A+SQL+Implicit+Type+Coercion

## FLIP-155 引入更便捷的Table API

Table API是用于流和批处理的统一的关系API，它与SQL共享相同的基础查询优化和查询执行堆栈。随着SQL中添加越来越多的特性和功能，Table API也变得越来越强大，因为它们之间共享了大多数优化和功能。 

对于Table API本身，社区也在不断改进它，包括但不限于以下方面：

- 我们支持几种基于行的操作，例如[FLIP-29中的](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=97552739)map / flatMap / aggregate / flatAggregate和基于列的操作，例如Flink 1.9中的addColumns / renameColumns / dropColumns / addOrReplaceColumns

- Expression DSL已在[FLIP-55](https://cwiki.apache.org/confluence/display/FLINK/FLIP-55%3A+Introduction+of+a+Table+API+Java+Expression+DSL)的Table API中引入，这大大提高了可用性。
- 在[FLIP-129中](https://cwiki.apache.org/confluence/display/FLINK/FLIP-129%3A+Refactor+Descriptor+API+to+register+connectors+in+Table+API)，它提出了重构现有的Descriptor API的方法，以填补Descriptor API和SQL DDL之间的功能空白。
- 从Flink 1.9开始，还支持Python Table API，它允许用户使用Python语言编写Table API程序。



在此FLIP中，我们希望通过引入一些新方法来不断改进Table API，因为Table API当前存在一些问题：Table API难以表达某些任务，例如重复数据删除，topn等，或当表中有数百列时，例如，空数据处理等，不容易表达

```java
Java：
table.deduplicate（new Expression [] {$（“ a”），$（“ b”）}，$（“ proctime”），true）

Python：
table.deduplicate（[table.a，table.b] ，table.proctime，True）
```

我感觉。。我更难懂了,Table API 优先级 最低吧

https://cwiki.apache.org/confluence/display/FLINK/FLIP-155%3A+Introduce+a+few+convenient+operations+in+Table+API

## FLIP-156 细粒度资源请求

**重点看！！**

??这个 mr 被 closed了 没合入吗？？

Flink当前采用一种**粗粒度的资源管理**方法，该方法将任务部署到预定义的通常相同的插槽中，而无需考虑每个插槽包含多少资源。通过[插槽共享](https://ci.apache.org/projects/flink/flink-docs-stable/concepts/flink-architecture.html#task-slots-and-resources)，可以将同一**插槽共享组（SSG）**中的任务部署到一个插槽中，而不管每个任务/操作员需要多少资源。在[FLIP-56中](https://cwiki.apache.org/confluence/display/FLINK/FLIP-56%3A+Dynamic+Slot+Allocation)，我们提出了**细粒度的资源管理**，它相对于工作负载的资源需求，利用具有不同资源的插槽来执行任务。

对于许多作业而言，就资源利用率和可用性而言，使用粗粒度资源管理并将所有任务简单地放入一个SSG中就足够了。

- 对于所有任务具有相同并行性的许多流作业，每个插槽将包含整个管道。理想情况下，所有管道应使用大致相同的资源，可以通过调整相同插槽的资源来轻松满足这些资源。
- 任务的资源消耗随时间而变化。当一个任务的消耗减少时，多余的资源可以被另一个消耗增加的任务使用。这被称为**削峰和填谷**效果，减少了所需的总资源。

但是，在某些情况下，粗粒度资源管理无法正常运行。

- 任务可能具有不同的并行性。有时，无法避免这种不同的并行性。例如，源/接收/查找任务的并行性可能受到外部上游/下游系统的分区和IO负载的限制。在这种情况下，任务少的插槽比任务整条插槽所需的资源少。
- 有时，整个管道所需的资源可能太多，无法放入单个插槽/任务管理器中。在这种情况下，需要将管道拆分为多个SSG，这些SSG可能并不总是具有相同的资源要求。
- 对于批处理作业，并非所有任务都可以同时执行。因此，管道的瞬时资源需求随时间而变化。

尝试执行具有相同插槽的所有任务可能会导致资源利用不理想。相同插槽的资源必须能够满足最高资源要求，这对于其他要求是浪费的。当涉及到昂贵的外部资源（如GPU）时，这种浪费变得更加难以承受。

因此，在这种情况下，需要细粒度的资源管理，它利用不同资源的插槽来提高资源利用率



https://cwiki.apache.org/confluence/display/FLINK/FLIP-156%3A+Runtime+Interfaces+for+Fine-Grained+Resource+Requirements

## FLIP-158

**重点看！！**

Incremental Checkpoint

建立一种方法，以大幅度减少跨状态后端的流应用程序的检查点间隔，无论规模大小，都可靠。即使是更大的规模（> 100个节点，状态TB），我们的目标间隔也是几秒钟。
根据用户对此功能的采用以及进一步的要求，此处的体系结构还可以作为将来进一步减少检查点间隔的基础。

更快的检查点间隔对流应用程序有很多好处：

- 减少恢复工作。检查点越频繁，恢复后需要重新处理的事件就越少。
- 事务接收器的延迟较低：事务接收器在检查点上提交，因此更快的检查点意味着更频繁的提交。
- 更可预测的检查点间隔：当前，检查点的长度取决于需要在检查点存储中保留的工件的大小。
  例如，如果RocksDB自上一个检查点以来仅创建了一个新的Level-0 SST，则该检查点将很快。
  但是，如果RocksDB完成新的压缩并为Level-3 / -4 / -5创建大型SST，则检查点将花费更长的时间。
- 频繁的检查点间隔使Flink在将接收器数据写入外部系统之前将其保存在检查点中（预写日志样式），而不会增加太多延迟。对于不能很好地公开事务API的系统，这可以简化接收器的设计。例如，由于卡夫卡（Kafka）的交易方式，特别是缺乏很好地恢复交易（而依赖于交易超时）的原因，一次恰好一次的卡夫卡接收器目前非常复杂。

此外，此处提出的方法还将有助于减少将RocksDB与增量检查点一起使用时可能发生的小文件碎片问题。

https://www2.cs.duke.edu/courses/cps296.4/fall13/838-CloudPapers/dean_longtail.pdf

https://issues.apache.org/jira/browse/FLINK-21352



https://cwiki.apache.org/confluence/display/FLINK/FLIP-158%3A+Generalized+incremental+checkpoints

## FLIP-159 

## Reactive Mode

**重点看！！！**

Reactive Mode

运行数天或更长时间的流作业通常会在其生命周期内遇到工作负载的变化。这些变化可能源于季节性高峰，例如白天与黑夜，工作日与周末或假日与非节假日，突发事件或产品的日益流行。这些更改中的某些更改比其他更改更可预测，但是所有这些更改的共同点在于，如果您希望为客户保持相同的服务质量，它们会更改您工作的资源需求。

即使您可以估计最大资源需求的上限，从一开始就使用最大资源几乎总是非常昂贵。因此，如果Flink可以利用在作业开始后可用的资源（TaskManagers），那就太好了。 同样，如果基础资源管理系统决定在其他位置需要一些当前分配的资源，则Flink应该不会失败，而是在资源被吊销后将其缩减规模。这样的行为会使Flink成为相应资源管理者的好公民。

理想情况下，Flink将控制资源（取消）分配。但是，并非每个Flink部署都知道底层的资源管理系统。此外，从应用程序开发人员（外部参与者）的角度确定实际资源需求可能会更容易。因此，我们提出了一种反应式执行模式，该模式使Flink可以通过向上或向下缩放作业来对新可用或已删除的资源做出反应，以尽可能地利用可用资源。

通过让外部服务监视某些指标，例如使用者延迟，总CPU利用率，吞吐量或延迟，Reactive模式使Flink用户可以实现强大的自动缩放机制。一旦这些指标超过或超过某个特定阈值，便可以从Flink群集中添加或删除其他TaskManager。这可以通过更改Kubernetes部署或[自动](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)伸缩组的[副本](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#replicas)因子来实现。

https://issues.apache.org/jira/browse/FLINK-10407

https://cwiki.apache.org/confluence/display/FLINK/FLIP-159%3A+Reactive+Mode

## FLIP-160

为了支持  Reactive Mode, 需要一种不同类型的调度程序, 该调度程序首先需要宣布所需的资源,并且在收到资源后才决定执行作业的实际并行度,这样的好处是,如果未满足所有必需的资源,此调度程序依旧可以调度资源,就算 TaskManagers 即是丢失了

**这个重点看！！！！**

附上 PR 并结合 公司的代码一起看

https://issues.apache.org/jira/browse/FLINK-21075

https://cwiki.apache.org/confluence/display/FLINK/FLIP-160%3A+Declarative+Scheduler

声明式调度程序将首先仅对流作业起作用。这将大大简化事情，因为我们总是必须安排所有操作员。此外，通过将每个故障视为重新启动整个拓扑的全局故障转移，我们可以进一步简化调度程序。无论如何，如果许多流拓扑不包含分离的图，则此故障转移行为是默认的。鉴于这些假设，我们希望开发以下调度程序：

调度程序将使用JobGraph，首先为其计算所需的资源。声明这些资源后，调度程序将等待，直到可用资源稳定为止。一旦资源稳定下来，调度程序就应该能够决定工作的实际并行度。一旦确定了并行性，并且执行与可用的插槽匹配，调度程序就会部署执行。

每当发生故障时，我们都会使整个作业失败并尝试重新启动它。重新启动是通过取消所有已部署的任务，然后按照与初始调度操作相同的代码路径重新启动JobGraph的调度来进行的。

与现有的流水线区域调度程序相比，此实现的明显回归是，我们始终在重新启动整个拓扑。对于尴尬的并行作业，由于正在运行的任务不需要重置为最新的检查点，因此可能不需要这样做。支持部分故障转移将是建议的调度程序的第一个扩展。支持部分故障转移的一种方法是在全局故障转移和本地故障转移之间引入区别。

- **全局故障转移**：重新启动整个拓扑，从而可以更改作业的并行性
- **本地故障转移**： 重新启动执行的子集，这不会改变操作员的并行性

如果系统由于没有足够的可用插槽而无法从本地故障转移中恢复，则必须升级该系统，以使其成为全局故障转移。全局故障转移将允许系统重新调整整个作业。

## FLIP-161

多环境的配置

允许通过环境变量覆盖此配置，可以使配置更加灵活

方案感觉并没有订

https://github.com/lightbend/config



https://cwiki.apache.org/confluence/display/FLINK/FLIP-161%3A+Configuration+through+envrionment+variables



## FLIP-162

许多与时间相关的函数（例如PROCTIME（），NOW（），CURRENT_DATE，CURRENT_TIME和CURRENT_TIMESTAMP）基于UTC + 0时区返回时间值。

引入 function配置 来影响其代码生成 ————这个加配置不太友好,直接配置全局时区？

要考虑时间窗口的 会话时区偏移量

还有 StreamRecord 中的 时间戳 也需要 是 基于时区的 

这块大致知道方案 以后再看

https://cwiki.apache.org/confluence/display/FLINK/FLIP-162%3A+Consistent+Flink+SQL+time+function+behavior



## Flip-163

社区 sql-client 支持的功能比较弱？

增加了 -i 去初始化 catalog表

连 -f 都不支持的吗？？？multi insert into 都不支持的？？？

理这块感觉很花时间, 先把 multi insert 和 common view理了吧

https://cwiki.apache.org/confluence/display/FLINK/FLIP-163%3A+SQL+Client+Improvements


##Flip-164

similar APIs in the Catalog interfaces such that catalog implementations can define table/views in a unified way
在 Catalog 层面定义 获取 schema的统一 API，可以给 DDL，DataStreamAPI,Catalog API 使用
https://cwiki.apache.org/confluence/display/FLINK/FLIP-164%3A+Improve+Schema+Handling+in+Catalogs