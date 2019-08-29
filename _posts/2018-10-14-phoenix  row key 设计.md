---
layout: post
title: "phoenix row key设计"
date: 2018-10-14
categories: BigData
tags: [phoenix]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
结合需求进行row key设计
<!-- more -->
### HBase基础
* Table：同传统数据库中的表是类似的，但他的不同之处在于它是基于SchemaLess的设计，比传统数据库表更加灵活。

* Region：将表横向切割成一个个子表，从而实现分布式的存储，子表在HBase中称作Region，它关联了数据的一个区间。

* Column Family：HBase可以将一行数据分成不同列的集合，这些列的集合称为Column Family，不同的Column Family文件被存储在不同的路径中。

* RegionServer：指HBase里面数据服务的进程，每个Region必须要分到RegionServer上才能提供正常的读写服务。

* MemStore：用来在内存中缓存一定大小的数据，达到一定大小后批量写入到底层文件系统中。

* HFile：HBase数据在底层分布式文件系统中的文件组织格式。

### HBase 存储结构
存储的逻辑视图如下
    
    |rowkey |  column family1   |  colume family2  |
    ------------------------------------------------
    |       | column1 |colume2  |  column3 |column4|
    |rowkey1| t:cell  |t:cell   |  t:cell  |t:cell |

* Row key ：表的主键，按照字典序排序。
* cf：在 HBase 中，列簇将表进行横向切割。
* column：属于某一个列簇，在 HBase 中可以进行动态的添加。
* cell : 是指具体的 Value 。
* Version(t):这个是指版本号，用时间戳（TimeStamp ）来表示。

hbase是kv数据库，那么key是如何构成的？
    
     |rowkey|cf|colume|timestamp|value|
同一个KEY可以有多个版本的Value值（可以通过配置来设置多少个版本）。查询的话是默认取回最新版本的那条数据，但是也可以进行查询多个版本号的数据，在接下来的进阶操作文章中会有演示。


### row key 作用
读写数据时通过RowKey路由到对应的Region，MemStore中的数据按RowKey排序，HFile中的数据按RowKey排序。


### row key设计原则
了解完HBase的存储结构,进入主题。
* RowKey长度原则：设计定长，越短越好。
* RowKey散列原则：为了避免查询热点问题，不建议将单调递增的时间字段放在RowKey的首位。（非要使用的话可以采用时间戳倒序的方式）
* RowKey唯一原则：保证唯一

保证这几个原则的前提下根据实际的需求表的特点进行RowKey设计。


### 加盐表，数据倒序，哈希
加盐表：加盐的原理是对RowKey添加一个散列的前缀，计算公式如下：

    new_row_key = ((byte) (hash(key) % BUCKETS_NUMBER) + original_key
    
数据倒序：类似时间这样递增有序的字段，可以通过数据反转，使其失去有序，产生随机性。

哈希： 将RowKey的部分或者全部值进行hash。以此打乱原有RowKey的有序。
