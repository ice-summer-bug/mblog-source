---
layout: post
title: Elastic Search 集群管理之动态扩容收缩
categories: elastic-search
tags: ['elastic search', '中间件']
date: 2019-03-16 22:00:00
description: Elastic Search 集群简单介绍及在线扩容和收缩
---

# `Elastic Search` 集群简单介绍

## `Elastic Search` 节点重启

***注意：本人使用的是 `Elastic Search` 的版本是 `5.6.x`, 不同版本的操作 API 可能不一致***

1. 禁止分配分片

```bash
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.enable": "none"
  }
}
'
```

2. 停止不必要的索引的创建，并发出同步刷新请求（非必要步骤）

___在重启节点的过程中，可以不停止索引的创建；但是如果你停止不必要的索引的创建，并发出同步刷新请求的话，分片的回复将会更加快速___


```bash
curl -X POST "localhost:9200/_flush/synced"
```

___同步刷新操作只能是尽最大努力执行成功，如果有挂起的索引操作，这个操作将会失败，建议多次执行同步刷新请求___

3. 修改节点配置，重启节点

重启完节点之后可以查看节点是否重启成功

```bash
curl -X GET "localhost:9200/_cat/nodes"
```

4. 恢复节点分配

```bash
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  }
}
'
```

5. 等待节点回复

使用如下指令查看集群状态，知道集群状态从 `yellow` 变为 `green`

```bash
curl -X GET "localhost:9200/_cat/health"
```

6. 重复上述步骤，重启其他需要重启的节点

## `Elastic Search` 集群动态新增节点

### 1. 准备在新的机器启动 `Elastic Search` 节点

#### 1.1 从原有节点上拷贝文件到新机器上

#### 1.2 修改如下配置

示例如下：将 `discovery.zen.minimum_master_nodes` 加一，在 `discovery.zen.ping.unicast.hosts:` 中新增 host `new.ip.3`

```yaml
discovery.zen.minimum_master_nodes: 3
discovery.zen.ping.unicast.hosts: ["ip1","ip2", "new.ip.3"]
```

#### 1.3 启动此节点

```bash
./bin/elasticsearch -d
```

#### 1.4 查看节点启动情况

```bash
curl -X GET "localhost:9200/_cat/nodes"
```

#### 1.5 等待集群回复

使用如下指令查看集群状态，知道集群状态从 `yellow` 变为 `green`

```bash
curl -X GET "localhost:9200/_cat/health"
```

### 2. 逐个重启其他节点

具体步骤见上述重启步骤，重启 `步骤3` 中修改 `config/elasticsearch.yml` 文件

将 `discovery.zen.minimum_master_nodes` 加一，在 `discovery.zen.ping.unicast.hosts:` 中新增 host `new.ip.3`

```yaml
discovery.zen.minimum_master_nodes: 3
discovery.zen.ping.unicast.hosts: ["ip1","ip2", "new.ip.3"]
```

## `Elastic Search` 集群动态下线节点

### 1. 告知集群分配分配的时候排除待下线节点

示例：排除 ip `10.0.0.1`

```bash
curl -XPUT localhost:9200/_cluster/settings -H 'Content-Type: application/json' -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
   }
}';echo
```

### 2. 查看分片分配情况


```bash
$curl -XGET 'http://localhost:9200/_nodes/NODE_NAME/stats/indices?pretty'
```

执行上述指令得到

