---
layout: post
title: "ElaticSearch Kibana 集群搭建"
date: 2019-01-22
categories: 大数据
tags: [ElasticSearch,Kibana]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。
<!-- more -->

## 1. 安装Elasticsearch
### 1.1 Elasticsearch简介
Elasticsearch是一个基于Apache Lucene(TM)的开源搜索引擎。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。它不但稳定、可靠、快速，而且也具有良好的水平扩展能力。

### 1.2 安装JDK或openJDK
Elasticsearch依赖于java，所以，需要先安装java环境。官网要求java 8及以上，或者jdk1.8.0_131或以上。若低于此版本，会无法启动Elasticsearch。Elasticsearch将使用的java版本在JAVA_HOME设置。

__(1) 从java官网下载jdk包__

```
jdk-8u171-linux-x64.tar.gz
```

__(2) 解压__

```shell
$ sudo tar -xvf jdk-8u171-linux-x64.tar.gz
```

__(3) 将jdk目录复制到/usr/lib/下__

```shell
$ sudo cp -r  jdk1.8.0_161 /usr/lib/jdk1.8.0_171
```

__(4) 修改`/etc/profile`文件，配置环境变量__  

```shell
$ sudo vim /etc/profile
```

在最后加入：  

```shell
JAVA_HOME=/usr/lib/jdk1.8.0_171
JRE_HOME=/usr/lib/jdk1.8.0_171/jre
CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export JAVA_HOME JRE_HOME CLASS_PATH PATH
```

使配置生效：  

```shell
$ source /etc/profile
```

__(5) 检验安装是否成功__  

```shell
$ java -version #查看java版本
$ echo $JAVA_HOME #查看JAVA_HOME配置
```

### 1.3 安装Elasticsearch
下面按照elasticsearch官网的指南使用rpm包安装，也可以通过其它方式。  

Debian, Ubuntu等基于Debian的系统(官网指南)：[Install Elasticsearch with Debian Package](https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html)  

Red Hat, Centos, SLES, OpenSuSE等基于RPM的系统(官网指南)：[Install Elasticsearch with RPM](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html)  

__(1) 下载并安装签名公钥__   

```shell
$ rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

__(2) 创建文件`elasticsearch.repo`并修改内容__  

```shell
$ vim /etc/yum.repos.d/elasticsearch.repo

