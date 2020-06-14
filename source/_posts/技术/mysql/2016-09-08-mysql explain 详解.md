---
layout: post
title: mysql explain 介绍
categories: mysql
tags: mysql
date: 2016-09-08 21:00:00
description: mysql explain 指令说明
---

## `explain` 指令

在日常使用 mysql 的过程中，我们需要查看表结构设计，或者查看sql 执行计划；mysql 提供了一个指令 ———— `explain`，它有以下功能：

                1. 查看表结构信息
                2. 查看sql 执行计划信息 
                3. mysql 8.18 版本以上，还支持 [explain analyze]


###### 注意：<br/> 1. 在 `mysql 5.6` 之前 `explain` 只能分析 `select` 语句，其他类型的语句只能先改写为 `select` 语句再分析； 但是 `mysql 5.6` 及以后， `explain` 支持其他语句的解析 <br/> 2. `explain` 指令还有一个同名的指令 `describe`



### `explain` 查看表结构

`explain` 和 `describe` 都能查看表结构信息，实例如下

                mysql> explain TABLES;
                +-----------------+--------------------------------------------------------------------+------+-----+---------+-------+
                | Field           | Type                                                               | Null | Key | Default | Extra |
                +-----------------+--------------------------------------------------------------------+------+-----+---------+-------+
                | TABLE_CATALOG   | varchar(64)                                                        | YES  |     | NULL    |       |
                | TABLE_SCHEMA    | varchar(64)                                                        | YES  |     | NULL    |       |
                | TABLE_NAME      | varchar(64)                                                        | YES  |     | NULL    |       |
                | TABLE_TYPE      | enum('BASE TABLE','VIEW','SYSTEM VIEW')                            | NO   |     | NULL    |       |
                | ENGINE          | varchar(64)                                                        | YES  |     | NULL    |       |
                | VERSION         | int(2)                                                             | YES  |     | NULL    |       |
                | ROW_FORMAT      | enum('Fixed','Dynamic','Compressed','Redundant','Compact','Paged') | YES  |     | NULL    |       |
                | TABLE_ROWS      | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | AVG_ROW_LENGTH  | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | DATA_LENGTH     | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | MAX_DATA_LENGTH | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | INDEX_LENGTH    | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | DATA_FREE       | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | AUTO_INCREMENT  | bigint(21) unsigned                                                | YES  |     | NULL    |       |
                | CREATE_TIME     | timestamp                                                          | NO   |     | NULL    |       |
                | UPDATE_TIME     | datetime                                                           | YES  |     | NULL    |       |
                | CHECK_TIME      | datetime                                                           | YES  |     | NULL    |       |
                | TABLE_COLLATION | varchar(64)                                                        | YES  |     | NULL    |       |
                | CHECKSUM        | bigint(21)                                                         | YES  |     | NULL    |       |
                | CREATE_OPTIONS  | varchar(256)                                                       | YES  |     | NULL    |       |
                | TABLE_COMMENT   | text                                                               | YES  |     | NULL    |       |
                +-----------------+--------------------------------------------------------------------+------+-----+---------+-------+
                21 rows in set (0.00 sec)


`explain table_name` 指令可以说是 `show columns from table_name` 指令的缩写，可以查看表结构中的所有的列信息；

查看表结构信息我们还可以使用以下指令 

