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

                除了上述状态之外，线程还有个状态，就是 RUNNING(运行中)，但是 JDK 并没有提供这样一个状态，主要是因为 RUNNING 状态是个瞬时状态；JRE 很难快捷的区分出 RUNNING 和 RUNNABLE 状态

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

##### Thread#interrupt 方法

我们先看看方法上的说明文档：

```txt
Interrupts this thread.

Unless the current thread is interrupting itself, which is always permitted, the checkAccess method of this
thread is invoked, which may cause a SecurityException to be thrown.

If this thread is blocked in an invocation of the wait(), wait(long), or wait(long, int) methods of the 
Object class, or of the join(), join(long), join(long, int), sleep(long), or sleep(long, int), methods of
this class, then its interrupt status will be cleared and it will receive an InterruptedException.

If this thread is blocked in an I/O operation upon an InterruptibleChannel then the channel will be closed,
the thread's interrupt status will be set, and the thread will receive a
java.nio.channels.ClosedByInterruptException.

If this thread is blocked in a java.nio.channels.Selector then the thread's interrupt status will be set
and it will return immediately from the selection operation, possibly with a non-zero value, just as if 
the selector's wakeup method were invoked.

If none of the previous conditions hold then this thread's interrupt status will be set.

Interrupting a thread that is not alive need not have any effect.
------------------------------------------------------------------------------------------------------
中断这个线程。

当前线程中断自身的时候，始终被允许，否则 checkAccess 方法会被调用，可能会抛出 SecurityException 异常；   

1) 当前线程在被调动 Thread#sleep、Thread#join 或者 Object#wait 方法后处于阻塞状态，此时中断这个线程的时候，
它的中断状态会被清除，线程会接收到一个 InterruptedException；

2) 如果该线程阻塞在一个操作可中断的通道 InterruptibleChannel 的 I/O 操作上，这个通道将被关闭，该线程的中断
状态也会被标记，该线程也会接收到一个 java.nio.channels.ClosedByInterruptException 异常；

3）如果线程在 java.nio.channels.Selector 中被阻塞，线程的中断状态被标记，并且理解从选择操作中返回，可能是一个
非 0 的数值，就像 selector 的 wakeup 方法被调用

4）如果该线程不满足上述所有条件，仅标记线程的中断标识，不对线程造成其他影响

5）中断一个不存在的线程，将不会产生任何效果
```

###### 中断一个正在运行的普通线程

```java
public class InterruptDemo {

  public static void main(String[] args) throws InterruptedException {
    Thread thread1 = new Thread(new RunningThread());
    thread1.start();

    TimeUnit.MILLISECONDS.sleep(10);
    thread1.interrupt();
    System.out.println("RunningThread.isInterrupted: " + thread1.isInterrupted());
  }

  private static class RunningThread implements Runnable {

    @Override
    public void run() {
      while (true) {
      }
    }
  }
}
```

运行结果：

                RunningThread.isInterrupted: true

这里我们可以看到，主程序没有退出，线程一直在运行，只有线程的中断标识被标记了

###### 中断一个处于等待状态(`TIME_WAITNG`, `WAITING`)的线程

- 中断调用了 `Thread#sleep` 方法的线程

```java

public class InterruptDemo {

  public static void main(String[] args) throws InterruptedException {
    Thread thread2 = new Thread(new SleepingThread());
    thread2.start();

    TimeUnit.MILLISECONDS.sleep(10);
    thread2.interrupt();
    System.out.println("SleepingThread.isInterrupted: " + thread2.isInterrupted());
  }

  private static class SleepingThread implements Runnable {

    @Override
    public void run() {
      while (true) {
        try {
          TimeUnit.SECONDS.sleep(30);
        } catch (InterruptedException e) {
          System.out.println("do sth for InterruptedException...");
          e.printStackTrace();
        }
      }
    }
  }
}
```

运行结果

                SleepingThread.isInterrupted: false
                do sth for InterruptedException...
                java.lang.InterruptedException: sleep interrupted
                    at java.lang.Thread.sleep(Native Method)
                    at java.lang.Thread.sleep(Thread.java:340)
                    at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
                    at com.liam.learn.interruptdemo.InterruptDemo$SleepingThread.run(InterruptDemo.java:40)
                    at java.lang.Thread.run(Thread.java:748)

从结果可以看出，这个一直调用 `Thread#sleep` 方法的线程调用了 `interrupt` 方法之后，接收到了 `InterruptedException` 异常，但是线程的中断标志位并没有被表示为已中断，但是我们可以在线程中捕获 `InterruptedException` 并根据这个信息作出应对操作。

