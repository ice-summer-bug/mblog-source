---
layout: post
title: Java 线程池
categories: Java
tags: [Java, Thread]
date: 2020-06-26 21:00:00
description: 重读 JDK 源码，回顾线程池
---

# Java 线程池

## 线程池存在的意义

在机器的 CPU 核数越来越多的发展趋势下，为了更好的利用机器资源，我们就会想到创建更多的线程，理想状态下对于每个任务都去创建一个线程；但是 CPU 的数量也是有限的，创建了过多的线程会导致多个线程去竞争 CPU 资源，导致线程上下文的频繁切换，浪费系统资源；而且每个线程都有自己的内存空间，系统创建了过多的线程也会浪费系统资源；线程的创建和销毁也会消耗系统资源；

总而言之，线程是很稀有的资源，通过线程池我们可以减少线程的创建和销毁所消耗的时间和系统资源的开销，也避免了为大量线程分配内存，以及线程上下文的 `过度切换`；
将线程这种稀有资源池化使用也是开发人员的惯用方法，同理还有 `数据库连接池`，可以在接受一个任务之后，直接在资源池中获取一个线程，用完之后再归还。

线程池的好处主要是

- 线程资源的复用，降低资源消耗
- 提高响应速度，直接使用空闲线程，不用每个请求都要等待创建线程
- 提高线程的可管理性，对线程实现统一分配和实时监控

## `Executor` 的重要实现 ———— `ThreadPoolExecutor`

在 `J.U.C` 工具包中，线程池的实现主要是 `ThreadPoolExecutor`，下面个将对线程池进行更详细的介绍

### `ThreadPoolExecutor` 的参数

`ThreadPoolExecutor` 的构造方法的参数如下

| 参数名称 | 参数类型 | 参数说明 | 
|-|-|-|
| `corePoolSize` | int | 线程池核心线程数量，如果没有设置 `allowCoreThreadTimeOut` 参数，即使空间也会保留的线程数量 |
| `maximumPoolSize` | int | 线程池允许的最大线程数 |
| `keepAliveTime` | long | 空闲线程存活时间，空闲线程在终止之前等待新任务的最长时间；超出核心线程数的非核心线程会阻塞等待一段时间去获取任务等待队列中的任务，也就是说任务等待队列中的任务的等待时长最多为这个时间 |
| `unit` | TimeUnit | 空闲线程存活时间的单位 |
| `workQueue` | BlockingQueue | 用于缓存任务的阻塞等待队列 |
| `threadFactory` | ThreadFactory | 创建线程的工厂类 |
| `handler` | RejectedExecutionHandler | 线程池无法接受新任务时，拒绝新任务的策略，简称饱和策略 |
| `allowCoreThreadTimeOut` | boolean | 是否允许核心线程超过空闲存活时间后被销毁 |

***注意：`allowCoreThreadTimeOut` 参数不是 `ThreadPoolExecutor` 构造方法的参数，默认为 `false`***

#### `J.U.C` 提供的 `RejectExecutionHandler` 的实现

| 名称 | 说明 |
|-|-|
| AbortPolicy |	抛出一个 RejectedExecutionException 异常，中断任务提交 |
| DiscardPolicy | 忽略这个被拒绝的任务，什么都不做 |
| DiscardOldestPolicy | 将任务等待队列中最老的任务丢弃掉 |
| CallerRunsPolicy | 直接在主线程中调用被拒绝放入线程池的任务 |

### `ThreadPoolExecutor` 的状态

线程池的状态信息如下：

| 状态名称 | 状态特征描述 | 
|-|-|
| RUNNING(运行中) | 接受新任务而且处理缓存队列中的任务 |
| SHUTDOWN(待关闭) | 不接受新任务，但是处理缓存队列中的任务， |
| STOP(停止) | 不接受新任务，现有缓存队列中的任务不会处理，正在运行的任务会被中断 |
| TIDYING(整理) | 所有任务都被终止了，并且准备去调用 terminal() 钩子方法 |
| TERMINATED(终止) | terminal() 方法执行完毕，进入终态 |

状态流转流程如下

