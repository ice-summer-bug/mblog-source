---
layout: post
title: Flink 探究之路 ———— State
categories: flink
tags: [flink, state]
date: 2019-01-26 09:00:00
description: apache flink 中的 state
---

# Flink 中的 `state`(状态)

`flink` 提供了完善的数据保存机制，那就是 `state`，flink 中的 `Function` 和 `Operator` 在处理输入的过程中，将计算结果或存储到 `state` 中

## `state` 的分类

`flink` 中的 `state` 分为两种类型： `Key State` 和 `Operater State`。

## `Key State`

`Key State` 总是和 `key` 相关联，只有 `KeyedStream` 中的 `Funciton` 和 `Operator` 能使用它。

你可以将 `Key State` 看作是已经被分割的 `Operator State`，而且每个 state 分区将对应一个 `key`，每一个 `Key State` 在逻辑上绑定一个唯一的结构复杂的 `key`，这个 `key` 的组成是 `<parallel-operator-instance, key>`，这里的 `key` 可以在代码中自有指定生成方法；而且因为每一个 `key` 都“属于”一个 `keyed-operaotr` 的并行实例（也可称为 sub-task）；我们可以将这个 `key` 的机构简化成 `<operator, key>`。

`Key State` 可以更进一步被组织成 `Key Groups`, `Key Group` 是 flink 重新分配 `Key State` 的最小单位；`Key Group` 的数量和定义的最大并行度的数量一致（PS: 这里的最大并行度是当前 Operator 的并行度吗？还是当前运行环境中的最大并行度？），在运行时的 `keyed-operator` 的每一个并行实例会和 `key` 一起为一个或多个 `Key Group` 工作。

***对于任何一个支持并行的 `Operator`，key 相同的事件，将被同一个并行实例处理，也只会出现在同一个 `task slot` 中，`Flink` 将 `Key State` 组织成 `Key Groups`，每个 `Key` 及其对应的 `Key State` 永久和一个特定的 `Key group` 相绑定，并且，每一个 `task slot` 负责处理一个或者多个 `Key group` 中的 key。 当 `Flink job` 被重新调节时候，`Operator` 的并行度发生变化，需要对 `state` 进行重新分配，`Flink` 承诺 `state` 的重新分配，将按照 `Key group` 为最小单位进行重新分配 ***

## `Operator State`

每一个 `Operator State` 都会和一个 `Operator` 的并行实例绑定，`Kafka Consumer` 就是一个很好的使用flink 的 `Operator State` 的示例，每一个 `Kafka Consumer` 的并行实例都维护了一个 Map 类型的 `Operator State`，Map 的构成是 Kafka 的 `Topic Partition` 对应的 `Offset`。

`Operator State` 接口支持当并行度发生变化的时候，在并行示例之间进行重新分配，这里有多种不同的重新分配的方法。


        疑问：`Key State` 和 `Operator State` 在多个并行实例中是共享的？还是各自维护


## Raw and Managed State（原生和托管的状态)

`Key State` 和 `Operator State` 有两种存在方式 `Raw State`(原生状态) 和 `Managed State`(托管状态)。

`Manager State`（托管状态）：托管状态有flink 运行时控制的数据结构表示，比如内部哈希表或者 `RocksDB`，举例说明，`ListState`(List 结构的状态)、`MapState`（Map 结构的状态），flink 运行是将对状态进行编码并写入到 `checkpoint` 中。

`Raw State`(原生状态)：原生状态将被 `Operator` 保存在它本身的数据结构中，当 `checkpoint` 被触发的时候，原生状态将以字节队列的形式写入到 `checkpoint` 中，flink 不知道原生状态的数据结构，仅能看到原生的字节数组。

所有数据流函数都能使用托管状态，但是只有实现 `operators` 才能使用原生状态。推荐使用托管状态，因为当并行度发生变化的时候，flink 可以重新分配托管状态，同时还能更好的管理内存。

## 使用托管的 `Key State`

托管的 `Key State` 接口支持方位当前输入元素的 `Key` 范围内的不同类型的状态， 这意味着 `Key State` 只能在 `KeyedStream` 上使用，通过调用 `keyBy(...)` 方法获得 `KeyedStream`。

下面是各种类型的 `Key State` 的介绍，然后再看在程序中该如何使用它们。