[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

__(3) 安装Elasticsearch__  

```shell
$ yum install -y elasticsearch
```

> 这是通过yum方式安装，也可以使用rpm包离线安装。  
> `$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.3.rpm`  
> `$ rpm --install elasticsearch-6.2.3.rpm`  

__(4) 修改Elasticsearch的`JAVA_HOME`使之与上面安装的java相同__  

```shell
$ vim /etc/sysconfig/elasticsearch

JAVA_HOME=/usr/lib/jdk1.8.0_171
```

使之生效：

```shell
$ source /etc/sysconfig/elasticsearch 
```

__(5) 修改Elasticsearch的数据和日志目录的权限__  

```shell
# 创建数据目录并修改权限
$ mkdir /data/log
$ chown elasticsearch.elasticsearch /data/log

# 确认日志目录并查看权限
$ ll /var/log
drwxr-x--- elasticsearch elasticsearch elasticsearch

# 若日志目录不存在，需要创建并修改权限
$ mkdir /var/log/elasticsearch
$ chown elasticsearch.elasticsearch /var/log/elasticsearch
```

__(6) 修改Elasticsearch配置文件__  

```shell
$ vim /etc/elasticsearch/elasticsearch.yml
```

```yaml
cluster.name: oversea_hk_efk  # 集群名称
node.name: ek_i  # 节点名
path.data: /data/log  # 数据存储路径，请注意目录权限问题，如果不是数据节点，可以设置为/var/data/elasticsearch
path.logs: /var/log/elasticsearch  # 自身日志路径，请注意目录权限问题
network.host: 本机IP   # 本机IP，可设置为0.0.0.0或实际IP
discovery.zen.ping.unicast.hosts: ["master节点IP"]  # 单播列表，自动发现，只需要足够主节点即可，不包含端口号
discovery.zen.minimum_master_nodes: 2  # 最少要发现2个主节点，一般设置为(主节点数/2)+1
node.data: false  # 是否存储索引数据
node.master: true  # 是否有资格被选举成为主节点
node.ingest: true  # 是否担当ingest节点
action.destructive_requires_name: true  # 生产环境请设置此项，禁止_all或通配符*批量删除
indices.fielddata.cache.size: 50%  #数据节点内存熔断设置

```

__(7) 配置JVM选项__  

```shell
$ vim /etc/elasticsearch/jvm.options
```

```shell
# 两个数值必须相等
-Xms4g # -Xms:初始堆大小，默认为1g，修改为不超过总内存一半，一般设为一半
-Xmx4g # -Xmx:最大堆大小，默认为1g，修改为不超过总内存一半，一般设为一半
```

### 1.4 配置更多节点

对Elasticsearch集群中的其它所有节点重复1.2和1.3的步骤。必须严格控制所有节点的Elasticsearch版本号完全相同。1.3(6)修改Elasticsearch配置文件时，需要注意：

* 集群名称相同
* 节点名称不同
* 各节点角色需相应更改

### 1.5 启动Elasticsearch

```shell
root@vmlinux# systemctl daemon-reload
root@vmlinux# systemctl enable elasticsearch
root@vmlinux# systemctl start elasticsearch
```

或者：

```shell
root@vmlinux# cd /usr/shar/elasticsearch/bin
root@vmlinx:/usr/share/elasticsearch/bin# service elasticsearch start
```

此处请使用`service elasticsearch start`，而不是`./elasticsearch`，否则，可能会报错：`NoSuchFileException:/usr/share/elasticsearch/config`。

### 1.6 测试Elasticsearch

打开浏览器测试访问`http://node_ip:9200` 。如果Elasticsearch安装正常，将看到类似下面的信息。

``` json
{
  "name" : "ek_md1",
  "cluster_name" : "oversea_hk_efk",
  "cluster_uuid" : "HTgaF6EYRl6ZOMQY77DmzA",
  "version" : {
    "number" : "6.2.3",
    "build_hash" : "c59ff00",
    "build_date" : "2018-03-13T10:06:29.741383Z",
    "build_snapshot" : false,
    "lucene_version" : "7.2.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

### 1.7 重新配置Elasticsearch

如果需要修改配置文件：

```shell
$ vim /etc/elasticsearch/elasticsearch.yml
```

```yaml
new configuration...
```

重启生效：

```shell
$ systemctl restart elasticsearch
```

## 2. 安装Kibana

### 2.1 Kibana简介

Kibana是一个开源的分析与可视化平台，设计出来用于和Elasticsearch一起使用的。你可以用kibana搜索、查看、交互存放在Elasticsearch索引里的数据，使用各种不同的图表、表格、地图等kibana能够很轻易地展示高级数据分析与可视化。Kibana让我们理解大量数据变得很容易。它简单、基于浏览器的接口使你能快速创建和分享实时展现Elasticsearch查询变化的动态仪表盘。

### 2.2 安装Kibana

下面按照elasticsearch官网的指南使用rpm包安装，也可以通过其它方式。  

Debian, Ubuntu等基于Debian的系统(官网指南)：：[Install Kibana with Debian Package](https://www.elastic.co/guide/en/kibana/current/deb.html)  

Red Hat, Centos, SLES, OpenSuSE等基于RPM的系统(官网指南)：[Install Kibana with RPM](https://www.elastic.co/guide/en/kibana/current/rpm.html)    

__(1) 下载并安装签名公钥__   

```shell
$ rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

__(2) 创建文件`kibana.repo`并修改内容__  

```shell
$ vim /etc/yum.repos.d/kibana.repo

[kibana-6.x]
name=Kibana repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

__(3) 安装Kibana__  

```shell
$ yum install -y kibana
```

> 这是通过yum方式安装，也可以使用rpm包离线安装。  
> `$ wget https://artifacts.elastic.co/downloads/kibana/kibana-6.2.3-x86_64.rpm`  
> `$ rpm --install kibana-6.2.3-x86_64.rpm`  

__(4) 修改Kibana的运行日志目录权限__  

```shell
# 若日志目录不存在，请创建并修改权限
$ ll /var/log
$ mkdir /var/log/kibana
$ chown kibana.kibana /var/log/kibana
```

__(5) 修改kibana配置文件__  

```shell
$ vim /etc/kibana/kibana.yml

server.name: "my_kibana_tmp"    # Kibana名任取
server.host: "0.0.0.0"          # 请设为0.0.0.0，以使外部网络可访问
elasticsearch.url: "http://elasticsearch_master_ip:port"  # Elasticsearch集群任一主节点地址
logging.dest: /var/log/kibana/kibana.log   # Kibana运行日志路径
```

### 2.3 启动Kibana
使用下列命令启动Kibana，可以查看启动后的即时日志。

```shell
root@vmlinux# systemctl daemon-reload
root@vmlinux# systemctl enable kibana.service
root@vmlinux# systemctl start kibana
```

如果出现启动冲突，可以通过`systemctl stop kibana`来停止进程。

### 2.4 测试Kibana

打开浏览器测试访问[http://localhost:5601](http://localhost:5601) 。 如果Elasticsearch安装正常，将看到Kibana可视化平台界面。

### 2.5 重新配置Kibana

如果需要修改配置文件：

```shell
$ vim /etc/kibana/kibana.yml
```

```yaml
new configuration...
```

重启生效：

```shell
$ systemctl restart kibana
```