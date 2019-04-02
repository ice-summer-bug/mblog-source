---
layout: post
title: Flink 探究之路 ———— Flink Job 重启策略
categories: flink
tags: flink
date: 2019-01-26 09:00:00
description: apache flink 的
---

# Flink Job 重启策略

flink 提供多种重启策略，可以在 `flink-conf.yaml` 中通过配置 `restart-strategy` 参数设置默认使用的重启策略，也可以在 `job` 中指定重启策略。

flink 提供如下重启策略

1. 固定延时重启(Fixed delay)
2. 故障率重启(Failure rate)
3. 不重启(No restart)

当 `job` 没有开启 `checkpoint` 的时候，一定是使用 `不重启` 策略，如果 `job` 开启了 `checkpoint` 但是没有设置重启策略的时候，将使用 `固定延时重启策略`

### 固定延时重启(Fixed delay)

`job` 发生故障后，尝试重启 n 次，每次重启间隔固定时间 t，n 次之后失败。

1. 在 `flink-conf.yaml` 中设置重启策略

```java
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: 3 // 故障发生后尝试重启次数
restart-strategy.fixed-delay.delay: 10 s // 重启的间隔时间
```

2. `job` 中直接指定重启策略

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

### 故障率重启(Failure rate)

故障率重启策略在 `job` 发生故障后尝试重启，但是当在固定时间`failure rate interval` 内故障次数超过 `failure rate` 次后，`job` 被认定为故障。

举例说明：`job` 故障停止运行之后，在 5min 内重试 3次，每次间隔10s，如果3次之后依旧失败，则认定为故障

1. 在 `flink-conf.yaml` 中设置重启策略

```java
restart-strategy: failure-rate
restart-strategy.failure-rate.max-failures-per-interval: 3  // 故障后尝试重启次数
restart-strategy.failure-rate.failure-rate-interval: 5 min  // 故障后检查故障率的时间间隔
restart-strategy.failure-rate.delay: 10 s   // 故障后两次尝试重启的间隔时间
```

2. `job` 中直接指定重启策略

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
```

### 不重启策略(No Restart)

1. 在 `flink-conf.yaml` 中设置重启策略

```java
restart-strategy: none
```


2. `job` 中直接指定重启策略

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
```

###### 参考文档
https://ci.apache.org/projects/flink/flink-docs-stable/dev/restart_strategies.html

本人 flink 小白一枚，如有错漏之处，敬请斧正！
