---
layout: post
title: Spark SQL CodeGeneration
categories: [spark]
description: Spark SQL Code
keywords: spark
---

## Volcano Model



### Pros

Simple & clean, pipeline mode,operator independent easy to extend

### Cons

Too many virtual function calls

Poor code locality & complex book-keeping

not friendly with SIMD



虚函数是 C/C++ 里的概念,主要是为了实现多态,java里所有的普通函数都是虚函数,除非函数加了final/private.至于虚函数为什么cost这么高,大致是两个原因,1 是虚函数调用的机器指令更多，二是cpu cache 不友好

## Optimize Volcano Model

（http://www.vldb.org/pvldb/vol11/p2209-kersten.pdf）

Vectorized tuple processing  -> (DB2 BLU,columnar SQL Server,QuickStep)

Query compilation (data-entric code generation) -> (Apache Spark,Peloton)

向量化执行在memory-bound类的查询中更有优势，代码生成在calculation-heavy 类的查询中更有优势。但是总体来看,在OLAP场景中,向量化执行和代码生成的执行性能相近

## Vectorized vs query compilation

Vectorized vs compiled

Query compilation envolve （https://zhuanlan.zhihu.com/p/60965109）

Compare to orign volcona model, we generate code more close to machine code,and more friendly to machine

Relaxed Operator Fusion for In-Memory Database 

(http://www.vldb.org/pvldb/vol11/p1-menon.pdf)



## How to compile

SystemR (Machine code)  -> hard code

Hyper(LLVM IR) -> The LLVM compiler infrastructure project is a set of compiler and toolchain technologies

SparkSQL(VIrtual Machine bytecode) -> generate java code ->compile to bytecode with janio



## Efficiently Compiling Efficient Query Plans for Modern Hardware

Https://www.vldb.org/pvldb/vol4/p539-neuman.pdf



## Spark SQL code generation

Https://databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html

### Project Tungsten

Elimate cpu & memory bottleneck

1.Memory Management and Binary Processing (To tackle both object overhead and GC's inefficiency)

2.Cache-aware computation.(designing cache-friendly algorithms and data structures so Spark applications will spend less time waiting to fetch data from memory and more time doing useful work)

3.Code generation

### Janino compiler

Janino can not only compile a set of source files to a set of class file like JAVAC,but also compile a Java expression, a block, a class body, one.java file or a set of .java files in memory, load the bytecode and execute it directly in the same JVM

### Expression code generation

CodegenContext:记录将要生成代码中的各种元素,比如变量,函数等

CodeGenerator: 一个基类,对外提供代码生成的接口generate,相关的实现有七个,比如GeneratePredict就是实现谓词的codegen

FilterExec 为例

FilterExec.doExecute -> GeneratePredicate.generate -> GeneratePredicate.create

### WholeStage code generation

#### collapseCodegenStages

1.将支持codegen的算子pipeline 在一起,并在外层添加一个WholeStageCodegenExec

2.不支持codegen 的算子上添加一个适配器InputAdapter

#### codegenSUpport 

1.支持wholeStageCodegen 的算子需要实现该接口

2.CodegenSupport 主要包含consume/doConsume 和produce/doProduce两对方法,

consume 和produce 都是final类型,区别在于produce会调用doProduce方法,而consume 会调用父节点的doconsume方法