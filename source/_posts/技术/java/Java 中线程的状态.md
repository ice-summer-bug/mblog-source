---
layout: post
title: Java 中线程的状态
categories: Java
tags: [Java, Thread]
date: 2020-06-26 21:00:00
description: Java 中线程的状态
---

# Java 中的线程的状态

## 线程有哪些状态

在 JDK 源码中，我们可以看到 `java.lang.Thread.State` 定义了 `Thread` 的五种状态

```java
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```

- NEW
    线程刚刚被启动，还没有被执行的状态
- RUNNABLE
    线程达到了可执行状态，也就是说线程可以被 JVM 执行，但是还需要操作系统分配执行资源，例如处理器
- BLOCKED
    线程处于阻塞状态，等待获取到一个内置锁(`monitor lock` or `intrinsic lock`); 等待获取一个内置锁进入 `sychronized` 修饰的同步代码块或者同步方法；或者在当前线程调用了 `Object#wait` 方法之后等待重入同步代码块或者同步方法
- WAITING
    线程处于等待状态，在下面这些方法被调用之后，线程在这个状态下，只有等待另外一个线程进行指定操作之后才会被唤醒
    - `Object#wait()` 方法，没有 timeout 参数
    - `Thread#join()` 方法，没有 timeout 参数
    - `LockSupport#park()`  方法，没有 timeout 参数
- TIMED_WAITING
    线程处于指定了等待时长的等待状态，在下面这些方法被调用之后，线程在这个状态下，只有等待另外一个线程进行指定操作之后才会被唤醒
    - `Thread#sleep` 方法，指定了 timeout 时长
    - `Thread#join(long)` 方法，指定了 timeout 时长
    - `Object#wait(long)` 方法，指定了 timeout 时长
    - `LockSupport#parkNanos` 方法，指定了 timeout 时长
    - `LockSupport#parkUntil` 方法，指定了 timeout 时长
-  TERMINATED
    线程达到终态，正常执行结束，或者执行过程中遇到了异常而结束了

***除了上述状态之外，线程还有个状态，就是 `RUNNING(运行中)`***

## 线程的状态是如何流转的？

状态流转情况如图

![线程状态流转图](/assets/picture/thread.state.flow.png "线程状态流转图")

## 线程的状态的流转是怎么触发的？

### Object 中控制线程状态流转相关的方法

#### Object 中让线程进入等待状态(WAIING, TIME_WAITING)的方法
- java.lang.Object#wait(): 让当前线程一直等待
- java.lang.Object#wait(long timeout): 让当前线程等待指定时长，单位毫秒
- java.lang.Object#wait(long timeout, int nanos): 让当前线程等待指定时长，timeout 单位毫秒, nonas 为补充字段，设置等待的纳秒数

#### Object 中让线程退出等待状态的方法

- java.lang.Object#notify(): 唤醒等待当前对象的内置锁的一个线程
- java.lang.Object#notifyAll(): 唤醒所有等待状态的线程

### Thread 中控制线程状态流转相关的方法

#### Thread#join

- java.lang.Thread#join(): 让线程进入无限等待状态，等待线程死亡
- java.lang.Thread#join(long millis): 让线程进入无限等待状态，等待线程死亡，指定等待时间
- java.lang.Thread#join(long millis, int nanos): 让线程进入无限等待状态，等待线程死亡，指定等待时间, timeout 单位毫秒, nonas 为补充字段，设置等待的纳秒数

在 `Thread#join` 的核心实现中，还是通过调用 `Object#wait` 方法使得线程进入等待状态（`TIME_WAITING`, `WAITING`），等待线程死亡
```java
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) { // 不断调用 Object#wait 方法
                wait(0);  
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

#### `Thread#sleep`

`Thread#sleep` 方法会使当前正在执行的线程以指定的毫秒数暂停（暂时停止执行），具体取决于系统定时器和调度程序的精度和准确性。
但是线程会继续持有内置锁

- java.lang.Thread#sleep(long): 让线程休眠指定时长，单位毫秒
- java.lang.Thread#sleep(long, int): 让线程休眠指定时间，, timeout 单位毫秒, nonas 为补充字段，设置休眠的纳秒数

#### `Thread#start`

线程开始执行的入口方法，JVM 会去调度线程的 `run()` 方法

#### `Thread#run`

调用 Runnable 接口中的 `run()` 方法，Thread 的子类必须覆写这个方法

#### `Thread#yield`

告知调度器，当前线程愿意放弃处理器资源，让线程从 `RUNNING` 状态流转到 `RUNNABLE` 状态，调度器也可以忽略这个通知

#### 线程的中断 

线程中断相关的方法有三个

|方法名称|方法描述|
|-|-|
| interrupt | 将线程标记为中断 |
| isInterrupt | 测试线程是否标记为中断的 |
| interrupted | 将线程的中断标记清空 |