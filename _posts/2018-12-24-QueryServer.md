---
layout: post
title: "Phoniex queryserver"
date: 2018-12-24
categories: Phoniex
tags: [Phoniex]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
本文主要介绍:  
1.什么是queryserver  
2.为什么要使用queryserver  
3.怎么使用queryserver  
<!-- more -->
## 什么是queryserver？
### 首先了解一下Phoenix是什么？
~~~
• 社区公认的 HBase 上最合适的 SQL 层。
• Phoenix 为 HBase 赋能
• 提供JDBC接口，让HBase更易用
• 提供轻量级事务能力
• 提供操作性分析能力
• 提供二级索引能力
• 提供简单的tenancy能力 
~~~
### QueryServer作用与实现
~~~
• QueryServer的本质一个HTTP Server
• 代理请求并发送给Phoenix进行处理
• 基于Apache Calcite Avatica 实现
• QueryServer可以有一个或多个，通过负载均衡统一访问入口，并提高可用性。
~~~

## 为什么要使用queryserver
~~~
QueryServer自身无状态，扩展更容易
• 将计算资源移至到 Server 端，客户端可以运行在低配环境中
• 正真面向JDBC接口，客户端不再会直接访问 HBase/Zookeeper/HDFS
• 支持更多的native语言客户端访问 (go，python，C#)
~~~
### 客户端支持
~~~
• Go driver
    > https://github.com/Boostport/avatica
• C# driver
    > https://github.com/Azure/hdinsight-phoenix-sharp
    > https://www.nuget.org/packages/Microsoft.Phoenix.Client/1.0.0-preview
• Python DB API v2.0
    > https://bitbucket.org/lalinsky/python-phoenixdb
~~~

### 以python客户端为例！使用非常方便！
~~~
环境安装: pip install python-phoenixdb
使用示例:
import phoenixdb.cursor
database_url = 'http://xx.xx.xx.xx:8765/'
cursor = conn.cursor(phoenixdb.cursor.DictCursor)
cursor.execute(
    "select * from test limit 10"
)
xx = cursor.fetchall()
print(xx)
~~~

### 轻客户端链接 QueryServer
* 轻客户端 JDBC URL 格式：jdbc:phoenix:thin:url=<scheme>://<server-hostname>:<port>[;option=value...]

> scheme:传输协议，目前只支持 http，可以省略。     
> server-hostname : queryserver 或者 SLB 的 host。  
> port: 默认端口是8765，可以省略。  

* 访问方式：
> 通过 bin/sqlline-thin.py 脚本访问，提供交互式操作。  
> 用法：bin/sqlline-thin.py [[scheme://]host[:port]] [sql_file]  
> 示例：bin/sqlline-thin.py http://localhost:8765  
 

## 怎么使用queryserver
### queryserver部署（通过docker容器化，在同一机器上可以同时部署多个服务）
* 如果部署在 HBase 集群，可以省略以下步骤 1，2
1. 创建配置文件目录 hbase_conf(配置参数调整看相关链接)
2. 在 hbase_conf 目录下创建 hbase-site.xml文件，并配置 zk 地址：
~~~
 <configuration>
    <property>
      <name>hbase.zookeeper.quorum</name>
      <value>zk,zk2,zk3</value>
    </property>
    <property>
      <name>hbase.rpc.timeout</name>
      <value>60000000</value>
    </property>
    <property>
      <name>hbase.client.scanner.timeout.period</name>
      <value>60000000</value>
    </property>
    <property>
      <name>phoenix.query.timeoutMs</name>
      <value>60000000</value>
    </property>
     <property>
      <name>hbase.client.scanner.caching</name>
      <value>5000</value>
    </property>
     <property>
      <name>phoenix.mutate.batchSize</name>
      <value>1000</value>
    </property>
  </configuration>
~~~
3. 设置环境变量  
> vim /etc/profile 添加以下配置  
> export HBASE_CONF_DIR=/root/conf  
> source /etc/profile
4. 启动 queryserver
> bin\queryserver.py start

## 相关链接
[1. hbase-site.xml配置详解](https://blog.csdn.net/ningxuezhu/article/details/50547970)  
[2. phoenix 官网](https://phoenix.apache.org/)