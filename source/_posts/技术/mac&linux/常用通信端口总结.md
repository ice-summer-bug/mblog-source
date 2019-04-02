---
layout: post
title: 常用通信端口总结
categories: 端口
tags: [端口,linux]
date: 2018-01-18 21:00:00
description: 系统常用通信端口汇总
---

### 前言

今天在查看线上应用的时候发现一个不常见的端口占用

```bash
$netstat -tlp | grep 99573
tcp        0      0 0.0.0.0:24013           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:17134           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:sieve-filter    0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:18134           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 localhost:19134         0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:commplex-main   0.0.0.0:*               LISTEN      99573/java          
```

`sieve-filter` 这个是什么？`commplex-main` 又是什么？

## 通用常用端口列表

| 端口 | 使用协议 | 服务 | 说明 |
|-|-|-|-|
| 5000 | TCP/UDP | commplex-main | |
| 2000 | TCP/UDP | sieve-filter | |


结论：
        这里使用 `commplex-main` 和 `sieve-filter` 端口的是 `crashub`
