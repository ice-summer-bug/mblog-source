---
layout: post
title: Tomcat 日志配置说明
categories: Tomcat
tags: [Tomcat, Java Web, log]
date: 2018-09-28 00:00:00
description: Tomcat 中的日志简要说明
---

### 前言

日志应该是除了代码之外，程序员最好的朋友了，它可以帮助我们定位问题、修复bug，或者是确认服务是否正常运转；很多时候我们做一次部署只是为了加几行日志；

而 `Tomcat` 作为经久畅销的web 服务器，一直是web 开发者首选，而 `Tomcat` 的原生日志是我们判断这个服务器是否正常运转的重要数据。

### Java 日志组件

这里我们按照历史顺序简单介绍一下 `Java` 常用的日志组件，

`JUL`(Java Util Logging): 是 `jdk` 自带的log 实现组件，虽然是官方出品但是它并没有被广泛使用，主要是下面几个原因
1. `JUL` 出现的太晚了，2002年它才被放到 `jdk1.4` 中，当时已经有很多第三方的日志组件被广泛使用了
2. `JUL` 早期性能问题太明显，到 `JDK1.5` 才有所改善，但是它和其他第三方日志组件`logback`或`log4j2`相比也还是有差距
3. `JUL` 提供的功能不如第三方组件`logback`或`log4j2`完善

`log4j` 是在 `logback` 之前被广泛使用的日志实现组件，`log4j` 在设计上十分优秀，对后期的`Java` 日志框架有深远的影响，但是它在性能上存在缺陷；`logback` 出现之后就取代了 `log4j`

`JCL`(Apache Commons Logging): apache 提出的 `Log Facade`，只提供日志api，不提供实现，通过不同的 Adapter 来使用 `JUL` 或者 `log4j`；在打印日志的时候调用的都是 `JCL` 指定的api ，具体实现是看当前的 `classpath` 中有什么实现，如果什么都没有

`slf4j`(The Simple Logging Facade For Java): `slf4j` 是 `Ceki Gülcü` 开发的 `Log Facade`，主要是因为`Ceki Gülcü` 觉得作为日志统一接口的 `JCL` 设计的不合理：<br/>

下面这种写法不管是否输出 `debug` 级别的时候都需要做一次字符串拼接，如果这种代码被反复调用就会产生很多无用的字符串拼接，影响性能。

```Java
logger.debug("log:" + log);
```

而官方给出的最佳时间方式是这样的：

```Java
if (logger.isDebugEnabled()) {
  logger.debug("log:" + log);
}
```

怎么看都是反人类的设计，所以在 `slf4j` 中，设计的api 是这样的
```Java
logger.debug("log:{}", log);
```

`logback`: `logback` 也是`Ceki Gülcü` 开发的日志实现，在 `log4j` 的基础上进行了改进，提供了更好的性能实现，异步logger，Filter 等更能多的特性。

`Ceki Gülcü` 给我们开发了很好用的日志组件，但是现在有了两个 `Log Facade` 和三个流行的 `Log Implementation`，事情变的复杂了；`Ceki Gülcü` 作为一个完美主义者，为了我们能在不同的log 之间自由切换，他又开发了各种 `Adapater` 和 `Bridge` 来连接，这里盗用一张 `slf4j` 官网的图片

![图片](/assets/picture/slf4j_over.png "slf4j 桥接其他日志api关系图")

`log4j2`: `log4j2` 的开发维护人员不想看着 `log4j` 被 `slf4j/logback` 所取代，在设计上很大程度的模仿了 `slf4j/logback`，完全脱离`log4j1.x`，在性能上实现了很大的提升，作为一个高仿品这里不多介绍。


### `Tomcat` 的日志实现方法

`Tomcat` 内部整合的日志模块是 `JULI`，`JULI`是从 `JCL` fork 过来的一个重命名分支，默认被硬编码使用 `JUL` 作为日志实现，从而保证 `Tomcat` 本身的日志和业务日志实现完美隔离。
而`Tomcat`的日志的配置文件默认位置是 `${catalina.base}/conf/logging.properties`，如果无法读取或不存在的时候，就会去找`${java.home}/lib/logging.properties`；在web应用的范围内也有一个日志配置文件 `WEB-INF/classes/logging.properties`

```properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

## 全局申明，tomcat 可以使用的 Handler
handlers = 1catalina.org.apache.juli.FileHandler, 2localhost.org.apache.juli.FileHandler, 3manager.org.apache.juli.FileHandler, 4host-manager.org.apache.juli.FileHandler, java.util.logging.ConsoleHandler

## 在
.handlers = 1catalina.org.apache.juli.FileHandler, java.util.logging.ConsoleHandler

############################################################
# Handler specific properties.
# Describes specific configuration info for Handlers.
############################################################

## catalina.out catalina.yyyy-MM-dd.log 日志的级别、日志文件位置、日志文件名称前缀配置
1catalina.org.apache.juli.FileHandler.level = FINE
1catalina.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.FileHandler.prefix = catalina.

## localhost.yyyy-MM-dd.log 日志的级别、日志文件位置、日志文件名称前缀配置
2localhost.org.apache.juli.FileHandler.level = FINE
2localhost.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
2localhost.org.apache.juli.FileHandler.prefix = localhost.

## manager.yyyy-MM-dd.log 日志的级别、日志文件位置、日志文件名称前缀配置
3manager.org.apache.juli.FileHandler.level = FINE
3manager.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
3manager.org.apache.juli.FileHandler.prefix = manager.

## host-manager.yyyy-MM-dd.log 日志的级别、日志文件位置、日志文件名称前缀配置
4host-manager.org.apache.juli.FileHandler.level = FINE
4host-manager.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
4host-manager.org.apache.juli.FileHandler.prefix = host-manager.

## console 日志级别及格式设置
java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter


############################################################
# Facility specific properties.
# Provides extra control for each logger.
############################################################

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].handlers = 2localhost.org.apache.juli.FileHandler

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].handlers = 3manager.org.apache.juli.FileHandler

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].handlers = 4host-manager.org.apache.juli.FileHandler

# For example, set the org.apache.catalina.util.LifecycleBase logger to log
# each component that extends LifecycleBase changing state:
#org.apache.catalina.util.LifecycleBase.level = FINE

# To see debug messages in TldLocationsCache, uncomment the following line:
#org.apache.jasper.compiler.TldLocationsCache.level = FINE
```
