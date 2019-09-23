---
layout: post
title: "python 常见面试题整理（1）"
date: 2019-01-04
categories: Mongo
tags: [BigData]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
导出mongo数据过程记录
<!-- more -->
### mongoexport
    
    # 直接从某个表导出期望字段，生成CSV
    mongoexport --host 127.0.0.1 --db sampleData --collection eventV4 --csv --out events.csv --fields '_id,something'
    
    # 增加一个检索filter后导出CSV
    mongoexport --host 127.0.0.1 --db sampleData --collection eventV4 --queryFile ./range.json --csv --out events.csv --fields '_id,something'
    
### 参数
    Usage:
      mongoexport <options>
    
    Export data from MongoDB in CSV or JSON format.
    
    See http://docs.mongodb.org/manual/reference/program/mongoexport/ for more information.
    
    general options:
          --help                     print usage
          --version                  print the tool version and exit
    
    verbosity options:
      -v, --verbose                  more detailed log output (include multiple times for more verbosity, e.g. -vvvvv)
          --quiet                    hide all log output
    
    connection options:
      -h, --host=                    mongodb host to connect to (setname/host1,host2 for replica sets)
          --port=                    server port (can also use --host hostname:port)
    
    authentication options:
      -u, --username=                username for authentication
      -p, --password=                password for authentication
          --authenticationDatabase=  database that holds the user's credentials
          --authenticationMechanism= authentication mechanism to use
    
    namespace options:
      -d, --db=                      database to use
      -c, --collection=              collection to use
    
    output options:
      -f, --fields=                  comma separated list of field names (required for exporting CSV) e.g. -f "name,age"
          --fieldFile=               file with field names - 1 per line
          --type=                    the output format, either json or csv (defaults to 'json')
      -o, --out=                     output file; if not specified, stdout is used
          --jsonArray                output to a JSON array rather than one object per line
          --pretty                   output JSON formatted to be human-readable
    
    querying options:
      -q, --query=                   query filter, as a JSON string, e.g., '{x:{$gt:1}}'
      -k, --slaveOk                  allow secondary reads if available (default true)
          --forceTableScan           force a table scan (do not use $snapshot)
          --skip=                    number of documents to skip
          --limit=                   limit the number of documents to export
          --sort=                    sort order, as a JSON string, e.g. '{x:1}'

### 部分参数说明：
    -h:指明数据库宿主机的IP  
    -u:指明数据库的用户名
    -p:指明数据库的密码
    -d:指明数据库的名字
    -c:指明collection的名字
    -f:指明要导入那些列
    -- limit 导出的行数
    -- skip 从第几行开始导出
    -- sort 排序之类的

### mongoexport
对应导出工具，使用上基本和mongoexport差不多。想具体了解如何使用的[请戳这里](https://www.cnblogs.com/limingluzhu/p/4323146.html)

### 总结
使用下来比较简单没出什么幺蛾子！！！