- 中断了调用了 `Object#wait` 方法的线程

```java
public class InterruptDemo {

  public static void main(String[] args) throws InterruptedException {
    Object lock = new Object();
    Thread thread3 = new Thread(new WaitingThread(lock));
    thread3.start();

    TimeUnit.MILLISECONDS.sleep(10);
    thread3.interrupt();
    System.out.println("WaitingThread.isInterrupted: " + thread3.isInterrupted());

  }


  private static class WaitingThread implements Runnable {

    private Object lock;

    public WaitingThread(Object lock) {
      this.lock = lock;
    }

    @Override
    public void run() {
      synchronized (lock) {
        try {
          lock.wait();
        } catch (InterruptedException e) {
          System.out.println("do sth for InterruptedException...");
          e.printStackTrace();
        }
      }
    }
  }

}
```

运行结果是：

                do sth for InterruptedException...
                WaitingThread.isInterrupted: false
                java.lang.InterruptedException
                    at java.lang.Object.wait(Native Method)
                    at java.lang.Object.wait(Object.java:502)
                    at com.liam.learn.interruptdemo.InterruptDemo$WaitingThread.run(InterruptDemo.java:70)
                    at java.lang.Thread.run(Thread.java:748)

                Process finished with exit code 0


- 中断调用了 `Thread#join` 方法的线程

```java
public class InterruptDemo {

  public static void main(String[] args) throws InterruptedException {
    Thread thread4 = new Thread(new JoiningThread());
    thread4.start();
    TimeUnit.MILLISECONDS.sleep(10);
    thread4.interrupt();
    System.out.println("JoiningThread.isInterrupted: " + thread4.isInterrupted());

  }

  private static class JoiningThread implements Runnable {

    @Override
    public void run() {
      try {
        Thread.currentThread().join();
      } catch (InterruptedException e) {
        System.out.println("do sth for InterruptedException...");
        e.printStackTrace();
      }
    }
  }

}
```

运行结果是

                do sth for InterruptedException...
                JoiningThread.isInterrupted: false
                java.lang.InterruptedException
                    at java.lang.Object.wait(Native Method)
                    at java.lang.Thread.join(Thread.java:1252)
                    at java.lang.Thread.join(Thread.java:1326)
                    at com.liam.learn.interruptdemo.InterruptDemo$JoiningThread.run(InterruptDemo.java:91)
                    at java.lang.Thread.run(Thread.java:748)

调用 `Object#wait`、`Thread#join` 方法的线程和调用了 `Thread#sleep` 方法的线程被调用了 `interrupt` 方法的结果是一样的，都不会被标记为中断，都会接收到 `InterruptedException`

##### 中断机制的实现

`Thread#interrupt` 方法的实现如下，通过调用一个 native 方法去修改线程的中断标识，将线程标记位中断。

```java
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }

    private native void interrupt0();
```

而 `interrupt0` 的实现我们可以在 [openjdk](https://github.com/openjdk/jdk "OpenJDK") 源码中找到

首先是 `Thread.c` 中定义了 native 方法
```c
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};
```

而 `JVM_Interrupt` 方法的是实现是在 `jvm.cpp` 中

```cpp
JVM_ENTRY(void, JVM_Interrupt(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_Interrupt");

  ThreadsListHandle tlh(thread);
  JavaThread* receiver = NULL;
  bool is_alive = tlh.cv_internal_thread_to_JavaThread(jthread, &receiver, NULL);
  if (is_alive) {
    // jthread refers to a live JavaThread.
    receiver->interrupt();
  }
JVM_END
```

而 `JavaThread::interrupt` 的实现是 

```cpp
// interrupt support
void JavaThread::interrupt() {
  debug_only(check_for_dangling_thread_pointer(this);)

  // For Windows _interrupt_event
  osthread()->set_interrupted(true);

  // For Thread.sleep
  _SleepEvent->unpark();

  // For JSR166 LockSupport.park
  parker()->unpark();

  // For ObjectMonitor and JvmtiRawMonitor
  _ParkEvent->unpark();
}
```

这里 JVM 将线程的中断标识标记为中断

## 关于线程状态的一些常见文件

1. `sleep` 和 `wait` 方法的区别是什么？

        Thread#sleep 只会让线程进入休眠状态，但是不会放弃已经获得的内置锁；
        而 Object#wait 方法不仅会让线程进入等待状态，还会让线程释放内置锁

