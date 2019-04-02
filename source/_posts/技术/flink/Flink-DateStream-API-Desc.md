---
layout: post
title: Flink DateStream API
categories: flink
tags: flink
date: 2018-11-19 22:00:00
description: Apache Flink Datastream Api 分析
---

# checkpoint

_Chandy Lamport algorithm_

## barrier

## checkpoint configurations

### checkpoint mode  

1. exactly-once
2. at-least-once

<span style="color:red">是否可以给单个算子指定 `exactly-once` 或者 `at-least-once` </span>


### checkpoint interval

### checkpoint timeout  

### minimum time between checkpoints

### number of concurrent checkpoints

### externalized checkpoints

1. `ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`: 当 flink job 被取消的时候保存 `checkpoint`，也就是说当我们主动取消 job 的时候需要我手动删除
2. `ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`: 当 flink job 被取消的时候删除 `checkpoint`， 只有 job 状态是故障失败时 `checkpoint` 才会被保存。

#### retain state checkpoint

```yaml
state.checkpoints.dir: hdfs:///checkpoints/
```

# state

## keyed state

`Keyed Stream` 的状态

### ListState

List 类型的状态集合

### ValueState

单个状态

### MapState

Map 类型的状态集合

### ReducingState

合并统一类型的多个状态数据到一个状态数据

### AggregatingState

## operator state

`org.apache.flink.api.common.state.OperatorStateStore` 用于注册 operator state

`getUnionListState`: 获取分布式集群中的 `ListState`

`getListState`: 获取单点的 `ListState`



# state backend

1. asynchronus
2. synchronus

## memory state backends

缺点
1. 每个 `state` 大小限制是 5M
2. `state` 的大小不能超过 `akka frame size`
3. `aggregate state` 必须存放在 `Job Manager` 的内存中

## fs state backends

该模式需要配置文件系统URL，支持 `hdfs(hdfs://namenode:40010/flink/checkpoints)` 或 `本地文件(file:///data/flink/checkpoints)`
将未完成的数据存储在 `TaskManager` 的内存中，而将 `state snapshot` 写入到文件系统或者文件夹中，最小化元数据被保存在JobManager的内存中（在高可用模式下，被保存在元数据checkpoint中）。
该模式默认 `异步` 写入 `state backend`，也可以改为 `同步` 写入

```
new FsStateBackend(path, false); // true: 异步，false：同步
```
## rocksdb state backends

该模式需要配置文件系统URL，支持 `hdfs(hdfs://namenode:40010/flink/checkpoints)` 或 `本地文件(file:///data/flink/checkpoints)`
将未完成的数据存储在 `rocksDB` 中，而将 `state snapshot` 写入配置的文件系统或者文件夹中，最小化元数据被保存在JobManager的内存中（在高可用模式下，被保存在元数据checkpoint中）。

`RocksDBStateBackend` 只支持异步快照模式

缺点：
1. 因为RocksDB的JNI的API基于byte[]，状态中每个key和每个value所支持的最大值各为2^31字节。

***注意：state使用了RocksDB的合并算子（如ListState），状态的大小很容易累积超过2^31字节，下一次状态恢复就会失败。这是当前RocksDB JNI的局限性。***

**_RocksDBStateBackend是当前唯一一种提供增量checkpoint的state backend._**


## restart strategies（重启策略）

# Event Time

## Processing time

`flink` 开始处理事件的时间

## Event time

时间发生的原始时间，由`事件发生器`自主设置

## Ingestion time  

`Flink Source` 接收到事件的时间

# operator

## operator lifecycle

```code
// initialization phase
OPERATOR::setup
    UDF::setRuntimeContext
OPERATOR::initializeState
OPERATOR::open
    UDF::open

// processing phase (called on every element/watermark)
OPERATOR::processElement
    UDF::run
OPERATOR::processWatermark

// checkpointing phase (called asynchronously on every checkpoint)
OPERATOR::snapshotState

// termination phase
OPERATOR::close
    UDF::close
OPERATOR::dispose
```

***注意：`initializeState()`包含operator state的初始化（例如register keyed state），也包含任务失败后从checkpoint中恢复state的逻辑。***
