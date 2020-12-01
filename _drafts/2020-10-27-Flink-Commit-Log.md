---
layout: post
title: Flink Commit Log
categories: [flink]
description: Flink Commit Log
keywords: flink
---





1.Group by 清理的时候 在 prev= new的情况要发送，不清理的情况不需要发送，

如果启用了状态清理，我们必须发出消息，以防止下游操作员过早地退出状态。





```java
@Override
public void run() {
   if (modifiedOnProcessingTime) {
      handleTargetCallback();
   } else {
      synchronized (lock) {
         handleTargetCallback();
      }
   }
}
```

```java
/** The lock that guarantees that record emission and state updates are atomic,
 * from the view of taking a checkpoint. */
private final Object checkpointLock;
```





udaf acc的结果是  static  会 被用户改变 需要 copy





FlinkTypeFactory

Don't cache row types in type factory for Calcite

seenTypes.getOrElseUpdate((typeInfo, isNullable), createAdvancedType(typeInfo, isNullable))



在 source和 sink schema相同的情况下 ,会颠倒 Scan 

Schema MAP ANY BINARY之类的



class CRowSerializer(val rowSerializer: TypeSerializer[Row]) extends TypeSerializer[CRow] {



Function isDeterministic 





Micro Batch Mark not work is not active



SortedMapState



提供 自定义Source Sink 序列化 



Lookup join 



TimeZone



CodeGenerator



Hour  TimeScalarFunctionBase


Idle
 	public void emitMicroBatchMark(MicroBatchMark mark) {
 		serializationDelegate.setInstance(mark);
 
+		try {
+			recordWriter.broadcastEmit(serializationDelegate);
+		} catch (Exception e) {
+			throw new RuntimeException(e.getMessage(), e);
 		}
 	}

		@Override
		public void emitWatermark(Watermark mark) {
			watermarkGauge.setCurrentWatermark(mark.getTimestamp());
			if (streamStatusProvider.getStreamStatus().isActive()) {
				for (Output<StreamRecord<T>> output : outputs) {
					output.emitWatermark(mark);
				}
			}
		}
		
		
@Override
        		public void markAsTemporarilyIdle() {
        			synchronized (checkpointLock) {
        				streamStatusMaintainer.toggleStreamStatus(StreamStatus.IDLE);
        			}
        		}
        		
        		
        		
@Override
	public void toggleStreamStatus(StreamStatus status) {
		if (!status.equals(this.streamStatus)) {
			this.streamStatus = status;

			// try and forward the stream status change to all outgoing connections
			for (RecordWriterOutput<?> streamOutput : streamOutputs) {
				streamOutput.emitStreamStatus(status);
			}
		}
	}
	
	// mark the subtask as temporarily idle if there are no initial seed partitions;
		// once this subtask discovers some partitions and starts collecting records, the subtask's
		// status will automatically be triggered back to be active.
		if (subscribedPartitionsToStartOffsets.isEmpty()) {
			sourceContext.markAsTemporarilyIdle();
		}	
		
		
		
@Override
			public void onProcessingTime(long timestamp) {
				final long currentTime = timeService.getCurrentProcessingTime();

				synchronized (lock) {
					// we should continue to automatically emit watermarks if we are active
					if (streamStatusMaintainer.getStreamStatus().isActive()) {
						if (idleTimeout != -1 && currentTime - lastRecordTime > idleTimeout) {
							// if we are configured to detect idleness, piggy-back the idle detection check on the
							// watermark interval, so that we may possibly discover idle sources faster before waiting
							// for the next idle check to fire
							markAsTemporarilyIdle();

							// no need to finish the next check, as we are now idle.
							cancelNextIdleDetectionTask();
						} else if (currentTime > nextWatermarkTime) {
							// align the watermarks across all machines. this will ensure that we
							// don't have watermarks that creep along at different intervals because
							// the machine clocks are out of sync
							final long watermarkTime = currentTime - (currentTime % watermarkInterval);

							output.emitWatermark(new Watermark(watermarkTime));
							nextWatermarkTime = watermarkTime + watermarkInterval;
						}
					}
				}

				long nextWatermark = currentTime + watermarkInterval;
				nextWatermarkTimer = this.timeService.registerTimer(
						nextWatermark, new WatermarkEmittingTask(this.timeService, lock, output));
			}	
			
			
	Source		
@Override
		public void onProcessingTime(long timestamp) throws Exception {

			long minAcrossAll = Long.MAX_VALUE;
			boolean isEffectiveMinAggregation = false;

			List<KafkaTopicPartitionState<KPH>> nonIdleStates = getNonIdlePartitions();
			for (KafkaTopicPartitionState<?> state : nonIdleStates) {

				// we access the current watermark for the periodic assigners under the state
				// lock, to prevent concurrent modification to any internal variables
				final long curr;
				//noinspection SynchronizationOnLocalVariableOrMethodParameter
				synchronized (state) {
					curr = ((KafkaTopicPartitionStateWithPeriodicWatermarks<?, ?>) state).getCurrentWatermarkTimestamp();
				}

				minAcrossAll = Math.min(minAcrossAll, curr);
				isEffectiveMinAggregation = true;
			}

			// emit next watermark, if there is one
			if (isEffectiveMinAggregation && minAcrossAll > lastWatermarkTimestamp) {
				lastWatermarkTimestamp = minAcrossAll;
				emitter.emitWatermark(new Watermark(minAcrossAll));
			}

			// schedule the next watermark
			timerService.registerTimer(timerService.getCurrentProcessingTime() + interval, this);
		}
	}	
	
	
	
