---
layout: post
title: Flink 探究之路 ———— 容错机制，Checkpoint 和 Savepoint
categories: flink
tags: flink
date: 2019-01-26 09:00:00
description: apache flink 数据流容错机制介绍文档译文
---

# Flink 数据流容错机制译文

`Flink` 最吸引使用者的地方就是它提供的容错机制保证可以持续性的恢复数据流应用程序的`状态`。`Flink` 保证即使在失败的情况下，数据流中的每一条数据最终也能确保只会对状态数据响应一次（`exactly once`）。`响应一次` 的机制可以手动降级到 `至少响应一次`(`at least once`)。

`容错机制` 对分布式流式数据持续性的产生`快照`(`snapshot`)并存储。对于持有小型数据状态的数据流应用来说，产生 `快照` 的过程是很轻量级的，对于数据流的正常处理过程的影响微乎其微。数据流应用的 `状态` 数据可以存储到一个可配置的环境(`Master`节点中，或者 `HDFS` 中）。

当程序失败（机器、网络或者软件故障）的时候，`Flink` 将停止分布式数据流应用。然后再从最后一次成功的 `checkpoint` 中保存的 `状态`(`state`) 数据中恢复应用的所有 `算子`（`Operator`）。输入数据也被重置到最后一次成功的`快照`数据中保存的位置 。 `Flink` 保证并行数据流在重启之后处理的所有数据都不会是最近一次成功的 `checkpoint` 之前的数据。

    注意：
    1. `checkpointing` 功能默认是关闭的，需要手动配置，指定开启 `checkpointing`，具体操作说明详见：[Checkpointing 说明文档](https://ci.apache.org/projects/flink/flink-docs-master/dev/stream/state/checkpointing.html)
    2. 在 `Flink` 完成保证的基础上，数据流输入源 (`streaming source`)需要保障能回退到指定的最近一个位置。在 `Apache Kafka ` 提供这个能力的基础上，Flink 适配 Kafka 的 connector 利用这个能力实现容错机制。Flink
	连接器(Connectors)对容错机制的支持详见：[数据输入源和输出流的容错机制](https://ci.apache.org/projects/flink/flink-docs-master/dev/connectors/guarantees.html)
	
# `Checkpointing`

`Flink` 的 `容错机制` 简而言之就是持续不断的对 `分布式数据流` 和 `算子状态(Operator state)` 产生 `一致性` 的 `快照` 数据。这些 `快照` 数据系统遇到故障时，用于从错误状态中恢复的 `检查点` (`checkpoints`)。 `Flink` 产生 `快照` 数据的机制的详细描述如下： [Lightweight Asynchronous Snapshots for Distributed Dataflows](http://arxiv.org/abs/1506.08603 "Lightweight Asynchronous Snapshots for Distributed Dataflows")，该算法是在参考 [Chandy-Lamport algorithm](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf "Chandy-Lamport algorithm") 算法的基础上进行改进的，并针对 `Flink 执行模型` 进行量身定做。

## `Barriers` (`栅栏`)

`Flink` 的分布式快照的核心组成部分就是 `Barriers(栅栏)`，这些 `Barriers(栅栏)` 被插入到数据流中，和数据一起往下流。`Barriers(栅栏)` 不会影响数据流中数据的顺序，数据流保证严格有序。`Barriers(栅栏)` 将数据切分成两部分，前一部分的数据进入当前的快照数据(`snapshot`)中，后一部分的数据进入下一快照数据。每个 `Barriers(栅栏)` 都有一个 `ID`，这个 `ID` 就是 `Barriers(栅栏)` 前一个 `snapshot` 的 `ID`。`Barriers(栅栏)` 不会影响数据流的处理，所以非常轻量级。多个不同 `快照` 的多个 `Barriers(栅栏)` 可以在数据流中同时存在，即多个 `快照` 可以同时创建。

***问题：数据流中的数据也会进入 `快照` ？？？不应该是只包含状态数据吗？***

![Barriers](https://segmentfault.com/img/remote/1460000008129555 "Barriers")

`Barriers(栅栏)` 被插入到 `数据源`的并行数据流中。为快照 `n` 产生的 `Barriers` 注入的位置 S<sub>n</sub> 就是在源数据中包含这些快照数据的位置。例如，在 `Apache Kafka` 中这个位置就是在分区(`partition`) 中最后一条已消费数据的偏移位置。 这个位置 S<sub>n</sub> 将被上报给检查点协调器(`checkpoint coordinator`)，也就是 Flink 的 `Job Manager`。

然后 `Barriers(栅栏)` 流向下游数据流，当中间的`算子(Operator)` 从所有的上游输入流都接收到了 `快照 n` 的 `栅栏` 之后，向所有下游算子下发 `快照 n` 的 `栅栏`。当 `输出算子(sink operator)` （flink 有向无环图[DAG] 的尾节点）从它的所有上游输入流都接收到了 `快照 n` 的 `栅栏` 之后会检查点协调器发起 ACK 确认已接收到 `快照 n`。当所有的 `输出算子(sink operator)` 都发出了ACK 确认之后，`快照 n` 的数据被认为已经被处理完成了。

当 `快照 n` 已经被确认处理完成了，当前任务不会再向输入流请求获取 `快照 n` 之前的数据，因此这些数据将已经完成通过真个拓扑数据流。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/stream_aligning.svg "stream barriers aligning")

接收多个输入流的算子需要在快照的栅栏上对齐输入流，上图描述了如下特性：

- 当算子接收到其中一个上游输入流的 `快照 n` 的栅栏的时候，算子不会处理这个栅栏之后的任何数据，直到它从剩下的所有输入流都接收到 `快照 n` 的栅栏。否则  `快照 n` 和  `快照 n+1` 的数据将被混合在一起。

- 数据流向检查点协调器报告栅栏的时候会被缓存并搁置，这个数据流的数据不会被处理，而是放置到输入缓存中。

- 一旦从最后一个流收到了`barrier n`，这个算子会发送所有积压的记录（个人注：将barrier之前的数据都发送出去），然后发送快照n的barrier。

- 然后，它继续处理从所有输入流中的数据，先处理输入缓存中的数据，然后处理流中的数据。

## 状态 `State`

如果一个算子包含`状态`，那这个`状态`数据一定是 `快照` 的一部分，算子状态有不停的形式：

- `自定义状态`：通过转化函数（如 `map()` 或 `filter()`）来创建和修改状态数据，详见 [State in Streaming Applications](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/index.html "State in Streaming Applications")

- `系统状态`：这种状态时指的是算子计算过程中的一部分缓存数据。典型的例子就是 `窗口缓存`，系统收集窗口对应的数据到缓存，直到窗口计算或者发射。

算子在接收到所有上游输入流的栅栏之后，在向所有输出流发射栅栏之前对状态数据进行快照。此时栅栏之前的数据对状态的更改已经生效，并且栅栏之后的数据对状态的修改不会发生。由于快照的状态的数据可能会比较大，它可以存储到一个可配置的状态后端存储系统中。默认状态下，状态数据存储在 `JobManager` 的内存中，但是在生产环境还是需要配置成一个 `可靠` 的分布式存储系统（例如 `HDFS`）. 状态被存储之后，算子会确认其检查点完成，将 `快照` 的 `栅栏` 的数据发送给下游。

现在我们可以看一下 `快照` 中包含的数据。

- 对于并行的输入数据源，快照建立时数据流的偏移位置。
- 对于算子，快照包含了一个指向装填实际存储位置的指针。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/checkpointing.svg "checkpointing")

## Exactly Once vs. At Least Once 只响应一次还是还是至少响应一次

对齐操作可能会增大数据流应用的延时，一般来说，对齐产生的额外延时只有几毫秒的数量级，但是我们也发现过延迟显著增加的异常情况。对于要求延时非常低（几毫秒）的数据流应用，flink 提供在产生检查点的时候关闭对齐的开关。如果关闭对齐步骤，算子会在接收到一个上游的栅栏的时候就会产生一个快照，而不是等到其他上游的栅栏都到齐了再来生成快照。

当对齐被关闭的时候，算子在收到栅栏的时候也会持续的处理输入数据。也就是说：算子在会在产生 `检查点 n` 的时候，会处理属于 `检查点 n+1` 的数据。所以当故障恢复的时候，这部分数据会被重复处理，因为这些数据都属于 `检查点 n` 的快照数据，同时在 `检查点 n` 之后也会被回放而被再次处理。

				注意：
				对齐操作只会发生在多输入运算（join）或者多输出的算子（例如重分区，分流）的场景下。
				因此，对于普通的并行数据流操作（`map()`, `flatMap()`, fliter() 等），
				及时在 `至少响应一次（at least once）` 的模式下，也会保证 `只响应一次（exactly once）`


## Asynchronous State Snapshots 异步状态快照

上面所述的机制表明算子在存储快照数据到后端存储系统的时候会停止处理输入数据，这种同步产生状态快照的模式每次产生的快照的时候都会引入额外的延时。

我们完全可以让算子在快照数据的同时继续处理输入数据，让快照的存储在后台异步进行。为了做到异步状态快照，算子必须能保证产生一个状态数据对象被存储之后，后续对状态的修改不会影响这个状态数据对象。例如 `RocksDB` 中使用的 `写时复制（ copy-on-write ）` 类型的数据结构。

接收到输入数据的栅栏的时候，算子开始异步的快照复制出它的状态。算子立即向输出发射栅栏，并继续处理输入数据。当后台异步快照完成时，算子会向 `检查点协调器`（`checkpoints coordinator`, 也就是 `Job Manager`）确认检查点完成，现在检查点完成的充分条件是：所有的 `输出算子（sink）` 都接收到了栅栏，而且所有有状态的算子确认完成了状态数据的备份（这个确认操作可能会晚于栅栏到达 `输出算子（sink）`）。

详细的状态快照见： [State Backends](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/state_backends.html "State Backends")

## `故障恢复(Recovery)`

这种机制下的故障恢复就很简单：当发生故障的时候，`Flink` 选择最新完成的`检查点 k`。然后系统重新部署整个分布式数据流，给所有的算子提供快照在检查点中的状态数据用于恢复。输入流的读取位置被设置到从 S<sub>k</sub> 开始读取，对于 `Apache Kafka` 来说就是通知 `consumer` 从偏移位置 S<sub>k</sub> 开始消费消息。

## 算子快照的实现

算子产生快照的过程分为两个部分：`同步部分` 和 `异步部分`。

算子和状态后端存储系统(`State Backends`) 提供 `Java FutureTask` 用于快照。这个任务包含同步部分已经完成，异步部分还在等待的状态，检查点的异步部分在后台线程中被执行。

完成同步的算子仅仅返回一个已经完成的 `FutureTask`。如果需要异步执行，`FutureTask` 中的 `run()` 方法将被会调用。

这些 `FutureTask` 是可以取消的，这样就可以释放流和其他资源的消耗。

# 比较一下 `Checkpoint` 和 `Savepoint`

`Checkpoint` 和 `Savepoint` 都是 flink 提供的容错恢复机制，两个不管是命名还是使用方式都很类似，这里分别对两个进行一个简单的介绍并且对二者进行对比。

## `保存点(Savepoint)`

保存点是通过 `Flink` 的 `检查点机制(checkpointing mechanism)` 创建的，包含数据流任务的运行状态的一个一致性的快照数据。你可以用保存点去停止并重启任务、复制任务或者更新任务。

保存点由两个部分组成

1. 存储在一个稳定的存储介质（`HDFS`、`S3`等）上的，一般来说比较大的包含二进制文件的文件夹。这些二进制文件纯粹的保存任务的运行状态的快照数据。
2. 一个元数据文件（一般来说比较小），文件中保存了指向存储在稳定存储介质上的保存点的所有文件的指针（文件路径）。

## 算子命名的重要性

保存点中维护了一个 map 格式的数据

				Operator ID | State
				------------+------------------------
				source-id   | State of StatefulSource
				mapper-id   | State of StatefulMapper

通过算子的ID 能够获取到起对应的状态数据，在新增、删除算子或者修改算子的顺序的时候如果没有自定义命名，而是使用 flink 的默认命名方式，算子的数量和顺序的改变会影响重新启动的算子的ID，可能会导致算子ID 的状态数据匹配错误。强烈建议给每个算子设置一个  `ID`。

## 从保存点重启任务的时候算子修改会有什么结果?

### 新增一个有状态的节点

When you add a new operator to your job it will be initialized without any state. Savepoints contain the state of each stateful operator. Stateless operators are simply not part of the savepoint. The new operator behaves similar to a stateless operator.

新增的有状态的算子在重启的时候变现的和无状态的算子一样，因为保存点中没有这个算子对应的装填数据。

### 删除一个有状态的节点

默认情况下删除一个有状态的节点会导致重启失败，因为这个过程默认需要确保所有状态数据都要对应一个算子。如果你真的需要删除一个状态的节点，你需要在启动参数中加上参数 `--allowNonRestoredState (short: -n)`

```bash
$ bin/flink run -s :savepointPath -n [:runArgs]
```

### 重新排序有状态的节点 && 删除或者重新排序无状态的节点

算子指定ID 的情况下没问题，否则可能会导致状态数据匹配错误而重启失败

### 修改算子的并行度

`flink 1.2` 以上版本并且没有使用任何 `State` 相关的 `过时API` 没有问题。
如果保存点是在 `flink 1.2` 以下版本或者使用了 `State` 相关的 `过时API` 的代码中生成的，你需要升级 flink 版本并且替换 `State` 相关的 `过时API`，详见 [upgrading jobs and Flink versions guide](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/upgrading.html "upgrading jobs and Flink versions guide.")


## `保存点(Savepoint)` 和 `检查点(Checkpoint)` 的区别

从在概念上说，`保存点(Savepoint)` 和 `检查点(Checkpoint)` 的不同比较像备份不同于传统数据库从日志恢复。

下面是他们的不同点：

|| 检查点 | 保存点 |
|-|-|-|
| 目的 | 为出现异常情况 Flink 任务提供一个恢复机制确保任务能从部分故障中恢复 | 用户手动维护任务的时候触发应用重启等操作 |
| 设计要求 | 作为一个用于恢复的，被周期调用的方法，它的实现要求:<br/> 1. 创建过程很轻量级 <br/> 2. 可以快速的从检查点恢复故障  | 保存点的设计要求更着重于备份数据的便携性，而不是很关注创建过程的轻量级和快速恢复。 |
| 生命周期 | flink 直接管理检查点的生命周期，从创建到释放都不需要维护人用手动触发 | 维护人员手动触发保存点的创建和删除，所有者是维护人员。 |
| 删除 | 用户停止任务之后被删除（除非编码或配置声明保留） | 维护人员手动删除 |
| 具体实现 | 虽然目前保存点和检查点的代码实现和产生的文件格式都是一样的，但是使用 RocksDB 的检查点使用的保存格式不是flink 定义的，而是 RocksDB 自定义的格式，而且 RocksDB 的检查点支持增量检查点<br/><span style='color:red'>不支持 rescaling </span> | 不支持增量，支持 rescaling |

*** rescaling 的翻译存疑，猜测是改变并行度***


###### 译者注

本人对文档的翻译过程中，对Flink 相关名词翻译如下：

| 英文名词 | 译者翻译 |
|-|-|
| Connector| 连接器 |
| Data Sources| 数据输入源|
| Data Sinks | 数据输出流 |
| rescaling | - |

###### 参考文档

https://ci.apache.org/projects/flink/flink-docs-release-1.7/internals/stream_checkpointing.html<br/>
https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/checkpoints.html<br/>
https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/savepoints.html<br/>

本人 flink 小白萌新一枚，如有错漏之处，敬请斧正！
