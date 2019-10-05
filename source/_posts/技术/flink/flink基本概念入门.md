---
layout: post
title: Flink 基本概念入门
categories: flink
tags: flink
date: 2018-11-19 22:00:00
description: apache flink 基本概念介绍
---

# Flink 基本概念

## 基本名词

### `Job`

以 `jar` 包的形式在 `flink` 中提交的可运行程序

### `task`

### `stream`

`flink` 作为一个流数据处理的引擎，就是针对一个或多个 `stream` 进行流计算处理，再输出到一个或多个 `stream` 中去，这里的 `stream` 可以使 mq，也可以是文件、也可以直接是控制台输入\输出。

### `operator` && `task`

`flink` 流处理流程中的每个操作(如 `map`, `keyBy`, `sink`, `source`等)都是 `operator`

### `operator subtask`

每个 `operator` 可以分成多个 `operator subtask`，一个 `operator` 的并行度就是 `operator subtask` 的数量

### `operator chain`

`flink` 作为分布式运行系统，会将多个 `operator subtask` 关联成一个 `operator task`，这个过程就是 `operator chain`。

两个 `operator subtask` 能否关联起来，需要满足下列要求

1. 上下游的并行度一致
2. 下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）
3. 上下游节点都在同一个 slot group 中（下面会解释 slot group）
4. 下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等默认是ALWAYS）
5. 上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链接，Source默认是HEAD）
6. 两个节点间数据分区方式是 forward（参考理解数据流的分区）
用户没有禁用 chain



## 分布式运行时环境

### `JobManager`

`flink` 集群服务的 master 节点，用来协调分布式计算，负责进行任务调度，协调 checkpoints，协调错误恢复等等。一个集群中至少有一个 `JobManager`，如果有多个 `JobManager`，其中一个作为 `leader`，其余处于备用的状态。

### `TaskManager`

`flink` 集群的 worker 节点，真正执行 dataflow 中的 tasks，并且对 streams 进行缓存和交换，集群中至少需要一个 `TaskManager`，每个 `TaskManager` 是一个 JVM 进程。

### `Clients`

连接 `flink 集群` 的客户端，向 `flink 集群` 提交计算任务

### `Task Slots`

每个 `TaskManager` 都是一个 JVM 进程，可以在不同的线程运行一个或多个线程，每个 `TaskManager` 通过 `Task Slots` 来控制可以接收多少个tasks。
每个 `Task Slots` 代表 `TaskManager` 中一个固定的资源子集，如果 1 个 `TaskManager` 有 3 个 `Task Slots`，它会将他的内存资源划分为 3 份分配给每个 slot。
通过调整 `Task Slots` 的数量而调整 subtasks 之间的隔离方式。当每个 `TaskManager` 只有一个 `Task Slot` 的时候，意味着每个 `task group` 运行在独立的的JVM 中。当一个 `TaskManager` 有多个 `slot` 的时候，意味着多个 在同一 JVM 进程中的 task 将共享 TCP 链接和心跳信息，他们也能共享。


DateStream API

`source`: 数据来源

`sink`: 处理结果输出

`window`: 窗口
