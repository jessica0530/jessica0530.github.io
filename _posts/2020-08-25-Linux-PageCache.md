---
layout: post
title: Linux PageCache
categories: [linux]
description: Linux PageCache
keywords: pageCache
---

# 什么是 Page Cache

Page Cache 是内核管理的内存，也就是说，它属于内核不属于用户。

![linux-pagecache](/images/posts/linux-pagecache.png)

那咱们怎么来观察 Page Cache 呢？其实，在 Linux 上直接查看 Page Cache 的方式有很多，包括 /proc/meminfo、free 、/proc/vmstat 命令等，它们的内容其实是一致的

等式两边是PageCache

```java
Buffers + Cached + SwapCached = Active(file) + Inactive(file) + Shmem + SwapCached
```

在 Page Cache 中，Active(file)+Inactive(file) 是 File-backed page（与文件对应的内存页），是你最需要关注的部分。因为你平时用的 mmap() 内存映射方式和 buffered I/O 来消耗的内存就属于这部分，最重要的是，这部分在真实的生产环境上也最容易产生问题

```java

$ free -k
              total        used        free      shared  buff/cache   available
Mem:        7926580     7277960      492392       10000      156228      430680
Swap:       8224764      380748     7844016
```

free 命令中的 buff/cache 是由 Buffers、Cached 和 SReclaimable 这三项组成的，它强调的是内存的可回收性，也就是说，可以被回收的内存会统计在这一项



如果不用内核管理的 Page Cache，那有两种思路来进行处理：

第一种，应用程序维护自己的 Cache 做更加细粒度的控制，比如 MySQL 就是这样做的，你可以参考MySQL Buffer Pool ，它的实现复杂度还是很高的。对于大多数应用而言，实现自己的 Cache 成本还是挺高的，不如内核的 Page Cache 来得简单高效。

第二种，直接使用 Direct I/O 来绕过 Page Cache，不使用 Cache 了，省的去管它了。这种方法可行么？那我们继续用数据说话，看看这种做法的问题在哪儿？

# 为什么需要 Page Cache？

标准 I/O 和内存映射会先把数据写入到 Page Cache，这样做会通过减少 I/O 次数来提升读写效率。

第二次读取文件的耗时远小于第一次的耗时，这是因为第一次是从磁盘来读取的内容，磁盘 I/O 是比较耗时的，而第二次读取的时候由于文件内容已经在第一次读取时被读到内存了，所以是直接从内存读取的数据，内存相比磁盘速度是快很多的。**这就是 Page Cache 存在的意义：减少 I/O，提升应用的 I/O 速度**



# Page Cache 是如何“诞生”的

Page Cache 的产生有两种不同的方式：Buffered I/O（标准 I/O）；Memory-Mapped I/O（存储映射 I/O）。这两种方式分别都是如何产生 Page Cache 的呢？

![linux-pagecache诞生](/images/posts/linux-pagecache诞生.jpg)

从图中你可以看到，虽然二者都能产生 Page Cache，但是二者的还是有些差异的：

标准 I/O 是写的 (write(2)) 用户缓冲区 (Userpace Page 对应的内存)，然后再将用户缓冲区里的数据拷贝到内核缓冲区 (Pagecache Page 对应的内存)；如果是读的 (read(2)) 话则是先从内核缓冲区拷贝到用户缓冲区，再从用户缓冲区读数据，也就是 buffer 和文件内容不存在任何映射关系。

对于存储映射 I/O 而言，则是直接将 Pagecache Page 给映射到用户地址空间，用户直接读写 Pagecache Page 中内容。显然，存储映射 I/O 要比标准 I/O 效率高一些，毕竟少了“用户空间到内核空间互相拷贝”的过程。这也是很多应用开发者发现，为什么使用内存映射 I/O 比标准 I/O 方式性能要好一些的主要原因