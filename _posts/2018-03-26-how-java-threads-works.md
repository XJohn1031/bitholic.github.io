---
layout:     post
title:      "How Java Threads Works"
subtitle:   "理解Java中线程的工作方式"
date:       2018-03-26
author:     "bith01ic"
header-img: ""
tags:
    - Concurrency
    - Java
---


# How Java Threads Works

## 1. 线程简介

### 1.1 并发（Concurrency）的历史：

1. 在上古时期，那时的计算机还没有操作系统，程序只能从头到尾的运行并且能够获取计算机的所有资源。
2. 在操作系统出现以后，允许了多个程序作为不同的**进程（processes）**同时运行，操作系统给进程分配资源，线程间可以通过sockets, signal, shared memory, files等进行通信。
3. 随着计算机程序复杂度的提高，同一个进程也需要在同时运行不同的任务，这就激发了线程（threads）的出现。线程允许在同一个进程内并发执行多个程序流，并且它们共享进程的资源。

线程也被称为轻量级进程，在现代操作系统上，线程（而不是进程）往往作为CPU调度的基本单元。在现代的多核CPU上，线程能够做到真正意义上的同时运行（并行）。

> 并发（Concurencey）：多个任务以时间片的方式在一段时间内交替运行（上下文切换），在指定时刻，还是只有一个任务在执行。
> 
> 并行（Parallelism）：多个任务真正的同时运行。

### 1.2 线程带来的风险

1. 安全问题：
2. 活性问题：
3. 性能问题：

## 2. Java中的线程

Java中线程是以类`Thread`来表示的，在Java中创建线程的唯一方式就是创建一个`Thread`的实例。可通过调用线程的`start()`方法来启动这个线程。

没有正确同步的线程的行为往往会让人感到困扰。Java通过*Java programming language memory model*(有时会简称为the memory model)的语义描述多线程的行为，这个语义并没有规定多线程程序应该怎样运行，而是描述了多线程程序运行时可被允许的行为表现。

### 2.1 同步（Synchronization）

Java 语言提供了多种线程间通信的机制，最常见也最基础的就是通过 monitors 实现的同步机制。Java 中的每个对象都会与一个 monitor 相关联，线程可以 *lock* 和 *unlock* 这个 monitor。

> 线程 lock 了一个对象的 monitor 有时也称为线程拥有了 monitor 的锁或线程拥有了对象的锁，同样，线程 unlock 一个对象的 monitor 也称为线程释放了 monitor 的锁或线程释放了对象的锁。

在同一个时刻只能有一个线程能够拥有对 monitor 的锁，其它尝试 lock 这个 monitor 的线程都会被阻塞直到他们能够获得到锁。同一个线程可以多次 lock 同一个 monitor，每次 unlock都会释放一次对 monitor 的锁。

Java 中通过`synchronized`关键字来自动获取和释放 monitor 的锁，在`synchronized`修饰的方法或代码块开始运行时会自动尝试 lock 相应对象的 monitor，只有 lock 操作成功才会接着往下执行，当方法或代码块（不管是正常还是非正常）执行完后，会自动 unlock 相应的 monitor。

这里 monitor 对应的对象是：

+ 如果`synchronzied`修饰的是非静态方法，那么 monitor 对象是`this`指向的对象。
+ 如果`synchronzied`修饰的是静态方法，那么 monitor 对象是该类的`Class`对象。
+ 如果`synchronzied`修饰的是代码块，那么需要显示指定对象作为 monitor 对象。

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
+ 如果线程*t* 被interrupted，那么会抛出`InterruptedException`异常，并且*t* 的 interruption status会被设置为 false
+ 除去上面的情况，会发生:
    1. 线程*t* 会被添加到*m* 的 wait set 中，并释放对*m* 的*n* 次锁。
    2. 线程*t* 不会再执行任何操作直到它从*m* 的 wait sets 里移除。线程会在以下动作发生时从 wait sets 中移除：
        + *m* 的`notify()`方法被调用，而*t* 刚好作为被选中的线程。
        + *m* 的`notifyAll()`方法被调用
        + 线程*t* 的interrupt动作发生
        + 带超时的`wait()`在超时时
        + 虚假唤醒（spurious wake-ups），为了避免虚假唤醒，一般将判断条件放到循环中。
    3. 线程*t* 再对 *m* 进行 *n* 次锁。
    4. 如果线程*t* 是因为 interrupt 而从 wait set 中移除，那么线程*t* 的interruption status会被设置为 false ，并抛出`InterruptedException`异常。

#### 2.2.2 Notification

通知动作发生在 `notify()` 或 `notifyAll()` 方法被调用时，继续上面线程的例子说明通知动作发生时的状况：

+ 如果 *n* 是0，那么抛出`IllegalMonitorStateException`异常
+ 如果 *n* 大于0并且是`notify()`动作，那么如果 *m* 的 wait set 不是空的，那么会从 wait set 随机移除一个线程 *u*
+ 若果 *n* 大于0并且是`notifyAll()`动作，那么 *m* 的 wait set 中的所有线程都会被移除，注意虽然 wait set 中所有的线程都会被唤醒，但是在任意时刻只有一个线程能够重新获得 monitor 的锁。

#### 2.2.3 Interruptions

