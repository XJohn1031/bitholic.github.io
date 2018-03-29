---
layout:     post
title:      "JVM GC Algorithms"
subtitle:   "常见JVM垃圾回收算法介绍"
date:       2018-03-29
author:     "bith01ic"
header-img: ""
tags:
    - Java
    - JVM
---

## 关于GC

GC(Grabage Collection)是JVM中最核心的技术之一，随着多年的发展，GC算法出现了很多变种，适用于不同的环境，选择合适的算法对于新手来说往往是一个难点。虽然这些算法的实现有所不同，但GC的核心思想基本一致：**找到所有“活”（使用）的对象，清除所有“死”（未使用）的对象。**

## 标记“活”的对象

所有的GC算法都要首先能找到“活”的对象，早期的GC使用引用计数的方式，这种方式有很多缺陷，比如不能清除循环引用。而现在的GC普遍采用可达性分析算法，首先来看一幅图：

![GC_ROOTS](/img/in-post/JVM_GC_Algorithms/GC_ROOTS.png)

从图中可以看出，我们可以从GC ROOTS开始找到所有有用的对象，那么首先看下什么样的对象可以作为有效的GC ROOTS：

+ VM栈中引用的对象，或者说当前所有被调用方法的引用类型的参数、局部变量、临时值。
+ JNI(native栈)中引用的对象
+ 方法区中常量或静态变量引用的对象
+ 系统类（由bootstrap/system class loader加载的对象，比如 rt.jar 中的 `java.util.*`）
+ 其它

标记时就可以从这些GC ROOTS出发找到当前所有有用的对象，并将其标记为“活”对象。

**需要注意的是，GC进程在执行标记时，应用进程是需要被停止的，对于应用层来说，这个阶段被称为“Stop the World(STW)”。并且暂停的时间取决于“活”对象的数目，而不在于堆的大小。**

## 清除无用的对象

不同的GC算法主要的区别就在于如何清除无用对象，并且大致可以分为以下几类：

+ Sweep（清除）：这是简单的方式，它直接将标记为无用的对象释放。一般内部维持一个free-list，记录所有空闲的空间，但这种方式的问题在于会产生很多内存碎片，可能出现内存碎片加起来很大确出现OOM。

    ![Mark_Sweep](/img/in-post/JVM_GC_Algorithms/Mark_Sweep.png)
    
+ Compact（压缩、整理）：这种方式弥补了Sweep的缺点，在执行为清除后，将所有的对象都拷贝到一块，避免了内存碎片的产生。这种方法的缺点是会增加STW的时间，因为拷贝对象和更改引用地址需要额外的时间。

    ![Mark_Sweep_Compact](/img/in-post/JVM_GC_Algorithms/Mark_Sweep_Compact.png)
    
+ Copy（复制）：这种方式和Compact很像，都是重新分配活对象的空间，但是Copy一般使用两个空间，一般称为存活区，每次GC时都将存活的对象放到其中一个存活区里，这种方式的好处是拷贝可以和标记同时进行，降低STW的时间，缺点是需要两个存活区，为了能够存下活的对象，他们都得足够大，这就浪费了一些内存空间。
    
    ![Mark_and_Copy](/img/in-post/JVM_GC_Algorithms/Mark_and_Copy.png)
    
+ Inremental（增量）：增量不是一种新的算法，它的基本思想是如果GC一次性进行所有的垃圾清理会造成很长的STW，那么就将GC过程分为多个阶段，每次只收集部分区域。这种方式的缺陷是带来了线程切换的额外消耗，造成系统吞吐量下降。
+ 分代：分代也不是一种新的算法，它的基本思想是将内存分代，不同的代使用前面提到过的不同的算法。一般将内存分为新生代（Young Generation）和老年代（Old Generation），新生代一般存储刚产生的、存活周期短的对象，在新生代存活超过一定周期后对象会被放入老年代。在年轻代一般选择效率较高的复制算法，老年代一般选择标记-压缩算法。如：

    ![GC_generations](/img/in-post/JVM_GC_Algorithms/GC_generations.png)

## 详解分代

分代是一种重要的思想，
