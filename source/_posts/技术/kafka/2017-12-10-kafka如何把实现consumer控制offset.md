---
layout: post
title: kafka 中的偏移量
categories: kafka
tags: [kafka,Tools]
date: 2017-12-10 21:00:00
description: kafka 简介和偏移量说明
---

# kafka 中的偏移量

## 首先来了解一下Kafka所使用的基本术语：

`Topic`: Kafka将消息种子(Feed)分门别类，每一类的消息称之为一个主题(Topic).

`Producer`: 发布消息的对象称之为主题生产者(Kafka topic producer)

`Consumer`: 订阅消息并处理发布的消息的种子的对象称之为主题消费者(consumers)

`Broker`: 已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理(Broker). 消费者可以订阅一个或多个主题（topic），并从Broker拉数据，从而消费这些已发布的消息。

-------------------------------

## 话题和日志 (Topic和Log)

让我们更深入的了解Kafka中的Topic。

Topic是发布的消息的类别或者种子Feed名。对于每一个Topic，Kafka集群维护这一个分区的log，就像下图中的示例：

![图片](/assets/picture/kafka-partition.png "Kafka partition log")

每一个分区都是一个顺序的、不可变的消息队列， 并且可以持续的添加。分区中的消息都被分了一个序列号，称之为`偏移量`(offset)，在每个分区中此`偏移量`都是唯一的。

Kafka集群保持所有的消息，直到它们过期， 无论消息是否被消费了。 实际上消费者所持有的仅有的元数据就是这个`偏移量`，也就是消费者在这个log中的位置。 这个`偏移量`由`消费者`控制：正常情况当消费者消费消息的时候，`偏移量`也线性的的增加。但是实际`偏移量`由消费者控制，消费者可以将`偏移量`重置为更老的一个`偏移量`，重新读取消息。 可以看到这种设计对消费者来说操作自如， 一个消费者的操作不会影响其它消费者对此log的处理。

![图片](/assets/picture/kafka-offset.png "Kafka partition offset")


--------------------------------------

## 消费者(Consumers)

通常来讲，消息模型可以分为两种， `队列` 和 `发布-订阅` 式。 队列的处理方式是 一组消费者从服务器读取消息，一条消息只有其中的一个消费者来处理。在发布-订阅模型中，消息被广播给所有的消费者，接收到消息的消费者都可以处理此消息。Kafka为这两种模型提供了单一的消费者抽象模型： 消费者组 （consumer group）。 消费者用一个消费者组名标记自己。 一个发布在Topic上消息被分发给此消费者组中的一个消费者。 假如所有的消费者都在一个组中，那么这就变成了queue模型。 假如所有的消费者都在不同的组中，那么就完全变成了发布-订阅模型。 更通用的， 我们可以创建一些消费者组作为逻辑上的订阅者。每个组包含数目不等的消费者， 一个组内多个消费者可以用来扩展性能和容错。正如下图所示：
2个kafka集群托管4个分区（P0-P3），2个消费者组，消费组A有2个消费者实例，消费组B有4个。

![图片](/assets/picture/kafka-customer-type.png "Kafka customer type")

正像传统的消息系统一样，Kafka保证消息的顺序不变。 再详细扯几句。传统的队列模型保持消息，并且保证它们的先后顺序不变。但是， 尽管服务器保证了消息的顺序，消息还是异步的发送给各个消费者，消费者收到消息的先后顺序不能保证了。这也意味着并行消费将不能保证消息的先后顺序。用过传统的消息系统的同学肯定清楚，消息的顺序处理很让人头痛。如果只让一个消费者处理消息，又违背了并行处理的初衷。 在这一点上Kafka做的更好，尽管并没有完全解决上述问题。 Kafka采用了一种分而治之的策略：分区。 因为Topic分区中消息只能由消费者组中的唯一一个消费者处理，所以消息肯定是按照先后顺序进行处理的。但是它也仅仅是保证Topic的一个分区顺序处理，不能保证跨分区的消息先后处理顺序。 所以，如果你想要顺序的处理Topic的所有消息，那就只提供一个分区。

--------------------------------------

## 偏移量(Offsets)的管理

如上文所述，kafka为分区中的每条消息保存一个 `偏移量（offset）`，这个`偏移量`是该分区中一条消息的唯一标示符。也表示消费者在分区的位置。例如，一个位置是5的消费者(说明已经消费了0到4的消息)，下一个接收消息的偏移量为5的消息。实际上有两个与消费者相关的“位置”概念：

消费者的位置给出了下一条记录的偏移量。它比消费者在该分区中看到的最大偏移量要大一个。 它在每次消费者在调用poll(long)中接收消息时自动增长。

