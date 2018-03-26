---
layout:     post
title:      "How Java Threads Works"
date:       2018-03-26
author:     "bith01ic"
header-img: "img/how_java_threads_workds_bg.jpg"
tags:
    - Java
    - 并发
---


# How Java Threads Works

## 1. 线程简介

### 1.1 并发（Concurrency）的历史：

1. 在上古时期，那是的计算机还没有操作系统，程序只能从头到尾的运行并且能够获取计算机的所有资源。
2. 在操作系统出现以后，允许了多个程序作为不同的**进程（processes）**同时运行，操作系统给进程分配资源，线程间可以通过sockets, signal, shared memory, files等进行通信。
3. 随着计算机程序复杂度的提高，同一个进程也需要在同时运行不同的任务，这就激发了线程（threads）的出现。线程允许在同一个进程内并发执行多个程序流，并且它们共享进程的资源。

线程也被称为轻量级进程，在现代操作系统上，线程（而不是进程）往往作为CPU调度的基本单元。在现代的多核CPU上，线程能够做到真正意义上的同时运行（并行）。

> 并发（Concurencey）：多个任务以时间片的方式在一段时间内交替运行（上下文切换），在指定时刻，还是只有一个任务在执行。
> 
> 而并行（Parallelism）：多个任务真正的同时运行。

### 1.2 线程带来的风险

1. 安全问题：
2. 活性问题：
3. 性能问题：

## 2. Java中的线程

Java中线程是以类`Thread`来表示的，在Java中创建线程的唯一方式就是创建一个`Thread`的实例。可通过调用线程的`start()`方法来启动这个线程。

没有正确同步的线程的行为往往会让人感到困扰。Java通过*Java programming language memory model*(有时会简称为the memory model)的语义描述多线程的行为，这个语义并没有规定多线程程序应该怎样运行，而是描述了多线程程序运行时可被允许的行为表现。

### 2.1 同步（Synchronization）

Java语言提供了多种进程间通信的机制，最常见也最基础的就是通过monitors实现的同步机制。Java中的每个对象都会与一个monitor相关联，线程可以*lock*和*unlock*这个monitor。

> 线程lock了一个对象的monitor有时也称为线程拥有了monitor的锁或线程拥有了对象的锁，同样的，线程unlock一个对象的monitor也称为线程释放了monitor的锁或线程释放了对象的锁。

在同一个时刻只能有一个线程能够拥有对monitor的锁，其它尝试lock这个monitor的线程都会被阻塞直到他们能够获得到锁。同一个线程可以多次lock同一个monitor，每次unlock都会释放一次对monitor的锁。

Java中通过`synchronized`关键字来自动获取和释放monitor的锁，在`synchronized`修饰的方法或代码块开始运行时会自动尝试lock相应对象的monitor，只有lock操作成功才会接着往下执行，当方法或代码块（不管是正常还是非正常）执行完后，会自动unlock相应的monitor。

这里monitor对应的对象是：

+ 如果`synchronzied`修饰的是非静态方法，那么monitor对象是`this`指向的对象。
+ 如果`synchronzied`修饰的是静态方法，那么monitor对象是该类的`Class`对象。
+ 如果`synchronzied`修饰的是代码块，那么需要显示指定对象作为monitor对象。

示例：

```java
public class Monitor {

    private static int b = 0;

    private static final Object staticLock = new Object();

    private int a = 0;

    private final Object lock = new Object();

    // 1: 修饰非静态方法, monitor 对象是 this
    public synchronized void add() {
        a++;
    }

    // 2: 修饰非静态方法内部代码块, monitor 对象是 this, 等价于1
    public void innerAdd() {
        synchronized (this) {
            a++;
        }
    }

    // 3: 修饰非静态方法内部代码块并显示指定了 lock 为 monitor 对象
    public void innerAddWithOtherMonitor() {
        synchronized (lock) {
            a++;
        }
    }

    // 4: 修饰静态方法, monitor 对象是 Monitor.cass
    public synchronized static void staticAdd() {
        b++;
    }

    // 5: 修饰静态方法内部方法块，monitor 对象是 Monitor.class, 等价于4
    public static void staticInnerAdd() {
        synchronized (Monitor.class) {
            b++;
        }
    }

    // 6: 修饰静态方法内部方法块并显式指定了 staticLock 为 monitor 对象
    public static void staticInnerAddWithOtherMonitor() {
        synchronized (staticLock) {
            b++;
        }
    }
}
```

### 2.2 Wait Sets and Notification

Java 中每个对象除了都和一个 monitor 关联外, 还会和一个 wait set 相关联。wait set是一个线程集, 当一个对象新创建时，它的 wait set 是空的。添加线程到 wait set 和从 wait set 移除线程等基础操作都是原子的。wait set 状态受到以下几种方法影响：

+ `wait()`
+ `notify()`
+ `notifyAll()`
+ interruptions

#### 2.2.1 Wait

等待动作发生在 `wait()`, `wait(long millisecs)` 或 `wait(long millisecs, int nanosecs)` 方法被调用时，一个处在等待状态的线程可正常退出或通过抛出`InterruptedException`异常退出。

假设线程*t* 将在对象*m* 上执行了`wait()`方法（*n* 是*t* 在*m* 上已获取锁的次数），那么会有下面几种情况发生：

+ 如果*n* 是0， 那么会抛出`IllegalMonitorStateException`异常
+ 如果是定时的`wait()`方法，`millisecs`参数是负的或者`nanosecs`参数不在0-999999范围内，那么会抛出`IllegalArgumentException`异常
+ 如果线程*t* 被interrupted，那么会抛出`InterruptedException`异常，并且*t* 的interruption status会被设置为false
+ 除去上面的情况，会发生:
    1. 线程*t* 会被添加到*m* 的 wait set 中，并释放对*m* 的*n* 次锁。
    2. 线程*t* 不会再执行任何操作直到它从*m* 的 wait sets 里移除。线程会在以下动作发生时从 wait sets 中移除：
        + *m* 的`notify()`方法被调用，而*t* 刚好作为被选中的线程。
        + *m* 的`notifyAll()`方法被调用
        + 线程*t* 的interrupt动作发生
        + 带超时的`wait()`在超时时
        + 虚假唤醒（spurious wake-ups），为了避免虚假唤醒，一般将判断条件放到循环中。
    3. 线程*t* 再对 *m* 进行 *n* 次锁。
    4. 如果线程*t* 是因为 interrupt 而从 wait set 中移除，那么线程*t* 的interruption status会被设置为 false ，并抛出`InterruptedException`。

#### 2.2.2



        
## 参考文献

1. *The Java Language Specifications Java SE 10 edition*
2. *Java concurrency in practice*
3. *Thinking in Java 4th edition*