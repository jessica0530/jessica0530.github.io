---
layout: post
title: Java Volatile ThreadLocal
categories: [java]
description: Java Volatile
keywords: java
---
# Volatile

Java的内存模型实现总是从主存(即共享内存)读取变量，是不需要进行特别的注意 的。而在当前的 Java 内存模型下，线程可以把变量保存本地内存(比如机器的寄存器)中，而不是直 接在主存中进行读写。这就可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使 用它在寄存器中的变量值的拷⻉，造成数据的不一致。

![java-volatile-1](/images/posts/java-volatile-1.png)

![java-volatile-2](/images/posts/java-volatile-2.png)

volatile 关键字的主要作用就是保证变量的可⻅性

然后还有一个作用是防止指令重排序。

1. 原子性 : 一个的操作或者多次操作，要么所有的操作全部都得到执行并且不会收到任何因素的 干扰而中断，要么所有的操作都执行，要么都不执行。 synchronized 可以保证代码片段的原子 性。
2. 可⻅性 :当一个变量对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新 值。 volatile 关键字可以保证共享变量的可⻅性。
3. 有序性 :代码在执行的过程中的先后顺序，Java 在编译器以及运行期间的优化，代码的执行顺 序未必就是编写代码时候的顺序。 volatile 关键字可以禁止指令进行重排序优化

# volatile的禁止指令重排序

我们都知道volatile关键字有两个语义：

保证内存可见性
禁止指令重排序

其中JVM对其禁止指令重排序在硬件层面的实现就是通过在volatile修饰的变量前后插入内存屏障。volatile变量的内存屏障规则如下：

在每个volatile写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障；
在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障；



# ThreadLocal

通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。如果想实现每一个线程都有自己的 专属本地变量该如何解决呢? JDK中提供的 ThreadLocal 类正是为了解决这样的问题。 **ThreadLocal** 类 主要解决的就是让每个线程绑定自己的值，可以将 **ThreadLocal** 类形象的比喻成存放数据的盒子，盒子 中可以存储每个线程的私有数据。 

如果你创建了一个 **ThreadLocal** 变量，那么访问这个变量的每个线程都会有这个变量的本地副本，这也 是 **ThreadLocal** 变量名的由来。他们可以使用 **get**() 和 **set**() 方法来获取默认值或将其值更改 为当前线程所存的副本的值，从而避免了线程安全问题。 



```java
      public void set(T value) {
         Thread t = Thread.currentThread();
         ThreadLocalMap map = getMap(t);
         if (map != null)
             map.set(this, value);
         else
             createMap(t, value);
     }
     ThreadLocalMap getMap(Thread t) {
         return t.threadLocals;
     }
```

最终的变量是放在了当前线程的 **ThreadLocalMap** 中，并不是存在 **ThreadLocal** 上， **ThreadLocal** 可以理解为只是 **ThreadLocalMap** 的封装，传递了变 量值。 ThrealLocal 类中可以通过 Thread.currentThread() 获取到当前线程对象后，直接通过 

getMap(Thread t) 可以访问到该线程的 ThreadLocalMap 对象。
 **ThreadLocal** 内部维护的是一个类似 **Map** 的 **ThreadLocalMap** 数据结构， **key** 为当前对象的 

**Thread** 对象，值为 **Object** 对象。 

# ThreadLocal内存泄漏问题

ThreadLocalMap 中使用的 key 为 ThreadLocal 的弱引用,而 value 是强引用。所以，如果 

ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清 理掉。这样一来， ThreadLocalMap 中就会出现key为null的Entry。假如我们不做任何措施的话， value 永远无法被GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap实现中已经考虑了这种 情况，在调用 set() 、 get() 、 remove() 方法的时候，会清理掉 key 为 null 的记录。使用完 

ThreadLocal 方法后 最好手动调用 remove() 方法 

```java
  static class Entry extends WeakReference<ThreadLocal<?jk {
             /** The value associated with this ThreadLocal. */
             Object value;
             Entry(ThreadLocal<?> k, Object v) {
                 super(k);
value = v; }
}
```

弱引用介绍: 

如果一个对象只具有弱引用，那就类似于可有可无的生活用品。弱引用与软引用的区别在于:只具 有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一 旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾 回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 

弱引用可以和一个引用队列(ReferenceQueue)联合使用，如果弱引用所引用的对象被垃圾回收， Java虚拟机就会把这个弱引用加入到与之关联的引用队列中 