- `ValueState<T>`：这里维护了单个 `T` 类型的可更新、可查看的状态值（由于上述输入元素的 `key` 的限定，每个 `key` 对应一个值），使用 `update(T t)` 方法更新状态， 使用 `value()` 方法获取状态值。

- `ListState<T>`：这里维护一个 `T` 类型的状态数组，通过 `update(List<T> list)` 更新整个状态数组，通过 `add(T t)` 来追加状态，通过 `addAll(List<T> list)` 来追加多个状态，通过 `get()` 来获取整个状态数组。

- `ReduceState<T>`：维护一个 `T` 类型的聚合状态值，聚合通过 `add(T t)` 方法添加的所有状态值，状态值的聚合通过自定义 `ReduceFunction` 来实现，通过 `get()` 来获取聚合状态值

        org.apache.flink.api.common.functions.ReduceFunction<T>
        T ReduceFunction#reduce(T value1, T value2): 将两个 value 值进行合并，返回合并结果


- `AggregatingState<IN, OUT>`：通过 `add(IN in)` 方法添加状态输入值，通过 自定义的 `AggregateFunction<IN, ACC, OUT>` 对输入值进行合并之后返回 `OUT` 类型的状态值，`AggregatingState` 和 `ReduceState` 类似，区别在于 `AggregatingState` 合并操作的输入数据类型和查询结果的数据类型可以不一致。

        org.apache.flink.api.common.functions.AggregateFunction<IN, ACC, OUT>
        <IN>：输入类型
        <ACC>：自定义累计计数器的类型
        <OUT>：累计结果的类型
        ACC createAccumulator()：创建一个累计计数器，开始合并数据
        ACC add(IN value, ACC accumulator)：将输入数据累加到累计计数器上
        OUT getResult(ACC accumulator)：查询累计计数结果
        ACC merge(ACC a, ACC b)：合并两个累计计数器的数据

- `MapState<UK, UV>`：维护 map 类型的状态集合，支持 map 类型数据结构的基本操作，支持 `put(UK k, UV v)`，`putAll(Map<UK, UV>)`，`get(UK k)`，`entries()`，`keys()`，`values()` 等操作。


***上述所有 State 都支持 通过 clear() 方法类清除已经保存的状态数据***

需要注意的是：
1. 上述数据对象都是用来和状态数据交互的，无论状态数据是在内存中还是在硬盘中，或者其他存储介质中。
2. `Key State` 中，所有状态数据都和 `key` 关联，状态数据查询结果取决于输入数据的`key`，所以在用户自定义函数（UDF, User Defined Function）中查询状态数据的时候，如果输入数据的 `key` 不一致，查询的状态数据结果也会不一致。

在创建上述类型的 `Key Stated` 的时候需要用到 `StateDescriptor`，`StateDescriptor` 包含了状态的 `名称`，`状态值数据类型`，有时候还会包含一个 `UDF`，例如 `ReduceFunction`。

我们可以在 `RichFunction` 中通过 `RuntimeContext` 访问状态数据。`RuntimeContext` 提供如下方法创建各种类型的状态数据。

- ValueState<T> getState(ValueStateDescriptor<T>)
- ReducingState<T> getReducingState(ReducingStateDescriptor<T>)
- ListState<T> getListState(ListStateDescriptor<T>)
- AggregatingState<IN, OUT> getAggregatingState(AggregatingStateDescriptor<IN, ACC, OUT>)
- MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)

### 状态生存时间(State Time-To-Live, TTL)

每一个 `Key State` 都可以设置一个生存时间，如果一个状态数据设置了生存时间，当状态数据过期的时候，转态数据将被清除。

所有类型的状态数据都支持 `生存时间(Time-To-Live)`，对于 `List` 和 `Map` 类型的状态数据，`生存时间` 的粒度为集合中的单个数据。

我们通过 `StateTtlConfig` 来配置 状态数据的生存时间。

```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();

ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```

`StateDescriptor` 默认是不支持设置状态生存时间的，通过 `enableTimeToLive(StateTtlConfig ttlConfig)` 方法来设置状态生存时间。

`StateTtlConfig` 中提供如下属性

- `StateTtlConfig.UpdateType`： 配置状态生存时间的 `上次访问时间` 的计算方式，`当前时间`和`上次访问时间`的时间间隔决定状态是否过期，取值如下：
  1. `Disabled`：不设置状态生存时间
  2. `OnCreateAndWrite`：将创建或者写入时间作为 `上次访问时间`，默认使用该选项
  3. `OnReadAndWrite`：将读取或者写入时间作为 `上次访问时间`

