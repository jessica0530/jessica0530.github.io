---
layout: post
title: Kubernetes 基础
categories: [kubernetes]
description: kubernete基础
keywords: kubernetes
---

# Kubernetes 架构及应用场景

K8s 是管理容器的容器化部署模式

https://jimmysong.io/kubernetes-handbook/guide/kubectl-cheatsheet.html

![kubernetes-constructs-concepts-architecture](/images/posts/kubernetes-constructs-concepts-architecture.jpg)



![Kubernetes-control-plane-taxonomy](/images/posts/Kubernetes-control-plane-taxonomy.jpg)



## K8s 在企业中的应用场景

###   1.自动化运维平台

-  中小型企业,使用 k8s 构建一套自动化的运维平台,实现降本增效

-  大型互联网公司更要使用、

  

  前提条件:SOA 架构,微服务架构模式下,服务拆分越来越多,部署维护的服务越来越多，如此多的服务如何去管理

####  面临问题

  ```
  1.如何对服务进行横向扩展
  
  2.容器宕机怎么办,数据怎么恢复
  
  3.重新发布新的版本如何更新,更新后,不影响业务
  
  4.如何监控容器
  
  5.容器如何调度创建
  
  6.数据安全性如何保证
  ```

  解决方案:使用 k8s管理容器,可以解决上面的问题

###  2.充分利用服务器资源



```
案例:并发量200请求,服务器 2 cpu 4G

   静态请求:150 --访问CDN,nginx，缓存

   动态请求: 50 -- 访问数据库,把数据读入内存

   估算服务器资源:只考虑内存,不考虑CPU切换时间

   

   150 * 2 M + 50* 10M = 800M -- 200 请求占用800M内存

   QPS:200 * 4 =800 如果重复利用服务器资源占用 3.2G  0.8G留给操作系统(其实不够用)

   实际上:800 QPS是无法达到的

   1. 相应时间-response time 再慢一些 并发量会下降
   2. Cpu 切换时间
   3. 内存

​    所以 QPS 估算为300,不能充分利用服务器资源
```

容器化的方式,充分利用服务器资源

容器是 更轻量级的虚拟机,部署效率更高

### 3.服务无缝迁移

测试——生产环境

把 JDK等服务做成镜像,用容器化的方式 做无缝迁移



## 云架构 & 云原生 

### 1)云和 k8s 是什么关系

 云 使用容器来构建的一套服务集群网络,云由很多的大量的容器构建

K8s 就是用来管理云中的容器。

### 2)云架构

   1.Iaas 基础设置即服务

​          用户:租用(购买,分配权限) 云主机,用户不需要考虑网络 ,DNS

​         运营商:提供网络,存储,DNS,这样服务就是基础设置服务

   2.Paas  平台即服务

​        MYSQL\ES\MQ

  3.SAAS 软件即服务

​            钉钉,财务管理

  4.Serverless

​         不需要服务器

站在用户的角度考虑,用户只需要云服务器即可,在云服务器所有的基础环境,软件环境都不需要

### 3)云原生

​        就是为了让应用程序(项目,服务软件)都运行在云上的解决方案,这样方案叫做云原生

​        特点:

​       1) 容器化  —— 所有的服务都必须部署在容器中

​       2）微服务—— web服务架构师 微服务架构

​       3） CI/CD

​       4)   DevOps

## Kubernets 架构原理

  1）kubernetes 是 Google 公司使用 go语言开发

  2) Kubernetes 架构

### master节点

1. Api server k8s网关

2. Scheduler 调度器,使用调度算法,把请求资源调度某一个node节点,

   调用 Api server 把指令存储再 etcd,kubelet去etcd 扫描相关信息,再去对应的node节点执行指令

3. Controller 控制器，维护k8s资源对象

4. etcd存储资源 对象

### Node节点

​    1.Docker 运营容器的基础环境,容器引擎

​    2.Kubelet 在每个Node节点  都存一份,在node节点上的资源操作指令由kubelet来执行

   从 etcd sao

   3.Kube-proxy 代理服务,负载均衡,多个pod之间来做负载均衡

   4.Fluentd 日志收集服务

   5.pod是k8s管理的基本单元(最小单元),pod内部是容器.k8s不直接管理容器,而是管理 pod

# Kubernetes 核心组件原理

## pod的核心原理

k8s通过pod来间接管理容器

### pod是什么

pod内部可以有一个容器或者多个容器

pod也是个容器,这个容器中装的是 docker创建的容器,pod用来封装容器的一个容器,pod是一个虚拟化的分组,pod有自己的IP地址,主机名,相当于一台独立沙箱环境。

Pod相当于独立主机,可以封装一个或者多个容器

通常情况下,在服务部署时候,使用pod来管理一组相关的服务(一个pod中要么部署一个服务,要么部署一组有关系的服务)

一组相关的服务  比如 nginx->web服务->mysql  在链式调用的调用链路上的服务,叫做一组的相关的服务

