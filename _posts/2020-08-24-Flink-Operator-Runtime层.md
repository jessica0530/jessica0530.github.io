---
layout: post
title: flink Operator组件层
categories: [flink]
description: flink Operator Runtime
keywords: flink Operator runtime层
---

# Flink-streaming-java



## Collector

Forward -DirectedOutput



## Function

### Agg

 ComparableAggregator

### Async

AsyncFunction

### Co

BroadcastProcessFunction

```
processElement
processBroadcastElement
```

### Sink

```
TwoPhaseCommitSinkFunction
```

### Source

continuousFileReader

### Graph

StreamGraphGenerator

### Operator

#### async

AsyncWaitOperator

queue

OrderedStreamElementQueue

#### Co

CoBroadcastWithKeyedOperator

#### Source

idle ？

```
@Override
public void markAsTemporarilyIdle() {
   synchronized (checkpointLock) {
      streamStatusMaintainer.toggleStreamStatus(StreamStatus.IDLE);
   }
}
if (subscribedPartitionsToStartOffsets.isEmpty()) {
			sourceContext.markAsTemporarilyIdle();
}
```

##### InternalTimerServiceImpl

```
/**
 * Processing time timers that are currently in-flight.
 */
private final KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> processingTimeTimersQueue;

/**
 * Event time timers that are currently in-flight.
 */
private final KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> eventTimeTimersQueue;
```

OneInputStreamOperator

TwoInputStreamOperator

### Transformation

OneInputTransformation

TwoInputTransformation

UnionTransformation

## Runtime

### io

#### Exactly-Once

CheckPointBarrierAligner

```java
// fast path for single channel cases
		if (totalNumberOfInputChannels == 1) {
			resumeConsumption(channelInfo);
			if (barrierId > currentCheckpointId) {
				// new checkpoint
				currentCheckpointId = barrierId;
				notifyCheckpoint(receivedBarrier, latestAlignmentDurationNanos);
			}
			return;
		}

		// -- general code path for multiple input channels --

		if (isCheckpointPending()) {
			// this is only true if some alignment is already progress and was not canceled

			if (barrierId == currentCheckpointId) {
				// regular case
				onBarrier(channelInfo);
			}
			else if (barrierId > currentCheckpointId) {
				// we did not complete the current checkpoint, another started before
				LOG.warn("{}: Received checkpoint barrier for checkpoint {} before completing current checkpoint {}. " +
						"Skipping current checkpoint.",
					taskName,
					barrierId,
					currentCheckpointId);

				// let the task know we are not completing this
				notifyAbort(currentCheckpointId,
					new CheckpointException(
						"Barrier id: " + barrierId,
						CheckpointFailureReason.CHECKPOINT_DECLINED_SUBSUMED));

				// abort the current checkpoint
				releaseBlocksAndResetBarriers();

				// begin a new checkpoint
				beginNewAlignment(barrierId, channelInfo, receivedBarrier.getTimestamp());
			}
			else {
				// ignore trailing barrier from an earlier checkpoint (obsolete now)
				resumeConsumption(channelInfo);
			}
		}
		else if (barrierId > currentCheckpointId) {
			// first barrier of a new checkpoint
			beginNewAlignment(barrierId, channelInfo, receivedBarrier.getTimestamp());
		}
		else {
			// either the current checkpoint was canceled (numBarriers == 0) or
			// this barrier is from an old subsumed checkpoint
			resumeConsumption(channelInfo);
		}

		// check if we have all barriers - since canceled checkpoints always have zero barriers
		// this can only happen on a non canceled checkpoint
		if (numBarriersReceived + numClosedChannels == totalNumberOfInputChannels) {
			// actually trigger checkpoint
			if (LOG.isDebugEnabled()) {
				LOG.debug("{}: Received all barriers, triggering checkpoint {} at {}.",
					taskName,
					receivedBarrier.getId(),
					receivedBarrier.getTimestamp());
			}

			releaseBlocksAndResetBarriers();
			notifyCheckpoint(receivedBarrier, latestAlignmentDurationNanos);
		}
```

#### AT_LEAST_ONCE

```java
final long barrierId = receivedBarrier.getId();

// fast path for single channel trackers
if (totalNumberOfInputChannels == 1) {
   notifyCheckpoint(receivedBarrier, 0);
   return;
}

// general path for multiple input channels
if (LOG.isDebugEnabled()) {
   LOG.debug("Received barrier for checkpoint {} from channel {}", barrierId, channelInfo);
}

// find the checkpoint barrier in the queue of pending barriers
CheckpointBarrierCount barrierCount = null;
int pos = 0;

for (CheckpointBarrierCount next : pendingCheckpoints) {
   if (next.checkpointId == barrierId) {
      barrierCount = next;
      break;
   }
   pos++;
}

if (barrierCount != null) {
   // add one to the count to that barrier and check for completion
   int numBarriersNew = barrierCount.incrementBarrierCount();
   if (numBarriersNew == totalNumberOfInputChannels) {
      // checkpoint can be triggered (or is aborted and all barriers have been seen)
      // first, remove this checkpoint and all all prior pending
      // checkpoints (which are now subsumed)
      for (int i = 0; i <= pos; i++) {
         pendingCheckpoints.pollFirst();
      }

      // notify the listener
      if (!barrierCount.isAborted()) {
         if (LOG.isDebugEnabled()) {
            LOG.debug("Received all barriers for checkpoint {}", barrierId);
         }

         notifyCheckpoint(receivedBarrier, 0);
      }
   }
}
else {
   // first barrier for that checkpoint ID
   // add it only if it is newer than the latest checkpoint.
   // if it is not newer than the latest checkpoint ID, then there cannot be a
   // successful checkpoint for that ID anyways
   if (barrierId > latestPendingCheckpointID) {
      markCheckpointStart(receivedBarrier.getTimestamp());
      latestPendingCheckpointID = barrierId;
      pendingCheckpoints.addLast(new CheckpointBarrierCount(barrierId));

      // make sure we do not track too many checkpoints
      if (pendingCheckpoints.size() > MAX_CHECKPOINTS_TO_TRACK) {
         pendingCheckpoints.pollFirst();
      }
   }
}
```

#### Process

StreamOneInputProcessor

StreamMultipleInputProcessor

StreamTwoInputProcessor

### Partition

KeyGroupStreamPartitioner

### StreamRecord

StreamElement/LatencyMarker/WaterMark

### Task

#### Mailbox

#### StreamTask

 AbstractTwoInputStreamTask

 OneInputStreamTask



