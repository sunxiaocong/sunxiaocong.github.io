---
layout: post
title: "通过kafka manager api监控topic的消费者组offset实现告警"
date: 2019-07-09
categories: kafka
tags: [kafka]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
通过kafka manager api监控topic的消费者组offset实现告警
<!-- more -->
### kafka manager api
kafka manager api route提供如下api。  
https://github.com/yahoo/kafka-manager/blob/master/conf/routes  
~~~
GET    /api/status/:c/brokers                               controllers.api.KafkaStateCheck.brokers(c:String)
GET    /api/status/:c/brokers/extended                      controllers.api.KafkaStateCheck.brokersExtended(c:String)
GET    /api/status/:c/topics                                controllers.api.KafkaStateCheck.topics(c:String)
GET    /api/status/:c/topicIdentities                       controllers.api.KafkaStateCheck.topicIdentities(c:String)
GET    /api/status/clusters                                 controllers.api.KafkaStateCheck.clusters
GET    /api/status/:c/:t/underReplicatedPartitions          controllers.api.KafkaStateCheck.underReplicatedPartitions(c:String,t:String)
GET    /api/status/:c/:t/unavailablePartitions              controllers.api.KafkaStateCheck.unavailablePartitions(c:String,t:String)
GET    /api/status/:cluster/:consumer/:topic/:consumerType/topicSummary   controllers.api.KafkaStateCheck.topicSummaryAction(cluster:String, consumer:String, topic:String, consumerType:String)
GET    /api/status/:cluster/:consumer/:consumerType/groupSummary          controllers.api.KafkaStateCheck.groupSummaryAction(cluster:String, consumer:String, consumerType:String)
GET    /api/status/:cluster/consumersSummary                controllers.api.KafkaStateCheck.consumersSummaryAction(cluster:String)
~~~

### 实现监控kafka offset
有了api接口，我们非常容易就实现了获取"totalLag"也就是消费者延迟数据量。
通过requests get获取接口返回实现如下
~~~
import requests
//请求参数
config = {
    'cluster_name': cluster_name,
    'consumer_name': consumer_name,
    'topic_name': topic_name,
    'host': host,
}
//请求接口
base_url = "{host}/api/status/{cluster_name}/{consumer_name}/{topic_name}/KF/topicSummary".format(
                    **config)                    
r = requests.get(base_url, auth=auth, verify=False)
response = json.loads(r.content)
~~~
接口返回值：
~~~
{
	'totalLag': x,
	'percentageCovered': xx,
	'partitionOffsets': [xx, xx, xx],
	'partitionLatestOffsets': [xx, xx, xx],
	'owners': ['consume1', 'consume2', 'consume3']
}
~~~

### 其他接口测试
* GET    /api/status/:c/brokers  
> {"brokers":[1,2,3]}   

* GET    /api/status/:c/topics
>{"topics":["topic1","topic2"]}

* GET    /api/status/:cluster/:consumer/:consumerType/groupSummary   获取这个组所有的topic消费者情况
> {"topic1":{"totalLag":0,"percentageCovered":100,"partitionOffsets":[],"partitionLatestOffsets":[],"owners":[]}，
"topic2":{"totalLag":0,"percentageCovered":100,"partitionOffsets":[],"partitionLatestOffsets":[],"owners":[]}}

* GET    /api/status/:cluster/consumersSummary   获取所有组名称以及type
> {
  "consumers" : [{
    "name" : "group1",
    "type" : "ZK"
  }, {
    "name" : "group2",
    "type" : "KF"
  }]
}

### 总结
每天进步一点点！！！