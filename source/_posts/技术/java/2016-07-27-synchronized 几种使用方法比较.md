---
layout: post
title: synchronized 关键字
categories: Java
tags: [JAVA 并发编程, Java]
date: 2016-07-27 21:00:00
description: java 并发基础，synchronized 关键字使用说明
---

## synchronized 几种使用方法比较

`synchronized` 关键字在 Java 中出现的很早， Java 提供的实现原子性的内置锁机制，就是用这个关键字实现。
`synchronized` 关键字可以修饰方法，也可以修饰一个代码块，实现一种互斥锁，这里就不做赘述。


下面讨论几个问题

#### 1. synchronized 关键字修饰 非静态方法， 锁定的是对象实例还是类？

```java
public synchronized void syncInnerReferenceLock1() {
    for (int i = 0; i < COUNT; i++) {
        System.out.println("syncInnerReferenceLock1: [count] -- " + i);
    }
}

public synchronized void syncInnerReferenceLock2() {
    for (int i = 0; i < COUNT; i++) {
        System.out.println("syncInnerReferenceLock2: [count] -- " + i);
    }
}

public static void main(String[] args) {
    final SyncLockDemo syncLockDemo1 = new SyncLockDemo();

    Thread t1 = new Thread(new Runnable() {
        public void run() {
            syncLockDemo1.syncInnerReferenceLock1();
        }
    });

    Thread t2 = new Thread(new Runnable() {
        public void run() {
            syncLockDemo1.syncInnerReferenceLock2();
        }
    });

    t1.start();
    t2.start();
}
```

上面的代码中有两个用 `synchronized`  关键字修饰的非静态方法， 在两个线程中运行这两个方法， 结果如下


        syncInnerReferenceLock2: [count] -- 0
        syncInnerReferenceLock2: [count] -- 1
        syncInnerReferenceLock2: [count] -- 2
        syncInnerReferenceLock2: [count] -- 3
        syncInnerReferenceLock2: [count] -- 4
        syncInnerReferenceLock1: [count] -- 0
        syncInnerReferenceLock1: [count] -- 1
        syncInnerReferenceLock1: [count] -- 2
        syncInnerReferenceLock1: [count] -- 3
        syncInnerReferenceLock1: [count] -- 4

        Process finished with exit code 0

    或

        syncInnerReferenceLock1: [count] -- 0
        syncInnerReferenceLock1: [count] -- 1
        syncInnerReferenceLock1: [count] -- 2
        syncInnerReferenceLock1: [count] -- 3
        syncInnerReferenceLock1: [count] -- 4
        syncInnerReferenceLock2: [count] -- 0
        syncInnerReferenceLock2: [count] -- 1
        syncInnerReferenceLock2: [count] -- 2
        syncInnerReferenceLock2: [count] -- 3
        syncInnerReferenceLock2: [count] -- 4

        Process finished with exit code 0

从结果可以看出，两个线程是串行执行的，我们可以猜测： 修饰非静态方法的 `synchronized` 关键字锁定的是这个类的对象实例，下面我们来验证这个想法：

```java
public synchronized void syncInnerReferenceLock1() {
  for (int i = 0; i < COUNT; i++) {
      System.out.println("syncInnerReferenceLock1: [count] -- " + i);
  }
}

public synchronized void syncInnerReferenceLock2() {
  for (int i = 0; i < COUNT; i++) {
      System.out.println("syncInnerReferenceLock2: [count] -- " + i);
  }
}

public static void main(String[] args) {
  final SyncLockDemo syncLockDemo1 = new SyncLockDemo();
  final SyncLockDemo syncLockDemo2 = new SyncLockDemo();

  Thread t1 = new Thread(new Runnable() {
      public void run() {
          syncLockDemo1.syncInnerReferenceLock1();
      }
  });

  Thread t2 = new Thread(new Runnable() {
      public void run() {
          syncLockDemo2.syncInnerReferenceLock2();
      }
  });

  t1.start();
  t2.start();
}
```

我们创建两个对象实例， 在两个线程中分别执行两个被 `synchronized` 关键字修饰的方法， 结果如下：

        syncInnerReferenceLock2: [count] -- 0
        syncInnerReferenceLock1: [count] -- 0
        syncInnerReferenceLock2: [count] -- 1
        syncInnerReferenceLock1: [count] -- 1
        syncInnerReferenceLock1: [count] -- 2
        syncInnerReferenceLock1: [count] -- 3
        syncInnerReferenceLock1: [count] -- 4
        syncInnerReferenceLock2: [count] -- 2
        syncInnerReferenceLock2: [count] -- 3
        syncInnerReferenceLock2: [count] -- 4

        Process finished with exit code 0