实现服务集群,只需要赋值多方pod的副本即可,k8s如果继续扩容,缩容,只需要控制pod的数量即可

### pod底层网络,数据存储是如何进行的

1.pod内部容器创建之前,必须先创建pause容器来共享网络和共享存储

2.服务容器之间 访问使用 localhost访问,相当于访问本地服务一样,性能非常高



## Deploy 部署对象

### 服务部署结构模型

![k8s-deployment](/images/posts/k8s-deployment.png)

### 滚动更新

 ReplicaSet副本控制器 控制pod副本的数量,项目的版本会不停的发版,版本的变化,如何做到服务更新?

部署模型:ReplicaSet不支持滚动更新,Deployment对象支持滚动更新,通常和ReplicaSet一起使用

Deployment
1.重新建立新的RS

2.重新建立新的Pod

3.再滚动更新



## ReplicaSet副本控制器

ReplicationController副本控制器

ReplicaSet 副本控制器

### 什么是副本控制器

副本控制器基本理解:控制pod副本(服务集群)的数量,永远与预期设定的数量保持一致即可,

例如:

   前提:replicas =3 (3个副本),会创建3个pod,如果pod死掉了,k8s马上会创建个新的pod来弥补数量

副本控制器:当副本设置为3的时候,副本控制器会永远保证副本数量为3，当有 pod服务宕机的时候,副本控制器会立马重新创建一个新的pod,永远保证副本数量为3



### RelicaSet & ReplicationController有什么区别

1.ReplicaSet可以使用 标签选择器(Pod有标签,app=web,release=stable) 选择维护一组相关的服务

Selector:

​          app=web (单项选择)

​         app=web,release=stable (复合选择)

2.ReplicationController

  单选

在新版的k8s中,建议使用 ReplicaSet作为副本选择器



## StatefulSet 

和deployment相似,是部署有状态的服务

StatefulSet保证pod重新建立后,hostname不会发生变化,pod就可以通过hostname来关联维护的数据

![k8s-statefulset](/images/posts/k8s-statefulset.png)



MYSQL使用容器化部署,会存在的问题

1.容器有生命周期的,一旦宕机,数据会丢失

2.pod 来部署,pod也有生命周期,数据依旧会丢失

k8s来说,不能使用 deployment部署模型部署有状态服务,deployment 来部署无状态的服务,

有状态的服务的部署,使用 statefulSet

状态服务的解释:

1. 有状态服务

   有实时的数据需要存储

   有状态服务集群中,把某一个服务抽离出去,一段时间后再加入机器网络,如果集群网络无法使用

# Kubernetes 的服务的注册与发现



## pod在生产环境中的访问流程

### pod的结构

pod相当于一个容器,有独立的ip地址,也有自己的hostname,利用namespace进行资源隔离,独立沙箱环境

### pod网络

​    1.pod有自己独立的ip地址

   2.pod内部容器之间访问采用 localhost访问

#### pod如何堆外提供服务访问的

   pod(PODIP,hostname),pod 是个虚拟的资源对象(进程),没有对应实体(物理机,物理网卡)与之对应,无法直接对外提供服务

解决方案

​    Pod 想要对外提供服务,必须绑定物理机端口(在物理机上开启端口,让这个端口和pod的端口进行映射),这样可以通过物理机进行数据包的转发

1.先通过物理机的ip+port进行访问

2.数据包转发

#### pod如何实现访问的负载均衡

pod是一个进程,是有生命周期的(宕机,版本更新),都会创建新的pod(ip地址发生变化,hostname发生变化,所以用 nginx是不行的)

##### 什么是Service资源对象

POD IP pod的ip地址

NODE IP 物理机的IP地址

Cluster IP 虚拟IP,是由kubernetes抽象出的service对象,这个service对象就是个VIP （虚拟IP）对象

##### 如何利用service来实现负载均衡的

Service VIP -虚拟IP

进程-资源对象  10.12.22.16：80 转发给下面的pod

###### 转发流程

1.service和pod都是一个进程,service也不能对外网提供服务

2.service和pod之间可以直接进行通信,它们的通信属于局域网通信

3.把请求交给service后,serice使用(iptables,ipvs)做数据包的分发



1）访问物理机 ->把请求交给service->转发给pod



###### service对象是如何和pod进行关联的

service和一组pod副本

Selector: app=x 

Endpoints[10.244.1.1,10.244.1.2,10.244.1.3,xxxxx,xxxx]

如果pod宕机了,service如何发现pod发生了变化

通过kube-proxy 来监听所有的pod,会更新所有的 映射关系 来更新endpoint，这些信息都 存在 etcd



# Flink on kubernetes

https://kuboard.cn/install/install-k8s.html
本地已经搭建好了kubernetes
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=default





![flink-submitjob-on-k8s](/images/posts/flink-submitjob-on-k8s.jpg)





