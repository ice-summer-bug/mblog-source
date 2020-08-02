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

`AtomicInteger` 等工具类有一个问题，就是当 `AtomicInteger` 的值从 1 改为2 再从2 改为1 的时候，两次读取 `AtomicInteger` 的值都是 1 的情况下，无法确定 `AtomicInteger` 的值在两次查询之间是否被修改过，解决这个问题的思路也很简单，再加一个版本属性每次操作的时候，版本递增；或者新增一个时间戳字段，每次修改记录时间戳，JDK 采用的是方法二，提供了工具类 `AtomicStampedReference<V>`，它将需要被原子操作的属性 `reference` 和 int 类型的时间戳封装成 `Pair`，再对 `Pair` 进行 CAS 操作保证原子性

### Atomic 工具类的自旋性能问题

AtomicInteger 等工具类利用 `Unsafe` 提供的 `CAS` 操作实现了在不加锁的情况下保证原子性操作，但是部分操作再单次 `CAS` 操作选择使用 while 循环实现自旋直至成功，如 `sun.misc.Unsafe#getAndAddInt` 

```java
  public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
      var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
  }
```

这个方法在高并发的情况下，多个线程同时执行 `CAS` 的且多次失败的情况下，每个线程都在自旋占用 CPU 资源；`Doug Lea` 大神也发现了这个问题，所以在 `JDK1.8` 中新增了 `LongAdder`、`LongAccumulator`、`DoubleAdder`、`DoubleAccumulator` 工具类

#### LongAdder 等工具类分段操作降低竞争热度

这里对 `LongAdder` 的实现进行一个简单的分析，其他工具类和它的实现思路是基本一样的；

![](/assets/picture/LongAdder.method.list.jpg "LongAdder 的方法列表")

`LongAdder` 提供的方法列表中仅提供了单个操作接口，没有提供 `AtomicInteger` 等工具类中的 `getAndIncrement()` 或者 `incrementAndGet()` 这种复合的原子操作接口，通过 `increment()` 和 `longValue()` 组合实现，但是不具有原子性，这也是 `LongAdder` 区别于 `AtomicInteger` 等工具类的不足；

下面来看看 `LongAdder` 中的主要方法

```java
public class LongAdder extends Striped64 implements Serializable {

    /**
     * Equivalent to {@code add(1)}.
     */
    public void increment() {
        add(1L);
    }

    /**
     * Equivalent to {@code add(-1)}.
     */
    public void decrement() {
        add(-1L);
    }
}
```

这里可以看到，`increment()` 和 `decrement()` 方法的试下都是基于 `add(long x)` 方法，下面来看看 `add(long x)` 方法：

```java
    /**
     * Adds the given value.
     *
     * @param x the value to add
     */
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
```

`add` 方法首先会尝试调用 `casBase` 方法，`casBase` 方法是父类 `Striped64` 中的方法

```java
    /**
     * CASes the base field.
     */
    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }
```

它也是使用 `Unsafe` 中的 `CAS` 方法，实现原子性操作，但是这里只会尝试一次 `CAS` 操作，失败之后就会进入下一步；在下一步操作中，首先是将变量 `uncontended` 变量置为 ture，从名字就可以看出来这个变量是标识没有竞争，下面的逻辑就是判断是否存在多线程竞争了；

下面的判断条件都是或逻辑