这里可以看到，连个线程是并行执行的，没有丝毫互斥的现象，从而论证了上面的猜测！


#### 2.  `synchronized` 关键字修饰 `静态方法`， 锁定的是对象实例还是类？
看了第一个问题，我们能不能说， `synchronized` 关键字修饰 `静态方法`， 锁定的是类呢？ 让我们看看

```java
public static synchronized void syncLock1() {
    for (int i = 0; i < COUNT; i++) {
        System.out.println("syncLock1: [count] -- " + i);
    }
}

public static synchronized void syncLock2() {
    for (int i = 0; i < COUNT; i++) {
        System.out.println("syncLock2: [count] -- " + i);
    }
}

public static void main(String[] args) {

    Thread t1 = new Thread(new Runnable() {
        public void run() {
            syncLock1();
        }
    });

    Thread t2 = new Thread(new Runnable() {
        public void run() {
            syncLock2();
        }
    });

    t1.start();
    t2.start();
}
```

这里把 `1` 中的两个方法修改成静态方法， 在两个线程中分别执行两个方法，结果如下：

        syncLock1: [count] -- 0
        syncLock1: [count] -- 1
        syncLock1: [count] -- 2
        syncLock1: [count] -- 3
        syncLock1: [count] -- 4
        syncLock2: [count] -- 0
        syncLock2: [count] -- 1
        syncLock2: [count] -- 2
        syncLock2: [count] -- 3
        syncLock2: [count] -- 4

        Process finished with exit code 0

    或

        syncLock2: [count] -- 0
        syncLock2: [count] -- 1
        syncLock2: [count] -- 2
        syncLock2: [count] -- 3
        syncLock2: [count] -- 4
        syncLock1: [count] -- 0
        syncLock1: [count] -- 1
        syncLock1: [count] -- 2
        syncLock1: [count] -- 3
        syncLock1: [count] -- 4

        Process finished with exit code 0

可以从结果中看出，两个线程是串行执行，存在明显的互斥，`synchronized` 关键字修饰 `静态方法`， 锁定的确实是类

#### 3. synchronized(reference) {} 和 synchronized(Clazz.class) {}
synchorinizable 修饰类和对象的区别是什么呢？

在我看来， synchorinizable 修饰类和对象，可以抽象为 `类锁` 和 `对象锁`

下面上代码

这是一个 `类锁` 的范例
```java
public static void syncClazzLock () {
    synchronized (SyncLockDemo.class) {
        for (int i = 0; i < COUNT; i++) {
            System.out.println("syncClazzLock: [count] -- " + i);
        }
    }
}
```
这是个 `对象锁`
```java
public void syncReferenceLock() {
    synchronized (this) {
        for (int i = 0; i < COUNT; i++) {
            System.out.println("syncReferenceLock: [count] -- " + i);
        }
    }
}
```

调用上面的锁

```java
public static void main(String[] args) {
     final SyncLockDemo syncLockDemo = new SyncLockDemo();

     Thread t1 = new Thread(new Runnable() {
         public void run() {
             syncClazzLock();
         }
     });

     Thread t3 = new Thread(new Runnable() {
         public void run() {
             syncLockDemo.syncReferenceLock();
         }
     });

     t1.start();
     t3.start();
 }
```

程序运行结果

    syncReferenceLock: [count] -- 0
    syncClazzLock: [count] -- 0
    syncReferenceLock: [count] -- 1
    syncClazzLock: [count] -- 1
    syncReferenceLock: [count] -- 2
    syncClazzLock: [count] -- 2
    syncReferenceLock: [count] -- 3
    syncReferenceLock: [count] -- 4
    syncClazzLock: [count] -- 3
    syncClazzLock: [count] -- 4

    Process finished with exit code 0

从结果可以看出， 这两个线程的运行顺序是并行的，并不互斥。 因为 `syncClazzLock` 中 `synchronized` 关键字修饰的是类， 这个  `类锁` 作用于类的范围， 作用于类的静态方法、类的class 对象、类的静态代码块；而 `syncReferenceLock` 方法中 `synchronized` 关键字修饰的是对象， 这个 `对象锁` 最用于对象的范围，作用的对象示例的方法或一个对象实例上
