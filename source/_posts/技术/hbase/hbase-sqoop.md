---
layout: post
title: HBase 简介和使用 Sqoop 同步 Mysql 数据到 HBase
categories: HBase
tags: [HBase, Sqoop]
date: 2018-08-26 12:00:00
description: 了解HBase 基本操作，实践从 Mysql 同步数据到HBase
---

### `HBase` 数据模型

#### `Namespace`: 命名空间
类似于关系型数据库中的 `database schema`

#### `Table`: 表
一个 `Namespace` 下有多个表，一个表可以包含多个行

#### `Row`: 行
在 `HBase` 中 `Row` 由一个 `Row Key` 和一个或多个列及其值组成，数据值的存储按照 `Row Key` 的字典顺序存储的。

#### `Column`: 列
在 `HBase` 中， 每个列有它所属的 `Column Family(列簇)`， 以及`Column Qualifier(列修饰符)`, 列名组成是 `Column Family:Column Qualifier`

#### `Column Family`: 列簇
在 `HBase` 中将列进行分类，每个列都有它所属的`列簇`，`列簇` 把列和相应的值物理上联合在一起。创建表的时候，必须指定至少一个 `列簇`。每个列出是一个存储属性的集合，

#### `Column Qualifier`: 列修饰符
`列簇` 和 `列修饰符` 才是实际意义上的列唯一标识，假设存在 `列簇` content, 可以存在 `列修饰符` xml, 组成一个唯一的列标识 `content:xml`；创建表的时候，`列簇` 已经被指定了，但是 `列修饰符` 是可变的，可以再 `put` 指令中随意指定属于 `列簇` 的 `列修饰符`。

#### `Cell`
一个Cell是行，列簇和列修饰符的组合，并且包含一个值和时间戳，时间戳代表着值的版本。

#### `Timestamp`（时间戳）
一个时间戳是连同值一起被写入的，是值版本的唯一标识，默认情况下，时间戳表示数据写入时RegionServer的时间，但是当你在写数据到Cell的时候，你可以指定一个不同的时间戳。

### `HBase` 常用指令

```shell
> create_namespace 'n1' //  创建一个 namespace n1
> list_namespace        //  列出所有的 namespace
> create 'n1:t1', 'CF1', 'CF2' // 创建表 t1
> list_namespace_tables 'n1'   // 列出 namespace n1 下的所有 table
> describe 'n1:t1'      //  查看表结构
> put 'n1:t1', 'rk', 'CF1:name', 'test' // 往表 n1:t1 中 row key 是 rk 的行中插入列名称是 CF1:name 的值 'test'
> get 'n1:t1', 'rk' // 获取表 'n1:t1' 中 row key 是 rk 的所有数据
> scan 'n1:t1'      // 模糊查看表 n1:t1
> scan 'n1:t1', FILTER=>"ColumnPrefixFilter('name') AND ValueFilter(=,'substring:test')"  // 模糊查询，列修饰符前缀为name 且值中包含字段 test 的数据
> delete 'n1:t1', 'rk', 'CF1:name' // 删除 row key 是 rk 列 `CF1:name` 的数据
> disable 'n1:t1'      // 禁用表 n1:t1，被被删除之前必须先被禁用
> is_enabled 'n1:t1'   // 查看表 n1:t1 是否可用
> is_disabled 'n1:t1'  // 查看表 n1:t1 是否被禁用
> enable 'n1:t1'       // 启用表 n1:t1
> drop 'n1:t1'         // 删除表 n1:t1，注意：只能删除被禁用的表
> drop_namespace 'n1'  // 删除命名空间 n1，注意：只能删除没有表的 namespace
```

### `sqoop` 导出 `mysql` 数据到 `HBase`

```shell
export HADOOP_CLASSPATH=/absolute/path/to/mysql-connector-java-5.1.15.jar  
sqoop import --connect jdbc:mysql://ip:port/database_name --username 'username' --password 'password' --table 'table_name' --columns "id,name,code,description" --hbase-table 'test:hbase_table_name' --hbase-create-table --hbase-row-key 'id,code' --column-family info
```

上述命令行解析

1. 设置 `HADOOP_CLASSPATH`<br/>
首先需要设置 `HADOOP_CLASSPATH`，值是 `mysql-connector-java-5.1.15.jar` 的绝对路径，否则会报错：`java.lang.RuntimeException: Could not load db driver class: com.mysql.jdbc.Driver`

2. `--connect`<br/>
连接数据库的url，从这个数据库中导出数据

3. `--username`<br/>
数据库用户名

4. `--password`<br/>
数据库密码

5. `--table`<br/>
导出数据的源数据库表

6. `--columns`<br/>
本次导出的数据，可以一次导出多列，用逗号分隔，导出的列在hbase 中属于 `--column family` 参数指定的列簇，列名称是  `column family:mysql表中的列名`，需要注意的是，如果没有指定参数 `--hbase-row-key`，在hbase 表中的row key 将是 `--columns` 中第一列。

7. `--hbase-table`<br/>
本次导入数据的 hbase 表，需要注意的是导入数据的hbase 表可以不存在，但是hbase 表所属的 namespace 必须是存在的，否则会报错：<br/>
        Import failed: org.apache.hadoop.hbase.NamespaceNotFoundException: org.apache.hadoop.hbase.NamespaceNotFoundException: 'namespace'

8. `--hbase-create-table`<br/>
如果导入数据的表不存在，则创建该表

9. `--hbase-row-key`<br/>
设置 hbase 中的 `Row Key`，参数值是mysql 表中的列名，可以设置多个列合并成 `Row Key`, 用逗号分隔

10. `--column-family`<br/>
指定导入数据所属的列簇，每次导入数据只能导入属于同一个`列簇` 的数据，如果 mysql 表中数据属于多个 `列簇`，只能通过多条指令分批导入。

***注意：上述指令没有指定列分隔符和行分隔符，默认的列分隔符是 `'\001'`，在less 中显示是 `^A`；默认的行分隔符是 `'\n'`。***