![ThreadPoolExecutor 状态流转图](/assets/picture/thread.pool.executor.state.flow.png "ThreadPoolExecutor 状态流转图")

#### `shutdown()` 方法

在这个方案中，我们会将线程池状态置为 SHUTDOWN，中断闲置线程，但是还会继续处理

```java
  public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 确认线程可访问
        checkShutdownAccess();
        //将状态设置为 SHUTOWN
        advanceRunState(SHUTDOWN);
        // 中断线程池中的空闲的线程
        interruptIdleWorkers();
        // 调用钩子方法
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
  }

  private void interruptIdleWorkers() {
      interruptIdleWorkers(false);
  }

  private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // 确认线程没有中断，并且处于闲置状态之后再中断
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

#### `shutdownNow()` 方法

`shutdownNow()` 方法中，除了将线程池状态置为 STOP 的之外，还会主动去中断工作线程，并且将任务阻塞等待队列清空

```java
  public List<Runnable> shutdownNow() {
      List<Runnable> tasks;
      final ReentrantLock mainLock = this.mainLock;
      mainLock.lock();
      try {
          // 确认线程可访问
          checkShutdownAccess();
          //将状态设置为 STOP
          advanceRunState(STOP);
          // 中断线程池中的所有工作线程，无论是否在运行
          interruptWorkers();
          // 将任务阻塞等待队列清空
          tasks = drainQueue();
      } finally {
          mainLock.unlock();
      }
      // 尝试终止线程池，因为当前线程池状态是 STOP，
      tryTerminate();
      return tasks;
  }

  private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock; 
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
  }

  private final class Worker
      extends AbstractQueuedSynchronizer
      implements Runnable
  {
    void interruptIfStarted() {
        Thread t;
        // 中断所有可以中断的线程
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
  }

```

### `ThreadPoolExecutor` 的任务提交流程

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

从上面代码的注释可以了解到，任务提交流程的逻辑如下：

1. 当线程池中线程数量小于核心线程数量时，直接创建创建新的线程
2. 当线程池中线程数量大于核心线程数量时，如果线程缓存存队列没有满，则将任务加入缓存队列
3. 当线程池中线程数量大于核心线程数量时，如果线程缓存队列满了，而且线程数量小于线程池最大容量的时候直接新建线程
4. 当线程池中线程数量大于核心线程数量时，如果线程缓存队列满了，而且线程数量大于线程池最大容量的时候，调用拒绝处理器，拒绝新增任务


具体的流程可以看下图：

![ThreadPoolExecutor 任务提交流程图](/assets/picture/thread.pool.executor.execute.flow.svg "ThreadPoolExecutor 任务提交流程图")

上面这个图只是线程池任务提交流程的概要图，具体的为任务新建线程的逻辑在 `addWoker` 方法中

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 线程状态为 SHUTDOWN，提交非空任务时， 直接退出
            // 线程状态为 SHUTDOWN，并且等待队列为空，此时提交空任务的时候直接退出
            // 线程状态为 STOP、TIDYING、TERMINATED 状态时，直接退出
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

        private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```

将上面的代码分析一下就得到了下图

![ThreadPoolExecutor 任务提交详细流程图](/assets/picture/thread.pool.executor.state.detail.flow.svg "ThreadPoolExecutor 任务提交详细流程图")

### `ThreadPoolExecutor` 是如何收缩的？

在任务执行过程中，线程池会根据状态去选择性的收缩大小，如下代码，
任务的线程在运行的时候，执行完了自身任务之后，会从线程池中取出等待的任务进行执行；在从任务等待队列中获取任务的过程中会根据情况对线程池进行收缩

```java

    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        public void run() {
            runWorker(this);
        }
    }

    final void runWorker(Worker w) {
      Thread wt = Thread.currentThread();
      Runnable task = w.firstTask;
      w.firstTask = null;
      w.unlock(); // allow interrupts
      boolean completedAbruptly = true;
      try {
          // 当前需要执行的任务执行完毕之后，会从任务阻塞等待队列中提取待运行的任务并执行
          while (task != null || (task = getTask()) != null) {
            // 省略
            try {
              // 省略
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
          }
          completedAbruptly = false;
      } finally {
          processWorkerExit(w, completedAbruptly);
      }
    }


    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 当线程池状态为 SHUTDOWN，并且等待队列为空时，线程池将被清空
            // 当线程池状态为 STOP 或者 TIDYING 或者 TERMINATED 时，忽略等待队列中的任务，线程池将被清空
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            /**
            * 1. 线程状态为 RUNNING
            *     1) 线程数量超过了线程池最大容量，线程池容量减一
            *     2) 线程数量超过了核心线程数，且上一循环中获取线程超时了，线程池容量减一
            *     3) 线程数量小于核心线程数且大于1，但是核心线程允许闲置超时销毁时，线程池容量减一
            *     4)线程数量小于等于1，但是核心线程允许闲置超时销毁时，且上一循环中获取线程超时了，且任务缓存队列为空，线程池容量减一
            * 2. 线程状态为 SHUTDOWN, 但是任务缓存队列不为空
            *     1) 线程数量超过了线程池最大容量，线程池容量减一
            *     2) 线程数量超过了核心线程数，且上一循环中获取线程超时了，线程池容量减一
            *     3) 线程数量小于核心线程数且大于1，但是核心线程允许闲置超时销毁时，线程池容量减一
            **/
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

```

从上面的代码可以看出，线程池的线程在执行完当前任务之后会去从任务阻塞等待队列中提取待运行的任务，这个时候线程池会在不同的状态下，根据情况进行收缩

- 线程池状态为RUNNIING
  - 线程数量超过了线程池最大容量，线程池容量减一
  - 线程数量超过了核心线程数，且上一循环中获取线程超时了（等待队列可能为空），线程池容量减一
  - 线程数量小于核心线程数且大于1，且上一循环中获取线程超时了（等待队列可能为空），但是核心线程允许闲置超时销毁时，线程池容量减一
  - 线程数量小于核心线程数且大于1，但是核心线程不允许闲置超时销毁时，线程池会阻塞在从任务等待队列中提起任务的操作上，除非被中断
  - 线程数量小于等于1，但是核心线程允许闲置超时销毁时，且上一循环中获取线程超时了（等待队列可能为空），且任务等待队列为空，线程池容量减一
- 线程池状态为 SHUTDOWN，并且任务缓存队列为空的时候，CAS 自旋将线程池大小降到 0 为止
- 线程池状态为 SHUTDOWN，且任务缓存队列不为空
  - 线程数量超过了线程池最大容量，线程池容量减一
  - 线程数量超过了核心线程数，且上一循环中获取线程超时了（等待队列可能为空），线程池容量减一
  - 线程数量小于核心线程数且大于1，但是核心线程允许闲置超时销毁时，线程池容量减一
- 线程池状态为 STOP 或者 TIDYING 或者 TERMINATED 时，CAS 自旋将线程池大小降到 0 为止

***注意：上面说到的超时，就是超过了线程池的闲置超时时间***

        总结一下：
        1. 线程池在运行中(RUNNING)状态时，
        1) 当线程数超过线程池最大容量时，线程池容量减一，确保线程池容量限制
        2) 当线程数超过了核心线程数，而且暂时没有等待运行的任务，线程池容量减一，让线程数逐步
        回归核心线程数；
        3) 当线程数小于核心线程数且大于1，而线程池允许核心线程闲置超时销毁，而且暂时没有等待
        运行的任务，线程池容量也会减一，让线程数逐步减少到 1；
        4) 当线程数小于核心线程数且大于1，但是运行中的线程池如果核心线程不允许闲置超时销毁，
        线程池会阻塞在从等待队列中提取待运行的任务，除非被中断；
        5) 当这个运行中的线程池中的线程数减少到 1 的时候，如果核心线程允许闲置超时销毁，而且
        暂时没有等待运行的任务，且当前任务等待队列为空，线程池容量减一，线程池为空了
        6) 当这个运行中的线程池中的线程数减少到 1 的时候，但是运行中的线程池如果核心线程不允
        许闲置超时销毁，线程池会阻塞在从等待队列中提取待运行的任务，除非被中断；
        2. 线程池在待关闭(SHUTDOWN)状态时，
        1) 如果任务等待队列为空，线程池将被清空；如果任务等待队列不为空，还是处理等待任务，当
        线程数超过线程池最大容量时，线程池容量减一，保证线程数不超过最大容量；
        2) 当线程数超过了核心线程数，而且暂时没有等待运行的任务，线程池容量减一，让线程数逐步
        回归核心线程数；
        3) 当线程数小于核心线程数且大于 1 的时候，如果核心线程允许闲置超时销毁，而且暂时没有
        等待运行的任务，线程池容量减一，让线程数逐步减少到 1；
        4) 当线程数减少到 1 时，因为线程池处于待关闭状态，且任务等待队列不为空，这个线程会被
        保留用于处理等待运行的任务
        3. 线程池在停止(STOP)、整理(TIDYING)、终止(TERMINATED) 状态下，不会接受新任务，
        也不会处理等待队列中的任务，直接将线程池容量逐步清空


