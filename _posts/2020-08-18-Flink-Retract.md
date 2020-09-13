---
layout: post
title: Flink Retract详解
categories: [flink]
description: flink
keywords: flink
---
# 流计算的本质
对无边界，无序的数据源，允许按数据本身的特征进行窗口计算，得到基于事件发生时间的有序结果，并能在准确性、延迟程度和处理成本之间调整。
流计算的本质，就是平衡正确性，延时和资源这三者的关系

# 流计算处理数据步骤 
处理数据 可以分为4个步骤: 
## WHAT
What results are calculated? = transformations.
## WHERE
Where in event time are results calculated? = windowing.  

决定哪些事件发生时间段（where）的数据被分组到一起来进行聚合操作，在事件时间的哪个地方计算结果

## WHEN
When in processing time are results materialized? = triggers + watermarks.  

决定在什么处理时间（when）窗口的聚合结果被处理输出成一个窗格,在什么时间点，可以输出结果

## WHEN 3种触发方式
Early/On-time/Late trigger：
Zero or more early panes：在watermark经过窗口之前，即周期性的输出结果。这些结果可能是不准的()，但是避免了watermark 输出太慢的问题。
无窗口”或无界聚合和流内部连接，窗口化（具有早期发射）聚合和流内部连接(inner join)

A single on-time pane：仅在watermark通过窗口结束时触发一次。这时的结果可以看作是准确的。

Zero or more late panes：在watermark通过窗口结束边界之后，如果这个窗口有late event，也可以触发计算。这样就可以随时更新窗口结果，避免了输出太快导致的结果不对的问题

## How
How do refinements of results relate? = Disard,Acc,AccRetract

# How - 修正数据的方式  

## Discard 抛弃

Discard 抛弃 窗口触发后，窗口内容被抛弃，而之后窗口计算的结果和之前的结果不存在相关性。当下游的数据消费者,另外，抛弃因为不需要缓存历史数据，因此对比其他两种模式，抛弃模式在状态缓存上是最高效的。

## AccMode 累积

AccMode 累积：触发后，窗口内容被完整保留住持久化的状态中，而后期的计算结果成为对上一次结果的一个修正的版本。这种情况下，当下游的消费者收到同一个窗口的多次计算结果时，会用新的计算结果覆盖掉老的计算结果。这也是Lambda架构使用的方式，流处理管道产出低延迟的结果，之后被批处理管道的结果覆盖掉。

## AccRetract 累积和撤回

AccRetract  累积和撤回：

早期的计算结果随着输入的增加可能变得无效,触发后，在进行累积语义的基础上，计算结果的一份复制也被保留到持久化状态中。

当窗口将来再次触发时，上一次的结果值先下发做撤回处理，然后新的结果作为正常数据下发。

如果数据处理管道有多个串行的GroupByKeyAndWindow操作时，撤回是必要的，因为同一个窗口的不同触发计算结果可能在下游会被分组到不同键中去。

在这种情况下，除非我们通过一个撤回操作，撤回上一次聚合操作的结果，否则下游的第二次聚合操作会产生错误的结果。

## 修正方式的成本
Discarding < AccMode < AccRetractMode
不缓存历史数据 (窗口 无早期和Late record) < history state 进行累积更新< history state ,保存上一次结果,发两个数据 导致网络流量增加 

# Retract优化规则
Retract优化规则, 如果earliy late,情况下 如何选择AccMode,AccRetractMode，如果遇到late event，要如何修改窗口之前输出的结果呢？


Discarding（抛弃）：每个窗口产生输出之后，其state都被丢弃。也就是各个窗口之间完全独立。比较适合下游是聚合类的运算，比如对整数求和。

Accumulating（累积）：所有窗口的历史状态都会被保存，每次late event到了之后，都会触发重新计算，更新之前计算结果。这种方式适合下游是可更新的数据存储，比如HBase/带主键的RDS table等。

Accumulating & Retracting（累积&撤回）：Accumulating与第二点一样，即保存窗口的所有历史状态。撤回是指，late event到来之后，出了触发重新计算之外，还会把之前窗口的输出撤回。以下两个case非常适合用这种方式：

如果窗口下游是分组逻辑，并且分组的key已经变了，那late event的最新数据下去之后，不能保证跟之前的数据在同一个分组，因此，需要撤回之前的结果。

## 举例

动态窗口中，由于窗口合并，很难知道窗口之前emit的老数据落在了下游哪些窗口中。因此需要撤回之前的结果。
以例子中第二个窗口[12:02,12:04)为例，我们分别看看三种模式的输出结果：

|              | Discarding | Accumulating | Accumulating & Retracting |
| ------------ | ---------- | ------------ | ------------------------- |
| Inputs=[7,3] | 10         | 10           | 10                        |
| Inputs=[8]   | 8          | 18           | -10, 18                   |
| Total Sum    | 18         | 28           | 18                        |
|              |            |              |                           |