“已提交”的位置是已安全保存的最后偏移量，如果进程失败或重新启动时，消费者将恢复到这个偏移量。消费者可以选择定期自动提交偏移量，也可以选择通过调用commit API来手动的控制(如：commitSync 和 commitAsync)。

### 消费者如何提交 `偏移量(Offsets)`

1. 自动提交

这种方式只要在启动时配置属性 `enable.auto.commit` 为 `true` 即可， 示例代码如下：

```java
private static void main (String[] args) {
	Properties props = new Properties();
	props.put("bootstrap.servers", "localhost:9092");
	props.put("group.id", "test");
	props.put("enable.auto.commit", "true");
	props.put("auto.commit.interval.ms", "1000");
	props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
	props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
	KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
	consumer.subscribe(Arrays.asList("test"));
	System.out.println(consumer);
	while (true) {
	  ConsumerRecords<String, String> records = consumer.poll(100);
	  for (ConsumerRecord<String, String> record : records) {
	    System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(),
	      record.value());
	  }
	}
}
```

2. 手动提交

在一些场景中，`消费者` 需要自行判定消息是否被消费了，如果没有判断为没有消费（ps:可能是消费失败了，需要重试），`消费者` 不会改变 offset；只有 `消费者` 判定消费成功是，才手动调用 `commitSync()` 或 `commitAsync()` 方法去提交 `偏移量`； 当然此时我们需要把 `enable.auto.commit` 置为 false。

下面给出个小例子：

```java

Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("group.id", "test");
 props.put("enable.auto.commit", "false"); // 主动提交置为false
 props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
 consumer.subscribe(Arrays.asList("test"));
 System.out.println(consumer);
 while (true) {
	 ConsumerRecords<String, String> records = consumer.poll(100);
	 for (ConsumerRecord<String, String> record : records) {
		 System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(),
			 record.value());
		 // 逻辑处理
		 // ...
	 }
	 if (success) { // 如果判定消费成功，则手动提交offset到 broker
		 consumer.commitSync();
	 }
 }
```

### 消费者如果控制偏移量

在一些场景下，消费者需要控制自己要读取的偏移量，此时用户可以通过API手动设置开始读取的 `偏移量`

API 中提供以下方法：

1. 指定到某个分区的具体 offset
org.apache.kafka.clients.consumer.KafkaConsumer#seek(TopicPartition partition, long offset)

2. 指定到某些分区的开始
org.apache.kafka.clients.consumer.KafkaConsumer#seekToBeginning(Collection<TopicPartition> partitions)

3. 指定到某些分区的结束，从上次结束的位置开始消费
org.apache.kafka.clients.consumer.KafkaConsumer#seekToEnd(Collection<TopicPartition> partitions)

此时我们需要知道当前的 `Topic` 的偏移量信息，`Kafka` 为我们提供了很友好的工具 `Get Offset Shell`

`Get Offset Shell`
get offsets for a topic

```Shell
bin/kafka-run-class.sh kafka.tools.GetOffsetShell
```

required argument [broker-list], [topic]

| Option | Description |
| ------ | ----------- |  
| --broker-list [hostname:port,....] | REQUIRED: The list of hostname and [hostname:port] port of the server to connect to. |
| --max-wait-ms [Integer: ms] | The max amount of time each fetch request waits. (default: 1000) |
| --offsets [Integer: count] | number of offsets returned (default: 1) |
| --partitions [partition ids] | comma separated list of partition ids. If not specified, will find offsets for all partitions (default) |
| --time [Long: timestamp in milliseconds] | -1(latest) / -2 (earliest) timestamp; offsets will come before this timestamp, as in getOffsetsBefore  |
| --topic [topic] | REQUIRED: The topic to get offsets from. |

示例：

查询最近的offset

				➜  kafka_2.12-1.0.0 bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 -topic test --time -1

输出

				test:0:52

查询开始的的offset


				➜  kafka_2.12-1.0.0 bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 -topic test --time -2

输出

				test:0:0



### `偏移量`的存储

新版本的 `Kafka` 将偏移量信息存储在名为 `__consumer_offsets` 的topic 中,
也支持将 `偏移量` 信息存储在 `Zookeeper` 中</br>
通过设置属性 `offsets.storage` 控制，`offsets.storage` 属性可选值有 `kafka` 和 `zookeeper`

消费者也可以不使用 `Kafka` 提供的偏移量存储方案，可自定义存储方式，详见[官方文档](http://kafka.apache.org/0101/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html#rebalancecallback "官方文档")
