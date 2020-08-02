---
layout: post
title: Mysql InnoDDB 的 undo log 和 mvcc
categories: mysql
tags: [mysql,InnoDB]
date: 2020-07-01 22:00:00
description: Mysql InnoDDB 的 undo log 和 mvcc
---


# Mysql InnoDB  的事务

# 事务隔离级别



# undo log

InnoDB  的行记录中有三个隐藏字段，
DB_TRX_ID: 记录最近更新这条信息的事务 ID，字段长度 6字节
DB_ROLL_PTR: 回滚指针，指向保存在 `rollback_segment` 中的 `undo log`, 长度 7 个字节
DB_ROW_ID: 行ID，如果 InnoDB 中没有自增长的聚簇索引，这个值会被认位是行的唯一标识存储在索引中

