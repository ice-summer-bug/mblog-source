---
layout: post
title: JVM 基础知识
categories: Java
tags: [JVM, Java]
date: 2017-09-12 21:00:00
description: JVM 脚本工具参数说明和使用示例
---

# 如何标记对象是否死亡

1. 引用计数法

  缺点：无法处理循环引用

2. 根可达法

JVM 使用的就是根可达法


# 垃圾回收算法

| 垃圾回收算法 | 优点 | 缺点 | 适用范围 |
|-|-|-|-|
| 标记清除法 | - | 1.造成大量内存碎片; 2. 标记和清除的过程效率都很低 | 适用于老年代 |
| 标记整理法 | 1. 没有内存碎片; 2. 不会浪费内存空间 | 1. 效率更加低下 | 适用于存活率高的老年代 |
| 标记复制法 | 1. 效率提升，无需多次整理碎片 | 1. 浪费空间 | 适用于对象存活率低的新生代 |
| 分代回收法 | 1. 根据对象存活特点划分区域，适配最合适的回收算法 | - | JVM 内存空间 |

# JVM 内存结构

## jdk8 之前的 hotspot JVM 内存结构

- 程序计数器 ———— 程序计数器一块较小的内存空间，它可以看作是  ***当前线程所执行的字节码的行号指示器***。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能会通过更高级的方式去实现），字节解释器的工作就是改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器去完成。为了线程切换后能恢复到正确的执行位置，***每条线程都需要有一个独立的程序计数器，独立存储。*** 这个区域不会出现 `OutOfMemoryError`；如果执行的是 native 方法，程序计数器的数值为 undefined。
- 栈
    - 虚拟机栈 ———— 虚拟机栈也是线程私有的，它的生命周期和线程相同。虚拟机栈描述的是 Java 方法执行的内存模型：每个方法被执行的时候都会同时创建一个 `栈帧(Stack Frame)` 用于存储 `局部变量表`、`操作数栈`、`动态链接`、`方法出口` 等消息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。虚拟机栈可能会抛出 `OutOfMemoryError` 异常和 `StackOverflowError` 异常。
        - 局部变量表 ———— 存放编译期可知的各种基本类型数据（boolean、byte、char、short、int、long、double）、对象引用和 returnAddress 类型，其中 64 位长度的 long 和 double 类型的数据会占用 2 个局部变量空间。
    - 本地方法栈 ———— 本地方法栈和虚拟机栈的区别是，本地方法栈是为虚拟机使用到的 native 方法服务的，虚拟机规范对本地方法栈中方法使用的语言、使用方式没有强制规定，虚拟机可以自行实现；本地方法栈也可能会抛出 `OutOfMemoryError` 异常和 `StackOverflowError` 异常。
- 堆 ———— JVM 管理的内存中最大的一块，由所有线程共享，在虚拟机启动的时候创建；堆唯一的目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。为了便于垃圾回收，堆使用分代收集算法，分为新生代和老年代；
    - 新生代 ———— 新生代进一步细分为 `Eden` 和 `Survivor`，`HotSpot 虚拟机`的实现中，`Survivor` 分为 `Survivor From` 和 `Survivor To`，三者的比例是 `Eden` : `Survivor From` : `Survivor To` 是 8:1:1；当 Survivor 空间不足的时候需要依赖老年代进行分配担保（也就是说大对象直接分配到老年代）。新生代使用标记复制收集法，对象实例先分配在 `Eden`，存活之后，将实例复制到其中一块 `Survivor` 区域，再将 `Eden` 区域进行清空。
    - 老年代 ———— 老年用于存放超过 `Eden` 可分配空间的对象；存放新生代 `Survivor` 中内存不足时，老年代空间担保的对象；存放超过一定年龄的对象
- 永生代（方法区）———— 线程共享的内存区域，存储已经被虚拟机加载的类信息、常量、静态变量、即时编辑器编译后的代码等数据


## jdk8 之后的 JVM 内存结构

- 程序计数器、虚拟机栈、本地方法栈、堆和 jdk8 之前的 JVM 内存结构是一致的，只有方法区变成了 `元空间`

## 元空间和永久代的区别

1. 元空间使用的本地内存空间，可以动态扩充
2. java8 把静态常量放到到 java heap
3. java8 把符号引用放到了 native heap
4. java8 把字面量转移到了 java heap
5. 元空间和堆不相邻，full gc 的时候不会收集元空间


