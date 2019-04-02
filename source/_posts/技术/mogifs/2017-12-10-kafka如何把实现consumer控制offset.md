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


***todo...***

## `mogilefs` 基本概念

`tracker节点`:<br/>
`database节点`:<br/>
`storage节点`:<br/>


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
