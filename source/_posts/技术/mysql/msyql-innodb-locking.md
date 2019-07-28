---
layout: post
title: mysql InnoDB 锁
categories: mysql
tags: mysql
date: 2019-07-01 22:00:00
description: mysql InnoDB 存储引擎中的锁
---

## InnoDB 支持的锁

1. 共享锁(S)和排它锁(X)
2. 意向锁
3. 行锁
4. 间隙锁（GAP 锁）
5. Next-Key Locks
6. 插入意向锁
7. 自增长字段锁
8. Predicate Locks for Spatial Indexes


## 行级锁

行级锁，实质上是锁定在一个索引记录上，`select ... for update` 会加锁, `delete`, `update` 也会加锁


## GAP 锁

间隙锁也是作用在索引记录上，但是不同的是， GAP 锁锁定的是两条索引记录之间的间隙，而没有锁定索引记录本身

## Next-Key 锁

`Next-Key 锁`是行级锁和 GAP 锁的集合，它会锁定命中的索引记录以及这个索引记录前的间隙。

