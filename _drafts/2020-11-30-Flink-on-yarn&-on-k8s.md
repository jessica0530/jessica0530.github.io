---
layout: post
title: Flink on yarn & on k8s
categories: [flink]
description: flink on yarn
keywords: flink
---


https://cloud.tencent.com/developer/article/1528548

超卖

JobManager的功能主要有：

将 JobGraph 转换成 Execution Graph，最终将 Execution Graph 拿来运行；
Scheduler 组件负责 Task 的调度；
Checkpoint Coordinator 组件负责协调整个任务的 Checkpoint，包括 Checkpoint 的开始和完成；
通过 Actor System 与 TaskManager 进行通信；
其它的一些功能，例如 Recovery Metadata，用于进行故障恢复时，可以从 Metadata 里面读取数据。

在介绍 Yarn 之前先简单的介绍一下 Flink Standalone 模式，这样有助于更好地了解 Yarn 和 Kubernetes 架构。

在 Standalone 模式下，Master 和 TaskManager 可以运行在同一台机器上，也可以运行在不同的机器上。
在 Master 进程中，Standalone ResourceManager 的作用是对资源进行管理。当用户通过 Flink Cluster Client 将 JobGraph 提交给 Master 时，JobGraph 先经过 Dispatcher。
当 Dispatcher 收到客户端的请求之后，生成一个 JobManager。接着 JobManager 进程向 Standalone ResourceManager 申请资源，最终再启动 TaskManager。
TaskManager 启动之后，会有一个注册的过程，注册之后 JobManager 再将具体的 Task 任务分发给这个 TaskManager 去执行。
以上就是一个 Standalone 任务的运行过程。


Yarn 模式特点

Yarn 模式的优点有：

资源的统一管理和调度。
Yarn 集群中所有节点的资源（内存、CPU、磁盘、网络等）被抽象为 Container。
计算框架需要资源进行运算任务时需要向 Resource Manager 申请 Container，
Yarn 按照特定的策略对资源进行调度和进行 Container 的分配。
Yarn 模式能通过多种任务调度策略来利用提高集群资源利用率。
例如 FIFO Scheduler、Capacity Scheduler、Fair Scheduler，并能设置任务优先级。
资源隔离。Yarn 使用了轻量级资源隔离机制 Cgroups 进行资源隔离以避免相互干扰，一旦 Container 使用的资源量超过事先定义的上限值，就将其杀死。
自动 failover 处理。例如 Yarn NodeManager 监控、Yarn ApplicationManager 异常恢复。

Yarn 模式虽然有不少优点，但是也有诸多缺点，例如运维部署成本较高，灵活性不够。


它的过程也类似，Resource Manager扮演Master的角色，会把task分配到不同的机器上面，我们用容器做一个分装，让它们在不同的机器上一起跑。


容器有点像虚拟机Virtual Machine(VM)，
但容器比VM更轻量化，比起VM搭建的速度更快。
容器能把你的代码、Runtime和Libraries打包成一个Bundle再做成镜像(image)，
你可以把这个镜像想象成存放在磁盘上的一个文件。
你可以通过Docker把镜像运行起来。容器提供了隔离的功能，即跑在容器内的应用，看不到容器外的应用的信息。


Kubernetes有一个好的地方就是它把多个容器抽象成了Pod，方便用户管理。

Kubernetes–基本概念

Kubernetes（k8s）中的 Master 节点，负责管理整个集群，含有一个集群的资源数据访问入口，还包含一个 Etcd 高可用键值存储服务。Master 中运行着 API Server，Controller Manager 及 Scheduler 服务。

Node 为集群的一个操作单元，是 Pod 运行的宿主机。Node 节点里包含一个 agent 进程，能够维护和管理该 Node 上的所有容器的创建、启停等。Node 还含有一个服务端 kube-proxy，用于服务发现、反向代理和负载均衡。Node 底层含有 docker engine，docker 引擎主要负责本机容器的创建和管理工作。

Pod 运行于 Node 节点上，是若干相关容器的组合。在 K8s 里面 Pod 是创建、调度和管理的最小单位。


Kubernetes 是 Google 开源的容器集群管理系统，其提供应用部署、维护、扩展机制等功能，利用 Kubernetes 能方便地管理跨机器运行容器化的应用。
Kubernetes 和 Yarn 相比，相当于下一代的资源管理系统，但是它的能力远远不止这些。




Kubernetes和Yarn的最大区别是前者主要负责long running service 任务，后者则是跑一些batch jobs


Flink之所以选择K8S来作为底层的资源管理来为用户提供服务主要原因有以下几点：

Flink特性：
首先Flink是大数据类应用，与传统大数据应用如Spark、Hadoop、MapReduce以及Hive等不同的是，
Flink是常驻进程，其类似于在线业务的App，作业发布后修改频率比较低，，这就要求执行作业的worker长时间稳定运行。

另外，与其他批处理作业相比，流作业任务一般应用于实时风控和实时推荐的业务场景下，其重要度更高，稳定性要求也更高。
K8S优势：K8S设计的初衷是为在线应用服务，
目标是为了帮助在线应用更好地发布和管理，实现资源隔离；
其次，目前K8S具备一定的生态优势，目前很多用户已经开始或尝试开始使用K8S来管理在线应用；
K8S可以很好地集成其他集群维护工具，如监控工具普罗米修斯，同时在资源弹性方面，K8S可以很方便地进行扩缩容。



