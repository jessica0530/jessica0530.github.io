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