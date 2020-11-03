---
layout: post
title: ClickHouse
categories: [clickhouse]
description: ClickHouse
keywords: clickhouse
---

# ClickHouse



https://clickhouse.tech/docs/en/introduction/distinctive-features/

https://clickhouse.tech/benchmark/dbms/



## LSM树原理

把一棵大树拆分成N棵小树，它首先写入内存中，随着小树越来越大，内存中的小树会flush到磁盘中，磁盘中的树定期可以做merge操作，合并成一棵大树，以优化读性能



