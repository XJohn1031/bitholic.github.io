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

GC(Grabage Collection)是JVM中最核心的技术之一，随着多年的发展，GC算法出现了很多变种，适用于不同的环境，选择合适的算法对于新手来说往往是一个难点。虽然这些算法的实现有所不同，但其核心思想都可总结为两步：

1. **找到所有“活”（使用）的对象**
2. **清除所有“死”（未使用）的对象。**

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

不同的GC算法主要的区别就在于如何清除无用对象，并且大致可以分为以下几种方式：

+ Sweep（清除）：这是最简单的方式，它直接将标记为无用的对象释放。一般内部维持一个free-list，记录所有空闲的空间，这种方式的问题在于会产生很多的内存碎片，可能出现内存碎片加起来很大但确出现OOM的情况。

    ![Mark_Sweep](/img/in-post/JVM_GC_Algorithms/Mark_Sweep.png)
    
+ Compact（压缩/整理）：这种方式弥补了Sweep的缺点，在执行清除后，将所有活着的对象都复制到一块，避免了内存碎片的产生。这种方法的缺点是增加了STW的时间，因为拷贝对象和更改引用地址需要花费额外的时间。

    ![Mark_Sweep_Compact](/img/in-post/JVM_GC_Algorithms/Mark_Sweep_Compact.png)
    
+ Copy（复制）：这种方式和Compact很像，都是重新分配活对象的空间，但是Copy一般使用两个空间，称为存活区，每次GC时都将存活的对象放到其中一个存活区里，这种方式的好处是拷贝可以和标记同时进行，降低了STW的时间，缺点是需要两个存活区，并且他们都得足够大以存放活对象，这就浪费了一些内存空间。
    
    ![Mark_and_Copy](/img/in-post/JVM_GC_Algorithms/Mark_and_Copy.png)
    
+ Inremental（增量）：增量不是一种新的算法，它的基本思想是如果GC一次性进行所有的垃圾清理会造成很长的STW，那么就将GC过程分为多个阶段，每次只用前面提到的算法来收集部分区域。这种方式的缺陷是带来了线程切换的额外消耗，造成系统吞吐量下降。
+ Generational（分代）：分代也不是一种新的算法，它的基本思想是将内存分代，不同的代使用前面提到过的不同的算法。一般将内存分为新生代（Young Generation）、老年代（Old/Tenured Generation），永久代（Permanent Generation），新生代一般存储刚产生的、存活周期短的对象，在新生代存活超过一定周期后对象会被放入老年代，永久代一般存储类型或方法信息（方法区），永久代现在也叫元空间（Metaspace）。在年轻代一般选择效率较高的复制算法，老年代一般选择标记-压缩算法。比较典型的分代如下示，后面再进行详细分析。

    ![GC_generations](/img/in-post/JVM_GC_Algorithms/GC_generations.png)

在选择GC算法时经常要考虑三个非常重要的方面：

+ Throughput（吞吐量）：在单位时间内能够处理的任务量，比如一些统计应用要求在一定时间内处理的任务量越多越好，允许个别任务响应很慢。
+ Latency（响应速度）：任务的平均响应时间越短越好，比如一般的web应用要求最快的响应速度。
+ Capacity（容量）：容量主要针对在使用一些小内存的机器时占用更少的内存。

## JVM上使用的垃圾回收器

### GC算法的分类
从不同的角度分析分代GC，可以将其分为不同的类型：

+ 按线程数分可以分为串行垃圾回收器（Serial）和并行垃圾回收器（Parallel）。串行垃圾回收器每次只使用一个线程进行垃圾回收，并行垃圾回收器每次将使用对个线程进行垃圾回收，在并行能力较强的CPU上，使用并行垃圾回收器可以减低STW时间。
+ 按工作模式分可以分为并发式垃圾回收器和独占式垃圾回收器。并发式垃圾回收器与应用程序交替工作，尽可能的减少STW时间。而独占式则不管这一点，开始后就一直运行直到回收结束。
+ 按工作的区域分又可以分为新生代垃圾回收器和老年代垃圾回收器。
+ 按内存碎片处理方式又可分为压缩式垃圾回收器和非压缩式垃圾回收器。

先来看下目前JVM（Java 8）中常用的GC算法（后面会详细分析每一种）：

| 新生代 | 老年代 | JVM参数 |
|----| --- | --- |
| Incremental | Incremental | -Xincgc |
| Serial | Serial | -XX:+UseSerialGC |
| Parallel Scavenge | Serial | -XX:+UseParallelGC -XX:-UseParallelOldGC |
| Parallel Scavenge | Parallel Old | -XX:+UseParallelGC -XX:+UseParallelOldGC |
| Serial | CMS | -XX:-UseParNewGC -XX:+UseConcMarkSweepGC |
| Parallel New | CMS | -XX:+UseParNewGC -XX:+UseConcMarkSweepGC |
| G1 | G1 | -XX:+UseG1GC |

