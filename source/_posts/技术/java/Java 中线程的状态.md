---
layout: post
title: Java 中线程的状态
categories: Java
tags: [Java, Thread]
date: 2019-03-01 21:00:00
description: Java 中线程的状态
---

# Java 中的线程的状态

## 线程有哪些状态

在 JDK 源码中，我们可以看到 `java.lang.Thread.State` 定义了 `Thread` 的五种状态

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
    - `Object#wait(long)` 方法，指定了 timeout 时长
    - `Thread#join(long)` 方法，指定了 timeout 时长
    - `LockSupport#parkNanos` 方法，指定了 timeout 时长
    - `LockSupport#parkUntil` 方法，指定了 timeout 时长
-  TERMINATED
    线程达到终态，正常执行结束，或者执行过程中遇到了异常而结束了

## 线程的状态是如何流转的？



## 线程的状态的流转是怎么触发的？
