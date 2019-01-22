---
layout: post
title: "ElaticSearch Kibana 使用与维护"
date: 2019-01-22
categories: 大数据
tags: [ElasticSearch,Kibana]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

ElasticSearch的数据主要是在Kibana中做展示，本文主要介绍如何使用Kibana来操作ElasticSearch

<!-- more -->

## 1. Kibana Console

### 1.1 Kibana Console 简介
Kibana提供了Console UI来通过REST API与Elasticsearch交互，Console位于Kibana的Dev Tools栏下。Console有两个主要区域，左边是编辑区用来书写REST请求，右边用来显示请求返回结果。
### 1.2 设置索引模板
```
PUT /_template/shard_template 
{
    "order" : 1,
    "index_patterns": [
    "dq2_orm*",
    "dq2_dq_orm*",
    "dq2_frontend*",
    "dq2_session*"
    ],
    "settings": {
        "index": {
            "number_of_shards": "8"
        }
    }
}

```

### 1.3 查看集群信息

```
GET /_cat/master?v   #获取主节点信息
GET /_cat/health?v  #获取集群健康状态
GET /_cat/nodes?v    #获取节点信息
GET /_cat/allocation?v #获取节点磁盘信息
```

### 1.4 索引信息

```
GET /_cat/indices?v&&s=index
```

### 1.5 分片操作

__(1) 查看索引对应的分片数__

```
GET /_cat/shards/hwsjus_grand-2018.11.01?v&s=node
```

__(2) 如果分片分布不均匀采用如下命令手动分片__

```
POST /_cluster/reroute
{
    "commands" : [
        {
        "move" : {
        "index" : "hwsjus_session-2018.11.01",
        "shard" :4,
        "from_node" : "es_d4",
        "to_node" : "es_d1"
        }
    }
    ]
}
```

__(3) 没有分片的__

```
POST /_cluster/reroute
{
    "commands" : [
        {
          "allocate_replica" : {
                "index" : "hwsj_grand-2018.10.25", 
                "shard" : 6,
                "node" : "es_d7"
          }
        }
    ]
}
```

__(4) 希望取消每个分片过程__  

```
POST /_cluster/reroute
{
    "commands" : [
        {
           "cancel" :   
                {  
                  "index" : "test", "shard" : 0, "node" : "node1"  
                }  
        }
    ]
}
```

## 2. 查询

### 2.1 空搜索

Elasticsearch将返回在请求超时前收集到的结果。
    超时不是一个断路器（circuit breaker）
    需要注意的是timeout不会停止执行查询，它仅仅告诉你目前顺利返回结果的节点然后关闭连接。在后台，其他分片可能依旧执行查询，尽管结果已经被发送。
    使用超时是因为对于你的业务需求（译者注：SLA，Service-Level Agreement服务等级协议，在此我翻译为业务需求）来说非常重要，而不是因为你想中断执行长时间运行的查询。
```
GET /_search?timeout=10ms
```

### 2.2 分页所搜

```
GET /_search?size=5
GET /_search?size=5&from=5
GET /_search?size=5&from=10

GET /_search
{
  "from": 30,
  "size": 10
}

```

### 2.3 DSL语句

#### 2.3.1 基础格式

```Javascript
GET /_search
{
    "query": YOUR_QUERY_HERE
}
```

#### 2.3.2 空查询

```Javascript
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```

#### 2.3.4 查询子句

```Javascript
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}

{
    "match": {
        "tweet": "elasticsearch"
        }
}
```

#### 2.3.5 合并多子句

- 叶子子句(*leaf clauses*)(比如`match`子句)用以在将查询字符串与一个字段(或多字段)进行比较

- 复合子句(*compound*)用以合并其他的子句。例如，`bool`子句允许你合并其他的合法子句，`must`，`must_not`或者`should`，如果可能的话：

```Javascript
{
  "bool": {
      "must":     { "match": { "tweet": "elasticsearch" }},
      "must_not": { "match": { "name":  "mary" }},
      "should":   { "match": { "tweet": "full text" }}
  }
}
```

#### 2.3.6 查询与过滤
* 查询一般条件为 match 该会计算 相关的sorce   
* 过滤条件为Term 自会根据索引  性能非常高 不在意里面具体字段
* 排序

```Javascript
GET /_search
{
    "query" : {
        "filtered" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" ,"mode":"min"}},
        { "_score": { "order": "desc" }}
    ]}
```

#### 2.3.7 常用dsl模板

```json
{
	"query": {
		"bool": {
			"filter": {
				"bool": {
					"must": [{
						"range": {
							"@timestamp": {
								"gte": "now-55s",
								"lte": "now"
							}
						}
					}],
					"should": [{
						"match": {
							"dai": "11"
						}
					}, {
						"match": {
							"dafjkld": "222"
						}
					}]
				}
			},
			"must": [{
				"match": {
					"message": "07f8b5bcd28011e8b5c00242ac110009"
				}
			}, {
				"match": {
					"trace_id": "d280"
				}
			}],
			"must_not": [{
				"match": {
					"TF": "11"
				}
			}]
		}
	}
}
```