#### 如何查看GC信息
1. 可通过JDK自带的工具jvisualvm查看可视化的GC过程
2. 可通过设置JVM启动参数`-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps`来打印GC信息。

>在翻阅文档的时候无意发现：JDK参数中-X开头的参数表示非标准的功能设置并且可能在后续JDK版本中被移除。-XX开头的参数表示非稳定的功能设置，行为可能在未提醒用户的情况下被改变。

#### Minor GC 、Major GC 与 Full GC

+ Minor GC: 一般将发生在新生代（包括Eden和Survivor区域）的内存回收称为 Minor GC。关于Minor GC需要注意的几点有：
    1. 当JVM无法为一个新的对象分配空间时会触发 Minor GC，比如当Eden区满了的时候。
    2. 在执行 Minor GC 操作时并不会影响到永久代，从永久代到年轻代的引用会被当做GC ROOTS，年轻代到永久代的引用在标记阶段会被直接忽略。
    3. 可能与感官相违背的是 Minor GC 带来的 STW 时间并不会特别久，因为新生代中大部分的对象都不符合GC存活条件，反而执行GC的过程会很快。

+ Major GC: 清理老年代
+ Full GC: 清理整个堆空间，包括新生代、老年代和元空间，通常发生在触发一次Minor GC时，发现统计数据说之前Minor GC的平均晋升大小比目前老年代的剩余的空间大，则不会触发Minor GC而改为触发Full GC。


#### Serial GC

串行垃圾回收器在新生代使用 mark-copy 算法，在老年代使用 mark-sweep-compact 算法。这种GC方式是单线程的，不能充分利用多核CPU的并行能力，并且在其执行时，会一直STW。这种方式的实现相对简单，没有线程切换的开销，在单CPU或较小内存的机器上性能表现可以超过并行回收器和并发回收器。

![SerialGC](/img/in-post/JVM_GC_Algorithms/SerialGC.png)

下面分析几个Serial GC的日志：

>2018-03-29T11:32:52.768-0800: 74.708: [GC (Allocation Failure) 2018-03-29T11:32:52.768-0800: 74.708: [DefNew: 35074K->2K(39424K), 0.0008172 secs] 37049K->1977K(126848K), 0.0008542 secs] [Times: user=0.06 sys=0.00, real=0.06 secs]

1. `2018-03-29T11:32:52.768-0800`: GC开始的时间
2. `74.708`: 相对于JVM启动时的GC开始时间，单位为秒
3. `GC`: 区分Minor GC和Full GC, 这里表示Minor GC
4. `Allocation Failure`: 发生GC的原因，这里表示触发GC的原因是新生代分配新的内存空间失败。
5. `DefNew`: 使用的垃圾回收器名称，这个奇怪的名字表示单线程的mark-copy STW收集器。
6. `35074K->2K`: 垃圾回收前后新生代的大小
7. `(39424K)`: 新生代的总大小
8. `37049K->1977K`: 垃圾回收前后堆的大小
9. `(126848K)`: 堆的总大小
10. `0008542 secs`: GC执行的时间
11. `[Times: user=0.00 sys=0.00, real=0.00 secs]`: 不同GC事件占据的时间，user表示垃圾回收线程消耗的CPU时间，sys表示花费在系统调用或等待系统事件的时间，real表示STW的时间。

#### Parallel GC

并行垃圾回收器在新生代使用 mark-copy 算法，在老年代使用 mark-sweep-compact 算法，但是这种方式会使用多个线程来执行垃圾回收动作，因此称为“Parallel”，这种方式的回收时间相对较少，可通过`-XX:ParallelGCThreads=N`调整Parallel GC的线程数，默认值是机器CPU的核数。这种方式可以最大化吞吐量。

![ParallelGC](/img/in-post/JVM_GC_Algorithms/ParallelGC.png)

#### CMS (Concurrent Mark and Sweep)

CMS垃圾回收器在年轻代使用parallel mark-copy 算法，在老年代使用 concurrent mark-sweep 算法。这种算法主要解决的问题是减少清理老年代的暂停时间，他并不会整理老年代的内存空间，使用free-list来管理空闲内存，在mark-sweep阶段的大部分操作都与应用程序并发运行。这种方式可以做大话降低延迟。

![CMS](/img/in-post/JVM_GC_Algorithms/CMS.png)


CMS在对老年代执行垃圾回收时分为以下几个步骤：

1. 起始标记：这个阶段会STW，这个阶段的目的是标记所有由GC ROOTS或年轻代指向的第一个引用。
2. 并发标记：