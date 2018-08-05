---
layout: post
title: mysql explain 介绍
categories: mysql
tags: mysql
date: 2016-09-08 21:00:00
description: mysql explain 参数说明
---

mysql 为我们提供了一个很好用的 sql 语句分析工具 `explain` , 它可以帮助我们选择更好的索引和写出更优化的查询语句。

`explain` 会显示sql 语句如何使用索引

###### 注意：在 `mysql 5.6` 之前 `explain` 只能分析 `select` 语句，其他类型的语句只能先改写为 `select` 语句再分析； 但是 `mysql 5.6` 及以后， `explain` 支持其他语句的解析！！！

先来个使用范例

    exlain select sql
    如：
    explain select * from grade_info where teacher_id = '1';


下面看看 `explain` 的执行结果：
![图片](/assets/picture/mysql_index_a.png "使用索引第一列的情况")
`explain` 的结果有这些列<br />
 `id`, `select_type`, `table`, `partitions`, `type`, `possible_keys`, `key`, `key_len`, `ref`, `rows`, `filtered`, `Extra`

#### id
sql 语句执行顺序号， 就是sql 语句的执行顺序
![图片](/assets/picture/mysql_nested_sql_explain.png "mysql嵌套语句")
这里可以看到 id 的变化

#### select_type
