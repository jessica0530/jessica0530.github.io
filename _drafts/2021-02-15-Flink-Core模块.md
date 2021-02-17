---
layout: post
title: Flink CORE模块
categories: [flink]
description: flink core
keywords: flink
---

## eventTime

Watermark 

onEvent

onPeriodicEmit  

WatermarkStrategyWithIdleness

WatermarkStrategyWithTimestampAssigner

## Function

基础function,richfunction(open,getruntime,close)

## Io

### Compression

能创建  bzip2Inputstream中的Factory,

主要用的是 apache commons-compress中的io

Flink-connector-files 中 根据  file.gzip 后缀 会 创建相应的inputstream



### Inputformat/outputformat

```java
inputformat  

public abstract class XXXXXInputFormat<T> extends FileInputFormat<T>
        implements CheckpointableInputFormat<FileInputSplit, Tuple2<Long, Long>>
  
outputformat
  
public abstract class FileOutputFormat<IT> extends RichOutputFormat<IT>
        implements InitializeOnMaster, CleanupWhenUnsuccessful  
  
  InitializeOnMaster/FinalizeOnMaster
```



### inputSplitAssigner

```java
LocatableInputSplitAssigner
```

## operators

BulkIteration

CoGroupOperatorBase

CrossOperator

FilterOperator

GroupCombineOperator

GroupReduceOperator

JoinOperatorBase

MapOperatorBase

OuterJoinOperatorBase

PartitionOperatorBase

ReduceOperatorBase

SortPartitionOperatorBase

## restartStrategies

```
ExponentialDelayRestartStrategy

FailureRateRestartStrategy

FallbackRestartStrategy

FixedDelayRestartStrategy
```

## State

KeyedStateStore

OperatorStateStore

AppendingState/ListState/MapState/ReducingState/ValueState

StateTtl

## typeinfo



## core

localfilesystem recoverable

Memory  memorySegment







