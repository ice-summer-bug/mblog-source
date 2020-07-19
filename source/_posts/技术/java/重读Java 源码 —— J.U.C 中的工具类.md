---
layout: post
title: J.U.C 包中的工具类
categories: Java
tags: [Java, J.U.C]
date: 2020-06-26 21:00:00
description: 重读 JDK 源码，回顾 J.U.C 包中的工具类
---

# J.U.C 包中的工具类

## atomic 包中的支持原子操作的类

在 Java 中 i++ 这种数字的自增操作不是原子性操作， i ++ 本质上是三个操作，获得当前变量的值，对当前值进行加一，再写回新的值；在没有额外资源可以使用的情况下，只有操作才能保证 `读-改-写` 操作的原子性，为了解决这种问题，j.u.c 提供了 atomic 包，其中包含了多种类型的具有原子性的工具类

### 从 AtomicInteger 开始看 Atomic 工具类

首先我们可以看看 `AtomicInteger`，其他的 Atomic 工具类的实现也和 `AtomicInteger` 类似，首先看看 `AtomicInteger` 的方法和属性

![](/assets/picture/atomic.integer.method.list.jpg "AtomicInteger 的方法及属性")

这里首先看看和 i ++ 功能一致的方法 `incrementAndGet()`，

```java
public class AtomicInteger extends Number implements java.io.Serializable {

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

}
```

从代码可以看到，这里主要是依赖 `Unsafe` 这个底层工具类，从下面的代码可以看出，这里通过 `Unsafe` 提供的 `CAS` 实现了原子性操作

```java
  public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
      var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
  }
```

其他方法也和 `incrementAndGet()` 方法类似，都是依赖 `Unsafe` 提供的 `native` 的 `CAS` 方法实现的原子性操作；

除了 `AtomicInteger` 之外， `j.u.c` 还提供了 `AtomicIntegerArray`、`AtomicLong`、`AtomicLongArrray`、`AtomicBoolean`、`AtomicReference`、`AtomicReferenceArray`，可以对 int\[\]、long、long\[\]、boolean、T、T\[\] 等类型的数据进行原子性修改， 主要方法有：

- 数字类型相关的方法:
    - getAndIncrement
    - getAndDecrement
    - incrementAndGet
    - decrementAndGet
    - addAndGet
    - getAndAdd
- 原子性工具类通用方法
    - accumulateAndGet: 通过 accumulatorFunction自定义加和逻辑，对当前数据和参数进行加和，先加和再返回值
    - compareAndSet
    - getAndAccumulate: 通过 accumulatorFunction自定义加和逻辑，对当前数据和参数进行加和，返回加和前的值
    - getAndSet
    - getAndUpdate
    - updateAndGet
    - weakCompareAndSet

### Atomic 工具类的 ABA 问题及如何解决

## Lock

### ReentrantLock

### ReentrantReadWriteLock

### Condition

### LockSupport

## CountDownLatch

## CyclicBarrier

## CompletionService

## ThreadLocalRandom

## TimeUnit