说了这么多我们直降到了线程池的数量被减少了，但是 ***闲置的线程是怎么被清除的呢？***

让我们再回看一下 `getTask()` 方法中在将线程池的容量收缩之后，马上返回了一个 `null`
![](/assets/picture/thred.pool.get.task.reuturn.null.after.decrement.jpg "getTask() 线程池容量收缩后返回 null")

获取到的待运行任务为 null 时，就要开始清除这个线程了
![](/assets/picture/thread.pool.executor.process.worker.exit.jpg "待运行任务为 null 时处理退出线程")

下面就是 `processWorkerExit` 方法的逻辑了

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w); // 将线程从线程池中删除
        } finally {
            mainLock.unlock();
        }

        // 尝试终止线程池
        tryTerminate();

        // 线程池状态时 RUNNING 或者 SHUTDOWN 且线程运行过程中没有抛出异常时
        // 当线程池不允许核心线程闲置超时被销毁时，确保线程池有核心线程数的线程
        // 当线程池允许核心线程闲置超时被销毁时，确保线程池至少有一个线程
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

          总结：
          当等待队列中暂时没有待执行任务时，将线程池数量收缩之后，这里会将当前工作线程从线程池中清除掉
          当线程池状态是 SHUTDOWN 且任务等待队列为空时，这里还会尝试去终止线程池
          当线程池状态是 RUNNING 或者状态是 SHUTDOWN 且任务队列不为空是，我们需要有一定的线程去处理任务；
          这个时候当线程池允许核心线程闲置超时被销毁时，如果线程池为空，向线程池新增一个不带任务的线程；
          当线程池不允许核心线程闲置超时被销毁时，如果线程数量小于核心线程数，也向线程池新增一个不带任务的线程；

          总之是当线程池还在运行中，或者待关闭需要处理等待队列中的待运行任务时，保证线程池有一定的线程去处理
          这些任务

