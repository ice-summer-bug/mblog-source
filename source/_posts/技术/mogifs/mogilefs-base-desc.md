---
layout: post
title: mogilefs 基础介绍
categories: mogilefs
tags: [mogilefs]
date: 2019-01-10 21:00:00
description: mogilefs 基础介绍
---

# `mogilefs`

## `mogilefs` 介绍

MogileFS是一个开源的分布式文件存储系统，由LiveJournal旗下的DangaInteractive公司开发。Danga团队开发了包括 Memcached、MogileFS、Perlbal 等多个知名的开源项目。目前使用MogileFS 的公司非常多，如日本排名先前的几个互联公司及国内的yupoo(又拍)、digg、豆瓣、1号店、大众点评、搜狗和安居客等，分别为所在的组织或公司管理着海量的图片。

## `mogilefs` 基本概念

### `mogilefs` 组成架构

`tracker节点`: `tracker节点` 是应用程序 `mogilefsd`，通过数据库来保存元数据，提供API 响应客户端 <br/>
`storage节点`: `storage节点` 是应用程序 `mogstored`，本质上是一个 `WebDAV` 服务，默认监听 `7500` 端口，接受客户端的文件存储请求 <br/>
`database`: 用于存储元数据的 `mysql` 数据库 <br/>

![](https://s1.51cto.com/images/20171204/1512393806362708.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk= "mogilefs 结构图")

### `mogilefs` 数据存储相关概念

`domain`: `mogilefs` 划分数据存储的容器，每一个 domain 中的数据的  key 不能重复，但是不同 domain 中数据的 key 可以重复；domain 的划分可以按照数据类型划分，也可以按照业务类型划分。<br/>
`host`: `mogilefs` 的每一个存储节点被称为一个 `host`，每个 `host` 有自己的 ID<br/>
`device`: 每个存储节点可以挂靠多个 `device`， 每个 `device` 对应一个时间存储数据的存储设备（硬盘），每个 `device` 有自己的ID<br/>
`class`: 对于存储在mogilefs 上的文件进行分类，同一 `domain` 下的文件可以分类不同类型，但是key 不同重复，上传文件时，可指定 `class`， 默认为 `default` <br/> 

## `mogilefs` 基本操作

### `mogadm`

#### `mogadm check`

检查 `mogilefs` 集群状态

        $mogadm check
        Checking trackers...
        127.0.0.1:7001 ... OK

        Checking hosts...
        [ 1] storage1 ... OK

        Checking devices...
        host device         size(G)    used(G)    free(G)   use%   ob state   I/O%
        ---- ------------ ---------- ---------- ---------- ------ ---------- -----
        [ 1] dev1           374.702     29.018    345.684   7.74%  writeable   0.0
        ---- ------------ ---------- ---------- ---------- ------
             total:   374.702     29.018    345.684   7.74%

#### `mogadm domain`

`mogilefs` 集群 `domain` 管理

```bash
mogadm domain add domain1 # 新增 domain
mogadm domain delete domain1 # 删除 domain
mogadm domain list # 列出所有 domain
```

#### `mogadm host`

`mogilefs` 集群 `host` 管理

```bash
# 新增 host
mogadm --trackers=127.0.0.1:7001 host add host1 --ip=[$ip] --port=7500 --status=alive 
mogadm host delete host1 # 删除 host, 不指定tracker 的时候默认使用本机 `127.0.0.1:7001`
mogadm host list # 列出所有 host
```

#### `mogadm device`

`mogilefs` 集群 `device` 管理

##### `device` 列出

```bash
mogadm device list # 列出所有 host
```

##### 新增 `device` 

下面的例子中新增 `device`

1. 进入 `/etc/mogilefs/mogstored.conf` 文件中配置的 `docroot`，例如 `cd /home1/mogdata`

2. 如果待挂载的 `device`（ID 命名为 dev2） 所在的硬盘和 `/home1` 所在硬盘不一致(例如 `/home2`)，需要在 `/home2` 中新建目录 `/home2/mogdata/dev2`，并在 `/home1/mogdata` 中创建软链

```bash
mkdir -pv /home2/mogdata/dev2;
cd /home1/mogdata;
ln -s /home2/mogdata/dev2 dev2;
mogadm --trackers=127.0.0.1:7001 device add host1 dev2
```

##### `device` 状态修改

```bash
# dev2 磁盘故障
mogadm --trackers=127.0.0.1:7001 device mark host1 dev2 dead # 将 dev2 置为死亡，不可用
# dev2 磁盘故障恢复，无法直接从 dead 更换到 alive，需先过渡到 down 状态
mogadm --trackers=127.0.0.1:7001 device mark host1 dev2 down # 将 dev2 置为 down 状态
mogadm --trackers=127.0.0.1:7001 device mark host1 dev2 alive # 将 dev2 置为 alive 状态
```

##### `device` 列表状态查询

```bash
mogadm --trackers=127.0.0.1:7001 device summary
```

### 文件上传

```bash
mogupload --trackers="127.0.0.1:7001" --domain=[${domain}] --key=[${key}] --file=[${file}] --class=[${class}]
```

`--domain`: 指定上传文件的 `domain`
`--key`: 指定上传文件的 `key`
`--file`: 指定上传文件的路径
`--class`: 指定上传文件的 `class`，可选项，默认值为 `default`

### 文件下载

```bash
mogfetch --trackers=host --domain=[${domain}] --key=[${key}] --file=[${file}]
```

`--domain`: 指定上传文件的 `domain`
`--key`: 指定上传文件的 `key`
`--file`: 指定下载文件的路径及名称

### 文件查看

```bash
mogfileinfo  --trackers="127.0.0.1:7001" --domain=[${domain}] --key=[${key}]
```

`--domain`: 指定上传文件的 `domain`
`--key`: 指定上传文件的 `key`

输出结果：

    - file: [$file]
    class:                   default
    devcount:                    1
    domain:                 [$domain]
    fid:                   30
    key:      [$key]
    length:                  608
    - http://127.0.0.1:7500/dev1/0/000/000/0000000030.fid