中断动作发生在 `Thread.interrupt()` 方法被调用时，假设线程*t* 会调用线程*u* 的`u.interrupt()`，其中的*t* 和*u* 可以是同一个线程，中断动作会将线程*u* 的中断标志（interruption status）设置为true。其次如果线程在某个对象 *m* 的 wait set 里，那么 *u* 将从这个 wait set 里移除，这样会使 *u* 从睡眠状态被唤醒，重新获得 *m* 的 monitor 锁后抛出`InterruptedException`异常。

`Thread.isInterrupted()` 可以获取线程的中断状态，`Thread.interrupted()` 可以获取线程的中断状态并重置中断状态为false。

#### 2.2.4 Waits, Notification 和 Interruption 的交互

如果一个处于 waiting 状态的线程同时收到了通知动作和中断动作，那么它可能会发生：

1. 从`wait()`中正常返回，挂起interrupt动作（此时调用`Thread.interrupted()`会返回 true ）
2. 从`wait()`中返回并抛出`InterruptedException`异常

同样，通知动作也不会因为中断而丢失，假定一个线程集*s* 在对象*m* 的 wait set 里，另外一个线程调用了 *m* 的`notify()`方法，则可能会发生：

1. *s* 中至少有一个线程正常返回；
2. *s* 中的所有线程都返回并抛出`InterruptedException`异常。

注意一个线程如果同时被`notify()`中断或唤醒，那么该线程会返回并抛出`InterruptedException`异常，这时 wait set 里的另外一个线程需要被通知。

### Sleep and Yield

`Thread.sleep()` 会使当前运行的线程睡眠（暂停执行）一段时间，但不会释放已拥有的 monitor 的锁。值得注意的是`Thread.sleep()`和`Thread.yield()`都没有保证同步机制，比如编译器可能不会将寄存器上已写的值保存到内存中去，也不会保证在调用`Thread.sleep()`或`Thread.yield()`后重新载入值。

### 线程状态

Java 给线程定义了6种状态：

+ NEW: 新建，一个线程被创建但是还未调用其`start()`方法时的状态
+ RUNNABLE: 可运行，一个线程在JVM中运行时的状态
+ BLOCKED: 阻塞，一个线程因为等待获取 monitor 的锁而被阻塞时的状态
+ WAITING: 等待，一个线程无限期的等待其他线程对其进行特定操作时的状态，比如调用没有带时间参数的`wait()`或`join()`方法
+ TIEMED_WAITING: 带超时的等待，一个线程在一定时间范围内等待其他线程对其进行特定操作时的状态，比如调用带时间参数的`wait()`、`sleep()`或`join()`方法
+ TERMINATED: 终止，一个线程退出后所处的状态

示例：

```java
import java.util.concurrent.TimeUnit;

public class ThreadStatus {

    public static synchronized void lock() {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException ex) {
            System.out.println("interrupted");
        }
    }

    public synchronized void waitNotify() {
        try {
            this.wait();
        } catch (InterruptedException ex) {
            System.out.println("interrupted");
        }
    }

    public synchronized void waitNotifyWithTimeout() {
        try {
            this.wait(5000);
        } catch (InterruptedException ex) {
            System.out.println("interrupted");
        }
    }

    public static void main(String[] args) throws Exception {
        final ThreadStatus threadStatus = new ThreadStatus();
        Thread thread1 = new Thread(() -> {
            System.out.println("NEW");
        });

        Thread thread2 = new Thread(() -> {
            System.out.println("RUNNABLE");
            int result = 0;
            for (int i = 1; i < Integer.MAX_VALUE; i++) {
                result = i * i / i;
            }
            System.err.println(result);
        });

        Thread lockThread = new Thread(ThreadStatus::lock);

        Thread thread3 = new Thread(() -> {
            System.out.println("BLOCKED");
            ThreadStatus.lock();
        });

        Thread thread4 = new Thread(() -> {
            System.out.println("WAITING");
            threadStatus.waitNotify();
        });

        Thread thread5 = new Thread(() -> {
            System.out.println("TIMED_WAITING");
            threadStatus.waitNotifyWithTimeout();
        });

        Thread thread6 = new Thread(() -> {
            System.out.println("TERMINATED");
        });

        thread2.start();
        lockThread.start();
        thread3.start();
        thread4.start();
        thread5.start();
        thread6.start();
        TimeUnit.MILLISECONDS.sleep(10);

        System.out.println("thread1 state: " + thread1.getState());
        System.out.println("thread2 state: " + thread2.getState());
        System.out.println("thread3 state: " + thread3.getState());
        System.out.println("thread4 state: " + thread4.getState());
        System.out.println("thread5 state: " + thread5.getState());
        System.out.println("thread6 state: " + thread6.getState());
        thread4.interrupt();
        thread5.interrupt();
    }
}
```

其输出为: 

```shell
...
thread1 state: NEW
thread2 state: RUNNABLE
thread3 state: BLOCKED
thread4 state: WAITING
thread5 state: TIMED_WAITING
thread6 state: TERMINATED
...
```
        
## 参考资料

1. *The Java Language Specifications Java SE 10 edition*
2. *Java concurrency in practice*
3. *Thinking in Java 4th edition*