***注意：线程池中的闲置线程在达到超时时间之后不一定会立刻被清除掉，而是在超时时间之后的某个时刻被清除掉***

### `ThreadPoolExecutor` 执行任务过程中的扩展操作

`ThreadPoolExecutor` 线程池在执行任务的过程中，支持进行一些扩展的插入操作

- `terminated()`: 线程池终止前的插入操作

在上面的状态流转说明中，线程池从 `TIDYING` 流转到 `TERMINATED` 状态之前，就是去调用 `terminated()` 方法，下面的代码就是具体逻辑

```java
    final void tryTerminate() {
        for (;;) {
            // 省略
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated(); // 扩展插入操作
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

- `beforeExecute()` 和 `afterExecute()`: 任务执行前后的插入操作

```java
    final void runWorker(Worker w) {
        // 省略
        try {
            while (task != null || (task = getTask()) != null) {
                // 省略
                try {
                    beforeExecute(wt, task); // 执行任务之前的插入操作
                    Throwable thrown = null;
                    try {
                        task.run();
                    } finally {
                        afterExecute(task, thrown); // 执行任务之后的插入操作
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```



### 通过 `Executors` 创建 `ThreadPoolExecutor`

#### CachedThreadPool

```java
  public static ExecutorService newCachedThreadPool() {
          return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                        60L, TimeUnit.SECONDS,
                                        new SynchronousQueue<Runnable>());
  }
```

无界线程池，它的特性是

- corePoolSize 是 0
- workQueue 任务等待队列是 `SynchronousQueue`，不存储任何元素，放入元素的时候直接返回 false
- maximumPoolSize 是 INTEGER.MAX_VALUE

上述特点意味着，向线程池提交任务之后，即超过了核心线程数，可以加入任务等待队列，但是 `SynchronousQueue` 不支持存储任何元素，线程池容量向 maximumPoolSize 发展，也就是 Integer.MAX_VALUE，线程池开始无限新建新线程

#### FixedThreadPool

固定大小的线程池的特性是

- 任务阻塞等待队列的长度是无界的

这意味着，线程池中线程数量达到核心线程数后，新提交的任务会进入无界等待队列(长度为 `Integer.MAX_VALUE`)，这意味 `线程池最大容量` 参数失去了意义，线程池中的线程数量不会超过核心线程数；

- 闲置线程的存活时间是 0

这表示，运行中的线程从任务等待队列中提取任务时，不会阻塞等待，

用户指定线程池和核心线程数和线程池最大容量，而且核心线程数和线程池容量一致，任务等待阻塞队列的长度是无界的，这意味着当线程数量超过核心线程数之后，新提交的任务被放入任务阻塞等待队列

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

***注意：阿里巴巴 Java 开发规范要求我们不要使用 `Executors` 去创建上述两种 `ThreadPoolExecutor`***

![](/assets/picture/why.not.use.executors.jpg "Why can not use Executors to create ThreadPoolExecutor")


#### SingleThreadExecutor

```java
  public static ExecutorService newSingleThreadExecutor() {
          return new FinalizableDelegatedExecutorService
              (new ThreadPoolExecutor(1, 1,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>()));
  }
```

单线程线程池，它的特点是

- 核心线程数和线程池容量都是 1
- 任务等待队列是无界的阻塞队列

这意味着 ———— 向线程池提交任务之后，会创建唯一的线程，然后任务被放入阻塞等待队列，然后被唯一的线程一个个的处理

### 关于线程池参数

#### 线程池大小

关于线程池的参数，所有所有资料都在告诉我们，线程池中的任务分为 IO 密集型和 CPU 密集型；在 《Java 并发编程实战》一书的 8.2 章节中我们可以学习到下面的结论：

CPU 密集型的任务特点是任务有大量计算，消耗 CPU 资源，CPU 占用率很高；执行这种任务的时候线程数过多，会发生很多线程抢占CPU 的情况，线程的上下文切换也比较浪费资源；所以对于 CPU 密集型的任务，我们需要的是 N<sub>cpu</sub> + 1 个线程，这里比 CPU 个数多一个的原因是，***即使当计算密集型的线程偶尔由于页缺失故障或者其他原因而暂停时，这个额外的线程也能确保CPU的时钟周期不会被浪费。***

而对于其他任务，CPU 计算操作和 IO 操作是混合在一起的，对于这种CPU 使用率不会特别高的任务，线程池的容量可以大一点，对于这种任务多很多文章都说线程池容量可以设置成 2N，这个值没有什么准确的依据，只是个经验预估值，没什么参考价值；针对 IO密集型的任务，我们需要关注的是 CPU 计算时间和 CPU 等待时间的比例，以及 CPU  的目标使用率

---------------------------------------------------
线程池容量 = N<sub>cpu</sub> * U<sub>cpu</sub> * (1 + W/C)
N<sub>cpu</sub>: CPU 个数
U<sub>cpu</sub>: 目标 CPU 使用率
W: CPU 等待时间
C: CPU 计算时间

---------------------------------------------------

举例说明，现在有个任务，CPU 计算的时间是 10ms， IO 操作的时间是 90ms，理想情况下我们以 CPU 利用率 100% 为假定目标，这个时候对于一个 8 核的机器：
线程池容量 = 8 * 1 * (1 + 90/10) = 80；所以这里我们需要 80 个线程

***线程池的大小是根据目标 CPU 使用率计算出来的，所以这就是我们的能设置的最大的线程池容量了***

#### 任务等待队列

##### 任务等待队列实现的选择

任务等待队列的类型是阻塞队列，下面先介绍一下阻塞队列，再说明如何选择任务等待队列的实现

##### 阻塞队列的特性

先看看 `BlockingQueue` 中的方法
![](/assets/picture/blocking.queue.method.list.jpg "阻塞队列的方法列表")

下面是具体一点的方法列表

| 方法签名 | 方法介绍 | 返回值 | 是否抛出异常 | 附加说明 |
|-|-|-|-|-|
| boolean add(E e) | 向队列中添加元素 | 添加成功则返回 true | 队列容量不足时，抛出 IllegalStateException | 使用有界队列的时候，推荐使用 offer(E) 方法 |
| boolean offer(E e) | 向队列中添加元素 | 添加成功返回true， 否则返回 false | - | 使用有界队列的时候，优先于 add(E) 方法被使用 |
| boolean offer(E e, long timeout, TimeUnit) | 向队列中添加元素 | 添加成功返回true， 否则返回 false，超时也返回 false | 被中断时抛出 InterruptedException | 使用有界队列的时候，优先于 add(E) 方法被使用 |
| void put(E e) | 向队列中添加元素 | 无 | 被中断的时候抛出 InterruptedException | 一直阻塞，直至成功或失败 |
| E remove() | 删除队列头第一个元素 | 删除成功时，返回被删除的元素 | 队列头为空时抛出 NoSuchElementException | - |
| boolean remove(E e) | 删除指定元素 | 删除成功时返回 true，反正返回 false | - | - |
| E poll() | 删除并返回队列头元素或者null | 返回队列头元素，队列为空时返回 null | - | - |
| E poll(long timeout, TimeUnit unit) | 删除并返回队列头元素或者null | 返回队列头元素，队列为空或者超时的时候返回 null | 被中断时抛出 InterruptedException | 没有命中元素时，阻塞直至超时 |
| E take() | 删除并返回队列头元素或者null | 返回队列头元素 | 被中断时抛出 InterruptedException | 一直阻塞，直至成功或失败 |
| E element() | 返回队列头元素，但是不删除 | 返回队列头元素 | 没有命中任何元素的时候抛出 NoSuchElementException | - |
| E peek() | 返回队列头元素或者 null，但是不删除 | 返回队列头元素或者null | - | - |

阻塞队列的特性就会提供了上述方法，分类以下几类

- 一直阻塞等待的方法，可以被中断，put(E e) 新增元素，take() 获取并删除头元素
- 阻塞超时的方法，可以被中断，offer(E e, long timeout, TimeUnit) 新增元素， poll(long timeout, TimeUnit unit) 获取并删除头元素
- 抛出异常，add(E e) 新增元素，remove() 删除元素，element() 获取头元素
- 返回特征值，offer(E e) 新增失败返回false；poll() 没有命中时返回 null；peek() 没有命中时返回 null

|方法\处理方式|抛出异常|返回特殊值|一直阻塞(可中断)|超时退出(可中断)|
|-|-|-|-|-|
|插入元素|add(e)|offer(e)|put(e)|offer(e, time, unit)|
|返回队列头元素，并从队列中移除|remove()|poll()|take()|poll(time,unit)|
|返回队列头元素，但不删除|element()|peek()|不可用|不可用|

方法的选用取决于我们是否需要立即同步获取操作结果，可以等待一定时长，还是可以一直等待

###### 阻塞队列的实现

先看看类图

![](/assets/picture/blocking.queue.implements.class.diagram.png "阻塞队列的主要实现的类图")


主要实现的简要说明：

| 类名 | 特性 | 优点 | 缺点 |
|-|-|-|-|
| LinkedBlockingQueue | 由单向链表实现的先进先出的阻塞队列 | 可阻塞，入出队列锁分离，效率高，支持容量限制，可以作为无界队列的实现 | 由于存在锁机制，同时链表需要遍历才能定位一个元素，因此效率有一定的影响 |
| ArrayBlockingQueue | 由数组实现的先进先出的阻塞队列 | 可阻塞，支持容量限制，遍历元素的效率更快，可以作为有界队列的实现 | 容量固定，不支持扩容，出入队列不能同时进行 |
| SynchronousQueue | 每一个插入操作都必须等待另一个线程的移除操作，反之亦然；这意味同步队列没有容量，或者说容量为 1，元素被生产者放入同步队列之后，期待消费者移除处理之后生产者才能继续放入元素 | 可阻塞，快速交换元素 | 内部没有容量 |
| PriorityBlockingQueue | 按照自然排序实现的阻塞队列，在元素需要排序的情况下是唯一的选择 | 可阻塞，元素有序，支持扩容 | 出入队列比较慢，效率比较低，基于数组的实现每次扩容都需要复制数组，同时容量不能减小，入队列不能被阻塞 |
| DelayQueue | 基于 PriorityBlockingQueue 实现的延时处理队列，每个元素都有一个延时时间，当且仅当延时时间过期才能出队列 | 可阻塞，可延时 | 效率低，入队列不能被阻塞 |

###### 阻塞队列的使用场景

从阻塞队列的特性来看，
`LinkedBlockingQueue` 是无界阻塞队列的首选，但是线程池不推荐使用无界队列；
为了更好的控制线程池，需要一个有界阻塞队列，`ArrayBlockingQueue` 是实现首选；
`PriorityBlockingQueue` 支持队列中元素排序，对于需要处理不同优先级的任务的线程池，最优选择是优先级阻塞队列，最好设置上长度；
`DelayQueue` 主要作用是元素需要延时被延时处理，只有线程池需要处理的任务需要等待一定时间才能处理时，才会选择这种阻塞队列的实现；

***结论：一般情况下，使用有界阻塞队列 `ArrayBlockingQueue` 作为线程池的任务等待队列***


##### 任务等待队列的长度和闲置线程存活时间

为了线程池的可控，我们给线程池的任务等待队列设置一个可接受的大小，否则像 CachedThreadPool 一样设置一个无界阻塞队列，很容易就会因为队列过程而 OOM；考虑等待队列长、闲置线程存活时间和线程池容量这三个参数的时候，我们需要综合考虑，因为线程数超过核心线程数之后再提交的任务，会先先尝试放入任务等待队列，队列空间不足就会去创建非核心线程，非核心线程从阻塞队列中提取任务的等待时长就是 `闲置线程的存活时间`，超过这个时间非核心线程就可能会被回收，也就是说 ***任务等待队列的核心功能就是将核心线程暂时无法忙碌无法处理的任务缓存起来；核心线程忙碌的情况下，如果等待队列没有空的情况下，队列中的任务都需要等待线程来处理*** ；

任务等待队列的长度，首先需要保证能够缓存所有的任务，其次缓存在队列中的任务的等待时长必须在业务可接受范围内；
- 在核心线程忙碌，任务进入等待队列的阶段，任务处理的最大时长是 

----------------------------------------------------
T<sub>task</sub> = ceil(workQueue.length / coorPoolSize) * T<sub>process</sub> + T<sub>process</sub>
T<sub>task</sub>: 核心线程忙碌时，任务处理总时长
workQueue.length: 任务等待队列的长度
coorPoolSize: 核心线程数
T<sub>process</sub>: 任务实际处理时长
ceil(): 向上取整的函数

----------------------------------------------------

- 在任务等待度列满了之后，创建了非核心线程之后，任务处理的最大时长是

----------------------------------------------------
T<sub>task</sub> = ceil(workQueue.length / poolSize) * keepAliveTime + T<sub>process</sub>
T<sub>task</sub>: 核心线程忙碌时，任务处理总时长
workQueue.length: 任务等待队列的长度
poolSize: 线程池中工作线程数
keepAliveTime: 闲置线程存活时间
T<sub>process</sub>: 任务实际处理时长
ceil(): 向上取整的函数

----------------------------------------------------

