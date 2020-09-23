---
layout: post
title: Flink mailBox
categories: [flink]
description: flink mailBox
keywords: flink
---

# StreamTask Threading-model on a mailbox-based approach

## 目标

基于 mailbox 来简化 流任务的线程模型

当前Flink 的流任务的线程模型,会出现 多条线程 想同事访问对象的State,

比如 StreamRecord的处理和 CheckPoint Trigger 通过 全局 锁 checkpointLock 来互斥,

而且会把锁 传到 代码的许多位置,会给面向 用户的API提供,如果不获取锁会产生数据上的细微的错误.



而在mailbox模式下,流任务中 所有 State的更改 都 从单线程中发生[mailbox thread],通过将动作 放入阻塞队列 来模拟并发动作. 该队列由单个主线程（mailbox thread）不断地探查是否有新动作。如果队列中有“并发”动作，则主线程将执行它。

## 现CheckPointLock的作用

在下面3个并发源之间 实现对 流任务的 组件状态的 互斥访问

1）Event Processing:Events,Watermarks,Barrier,Latency Marker,etc

2）CheckPoints:Source 上的 checkpoint 触发和 完成通知 移交给了 asyncCallDispatcher, Triggering/cancellation

3）Processing Time Timers:SystemProcessingTimeService 使用ScheduledExecutor 异步运行处理时间计时器

我们还可以确定，Checkpoint Lock的替代品不仅必须提供独占性，而且还必须像执行事件一样对关键部分进行原子执行

## MailBox具体组件

### Mail

表示一个需要执行的Action,

里面包括Runnable,Priority(不决定order),StreamTaskActionExecutor 调run方法

可执行的方法:tryCancel,run

### MailboxProcessor

此类封装了基于mailbox的执行模型的逻辑。

此模型的核心 #runMailboxLoop() 在循环中连续执行提供的MailboxDefaultAction。

在每次迭代中 该方法还会检查mailbox中是否有待处理的操作并执行这些操作。

该模型确保 default Action（例如记录处理）和mailbox Action（例如checkpoint trigger，timer firing）之间是单线程执行。

MailboxDefaultAction通过 MailboxController与此类交互，以将控制流更改传达到mailbox loop循环，

例如the default action的调用是临时的，或者 永久耗尽?。 

runMailboxLoop的设计围绕hot path （default action，no mail）越快越好的想法。这意味着所有mail检查和其他control flag（mailboxLoopRunning，suspendedDefaultAction）始终连接到#hasMail，表示true。这意味着 *可以直接在mailbox线程中更改control flag，但是我们必须确保至少有一个action 在mailbox中，以便这个 changes可以被picked up。对于所有其他线程的control changes更改，这必须通过mailbox action 发生，情况会自动如此。 

### TaskMailboxImpl 

多写单独的阻塞队列

### MailboxExecutorImpl

执行器 包含 processor 和 mailbox

方法

#### execute

#### yield

#### TryYield



## Proposed changes

### Changes in Stream Task

建议在流任务中引入“mailbox”作为StreamTask。mailbox的一种可能的初始实现方式是ArrayBlockingQueue 。稍后，我们可能会针对必须涵盖的多生产者单消费者案例评估更有效的实现，也许是基于环形缓冲区的Disruptor风格的实现。



该mailbox将位于流任务主线程中活动的中心，并且（对于大多数部分而言）将接管当前StreamTask＃run（）方法的角色，即它成为事件生成/处理的驱动程序。但是，与StreamTask＃run（）不同的是，此方法还将负责执行checkpointer event 和processing timer。所有这些事件都将简单地成为已排队在mailbox中的任务，并且流任务的主线程将不断从mailbox中提取并运行下一个事件。这样可以通过队列实现互斥执行。

由于我们希望能够在此模型中表示atomic部分，一种方法可以表示诸如在邮箱中排队的Runnable 对象之类的原子动作。

任务的主线程可以阻塞此类Runnable 的执行，并且生产者也可以阻止将新操作加入队列的尝试。第一种情况将与当前代码中的较长的关键部分在Checkpoint lock下阻塞的情况相对应。第二种情况是试图获取checkpoint lock的线程阻塞。



```java
BlockingQueue<Runnable> mailbox = ...


void runMailboxProcessing() {
    //TODO: can become a cancel-event through mailbox eventually
    Runnable letter;
    while (isRunning()) { 
        while ((letter = mailbox.poll()) != null) {
            letter.run();
        }

        defaultAction();
    }
}

void defaultAction() {
    // e.g. event-processing from an input
}
```