`show create table table_name`: 查看建表语句
`show table status [{in | from} db_name] [like 'pattern' | where expr]`: 查看数据库中的数据表概要信息，具体见[文档](https://dev.mysql.com/doc/refman/8.0/en/show-table-status.html "show table status")
`show index [{in | from} table_name]`: 查看表结构中的索引信息，具体见[文档](https://dev.mysql.com/doc/refman/8.0/en/show-index.html "show index")


### `explain` 查看执行计划情况

先来个使用范例

    exlain select sql
    如：
    explain select * from grade_info where teacher_id = '1';


下面看看 `explain` 的执行结果：
![图片](/assets/picture/mysql_index_a.png "使用索引第一列的情况")
`explain` 的结果有这些列<br />
 `id`, `select_type`, `table`, `partitions`, `type`, `possible_keys`, `key`, `key_len`, `ref`, `rows`, `filtered`, `Extra`

#### `explain` 查看执行计划输出字段概要说明

| 列名 | 说明 |
|-|-|
| id | sql 语句中多个嵌套语句的唯一标识，如果只有一条 sql 始终为 1|
| select_type | select 语句的类型，或其他语句的类型（较高版本中 explain 支持 delete,update,insert 等语句） | 
| table | 表名称，或者 sql 中表的别名 |
| partitions | 匹配的分区 |
| type | 连接类型 |
| possible_keys | 可能被用到的索引名称 |
| key | 实际被使用的索引的名称 |
| key_len | 被选用的索引实际被使用的长度 |  
| ref | 和索引比较的列，todo |
| rows | 将要被检查的行数的估计值 |
| filtered | 被“表”条件过滤的行数的百分比 |
| Extra | 附加信息 |

#### id
sql 语句执行顺序号， 就是sql 语句的执行顺序
![图片](/assets/picture/mysql_nested_sql_explain.png "mysql嵌套语句")
这里可以看到 id 的变化

#### select_type

select 语句的类型

***注意：非 `select` 语句的 `select_type` 就是该语句的类型，如 `delete` 语句的 `select_type` 就是 `delete` ***

|取值|说明|
|-|-|
| SIMPLE | 简单的查询语句（没有子查询，也没有 UNION） |
| PRIMARY | 在外层的select 语句 |
| UNION | UNION 查询中第二个或者第二个之后的 select 语句 |
| DEPENDENT UNION | UNION 查询中第二个或者第二个之后的, 并且依赖外部查询语句的 select 语句 |
| UNION RESULT | UNION 查询的结果 | 
| SUBQUERY | 子查询中的第一个 select 语句 |
| DEPENDENT SUBQUERY | 子查询中的第一个，依赖外部查询语句的 select 语句 |
| DERIVED | 派生表查询 |
| DEPENDENT DERIVED	| 依赖其他表的派生表查询 |
| MATERIALIZED ||
| UNCACHEABLE SUBQUERY | 查询结果不能被缓存，每一行查询结果都必须被外部查询语句评估的子查询语句 |
| UNCACHEABLE UNION	| UNION 查询中第二个或者第二个之后的属于 `UNCACHEABLE SUBQUERY` 类型的子查询语句 |


##### SIMPLE 没有UNION 也没有子查询的简单查询语句

    mysql> explain select * from Employee;
    +----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
    | id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
    +----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
    |  1 | SIMPLE      | Employee | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL  |
    +----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

##### PRIMARY 有子查询或者 UNION查询中最外层的查询语句

    mysql> explain select * from Employee where id = (select id from Employee where id = 5);
    +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    | id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    |  1 | PRIMARY     | Employee | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL        |
    |  2 | SUBQUERY    | Employee | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
    +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    2 rows in set, 1 warning (0.00 sec)

    mysql> explain select id from Employee where id = 5 union all select id from Employee2;
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    | id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    |  1 | PRIMARY     | Employee  | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
    |  2 | UNION       | Employee2 | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    7 |   100.00 | NULL        |
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    2 rows in set, 1 warning (0.00 sec)

##### UNION: UNION查询中第二个或者第二个之后的 select 语句 

    mysql> explain select id from Employee where id = 5 union all select id from Employee2 union all select id from Employee2;
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    | id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    |  1 | PRIMARY     | Employee  | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
    |  2 | UNION       | Employee2 | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    7 |   100.00 | NULL        |
    |  3 | UNION       | Employee2 | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    7 |   100.00 | NULL        |
    +----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    3 rows in set, 1 warning (0.00 sec)

##### DEPENDENT UNION: UNION 查询中第二个或者第二个之后的, 并且依赖外部查询语句的 select 语句

***DEPENDENT UNION 是什么意思？？？***


    mysql> explain select * from Employee where id in (select id from Employee where id = 5 union all select id from Employee2 union all select id from Employee2);
    +----+--------------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    | id | select_type        | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
    +----+--------------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    |  1 | PRIMARY            | Employee  | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    6 |   100.00 | Using where |
    |  2 | DEPENDENT SUBQUERY | Employee  | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
    |  3 | DEPENDENT UNION    | Employee2 | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    7 |    14.29 | Using where |
    |  4 | DEPENDENT UNION    | Employee2 | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    7 |    14.29 | Using where |
    +----+--------------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    4 rows in set, 1 warning (0.00 sec)

##### UNION RESULT: UNION 查询结果

    mysql> explain select * from Employee union select * from Employee2;
    +------+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
    |  id  | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
    +------+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
    |  1   | PRIMARY      | Employee   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL            |
    |  2   | UNION        | Employee2  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    7 |   100.00 | NULL            |
    | NULL | UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
    +------+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
    3 rows in set, 1 warning (0.01 sec)


##### SUBQUERY: 子查询中第一个 select 语句

    mysql> explain select * from Employee where id = (select id from Employee where id = 5);
    +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    | id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    |  1 | PRIMARY     | Employee | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL        |
    |  2 | SUBQUERY    | Employee | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
    +----+-------------+----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    2 rows in set, 1 warning (0.00 sec)

##### DEPENDENT SUBQUERY: 引用了外部查询结果或者变量的子查询

    mysql> explain select * from Employee where id in (select id from Employee where id = 5 union all select id from Employee2);
    +----+--------------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    | id | select_type        | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
    +----+--------------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    |  1 | PRIMARY            | Employee  | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    6 |   100.00 | Using where |
    |  2 | DEPENDENT SUBQUERY | Employee  | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
    |  3 | DEPENDENT UNION    | Employee2 | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    7 |    14.29 | Using where |
    +----+--------------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
    3 rows in set, 1 warning (0.00 sec)
    
关联子查询除了上面举的例子之外，`having`, `group by` 等其他语句引用外部查询结果或者变量的时候也会成为关联子查询

***`DEPENDENT SUBQUERY` 是低效的，如果能转化为 join 语句，效率会好很多***

##### DERIVED: 查询过程中的衍生表查询

    mysql> explain select * from Employee e join  (select id from Employee union select id from Employee2) a on e.id = a.id;
    +------+--------------+------------+------------+-------+---------------+-------------+---------+-----------+------+----------+-----------------+
    |  id  | select_type  | table      | partitions | type  | possible_keys | key         | key_len | ref       | rows | filtered | Extra           |
    +------+--------------+------------+------------+-------+---------------+-------------+---------+-----------+------+----------+-----------------+
    |   1  | PRIMARY      | e          | NULL       | ALL   | PRIMARY       | NULL        | NULL    | NULL      |    6 |   100.00 | NULL            |
    |   1  | PRIMARY      | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 4       | test.e.Id |    2 |   100.00 | Using index     |
    |   2  | DERIVED      | Employee   | NULL       | index | NULL          | PRIMARY     | 4       | NULL      |    6 |   100.00 | Using index     |
    |   3  | UNION        | Employee2  | NULL       | ALL   | NULL          | NULL        | NULL    | NULL      |    7 |   100.00 | NULL            |
    | NULL | UNION RESULT | <union2,3> | NULL       | ALL   | NULL          | NULL        | NULL    | NULL      | NULL |     NULL | Using temporary |
    +------+--------------+------------+------------+-------+---------------+-------------+---------+-----------+------+----------+-----------------+
    5 rows in set, 1 warning (0.00 sec)

##### DEPENDENT DERIVED: 依赖外部查询的衍生表查询

##### MATERIALIZED: 物化子查询

通过在内存中用临时表去缓存子查询结果来优化子查询结果来优化子查询效率，但当临时表过大的时候也会落盘到磁盘中；物化子查询通过临时表避免子查询每次都被触发，最好只被触发一次；

物化子查询的开关是可以通过变量设置的，变量 `optimizer_switch` 中的 `materialization`

    mysql> SELECT @@optimizer_switch\G
    *************************** 1. row ***************************
    @@optimizer_switch: index_merge=on,index_merge_union=on,
                        index_merge_sort_union=on,index_merge_intersection=on,
                        engine_condition_pushdown=on,index_condition_pushdown=on,
                        mrr=on,mrr_cost_based=on,block_nested_loop=on,
                        batched_key_access=off,materialization=on,semijoin=on,
                        loosescan=on,firstmatch=on,duplicateweedout=on,
                        subquery_materialization_cost_based=on,
                        use_index_extensions=on,condition_fanout_filter=on,
                        derived_merge=on,use_invisible_indexes=off,skip_scan=on,
                        hash_join=on

物化子查询触发场景有：

1. 如下形式sql 中，外部查询中的 `oe_N` 不为空或者内部子查询 `ie_N` 不为空，`N` 大于等于1
```sql
(oe_1, oe_2, ..., oe_N) [NOT] IN (SELECT ie_1, i_2, ..., ie_N ...)
```
2. 如下形式sql 中，外部查询中只有单个表达式 `oe`，内存子查询也只有一个表达式 `ie`，表达式可以为空
```sql
oe [NOT] IN (SELECT ie ...)
```
3. 查询结果为`UNKNOWN(NULL)` 表示 `false` 含义的`IN` 或 `NOT IN` 语句


***注意：物化子查询存在使用限制***
***1. 物化子查询必须是子查询结果和外部查询结果的变量类型是一致的，都是Integer 或者都是Long 或者其他类型***
***2. 物化子查询不支持查询结果类型是 BLOB 类型的子查询***


##### UNCACHEABLE SUBQUERY: 不可缓存的子查询，每一层外部查询都会触发一次子查询

##### UNCACHEABLE UNION: 不可缓存的 UNION 查询，每一层外部查询都会触发一次子查询


#### `table`: 查询的表名

1. 查询的表名
2. 查询中指定的表的别名
3. <derived N> 衍生临时表的名称，id 为 N 的表的部分数据的衍生临时表
4. <union M,N> id 为M，N 的表的连接查询结果
5. <subquery N> 物化子查询 N 的引用

    mysql> explain select e.id from Employee e join (select id from Employee union select id from Employee2) a on e.id = a.id;
    +----+--------------+------------+------------+-------+---------------+-------------+---------+-----------+------+----------+-----------------+
    | id | select_type  | table      | partitions | type  | possible_keys | key         | key_len | ref       | rows | filtered | Extra           |
    +----+--------------+------------+------------+-------+---------------+-------------+---------+-----------+------+----------+-----------------+
    |  1 | PRIMARY      | e          | NULL       | index | PRIMARY       | PRIMARY     | 4       | NULL      |    6 |   100.00 | Using index     |
    |  1 | PRIMARY      | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 4       | test.e.Id |    2 |   100.00 | Using index     |
    |  2 | DERIVED      | Employee   | NULL       | index | NULL          | PRIMARY     | 4       | NULL      |    6 |   100.00 | Using index     |
    |  3 | UNION        | Employee2  | NULL       | ALL   | NULL          | NULL        | NULL    | NULL      |    7 |   100.00 | NULL            |
    | NULL | UNION RESULT | <union2,3> | NULL       | ALL   | NULL          | NULL        | NULL    | NULL      | NULL |     NULL | Using temporary |
    +----+--------------+------------+------------+-------+---------------+-------------+---------+-----------+------+----------+-----------------+
    5 rows in set, 1 warning (0.00 sec)

    mysql> explain select * from Employee e1 where e1.id = any (select e2.id from Employee2 e2 where e2.Salary = e1.Salary);
    +----+--------------+-------------+------------+--------+---------------+------------+---------+---------------------------+------+----------+-------------+
    | id | select_type  | table       | partitions | type   | possible_keys | key        | key_len | ref                       | rows | filtered | Extra       |
    +----+--------------+-------------+------------+--------+---------------+------------+---------+---------------------------+------+----------+-------------+
    |  1 | SIMPLE       | e1          | NULL       | ALL    | PRIMARY       | NULL       | NULL    | NULL                      |    6 |   100.00 | Using where |
    |  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_key>    | <auto_key> | 12      | test.e1.Id,test.e1.Salary |    1 |   100.00 | NULL        |
    |  2 | MATERIALIZED | e2          | NULL       | ALL    | NULL          | NULL       | NULL    | NULL                      |    7 |   100.00 | NULL        |
    +----+--------------+-------------+------------+--------+---------------+------------+---------+---------------------------+------+----------+-------------+
    3 rows in set, 2 warnings (0.00 sec)

#### `partition` 分区信息（暂不介绍）

####  `type`: 连接类型

##### `system`: 查询的表只有一行数据，是 `const` 连接类型的一种特殊形式

##### `const`: 查询条件完全匹配唯一索引或者主键索引，且只有最多一行符合查询条件的数据

因为只有一行数据符合查询条件，所以后续的查询优化器可以把这个查询条件看作是常量的比较查询

##### `eq_ref`: 当前表和前一个表关联查询的时候，查询条件完全匹配不为空的唯一索引或者主键索引，且每个关联条件只有一行符合查询条件的数据

***注意: `eq_ref` 和 `const` 有一点不一样，前者要求匹配的唯一索引是不允许为空的唯一索引，而后者没这个限制***

`eq_ref` 是当前表和前一个表的关联查询，两者的关联查询支持 `=` 操作符；也可以说当前表和常量的 `=` 比较查询，示例如下：

```sql
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```

具体示例

```sql
CREATE TABLE `student` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(64) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '名称',
  `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='学生信息表';

CREATE TABLE `student_info` (
  `student_id` int(11) NOT NULL,
  `email` varchar(64) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT 'email',
  `phone` varchar(64) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '电话',
  PRIMARY KEY (`student_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='学生联系信息表';

CREATE TABLE `grade` (
  `student_id` int(11) NOT NULL,
  `course_id` int(11) NOT NULL DEFAULT '0',
  `grade` int(11) NOT NULL DEFAULT '0' COMMENT '成绩',
  UNIQUE KEY `student_course` (`student_id`,`course_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='学生成绩信息表'
```

    # grade.course_id = 1 且 grade.student_id = student.id 这个匹配条件下 每个<student_id, course_id(1)> 都只会查询到一个 grade 信息
    mysql> explain select * from grade g, student s where g.student_id = s.id and g.course_id = 1;
    +----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
    | id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref             | rows | filtered | Extra |
    +----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
    |  1 | SIMPLE      | s     | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL            |    2 |   100.00 | NULL  |
    |  1 | SIMPLE      | g     | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | test.s.id,const |    1 |   100.00 | NULL  |
    +----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+

    mysql> explain select s.id, si.student_id from student s,student_info si where s.id = si.student_id;
    +----+-------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------------+
    | id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref       | rows | filtered | Extra       |
    +----+-------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------------+
    | 1  | SIMPLE      | s     | <null>     | index  | PRIMARY       | PRIMARY | 4       | <null>    | 2    | 100.0    | Using index |
    | 1  | SIMPLE      | si    | <null>     | eq_ref | PRIMARY       | PRIMARY | 4       | test.s.id | 1    | 100.0    | Using index |
    +----+-------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------------+
    2 rows in set, 1 warning (0.00 sec)


##### `ref`: 通过 `=`, `>=`, `<=` 比较符号匹配非 `唯一索引` 和 `主键索引` 或者其他类型索引的左前缀查询到一行以上的数据，索引匹配是还可以是部分匹配

示例：建表语句见上一小节

    mysql> explain select * from grade g, student s where g.student_id = s.id;
    +----+-------------+-------+------------+------+---------------+---------+---------+-----------+------+----------+-------+
    | id | select_type | table | partitions | type | possible_keys | key     | key_len | ref       | rows | filtered | Extra |
    +----+-------------+-------+------------+------+---------------+---------+---------+-----------+------+----------+-------+
    |  1 | SIMPLE      | s     | NULL       | ALL  | PRIMARY       | NULL    | NULL    | NULL      |    2 |   100.00 | NULL  |
    |  1 | SIMPLE      | g     | NULL       | ref  | PRIMARY       | PRIMARY | 4       | test.s.id |    3 |   100.00 | NULL  |
    +----+-------------+-------+------------+------+---------------+---------+---------+-----------+------+----------+-------+
    2 rows in set, 1 warning (0.00 sec)


##### `fulltext`: 全文索引的连接方式

##### `ref_or_null`: 和 `ref` 高度相似，只是会在匹配的时候判断索引列是否为空

实例如下：

```sql
 CREATE TABLE `employee` (
  `Id` int(10) NOT NULL,
  `Salary` bigint(20) NOT NULL DEFAULT '0',
  `type` tinyint(4) DEFAULT '0' COMMENT '类型',
  `company_id` int(10) DEFAULT '0' COMMENT '公司id',
  PRIMARY KEY (`Id`),
  UNIQUE KEY `company_type` (`company_id`,`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```

    mysql> explain select * from employee where company_id = 1 or company_id is null;
    +----+-------------+----------+------------+-------------+---------------+--------------+---------+-------+------+----------+-----------------------+
    | id | select_type | table    | partitions | type        | possible_keys | key          | key_len | ref   | rows | filtered | Extra                 |
    +----+-------------+----------+------------+-------------+---------------+--------------+---------+-------+------+----------+-----------------------+
    |  1 | SIMPLE      | employee | NULL       | ref_or_null | company_type  | company_type | 5       | const |    3 |   100.00 | Using index condition |
    +----+-------------+----------+------------+-------------+---------------+--------------+---------+-------+------+----------+-----------------------+
    1 row in set, 1 warning (0.00 sec)


关联知识点：[Section 8.2.1.15, “IS NULL Optimization”.](https://dev.mysql.com/doc/refman/8.0/en/is-null-optimization.html "IS NULL Optimization")

##### `index_merge`: 索引合并优化

索引合并优化将单个表的多个范围查询合并到一个查询结果中，不支持跨表合并；合并可以是交集(`intersection`)、并集(`union`)或者交集的并集`(unions of intersection)`；

    索引合并优化存在如下限制：
    1. 在复查子查询中如果有深度嵌套的 AND/OR 语句，mysql 可能不会选择最优的优化算法，可以用如下方式拆分复杂语句
    
        (x AND y) OR z => (x OR z) AND (y OR z)
        (x OR y) AND z => (x AND z) OR (y AND z)
    
    2. 索引合并优化不能作用于全文索引(fulltext-indexes)

索引合并优化在 `explain` 指令输出列 `extra` 中会展示 

1. 并集 union(key1, key2)
2. 交集 intersection(key1, key2)

###### 交集访问算法

1. 对于包含 N 列的组合索引，需要用 and 连接 N 个索引列和常数进行比较

```sql
key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
```

2. InnoDB 表主键的范围查询

index merge 交集访问算法会在使用多个索引同时进行读取扫描，并产生这些被读取数据的交集

如果被查询的结果列被使用的索引完全覆盖了（即索引包含所有被查询列），就不会在读取扫描索引之后再次去读取数据文件中的行数据。例如：

```sql
SELECT COUNT(*) FROM t1 WHERE key1 = 1 AND key2 = 1;
```

如果被查询的结果列不能被使用的索引完全覆盖，被索引范围查询匹配到的索引数据对应的的数据文件中的行数据就会被扫描读取；

如果索引合并的查询条件中存在覆盖InnoDB表的主键索引的查询条件，这个查询条件不会被用来扫描读取数据文件中的行数据，而是用来过滤其他查询条件查询出的数据。

###### 并集访问算法

1. 对于包含 N 列的组合索引，需要用 or 连接 N 个索引列和常数进行比较

```sql
key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
```

2. InnoDB 主键索引的任何范围查询

3. 索引合并优化交集访问算法适用的条件，例如：
```sql
SELECT * FROM t1
  WHERE key1 = 1 OR key2 = 2 OR key3 = 3;

SELECT * FROM innodb_table
  WHERE (key1 = 1 AND key2 = 2)
     OR (key3 = 'foo' AND key4 = 'bar') AND key5 = 5;
```

###### 排序并集访问算法

通过 or 连接范围查询，且不不适用于并集访问算法的查询条件

```sql
SELECT * FROM tbl_name
  WHERE key_col1 < 10 OR key_col2 < 20;

SELECT * FROM tbl_name
  WHERE (key_col1 > 10 OR key_col2 = 20) AND nonkey_col = 30;
```

`排序并集访问算法` 和 `并集访问算法` 的区别在于 `排序并集访问算法` 首先要先获取所有满足条件的数据的 Row Id， 按照 Row Id 排序之后再返回查询结果

######  索引合并算法的参数化配置

可以通过 `optimizer_switch` 系统参数中的 `index_merge`、`index_merge_intersection`、`index_merge_union` 以及 `index_merge_sort_union` 标识控制 `Index Merge`的使用。
默认配置下，这些标识都是 `启用状态-on`。如果只想启用特定的某种算法，则可以设置index_merge为off，然后将相应启用算法的标识设置为on即可。

##### `ALL`: 全表扫描，按照行的顺序从头到尾读取去找到符合条件的行

##### `index`: 和全表扫描一样，但是是按照索引的存储顺序从头到尾读取去找到符合条件的行；优点是可以避免排序，缺点是会随机访问行

##### `unique_subquery`

这个连接类型在部分 IN 子查询语句中用于替换 `eq_ref` 连接类型

```sql
value IN (SELECT key_column FROM single_table WHERE some_expr)
```

`unique_subquery` 连接类型是完全可以用来提高子查询效率的索引查询方法

##### `index_subquery`

这个连接类型类似于 `unique_subquery`，只是子查询只作用于非唯一索引

##### `range`: 范围查询

通过索引查询给定范围的数据行，这个连接类型中的 `ref` 列是 `NULL`，`range` 范围查询类的连接类型用于索引通过如下比较符(=, <, >, <=, >=, BETWEEN, LIKE, IN()) 和常量比较的时候


#### `possible_keys`： 可能命中的索引，可能为空

#### `key`: 查询命中的索引，可能为空

`key` 的取值可能不是 `possible_keys` 的取值

#### `key_len`: 使用的索引的长度

使用的索引的长度决定能用的组合索引的长度，如果 `key` 为空， `key_len` 也为空

***索引存储方式决定了允许为空的列的索引长度可能会大于不能为空的列组成的索引***

#### `rows`: 查询语句为了查询出结果要读取的行数

对于 `InnoDB`, 这个行数是个估计值，可能不是实际值

#### `filtered`: 过滤行数

todo