```json
{
  "_nodes": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "cluster_name": "CLUSTER_NAME",
  "nodes": {
    "H447JUPgRSKZKQdzQ6H0gg": {
      "timestamp": 1551617699642,
      "name": "e93d15417",
      "transport_address": "IP:9300",
      "host": "IP",
      "ip": "IP:9300",
      "roles": [
        "master",
        "data"
      ],
      "indices": {
        "docs": {
          "count": 0,
          "deleted": 0
        },
        "store": {
          "size_in_bytes": 0,
          "throttle_time_in_millis": 0
        },
        "indexing": {
          "index_total": 0,
          "index_time_in_millis": 0,
          "index_current": 0,
          "index_failed": 0,
          "delete_total": 0,
          "delete_time_in_millis": 0,
          "delete_current": 0,
          "noop_update_total": 0,
          "is_throttled": false,
          "throttle_time_in_millis": 0
        },
        "get": {
          "total": 0,
          "time_in_millis": 0,
          "exists_total": 0,
          "exists_time_in_millis": 0,
          "missing_total": 0,
          "missing_time_in_millis": 0,
          "current": 0
        },
        "search": {
          "open_contexts": 0,
          "query_total": 0,
          "query_time_in_millis": 0,
          "query_current": 0,
          "fetch_total": 0,
          "fetch_time_in_millis": 0,
          "fetch_current": 0,
          "scroll_total": 0,
          "scroll_time_in_millis": 0,
          "scroll_current": 0,
          "suggest_total": 0,
          "suggest_time_in_millis": 0,
          "suggest_current": 0
        },
        "merges": {
          "current": 0,
          "current_docs": 0,
          "current_size_in_bytes": 0,
          "total": 0,
          "total_time_in_millis": 0,
          "total_docs": 0,
          "total_size_in_bytes": 0,
          "total_stopped_time_in_millis": 0,
          "total_throttled_time_in_millis": 0,
          "total_auto_throttle_in_bytes": 4320133120
        },
        "refresh": {
          "total": 641,
          "total_time_in_millis": 66,
          "listeners": 0
        },
        "flush": {
          "total": 237,
          "total_time_in_millis": 1
        },
        "warmer": {
          "current": 0,
          "total": 0,
          "total_time_in_millis": 0
        },
        "query_cache": {
          "memory_size_in_bytes": 0,
          "total_count": 0,
          "hit_count": 0,
          "miss_count": 0,
          "cache_size": 0,
          "cache_count": 0,
          "evictions": 0
        },
        "fielddata": {
          "memory_size_in_bytes": 0,
          "evictions": 0
        },
        "completion": {
          "size_in_bytes": 0
        },
        "segments": {
          "count": 0,
          "memory_in_bytes": 0,
          "terms_memory_in_bytes": 0,
          "stored_fields_memory_in_bytes": 0,
          "term_vectors_memory_in_bytes": 0,
          "norms_memory_in_bytes": 0,
          "points_memory_in_bytes": 0,
          "doc_values_memory_in_bytes": 0,
          "index_writer_memory_in_bytes": 0,
          "version_map_memory_in_bytes": 0,
          "fixed_bit_set_memory_in_bytes": 0,
          "max_unsafe_auto_id_timestamp": -9223372036854775808,
          "file_sizes": {}
        },
        "translog": {
          "operations": 0,
          "size_in_bytes": 0
        },
        "request_cache": {
          "memory_size_in_bytes": 0,
          "evictions": 0,
          "hit_count": 0,
          "miss_count": 0
        },
        "recovery": {
          "current_as_source": 0,
          "current_as_target": 0,
          "throttle_time_in_millis": 90997
        }
      }
    }
  }
}
```

可以看到查询结果中 `indices` 大部分都是0 ，这是因为步骤 1 排除了当前节点后触发了分配的重新分配

### 3. 查看集群健康情况

```bash
curl -XGET 'http://127.0.0.1:9200/_cluster/health?pretty'
```

确保集群集状态是 `green`

### 4. 重启现有节点

#### 4.1 禁止分配分片

```bash
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.enable": "none"
  }
}
'
```

#### 4.2 停止不必要的索引的创建，并发出同步刷新请求（非必要步骤）

___在重启节点的过程中，可以不停止索引的创建；但是如果你停止不必要的索引的创建，并发出同步刷新请求的话，分片的回复将会更加快速___


```bash
curl -X POST "localhost:9200/_flush/synced"
```

#### 4.3 修改节点配置，重启节点

修改 `config/elasticsearch.yml` 文件


示例如下：将 `discovery.zen.minimum_master_nodes` 减一，在 `discovery.zen.ping.unicast.hosts:` 中去除 host `new.ip.3`

```yaml
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping.unicast.hosts: ["ip1","ip2"]
```

重启完节点之后可以查看节点是否重启成功

```bash
curl -X GET "localhost:9200/_cat/nodes"
```

#### 4.4 恢复节点分配

```bash
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.enable": "all",
    "cluster.routing.allocation.exclude._ip" : "new.ip.3"
  }
}
'
```

#### 4.5 等待节点回复

使用如下指令查看集群状态，直到集群状态从 `yellow` 变为 `green`

```bash
curl -X GET "localhost:9200/_cat/health"
```

#### 4.6 重复上述步骤，逐步重启其他线上节点

### 5. 下线节点

```bash
ps -ef | grep elasticsearch
kill PID
```

### 6. 修改分片分配配置

删除配置  `"cluster.routing.allocation.exclude._ip" : "new.ip.3"`

```bash
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.exclude._ip" : null
  }
}
'
```

### 注意

排除分配分片的节点时候除了指定 `ip` 还可以指定节点名称 `name` 以及  `host`

还可以指定ip 匹配规则

如：
```bash
curl -XPUT localhost:9200/_cluster/settings -H 'Content-Type: application/json' -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._ip" : "10.0.0.*"
   }
}';echo

curl -XPUT localhost:9200/_cluster/settings -H 'Content-Type: application/json' -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._name" : "NODE_NAME"
   }
}';echo

```



###### 参考资料
[Rolling upgrades](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/rolling-upgrades.html "Rolling upgrades")
[How to remove node from elasticsearch cluster on runtime without down time](https://stackoverflow.com/questions/17268495/how-to-remove-node-from-elasticsearch-cluster-on-runtime-without-down-time "How to remove node from elasticsearch cluster on runtime without down time")
[Shard Allocation Filtering](https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-filtering.html "Shard Allocation Filtering")
[Elasticsearch: HOW-TO delete a (cluster) setting](https://stackoverflow.com/questions/33520384/elasticsearch-how-to-delete-a-cluster-setting "Elasticsearch: HOW-TO delete a (cluster) setting")
