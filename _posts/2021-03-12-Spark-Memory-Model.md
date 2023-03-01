---
layout: post
title: spark Memory Model
categories: [spark]
description: Spark Memory Model
keywords: spark
---

Spark Memory usages

Off-heap memory



compressd buffered etc.

Netty direct buffer.

Off-heap execution



Off-heap storage



## Executor memory model

Spark Memory (0.75  Storage Memory/Execution Memory)

User Memory (0.25)

Reserved Memory (300MB)



## Executor memory model  - a bit of history

2014(Dynamic Assignment, Static Memory Management)

2015(Project Tungsten)

2016.01(Off-heap Execution,Cooperative Spilling,Unified Memory Management)

2016.07(Off-heap Storage)



## Executor memory model -Execution vs Storage

Execution

Memory used for shuffle,joins,sorts,aggregations

Storage

Memory used to cache data that will be reused later



## How to arbitrate memory between execution and storage

Execution/storage

When execution is full, spill to disk

Storage is full,will evict LRU block to disk



## Problems of static assignment

1.execution can only use a fraction of the memory

2.efficient use of memory required user tunning(调优)

## Unified memory management



## Dynamic assignment between tasks

the share of each task depends on number of actively running tasks

if another task comes along so the first task will have to spill



Each task is now assigned 1/N of the memory,where N=4

Each task is no assigned 1/N of the memory, where N=2



Cooperative spilling (Sort forces Aggragate to spill a page to free memory)



## Execution memory model -Project Tungsten

Memory management and Binary Processing

In-memory binary date representation: row format and shuffle data

Cache-aware Computation

Faster sorting and hashing for aggregation,joins, and shuffle

Code Generation (not in this topic)

Faster expression evaluation and DataFrame/SQL operators



## Project Tungsten: row binary format

Native: 4 bytes with UTF-8 encoding

Java: 48 bytes