public void processWatermark1(Watermark mark) throws Exception {
		input1Watermark = mark.getTimestamp();
		long newMin = Math.min(input1Watermark, input2Watermark);
		if (newMin > combinedWatermark) {
			combinedWatermark = newMin;
			processWatermark(new Watermark(combinedWatermark));
		}
	}

	public void processWatermark2(Watermark mark) throws Exception {
		input2Watermark = mark.getTimestamp();
		long newMin = Math.min(input1Watermark, input2Watermark);
		if (newMin > combinedWatermark) {
			combinedWatermark = newMin;
			processWatermark(new Watermark(combinedWatermark));
		}
	}
	
	
	
offheap一般包括 directMemory(包括network缓存）+ metaspace + java 栈 + native memory


OffHeap：即在heap外动态申请的空间，我们将此部分空间分成两部分，一部分是以DirectByteBuffer形式申请的，
一部分是通过unsafe或者其他JNI库申请的。
前者可以通过Java控制(-XX:MaxDirectMemorySize)和监控(flink有此监控项)，
flink框架如果配置使用堆外内存，便是此种类型；
后者是我们无法控制和监控的，如果有发现堆外RSS异常（超出约1/4RAM的默认限制，或者持续增长），应及时排查用户代码或者引用的native库是否过渡使用堆外内存	


们在启动一个pod里的container时，会设置request和limit两个值来限制一个tm可以使用的内存。request的值主要是用作调度时的参考，只要有request的资源就可以调度起来。limit的值则是我们对container使用资源的一个限制，当node的内存足够时，我们是可以在不超过limit时一切正常的，当超过limit后，就会成为被Kill的候选者（node内存不足时会优先Kill），继续增长则会Kill。我们线上目前是将request和limit设置为同一个值，确保调度起来后在不超过limit时总会可用。

docker也可以限制swap的大小，但由于咱们默认没启用swap，所以暂不关注这个。此外，k8s也暂时没有对内核内存做隔离（进程在内核中分配的内存）,也没发现对虚拟内存做限制（即VIRT可以用到远远大于limit）。

docker的隔离主要以cgroups来实现，隔离机制可以可以参考CGROUP相关的机制。容器内的java进程默认无法感知cgroup对资源的限制，其看到的仍是宿主机的内存，但其申请或使用超过限制时，会被Kill。所以我们启动tm时，默认堆空间会固定占用0.75limit，剩余的供堆外占用。后续需要根据任务对堆外内存的需求调整这个比例	


心跳逻辑 

只能统一判断该进程是否还处于正常工作且能正常通信状态，不能区分是该进程高负载、
full GC、node/process crash还是网络抖动， 
心跳时间过短则过于敏感（很多异常都会导致重启），心跳时间过长则会导致作业断流时间加长，很难取舍。	


Sidecar设计模式	


SQL 
计算列 ,pk定义, createfunction,struct map column

try-cast

 val operatorCode = if (nullCheck) {
      s"""
         |${operand.code}
         |$resultTypeTerm $resultTerm;
         |boolean $nullTerm;
         |if (!${operand.nullTerm}) {
         |  try {
         |     $resultTerm = ${expr(operand.resultTerm)};
         |     $nullTerm = false;
         |  } catch (Exception ex) {
         |     $resultTerm = $defaultValue;
         |     $nullTerm = true;
         |  }
         |}
         |else {
         |  $resultTerm = $defaultValue;
         |  $nullTerm = true;
         |}
         |""".stripMargin
    }
    else {
      s"""
         |${operand.code}
         |$resultTypeTerm $resultTerm = ${expr(operand.resultTerm)};
         |""".stripMargin
    }

SqlWatermark Watermark(Span s) :
{
    SqlIdentifier timeField;
    Integer offset;
}
{
    <WATERMARK> <FOR> timeField = SimpleIdentifier() <AS> <WITHOFFSET>
    <LPAREN>
        timeField = SimpleIdentifier()
        <COMMA>
        offset = IntLiteral()
    <RPAREN>
    {
        return new SqlWatermark(s.add(this).pos(), timeField, offset);
    }
}

SqlNodeList StructSubColumns(Span s) :
{
}
{
    <STRUCT>
    <LT>
    {
        List<SqlNode> subColumns = Lists.newArrayList();
        SqlNode subColumn = null;
    }
    subColumn = TableColumn()
    { subColumns.add(subColumn); }
    (
        <COMMA> subColumn = TableColumn()
        { subColumns.add(subColumn); }
    )*
    <GT>
    { return new SqlNodeList(subColumns, s.addAll(subColumns).pos());  }
}

table 

struct 类型的字段

struct field struct
 if (field != null) {
else {
             // Throw an error if the table field was not found.
             // handle the condition that table name can be found in struct type field
            final SqlIdentifier prefix = identifier.skipLast(1);
            throw validator.newValidationError(prefix,
                    RESOURCE.tableNameNotFound(prefix.toString()));
          }

hint

SqlLiteral hint


LookupJoinAsyncFunction

PartialKeyGrouping

可以看到PartialKeyGrouping是一种CustomStreamGrouping，在prepare的时候，初始化了long[] targetTaskStats用于统计每个task 用过的次数
partialKeyGrouping如果没有指定fields，则默认按outputFields的第一个field来计算
这里使用guava类库提供的Hashing.murmur3_128函数，构造了两个HashFunction，然后计算哈希值的绝对值与targetTasks.size()取余数得到两个可选的taskId下标
然后根据targetTaskStats的统计值，取用过的次数小的那个taskId，选中之后更新targetTaskStats

int firstChoice = (int) (Math.abs(h1.hashBytes(key.toString().getBytes()).asLong()) % numPartitions);
int secondChoice = (int) (Math.abs(h2.hashBytes(key.toString().getBytes()).asLong()) % numPartitions);

常量函数 会一直返回同一个值不变

timezone

SqlFunction
  public static java.sql.Time internalToTime(int v) {
    return new java.sql.Time(v - LOCAL_TZ.getOffset(v));
  }
public static java.sql.Time internalToTime(Integer v) {
		return new java.sql.Time(v);
	}  
  
  
Runtime

hot-update  热更新
提交新的jar包, 不用停止旧的实例就让新的执行逻辑生效,减少切换时间

对于并行度之间无数据交换的作业，作业业务逻辑修改，需要重启作业时，期望能够尽可能小地减少流量抖动。
当前停止整个流，再重启的方式会造成消费中断，针对这类作业期望

 可以按并行度热更新一个运行中作业的执行逻辑（JobGraph结构不变的前提下）
	*1.* *灰度更新* - 支持不同并行度按不同的jar版本运行；

	2. 热更新 - *滚动更新*所有并行度为新版本，作业运行不中断；


声明式调度 

针对核心，高保障作业在资源不足的情况下，
存在无法保障的问题，
引入新的声明资源调度和基于改调度的降级方案，来为核心，高保障作业提供更高的保障。

核心高保障作业  会有 full restart重启的情况,碎片问题,会出现资源不足 

ResourceController会根据NodeManager 汇报上来的机器使用情况和Pod分布情况，
从优先级低的作业中选择影响作业数量最小，
影响作业最小的策略来选择作业来降级（降级有两种将资源调整到min和直接停止作业)， 选择不出来满足的资源则调用FM HA切换和告警。


碎片问题的解决方案



K8S 提供了  request和limit
本身提供了资源超卖的语义 

一个pod能最低给多少资源
最高给多少资源
会弹性的伸缩



通过资源的超卖,就能控制公有队列的资源能多卖出一部分

就能减少资源碎片了

本来只剩1C1G,我们也超卖出去卖给了需要2C2G的应用 

然后就没有1C1G的碎片了

因为同一个机器上的很多其他POD,他们实际都用不到他们需要的资源 

我们90%以上的 比如配置了4C4G,但是他们完全用不完,大部分时间都只是用了2C2G

这些作业没用的2C2G也被我们拿去卖了

16C32G的机器,被我们卖给了总的需要32C64G的总作业 



	