---
layout: post
title: yarn 集群日常问题汇总
categories: yarn
tags: yarn
date: 2018-11-19 22:00:00
description: yarn 集群日常问题汇总
---

# yarn 日志

## 如何确定 yarn 日志位置

1. 当 `yarn.log-aggregation-enable = true` 时，`yarn` 集群中的 `application` 的日志将被聚合到 `yarn.nodemanager.remote-app-log-dir` 指向的目录中去，保留时间为 `yarn.log-aggregation.retain-seconds`
<br/>
***`yarn.nodemanager.remote-app-log-dir` 指向的目录是 hdfs 目录***
2. 当 `yarn.log-aggregation-enable = false` 时，`yarn` 集群中的 `application` 的日志将被存储到 `yarn.nodemanager.log-dirs` 属性指向的在 `nodemanager nodes` 上的文件夹中，保留时间为 `yarn.nodemanager.log.retain-seconds`

## 为什么会有 application 没有日志

```log
yarn application does not have any log files.
```
