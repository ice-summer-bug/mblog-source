---
layout: post
title: Spring Boot auto-configuraion 浅析
categories: Spring
tags: [Spring, Spring Boot]
date: 2020-04-20 21:00:00
description: Spring Boot auto-configuraion 浅析
---

# Sprint Boot Auto-configuration 浅析

`Spring Boot` 的自动配置是结合各种 `starter` 一起使用的


### `Conditional` 注解

#### Class 相关的 `Conditional` 注解

- `@ConditionalOnClass`: `classpath` 中存在指定的 `Class`，可以指定多个 `Class`, 也可以通过类的全限定名指定
- `@ConditionalOnMissingClass`: `classpath` 中不存在指定的 `Class`，可以指定多个 `Class`, 也可以通过类的全限定名指定

#### Bean 相关的 `Conditional` 注解

- `@ConditionalOnBean`: 当前 Spring 上下文中存在指定名称