- `StateTtlConfig.StateVisibility`：配置失效状态数据的可见性，决定当获取的状态数据已经失效时，返回什么？配置项取值如下：
  1. `NeverReturnExpired` 不返回任何过期的状态数据，默认使用该选项
  2. `ReturnExpiredIfNotCleanedUp` 返回过期但是没有被清除的状态数据

- `StateTtlConfig.TimeCharacteristic`：配置状态数据生存时间使用的时间类型，目前只支持 `ProcessingTime`



        注意：
          1. `State backends` 将状态数据的最后修改时间戳和用户状态数据一起存储，这意味着`状态生存时间` 这一特性将增加状态存储的资源消耗。
          堆存储的 `State backends` 在内存中存储一个的额外的 java 对象，它包含一个long 类型的时间戳和一个用户状态数据对象。
          而 `RocksDB` 存储的`State backend` 则是为每一个状态数据（包含List或Map中的单个数据）额外存储 8 bytes 长度的时间戳数据
          2. 目前状态生存时间只支持 `ProcessingTime` 时间类型
          3. 当恢复状态的时候，如果状态数据之前没有设置状态生存时间，现在改为设置了状态生存时间；
          或者于此相反的情况下，将出现 `兼容性错误`，抛出 `StateMigrationException`
          4. 状态生存时间的配置数据，不是 `checkpoint` 和 `savepoint` 的一部分。但是它决定了当前运行任务如何对待 `checkpoint` 或者 `savepoint`。
          5. 声明生存时间的 map 类型的状态数据支持可序列话的null 类型的状态数据。如果 null 数据不支持序列化，可以使用 `NullableSerializer` 来包装数据，代价是在 `Serialized from` 增加额外的字节。


### 状态数据的清除

目前状态数据只有通过 `ValueState#value()` 方法被读取的状态数据会被删除，也就是说，如果状态数据一直没有被读取，就不会被删除！！！当然这个问题后期应该会被修复。目前的API 只支持在获取完成状态快照的时候清理状态数据的大小（也就是这时候清除过期的状态数据），具体配置方法如下：

```java

import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot()
    .build();
```

但是需要注意的是，当我们使用增量型 checkpoint，不支持`cleanupFullSnapshot()`！

***问题：8 bytes 不够存储时间戳吧？？？***

## 使用托管的 `Operator State`

想要使用托管的 `Operator State`，`Operator` 或者 `Function` 需要实现 `CheckpointedFunction` 接口，或者 `ListCheckPointed<T extend Serializable>` 接口。

### `CheckpointedFunction`

`CheckpointedFunction` 提供以下两个方法：

```java
/**
 * 每次需要生成的时候调用该方法
 */
void snapshotState(FunctionSnapshotContext context) throws Exception;

/**
 * 当用户自定义函数(User-Defined-Function,UDF) 初始化的时候调用此方法
 * 分为两种被调用的场景
 * 1. 自定义函数第一次被初始化的时候被调用
 * 2. 从历史 checkpoint 中恢复数据的时候会被调用
 * 因此此方法被用于初始化状态数据或者从checkpoint中恢复状态数据
 */
void initializeState(FunctionInitializationContext context) throws Exception;
```

现在 `Operator State` 只支持 list 类型的状态数据，其中保存的都必须是可序列化的状态数据，而且list 中的数据相互独立，这意味着 `Operator State` 支持重新分配。不同的 `Operator State` 访问方法决定了不同的重新分配的方式：

1. Even-split redistribution：每个算子返回一个状态数据的list集合。 当从 checkpoint 中恢复状态数据或者进行重新分配的时候，List 类型的状态数据将被切割成和并行示例数量一致的子链表，每个算子获得一个子链表，子链表可能为空，也可能包含一个或多个数据。例如，当一个并行度为1 算子，拥有一个 ListState，其中包含数据 e1、e2，当算子的并行改成 2 的时候，发生重新分配，ListState 被切分成两个子链，算子的并行实例1 获得包含 e1 的ListState, 并行实例2 获得包含 e2 的ListState。

2. Union redistribution：每个算子返回一个状态数据的 List 集合，包含所理由的状态数据。当发生重新分配或者从 checkpoint 中恢复状态数据的时候，每个算子都将获取到全部的状态数据。


