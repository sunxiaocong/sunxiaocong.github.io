---
layout: post
title: "hive表迁移到spark on hdfs集群（跨集群迁移）.md"
date: 2019-09-04
categories: BigData
tags: [spark]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
由于源数据在于旧的hbase集群的hdfs上，并未开启8020端口，没办法直接通过文件迁移到新集群。只能通过pyspark 读取hive表数据，写入新的集群的hdfs中。
<!-- more -->
### 迁移hdfs数据至新集群（虽然这种方式行不通，记录一下）
    hadoop distcp -skipcrccheck -update hdfs://xxx.xxx.xxx.xxx:8020/ user/risk hdfs://xxx.xxx.xxx.xxx:8020/user/risk
    -skipcrccheck 因本次迁移涉及低版本迁移高版本, 如果hadoop版本则不需要 
    -update 增量更新, 通过名称和大小比较,源与目标不同则更新
    
    完整迁移步骤参考：
    http://www.voidcn.com/article/p-bcawjjdz-vy.html
    
### pyspark 读取hive 写入新集群的方式
废话不说直接上代码

    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
    # @Time    : 2019/9/3 15:22
    __author__ = 'sxc'
    
    """yarn 连接模式"""
    import datetime
    from pyspark import SparkContext, SparkConf
    from pyspark.sql import SparkSession
    from pyspark.sql.functions import concat_ws
    
    value = "yarn"
    conf = SparkConf().setMaster(value)
    sc = SparkContext(conf=conf)
    sc.setLogLevel("ERROR")
    spark = SparkSession.builder.appName("my_name").enableHiveSupport().getOrCreate()
    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
    spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")
    
    df = spark.sql("")
    df.coalesce(1).write.format('parquet').mode("overwrite").save("hdfs:{新集群id}/{写入的路径}")
    
最重要的如下运行配置（阿里作业提交控制台上配置）
    
    --conf spark.hadoop.dfs.nameservices={源集群id},{目标集群id},{目标集群id2}
    --conf spark.hadoop.dfs.client.failover.proxy.provider.{目标集群id}=org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider
    --conf spark.hadoop.dfs.ha.automatic-failover.enabled.{目标集群id}=true
    --conf spark.hadoop.dfs.namenode.http-address.{目标集群id}.nn1={目标集群id}-master1-001.spark.rds.aliyuncs.com:50070
    --conf spark.hadoop.dfs.namenode.http-address.{目标集群id}.nn2={目标集群id}-master2-001.spark.rds.aliyuncs.com:50070
    --conf spark.hadoop.dfs.ha.namenodes.{目标集群id}=nn1,nn2
    --conf spark.hadoop.dfs.namenode.rpc-address.{目标集群id}.nn1={目标集群id}-master1-001.spark.rds.aliyuncs.com:8020
    --conf spark.hadoop.dfs.namenode.rpc-address.{目标集群id}.nn2={目标集群id}-master2-001.spark.rds.aliyuncs.com:8020
    
    --files /user/big_data_center_admin/daijt/hbase-site.xml
    --driver-memory 2G
    --driver-cores 1
    --executor-memory 1G
    --executor-cores 1
    --num-executors 20
    --name {名称}
    {执行脚本路径}
    
执行spark任务之后，就可以查看到新集群中的hdfs生成新的文件。


### 总结
