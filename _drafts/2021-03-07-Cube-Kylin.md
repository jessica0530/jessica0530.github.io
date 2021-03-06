---
layout: post
title: Cube Kylin
categories: [cube]
description: Cube Kylin
keywords: cube
---



# 场景

在很多业务场景下,数据提供者和数据使用者不是同一个部门,所以数据提供者对数据的查询模式并不是很熟悉,导致了Cube的设计和实际使用的切合度有一定的差距,这影响了Cube的资源利用率和查询性能

Cube-Planner 是基于各种维度组合的成本和效益,选择性价比高的维度组合集来构建 Cube（剪枝），

以提高Cube构建效率和查询性能.简单点说就是把资源花在构建性价比高的维度组合上。

该 feature 基于论文 Implementing Data Cubes Effiiciently(Lattices)的思想,并做了扩展

Cuboid: 一种维度组合

Cube: Cuboid的集合

Basic Cuboid： 包含所有维度和一个组合（简单理解就是大宽表）

Cuboid 的成本: 查询该 cuboid要扫的行数，表示为: C(i)

Cuboid 的收益:预计算出这个Cuboid,对这个Cube所有查询所能减少的查询成本



## 算法流程

### 贪心算法

贪心算法使用多轮迭代,每次选出当前状态下最优的一种维度组合加入 recommendCuboidList,假设有一个Cube,它有如下的Cuboid

### 基因算法

核心思想 是交叉变异,优胜劣汰,主要步骤包括选择-> 交叉->变异

## 算法选择

| Cuboid数量  | 算法     |      |
| ----------- | -------- | ---- |
| 0 ~ 2^8     | 无优化   |      |
| 2^8  - 2^23 | 贪心算法 |      |
| > 2^23      | 基因算法 |      |



## CuboidRecommender

### Phase1

 不需要预先构建CUbe数据，在第一次构建的时候，自动触发优化

构建的时候，在Extract Fact Table DIstinct COlumns 步骤,会使用 HLL算法 预估出优化前所有的Cuboid的行数，将该stats 放到HDFS的一个目录(代码:CuboidStatCalculator)

例如有C1，C2，2个维度,无聚合组,需要计算COUNT（DISTINCT C1，C2）

 COUNT（DISTINCT C1），COUNT（DISTINCT C2）

没有查询的stats,无查询权重,即假设所有Cuboid被查询的概率都相等

默认开启,构建自动触发

计算benefit 代码:BPUSCalculator

### Phase2

需要Cube 构建并使用过一段时间,会收集用户的查询历史,将其转化为Cuboid的击中次数记录下来,后续优化时会参考:

实际和其他一些统计信息存成一个特殊的Cube叫做System Cube

会记录查询应该击中那个Cuboid,以及实际击中的Cuboid

计算benefit的时候,会乘以Cuboid 被击中比例的权重

计算benefit的代码:PBPUSCalculator

## 存在的问题

1.扔需要用户建立模型（Join关系/维度/度量）,建立Cube 选择可能用到的维度/度量 这个 feature 的focus 在剪枝,无法做到全自动的推荐

2.一种可行的做法是第一次只建模Base cuboid，后续维度组合根据查询自动推荐,但是这种方式也需要提前定义号所有的度量,各个cuboid 只是维度组合不一样,度量都是之前Cube定义中的全量的Measure,不支持灵活的度量

3.一次性输入太多的Cuboid ,估算 Cuboid行数用的时间会很长