`Operator State` 的使用方法和 `Key State` 类似，都是通过 `StateDescriptor` 来初始化状态数据，下面这个例子使用了 `Even-split redistribution` 类型的重新分配模式的 `Operator State`

```java

public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        /**
        * 当从历史 checkpoint 中恢复状态数据的时候，需要读取保存历史状态数据
        * isRestored 方法可以查询当前是否缓存了历史状态数据
        */
        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
```

获取 `Operator State` 的入口是 `OperatorStateStore` 提供的方法，这些方法的命名能够见名之意，如果要使用 `Union redistribution` 重新分配模式的 `Operator State`，我们将使用 `getUnionListState(StateDescriptor stateDescriptor)` 访问状态数据。如果只是使用 `Even-split redistribution` 重新分配模式的 `Operaotor State`，调用 `getListState(StateDescriptor stateDescriptor)` 方法。

顺便说一句，`Key State` 也可以在 `initializeState` 方法中被初始化。

### `ListCheckpointed`

相对于 `CheckpointedFunction` 接口，`ListCheckPointed` 接口只支持 `Even-split redistribution`（偶切分重新分配模式）的 ListState。`ListCheckpointed` 接口包含如下两个接口:

```java
/**
* 返回一个状态数据的 List 用于生成 checkpoint，
* 如果状态数据保证不会被重新分区，可以永远返回 `Collections.singletonList(MY_STATE)`
*/
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

/**
* 重历史 checkpoint 恢复历史状态数据
*/
void restoreState(List<T> state) throws Exception;
```


## `CheckpointListener`

如果生成 `checkpoint` 的时候需要周知其他服务，可以使用 `CheckpointListener`。

***待补充***


## 广播状态(Broadcast State)

关于广播状态，简而言之，就是一个输入数据对下游的所有处理流程都有影响，需要周知所有下游处理算子。例如一个低流速数据处理规则输入流是高流速的数据数据流，规则输入流需要广播给处理数据的所有算子。

广播状态与其他 `Operator State` 之间有三个主要区别。与其余的 operator state 相反，广播状态：

- Map 的格式
- 每个算子只能有一条广播的输入流和一个非广播输入流
- 算子可以有多个不同名字的广播状态


### 广播状态相关 `API`

对于广播状态的使用[待补充](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/broadcast_state.html "广播状态")，这里重点突出如下注意事项。


### 重要注意事项

在使用广播状态时要记住以下4个重要事项：

- 使用广播状态，`Operator Task` 之间不会相互通信<br/>
  这也是为什么只有状态广播方 `(Keyed)-BroadcastProcessFunction` 能修改广播状态数据的内容；此外，用户需要保证所有并发实例上对于广播状态的输入源的处理逻辑是幂等的，否则不同的并发实例将拥有不一致的广播状态，导致处理结果等数据不一致。

- 不同的 `Operator Task` 中的广播状态的顺序可能不一致<br/>
  虽然 `Flink`  保证广告状态都会下发给所有算子，不会丢失，但是并不保证广播状态的顺序一致性。因此对于广播状态不能依赖于输入数据的顺序。


- 所有并行实例都会快照一份广播状态数据<br/>
  虽然所有并行实例中的广播状态都是一致的（正常使用的情况下），但是每个并行实例都会快找一份自己的广播数据，而不是只快照一份。这种设计是为了避免多个并行示例在恢复期间从单个文件读取数据而造成热点问题。但是也导致了随着并行度的增大，`checkpoint` 数据大小也会膨胀。`Flink` 保证数据恢复/扩容的时候不会产生重复的数据，也不会丢失数据。在以相同或者更小的并行度恢复时，每个 `task` 读取对应的 `checkpoint`，在以更大的并行度恢复时，每个 `task` 读取自己的 `checkpoint`，剩余新增的 `task` 会循环读取 `checkpoint`。

- `RocksDB state backend` 不支持广播状态<br/>
  广播状态目前在运行时保存在内存中。因为当前，`RocksDB state backends`还不支持广播状态。

这里谨期望广播状态能够进一步优化。


###### 参考文档

https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/<br/>
https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/state.html
https://stackoverflow.com/questions/45738021/flink-state-backend-keys-atomicy-and-distribution<br/>

本人 flink 小白一枚，如有错漏之处，敬请斧正！
