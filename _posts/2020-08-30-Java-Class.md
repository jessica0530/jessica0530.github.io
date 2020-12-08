---
layout: post
title: Java Class 和类加载过程
categories: [java]
description: Java Class
keywords: java
---

# 类文件

类文件结构

![java-class](/images/posts/java-class.png)

```java
 ClassFile {
u4 magic; //Class 文件的标志
u2 minor_version;//Class 的小版本号
u2 major_version;//Class 的大版本号
u2 constant_pool_count;//常量池的数量
cp_info constant_pool[constant_pool_count-1];//常量池
u2 access_flags;//Class 的访问标记
u2 this_class;//当前类
u2 super_class;//父类
u2 interfaces_count;//接口
u2 interfaces[interfaces_count];//一个类可以实现多个接口 u2 fields_count;//Class 文件的字段属性
field_info fields[fields_count];//一个类会可以有个字段
u2 methods_count;//Class 文件的方法数量
method_info methods[methods_count];//一个类可以有个多个方法
u2 attributes_count;//此类的属性表中的属性数 attribute_info attributes[attributes_count];//属性表集合
}
```

1. 魔数**:** 确定这个文件是否为一个能被虚拟机接收的 Class 文件。 

2. **Class** 文件版本 :Class 文件的版本号，保证编译正常执行。 

3. 常量池 :常量池主要存放两大常量:字面量和符号引用。 

4. 访问标志 :标志用于识别一些类或者接口层次的访问信息，包括:这个 Class 是类还是接口， 

   是否为 public 或者 abstract 类型，如果是类的话是否声明为 final 等等。 

5. 当前类索引**,**父类索引 :类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的 

   全限定名，由于 Java 语言的单继承，所以父类索引只有一个，除了 java.lang.Object 之 外，所有的 java 类都有父类，因此除了 java.lang.Object 外，所有 Java 类的父类索引都 不为 0。 

6. 接口索引集合 :接口索引集合用来描述这个类实现了那些接口，这些被实现的接口将
    按 implents (如果这个类本身是接口的话则是 extends ) 后的接口顺序从左到右排列在接口索引集合中。 

7. 字段表集合 :描述接口或类中声明的变量。字段包括类级变量以及实例变量，但不包括在方法 内部声明的局部变量

8. 方法表集合 :类中的方法。 

9. 属性表集合 : 在 Class 文件，字段表，方法表中都可以携带自己的属性表集合 

# 类加载过程

类加载过程:加载**->**连接**->**初始化。连接过程又可分为三步:验证**->**准备**->**解析。 

类加载过程的第一步，主要完成下面3件事情: 

1. 通过全类名获取定义此类的二进制字节流
2. 将字节流所代表的静态存储结构转换为方法区的运行时数据结构
3. 在内存中生成一个代表该类的 Class 对象,作为方法区这些数据的访问入口 

![类加载过程](/images/posts/类加载过程.png)

# 类加载器

JVM 中内置了三个重要的 ClassLoader，除了 BootstrapClassLoader 其他类加载器均由 Java 实现且 全部继承自 java.lang.ClassLoader : 

1. **BootstrapClassLoader(**启动类加载器**)** :最顶层的加载类，由C++实现，负责加载 %JAVA_HOME%/lib 目录下的jar包和类或者或被 -Xbootclasspath 参数指定的路径中的所有类。 
2. **ExtensionClassLoader(**扩展类加载器**)** :主要负责加载目录 %JRE_HOME%/lib/ext 目录下的jar 包和类，或被 java.ext.dirs 系统变量所指定的路径下的jar包。 
3. **AppClassLoader(**应用程序类加载器**)** :面向我们用户的加载器，负责加载当前应用classpath下 

的所有jar包和类。 

## 双亲委派模型

每一个类都有一个对应它的类加载器。系统中的 ClassLoder 在协同工作的时候会默认使用 双亲委派 模型 。即在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则 才会尝试加载。加载的时候，首先会把该请求委派该父类加载器的 **loadClass()** 处理，因此所有的请 求最终都应该传送到顶层的启动类加载器 **BootstrapClassLoader** 中。当父类加载器无法处理时，才由 自己来处理。当父类加载器为null时，会使用启动类加载器 BootstrapClassLoader 作为父类加载器 



双亲委派模型实现源码分析

双亲委派模型的实现代码非常简单，逻辑非常清晰，都集中在 java.lang.ClassLoader 的 loadClass() 中，相关代码如下所示。 

```java
private final ClassLoader parent;
 protected Class<?> loadClass(String name, boolean resolve)
         throws ClassNotFoundException
     {
synchronized (getClassLoadingLock(name)) { // 首先，检查请求的类是否已经被加载过 Class<?> c = findLoadedClass(name);
if (c WX null) {
                 long t0 = System.nanoTime();
                 try {
if (parent êX null) {//父加载器不为空，调用父加载器loadClass()方法处理 c = parent.loadClass(name, false);
} else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载 c = findBootstrapClassOrNull(name);
                     }
                 } catch (ClassNotFoundException e) {
//抛出异常说明父类加载器无法完成加载请求 }
if (c WX null) {
long t1 = System.nanoTime(); //自己尝试加载
c = findClass(name);
                     // this is the defining class loader; record the stats
                     sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                     sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                     sun.misc.PerfCounter.getFindClasses().increment();
} }
             if (resolve) {
                 resolveClass(c);
}  
 return c;
 }
```



# 双亲委派模型带来了什么好处

双亲委派模型保证了Java程序的稳定运行，可以避免类的重复加载(JVM 区分不同类的方式不仅仅根据 类名，相同的类文件被不同的类加载器加载产生的是两个不同的类)，也保证了 Java 的核心 API 不 被篡改。如果不用没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我 们编写一个称为 java.lang.Object 类的话，那么程序运行的时候，系统就会出现多个不同的 Object 类。 



# 如果我们不想用双亲委派模型怎么办

为了避免双亲委托机制，我们可以自己定义一个类加载器，然后重载 loadClass() 即可。 如何自定义类加载器**?** 

除了 BootstrapClassLoader 其他类加载器均由 Java 实现且全部继承自 java.lang.ClassLoader 。如 果我们要自定义自己的类加载器，很明显需要继承 ClassLoader  

加入自定义的类加载器来进行拓展，典型的如增加除了磁盘位置之外的Class文件来源，或者通过类加载器实现类的隔离、重载等功能