Discarding（抛弃）：同一个窗口的每次输出，都与之前的输出完全独立。本例子中，要算求和的话，只需要把窗口的每次输出都加起来即可。因此Discarding 模式对下游是聚合（SUM/AGG）等场景非常何时。

Accumulating（累积）：窗口的会把之前所有state都保存，因此同一个窗口的每个输出，都是之前所有数据的累积值。本例子中，该窗口第一次输出是10，第二次输入是8，之前的状态是10，所以输出是18。如果下游计算直接把两次输出加起来，结果就是错的。

Accumulating & Retracting（累积&撤回）：窗口的每个输出，都有一个累积值和一个撤回值。本例中，第一次输出10，第二次输出的是[-10,18]，因此下游把窗口的所有输出求和，会减去之前的重复值，得到正确结果18.

Retract是数据流的重要构建块，用于优化流式传输中的早期触发结果。

“早期发射”非常普遍，并且在许多流场景中广泛使用，例如“无窗口”或无界聚合和流内部连接，窗口化（具有早期发射）聚合和流内部连接(inner join)。

主要有两种情况需要撤消：

1）对密钥表进行更新（密钥是源表上的primaryKey（PK），或聚合中的groupKey / partitionKey）; 

2）当使用动态窗口（例如，会话窗口）时，由于窗口合并，新值可能正在替换多于一个的先前窗口。
如果没有 early-fired 就不需要回撤

一个场景 两种模式的处理结果

![image-20200913161844100](/images/posts/flink-retract.png)

## Retract 解决方案

从上面的例子中，我们了解到一些Operator需要额外的消息来帮助改进上游的早期触发结果。
为了帮助这些Operator，我们添加了附加信号，以指示来自上游表的数据是新键上的“添加”还是具有新值的现有键上的“替换”。
对于新键上的“添加，我们只需要发送一个数据并将其标记（每个数据的附加属性）作为累积消息。
对于替换，我们基本上必须发送两个数据，一个作为具有旧值的Retract消息，而另一个作为具有新值的Accumulate消息。 


### 表的Trait

#### Replace 表   

现有键上的"替换",对key 做替换操作,就是替换表  AccRetract(updateAsRetract  newRow,deleteRow ) (早期发射的)
No-Window GroupBy, Early-fired Window, Join left right 

#### Append  表   

新键上的添加 update (new Row) AccMode

![image-20200913162143467](/images/posts/flink-retract-2.png)

#### needRetract 

无法独立完成结果细化(改进)的运算符需要回撤消息。这通常是因为旧记录和新记录具有不同的Key，并且Operator无法在不收回消息的情况下收回旧记录
(Input of Aggregate),(Input of Sink),(join key 和上游是不一致的)

![image-20200913162456545](/images/posts/flink-retract-3.png)

### Retract Rule

解决退回问题的一种简单方法是让所有Replace表生成Retract消息，所有具有NeedRetract输入的运算符处理Retract消息。

但是"生成和处理撤消消息不是免费的”：

a）如果替换表始终生成撤消消息，则会导致网络流量加倍;

 b）对于大多数聚合体，通常处理回撤的执行计划通常比不处理回缩的执行计划更昂贵。例如，在MAX聚合中，如果需要处理退回，则聚合函数必须记住所有记录，否则当Retract消息要求收回当前MAX值时，系统将无法选择新MAX值（如果此MAX出现多次，则为相同的MAX值，或者是整个过去记录中的第二个MAX值）。因此，选择新MAX值的空间和计算成本非常昂贵

为了实现最小的空间/计算/网络成本，

**ReplaceTable +NeedRetract 即是AccRetractMode,其余的都是 AccMode**

![image-20200913162938587](/images/posts/flink-retract-4.png)

Flink中 如果判断是 needRetract和 ReplaceTable 这两个属性呢,每个Operator都有以下参数来识别

| 参数                 | 决定needRetract为true                                        | 决定Replace Table                                            | 决定是AccRetractMode                                      |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| needsUpdateAsRetract | 当前节点needsUpdateAsRetract为true的话,他的children都会是needRetract | 只决定 他的上游needRetract属性                               | NeedRetract 为 true并且 本身是 Replace Table 支持发回撤的 |
| produceUpdate        |                                                              | 是Replace Table                                              | 自己是 produceUpdates 并且 自己 是 needsUpdatesAsRetract  |
| ProduceRetractions   |                                                              | produceRetractions为True必为AccRetractMode                   | produceRetractions为True 必为 AccRetractMode              |
| consumeRetractions   |                                                              | 当前节点的子节点 是个AccRetractMode(生成或者转发Retraction信息),并且 他自己本身不会产生 Retraction 就只做转发工作的节点 |                                                           |
|                      |                                                              |                                                              |                                                           |