## 什么时候进入老年代
1. 对象初始大小超过新生代中 eden 可分配空间
2. 对象年龄超过一定程度，这个阈值可能是 `-XX:MaxTenuringThreslod` 参数设置的（默认值15），也可能是 survivor 中 50% 以上对象的年龄
3. 新生代 Survivor 中内存不足时，老年代进行空间分配担保，对象进入老年代

## 什么时候发生 老年代 GC
1. 老年代剩余可分配空间不足

## Young GC 的时候会扫描老年代吗？ Major GC 的时候会扫描新生代吗？

1. 老年代回收的时候会将新生代中的对象作为 GC ROOT，同理新生代回收的时候，也需要考虑对象是否被老年代中的对象引用
2. JVM 为了避免每次YOUNG GC 的时候都去扫描老年代，维护了一个 `CardTable`, 其中将引用了新生代对象的老年代对象的地址置为 `dirty`， YOUNG GC 时候只需要扫描置为 `dirty` 的老年代区域即可
3. 老年代GC 时候也需要扫描对象是否被新生代的对象引用

# 垃圾回收器对比

| 垃圾回收期 | 回收算法 | 特点  | 优点| 缺点 | 适用范围 |
|-|-|-|-|-|-|
| Serial | 标记复制 | 1. 单线程 2. 全程 STW | 1.单线程收集效率最高 | 1. 单线程 <br/>2. STW | 新生代 |
| ParNew | 标记复制 | Serial 的多线程版 | 1. 相比 Serial 来说，实现了多线程回收 <br/>2.可以和 CMS 配合使用 | 1. STW | 新生代 |
| Parallel Scavenge | 标记复制 | 1. 多线程; <br/>2.达到一个`可控`的`吞吐量` <sub>注1</sub>  <br/> 3. 自适应调节策略 | - | 1. STW 2.不能和CMS配合使用 | 新生代 |
| Serial Old | 标记整理 | 老年代 Serial，使用标记整理算法 | 1.单线程收集效率最高 | 1. 单线程 <br/>2. STW | 老年代 |
| Parallel Old | 标记整理 | 老年代的 Parallel Scavenge，使用标记整理算法 | 1. 配合 Parallel Scavenge 实现高吞吐量有限  | 1. STW | 老年代 |
| CMS | 标记清除 | 1. 专注于缩短STW 时间 | 1.多线程 2. STW 停顿时间短 | 1. 初始标记和重复标记的时候还是会 STW <br> 2. 产生大量碎片 <br/> 3. 并行清理的时候用户线程产生的垃圾称为浮动垃圾，只能在下一次回收中被清除 <br/> 4. 和用户线程抢CPU 资源  | 老年代 |
| G1 | 标记整理 | 将整个JAVA堆分为一定大小的独立区域（Region），将区域标识位新生代Eden区域、新生代 Survivor区域、老年代区域，跟踪这些区域的垃圾堆积程度，优先收集垃圾堆积程度高的区域 | 1.没有碎片 <br/> 2.精确控制停顿 3. 避免全区的垃圾回收，保证低停顿的同时不影响吞吐量 | 1. 单线程 <br/>2. STW | JVM 堆 |

***注1 吞吐量 = 运行用户代码的时间 / (运行用户代码的时间 + 垃圾收集时间)***


# CMS
![](/assets/picture/jvm.cms.processing.flow.png "CMS 垃圾收集过程")

1. 初始标记(STW) —— 从新生代出发标记可达的老年代对象，从GC ROOTS 出发标记老年代对象
2. 并发标记 —— 从 “初始标记” 标记的老年代出发，标记存活的老年代对象，将所有存活对象所在 Card 标记为 Dirty 状态;  后续只需要扫描这些对象，而不用扫描整个老年代 注意，这里不能标记出所有存活的数据，因为还有新晋升到老年代、或者直接在老年代分配的对象、或者更新老年代引用关系等等，需要重新标记
3. 预处理 —— 标记新分配的新生代对象可达的老年代对象，标记老年代内引用发生变化的card 为 dirty
4. 可中断的预处理 —— 循环处理标记新生代对象可达的老年代对象以及老年代内引用发生变化的对象，可以控制循环次数，循环时间上限为 5s，新生代eden 区使用率低于阈值（50%） 时循环退出；祈祷循环结束前进行一个 minor GC，也可以通过 `CMSScavengeBeforeRemark` 参数强制指定重新标记之前记性一次 minor GC
5. 并发重新标记(STW) —— 并发重新标记，标记新生代可达的老年代对象，标记GC ROOTS 可达的老年代对象，重新标记老年代的 Dirty Card 
6. 并发清除