- as == null ?  ——— 如果当前的 cells 为 null，则进入下一步 `longAccumulate` 方法
- (m = as.length - 1) < 0 ？———— 当前的 cells 是否为空，为空时直接进入下一步 `longAccumulate` 方法
- a = as[getProbe() & m]) == null || !(uncontended = a.cas(v = a.value, v + x) 
这一步稍微负责一点，首先通过 `getProbe()` 方法获取当前线程的 hash 标识，`getProbe()` 方法是从 `ThreadLocalRandom` 中复制来的获取线程的hash 标识的方法，通过 `getProbe() & m` 来为当前线程获取 Cell，如果获取成功则对这个 Cell 段进行 CAS 操作；再根据CAS 操作成功则表示当前不存在多线程竞争，直接退出，否则则进入下一步 `longAccumulate`方法；如果当前线程的 hash 标识映射到的 Cell 段为空，也会进入下一步 `longAccumulate` 方法；

上面的逻辑中出现了属性 `cells`，以及 `casBase` 方法中操作的属性 `base`， 这两个属性都来自 `LongAdder` 的父类 `Striped64`，`Striped64` 中定义了 `Cell` 段并管理了一组 `Cell` 段的数组和一个简单的 long 值；

```java
abstract class Striped64 extends Number {
  @sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
      try {
          UNSAFE = sun.misc.Unsafe.getUnsafe();
          Class<?> ak = Cell.class;
          valueOffset = UNSAFE.objectFieldOffset
              (ak.getDeclaredField("value"));
      } catch (Exception e) {
          throw new Error(e);
      }
    }
  }

  /**
  * Table of cells. When non-null, size is a power of 2.
  */
  transient volatile Cell[] cells;

  /**
  * Base value, used mainly when there is no contention, but also as
  * a fallback during table initialization races. Updated via CAS.
  */
  transient volatile long base;
}
```

从上面的代码可以看出 `Cell` 可以视为删减版的 `AtomicInteger`，它维护了个 long 类型的值，并提供了基于 `Unsafe` 的 `cas` 方法；`Striped64` 维护了一组 `Cell` 数组和一个 long 值，没有多线程竞争的情况下对 long 值进行修改，出现了多线程竞争的情况下，为线程匹配一个 `Cell` 段，对 `Cell` 段维护的数值进行修改，最后获取值的时候在对 `base` 和 `Cell[]` 中的数值进行求和；

```java
  public long sum() {
      Cell[] as = cells; Cell a;
      long sum = base;
      if (as != null) {
          for (int i = 0; i < as.length; ++i) {
              if ((a = as[i]) != null)
                  sum += a.value;
          }
      }
      return sum;
  }
```

从上面的逻辑可以简单分析出，`LongAdder` 的思路就是线程竞争不充分的时候，和 AtomicInteger 类似对单个值进行 CAS 操作的竞争，不同点在于不会自旋等待；在多线程情况下从多个线程自旋竞争同一个字段的 `CAS` 操作，优化为多个线程自旋竞争多个字段的 `CAS` 操作，而线程和 `Cell` 端的映射通过线程的 hash 标识进行再次hash 映射；而最后需要获取只的时候，需要对所有 `Cell` 中的数值以及 `base` 进行加和即可；唯一的不足是占用的内存空间更多了一点，但是在多线程高并发的情况下能提高效率，这点空间的占用也不算什么了；而且在线程竞争不激烈的情况下还是只使用了单个数值；

这里可以简单总结一下 ***低并发的时候 `AtomicInteger` 等工具类的性能和 `LongAdder` 的性能差不多；高并发的情况下，`LongAdder` 明显更高效***

下面还是会继续分析一下 `longAccumulate` 方法的逻辑

```java
  final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
      ThreadLocalRandom.current(); // force initialization
      h = getProbe();
      wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
      Cell[] as; Cell a; int n; long v;
      if ((as = cells) != null && (n = as.length) > 0) {
        if ((a = as[(n - 1) & h]) == null) {
          if (cellsBusy == 0) {       // Try to attach new Cell
            Cell r = new Cell(x);   // Optimistically create
            if (cellsBusy == 0 && casCellsBusy()) {
              boolean created = false;
              try {               // Recheck under lock
                Cell[] rs; int m, j;
                if ((rs = cells) != null &&
                  (m = rs.length) > 0 &&
                  rs[j = (m - 1) & h] == null) {
                  rs[j] = r;
                  created = true;
                }
              } finally {
                cellsBusy = 0;
              }
              if (created)
                break;
              continue;           // Slot is now non-empty
            }
          }
          collide = false;
        }
        else if (!wasUncontended)       // CAS already known to fail
          wasUncontended = true;      // Continue after rehash
        else if (a.cas(v = a.value, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
          break;
        else if (n >= NCPU || cells != as)
          collide = false;            // At max size or stale
        else if (!collide)
          collide = true;
        else if (cellsBusy == 0 && casCellsBusy()) {
          try {
            if (cells == as) {      // Expand table unless stale
              Cell[] rs = new Cell[n << 1];
              for (int i = 0; i < n; ++i)
                  rs[i] = as[i];
              cells = rs;
            }
          } finally {
              cellsBusy = 0;
          }
          collide = false;
          continue;                   // Retry with expanded table
        }
        h = advanceProbe(h);
      }
      else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
        boolean init = false;
        try {                           // Initialize table
          if (cells == as) {
            Cell[] rs = new Cell[2];
            rs[h & 1] = new Cell(x);
            cells = rs;
            init = true;
          }
        } finally {
          cellsBusy = 0;
        }
        if (init)
          break;
      }
      else if (casBase(v = base, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
        break;                          // Fall back on using base
    }
  }
```

这个方法比较长，逐步拆分看看

- 首先如果当前线程的 hash 标识没有初始化则进行初始化，并且将 wasUncontented 置为 true，表示当前没有竞争
- 接下来是个自旋死循环，保证 CAS 更新成功
- 循环中首先如果 `cells` 不为空 
  - 如果线程映射到的 `Cell` 为 null 且 `cellsBusy` 标识位标识 `cells` 不是在调整大小或者新建 `Cell` 的时候，尝试创建一个新的 `Cell`，加入到数据中
  - 如果线程映射到的 `Cell` 不为 null，则进行一次 `CAS` 尝试，成功了则返回
  - 如果 `cellsBusy` 标识了 `cells` 没有被操作，而且 `cells` 满了之后，对 `cells` 进行扩容，每次扩大两倍，然后进入下一次循环
- 如果 `cells` 为空而且成功将 `cellsBusy` 标识位置为 true 的时候，对 cells 初始化，长度为2，并且为当前现在映射的下标位置填充一个 `Cell`

上边的流程分析中始终多次出现了 `cellsBusy`，这个字段用于标识 `cells` 数组是否在调整大小或者被创建中，并且提供了 `CAS` 方法对这个标志位进行修改

        总结一下，需要进行原子性数据统计的时候，推荐使用 LongAdder 等代替 AtomicLong, 《阿里巴巴 Java 开发手册》 中也推荐使用 LongAdder

![](/assets/picture/alibaba.java.guide.suggestion.LongAdder.jpg "《阿里巴巴开发规范》推荐使用 LongAdder")

## CountDownLatch(锁存器)

`CountDownLatch` 工具类的作用，从注解中就能看出来

        A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.
        一个同步辅助工具类，允许一个或多个线程等待其他线程中一组操作完成后再进行其他操作。
        
下面看看方法列表

| 方法签名 | 返回值 | 方法说明 |
|-|-|-|
| CountDownLatch(int) | CountDownLatch | 构造方法，构造一个锁存器，并指定计数值 |
| countDown() | void | 对锁存器的计数值进行递减，线程安全，如果计数值减到了 0 则释放所有等待的线程 | 
| await() throws InterruptedException | void | 阻塞当前线程进入等待状态，直至锁存器的计数值达到零，除非当前线程被中断 | 
| await(long timeout, TimeUnit unit) throws InterruptedException | boolean, 锁存器减为0 时返回 true，如果超过指定时间计数值没有减为0 返回 false | 阻塞当前线程进入等待状态，直至锁存器的计数值达到零，除非线程被中断或者超过了指定的等待时间 | 
| getCount() | long | 返回锁存器的计数值 | 

从上面的方法列表可以看出，`CountDownLatch(锁存器)` 维护了一个计数值，可以让当前线程阻塞等待，直到计数值减为 0，这个数值也就是需要等待的线程的数量；通过这个同步工具类可以做到让一个或一组线程等待其他线程完成之后在进行在进行下一步的操作

```java
public class CountDownLatchDemo {

  public static void main(String[] args) throws InterruptedException {
    int threadCount = 5;
    CountDownLatch waitSignal = new CountDownLatch(threadCount);
    for (int i = 0; i < threadCount; i++) {
      new Thread(new WaitedTask("waited-thread-" + i, waitSignal)).start();
    }
    waitSignal.await();
    System.out.println("主线程等待其他线程执行完毕后, 执行后续动作");
  }


  private static class WaitedTask implements Runnable {

    private String name;
    private CountDownLatch countDownLatch;

    WaitedTask(String name, CountDownLatch countDownLatch) {
      this.name = name;
      this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
      try {
        System.out.println("线程 " + name + " 开始执行操作");
        int timeVal = ThreadLocalRandom.current().nextInt(5);
        TimeUnit.SECONDS.sleep(timeVal);
        System.out.println("线程 " + name + " 操作完成, 达到等待点");
        countDownLatch.countDown();
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }
}
```

        线程 waited-thread-0 开始执行操作
        线程 waited-thread-2 开始执行操作
        线程 waited-thread-1 开始执行操作
        线程 waited-thread-3 开始执行操作
        线程 waited-thread-4 开始执行操作
        线程 waited-thread-0 操作完成, 达到等待点
        线程 waited-thread-4 操作完成, 达到等待点
        线程 waited-thread-3 操作完成, 达到等待点
        线程 waited-thread-2 操作完成, 达到等待点
        线程 waited-thread-1 操作完成, 达到等待点
        主线程等待其他线程执行完毕后, 执行后续动作

        Process finished with exit code 0

在上面这个简单的示例中我们能看到正常情况下，`CountDownLatch` 可以做到让主线程等待一组线程完成一系列操作，但是这里也会有一个问题，假设这一组线程中有一个线程执行过程中出现了异常，程序还能继续执行吗？

```java
  private static class WaitedTask implements Runnable {

    private String name;
    private CountDownLatch countDownLatch;

    WaitedTask(String name, CountDownLatch countDownLatch) {
      this.name = name;
      this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
      try {
        System.out.println("线程 " + name + " 开始执行操作");
        int timeVal = ThreadLocalRandom.current().nextInt(5);
        TimeUnit.SECONDS.sleep(timeVal);
        if (name.contains("3")) {
          throw new IllegalStateException("一个用于测试的异常...");
        }
        System.out.println("线程 " + name + " 操作完成, 达到等待点");
        countDownLatch.countDown();
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }
```

这里简单模拟一个抛出异常的情况，再看看执行结果~

        线程 waited-thread-0 开始执行操作
        线程 waited-thread-2 开始执行操作
        线程 waited-thread-3 开始执行操作
        线程 waited-thread-1 开始执行操作
        线程 waited-thread-4 开始执行操作
        线程 waited-thread-4 操作完成, 达到等待点
        线程 waited-thread-0 操作完成, 达到等待点
        线程 waited-thread-1 操作完成, 达到等待点
        Exception in thread "Thread-3" 线程 waited-thread-2 操作完成, 达到等待点
        java.lang.IllegalStateException: 一个用于测试的异常...
          at com.liam.learn.CountDownLatchDemo$WaitedTask.run(CountDownLatchDemo.java:43)
          at java.lang.Thread.run(Thread.java:748)

结果很遗憾，一个线程抛出异常之后，主线程陷入了阻塞等待的状态，没办法继续执行后续操作~ 因为 `CountDownLatch(锁存器)` 的计数值始终不能减为 0，`CountDownLatch#await()` 方法一直处于一个阻塞状态；

## CyclicBarrier

## CompletionService

## ThreadLocalRandom

## TimeUnit

## Lock

### ReentrantLock

### ReentrantReadWriteLock

### Condition

### LockSupport



