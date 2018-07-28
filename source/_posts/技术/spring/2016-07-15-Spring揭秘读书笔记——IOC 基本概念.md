---
layout: post
title: Spring揭秘读书笔记 —— IOC 基本概念
category: 技术
tags: Spring
keywords: Spring揭秘
description:
---

### 前言
一直在使用Spring 这一开源项目， 但是在学习spring 的过程中一直没遇到让我心旷神怡的好书，看过 `《Spring技术内幕：深入解析Spring架构与设计原理（第2版）》`， 从源码角度讲解spring， 虽然也很不错， 但是略感枯燥。  另外就是开涛大神的 `跟我学Spring`(http://jinnianshilongnian.iteye.com/blog/1482071) 和 `跟我学Spring MVC`(http://jinnianshilongnian.iteye.com/blog/1617451), 这两个系列的博客，主要是从使用Spring 的角度出发，很适合初学者系列的学习Spring 的使用以及一些Spring 的原理。

###### 以上对两位大神的著作的评论仅属个人言论，欢迎大家指正

最近接触到一本比较 `古老` 的 Spring 学习书籍 ———— `《Spring揭秘》` , 这本书貌似现在已经停刊了，讲的 Spring 也是将的 `Spring 2.X` , 但是 Spring 的主要思想，在这本书中被作者用一种通俗易懂的语言表达的让人能一边看，一边笑着点头， 甚是舒畅。

----------

## IoC的基本概念读书笔记
#### IoC [控制反转] ———— 我们的理念是：让别人为你服务

书中的比喻很形象
![](/assets/picture/iocmetaphor.png "IoC形象比喻")

常见的 Ioc 实现方法
1. 构造方法注入
2. setter方法注入
3. 接口注入

三种方法中 `接口注入` 较为难理解
被注入的对象要想 Ioc Service Provider 为其注入依赖对象， 就要实现一个特定接口，特定的接口提供一个方法，用来为其注入一个依赖对象，这个特性接口就如同是上图比喻中的 "拿衣服的女朋友"

示例：
![](/assets/picture/ioc.interfaceinsert.demo.png)
FxNewsProvider 希望能被注入依赖 IFXNewsListener, 使用接口注入时，实现 FXNewsListenerCallable 接口  FXNewListenerCallable 接口提供了 injectNewsListener 方法, 这个方法的参数的类型就是 IFXNewsListenr

`接口注入`这种注入方式目前已经过时，不提倡使用


`构造方法注入` 在对象构造完之后，就会立即进入就绪状态，可以马上使用。但是如果依赖的对象比较多，构造方法的参数列表会比较长，而且`构造方法注入`底层实现还是基于反射机制，而反射机制对于构造方法中相同类型的参数处理会有困难；而且构造方法不能被继承，不能设置默认值


`setter方法注入` setter 方法参数单一，反射机制可以很好的支持，而且setter 方法能被继承，能够设置默认值
