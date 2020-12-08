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



![class-loader](/images/posts/class-loader.jpg)

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



# Native Method



## 与java环境外交互

有时java应用需要与java外面的环境交互。这是本地方法存在的主要原因，你可以想想java需要与一些底层系统如操作系统或某些硬件交换信息时的情况。本地方法正是这样一种交流机制：它为我们提供了一个非常简洁的接口，而且我们无需去了解java应用之外的繁琐的细节

## 与操作系统交互

JVM支持着java语言本身和运行时库，它是java程序赖以生存的平台，它由一个解释器（解释字节码）和一些连接到本地代码的库组成。然而不管怎样，它毕竟不是一个完整的系统，它经常依赖于一些底层（underneath在下面的）系统的支持。这些底层系统常常是强大的操作系统。通过使用本地方法，我们得以用java实现了jre的与底层系统的交互，甚至JVM的一些部分就是用C写的，还有，如果我们要使用一些java语言本身没有提供封装的操作系统的特性时，我们也需要使用本地方法

## Sun's Java

Sun的解释器是用C实现的，这使得它能像一些普通的C一样与外部交互。jre大部分是用java实现的，它也通过一些本地方法与外界交互。例如：类java.lang.Thread 的 setPriority()方法是用java实现的，但是它实现调用的是该类里的本地方法setPriority0()。这个本地方法是用C实现的，并被植入JVM内部，在Windows 95的平台上，这个本地方法最终将调用Win32 SetPriority() API。这是一个本地方法的具体实现由JVM直接提供，更多的情况是本地方法由外部的动态链接库（external dynamic link library）提供，然后被JVM调用。



可以将native方法比作Java程序同Ｃ程序的接口，其实现步骤：
１、在Java中声明native()方法，然后编译；
２、用javah产生一个.h文件；
３、写一个.cpp文件实现native导出方法，其中需要包含第二步产生的.h文件（注意其中又包含了JDK带的jni.h文件）；
４、将第三步的.cpp文件编译成动态链接库文件；
５、在Java中用System.loadLibrary()方法加载第四步产生的动态链接库文件，这个native()方法就可以在Java中被访问了



**JNI**

开始本篇的内容之前，首先要讲一下JNI。Java很好，使用的人很多、应用极 广，但是Java不是完美的。Java的不足体现在运行速度要比传统的C++慢上许多之外，还有Java无法直接访问到操作系统底层如硬件系统，为此 Java提供了JNI来实现对于底层的访问。JNI，Java Native Interface，它是Java的SDK一部分，JNI允许Java代码使用以其他语言编写的代码和代码库，本地程序中的函数也可以调用Java层的函 数，即JNI实现了Java和本地代码间的双向交互。

 

**Native**

JDK开放给用户的源码中随处可见Native方法，被Native关键字声明的方法说明该方法不是以Java语言实现的，而是以本地语言实现的，Java可以直接拿来用。这里有一个概念，就是本地语言，本地语言这四个字，个人理解应该就是可以和操作系统直接交互的语言。

