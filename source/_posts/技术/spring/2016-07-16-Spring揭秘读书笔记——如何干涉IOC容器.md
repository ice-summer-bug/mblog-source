---
layout: post
title: Spring揭秘读书笔记 —— 如何干涉IOC容器
categories: Spring
tags: [Spring,《Spring揭秘》]
date: 2016-07-16 21:00:00
description: 《Spring揭秘》学习系列，IOC 如期提供的干涉方式
---

### 前言

Spring IOC 容器在加载 Bean 的时候，可以大致分为两个阶段
1. 容器启动阶段
2. Bean 实例化阶段

### 插手容器启动阶段

容器启动阶段简而言之就是将 通过注解或者在 xml 文件中配置信息，解析转化为 `BeanDefinition`， 再通过 `BeanDefinitionRegister` 注册到容器中
这个阶段主要是一些收集准备工作

`Spring` 提供给我们插手这一阶段的方式是： 实现 `BeanFactoryPostProcessor`
