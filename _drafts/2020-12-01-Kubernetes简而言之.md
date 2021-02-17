---
layout: post
title: Kubernetes 简而言之
categories: [k8s]
description: k8s 
keywords: k8s
---

Kubernetes提供了一个平台或工具来帮助你快速协调或扩展容器化应用,

https://jimmysong.io/kubernetes-handbook/cloud-native/from-kubernetes-to-cloud-native.html

一种可以一次性将变更传递给所有受控容器的方法，同时也需要一种可以轻松地调度可用容器的方法，这个过程还必须要是自动化的

特别是在Docker 容器

良好隔离、资源分配与编排管理的优势







容器和容器化

整个应用程序的每个组件被单独打包成一个个独立的单元，这个单元就是所谓的容器

通过这种方式，可以将代码和应用服务从底层架构中分离出来，实现了完全的可移植性（在任何操作系统或环境上运行应用的能力）

Ubuntu 操作系统就是一个单元（容器）。MySQL 数据库是另一个容器，Vue 环境和随之而来的库也是一个容器



可以把容器看成是整个应用堆栈中的一层，每层都依赖于下层的单元

容器可以让你创建独立、隔离的工作单元，可以独立运行



Pod

一个 pod 允许你把多个容器，并指定它们如何组合在一起来创建应用程序。而这也进一步明确了 Docker 和 Kubernetes 之间的关系 —— 一个 Kubernetes pod 通常包含一个或多个 Docker 容器，所有的容器都作为一个单元来管理

ReplicationController 或 ReplicaSet 是 Kubernetes 的另一个关键功能。它是负责实际管理 pod 生命周期的组件 —— 当收到指令时或 pod 离线或意外停止时启动 pod，也会在收到指示时杀死 pod，也许是因为用户负载减少。所以换句话说，ReplicationController 有助于实现我们所期望的指定运行的 pod 数量的状态

Kubernetes是一个多租户的云平台，因此必须对用户的权限加以限制，对用户空间进行隔离。Kubernetes中的隔离主要包括这几种：

网络隔离：需要使用网络插件，比如flannel, calico。
资源隔离：kubernetes原生支持资源隔离，pod就是资源隔离和调度的最小单位，同时使用namespace限制用户空间和资源限额。
身份隔离：使用RBAC-基于角色的访问控制，多租户的身份认证和权限控制

每个物理queue内可以含有多种机型，如是否为SSD机器，通过label/taints标识。

每个物理queue划分为1个或多个namespace，对每个namespace进行资源限制、统计及监控


## 服务网格

用于管理服务之间的网络流量，是云原生的网络基础设施层

服务网格利用容器之间的网络设置来控制或改变应用程序中不同组件之间的交互

**行动指南**

1. 根据环境（比如开发、测试、生产）划分`namespace`，也可以根据项目来划分
2. 再为每个用户划分一个`namespace`、创建一个`serviceaccount`和`kubeconfig`文件，不同`namespace`间的资源隔离，目前不隔离网络，不同`namespace`间的服务可以互相访问
3. 创建yaml模板，降低编写Kubernetes yaml文件编写难度
4. 在`kubectl`命令上再封装一层，增加用户身份设置和环境初始化操作，简化`kubectl`命令和常用功能
5. 管理员通过dashboard查看不同`namespace`的状态，也可以使用它来使操作更便捷
6. 所有应用的日志统一收集到ElasticSearch中，统一日志访问入口
7. 可以通过Grafana查看所有namespace中的应用的状态和kubernetes集群本身的状态
8. 需要持久化的数据保存在分布式存储中，例如GlusterFS或Ceph中



使用Kubernetes原生调度的spark on kubernetes是对原先的spark on yarn和yarn on docker的改变是革命性的，主要表现在以下几点：

1. **Kubernetes原生调度**：不再需要二层调度，直接使用Kubernetes的资源调度功能，跟其他应用共用整个kubernetes管理的资源池；
2. **资源隔离，粒度更细**：原先yarn中的queue在spark on kubernetes中已不存在，取而代之的是kubernetes中原生的namespace，可以为每个用户分别指定一个namespace，限制用户的资源quota；
3. **细粒度的资源分配**：可以给每个spark任务指定资源限制，实际指定多少资源就使用多少资源，因为没有了像yarn那样的二层调度（圈地式的），所以可以更高效和细粒度的使用资源；
4. **监控的变革**：因为做到了细粒度的资源分配，所以可以对用户提交的每一个任务做到资源使用的监控，从而判断用户的资源使用情况，所有的metric都记录在数据库中，甚至可以为每个用户的每次任务提交计量；
5. **日志的变革**：用户不再通过yarn的web页面来查看任务状态，而是通过pod的log来查看，可将所有的kuberentes中的应用的日志等同看待收集起来，然后可以根据标签查看对应应用的日志；