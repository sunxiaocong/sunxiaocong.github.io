---
layout: post
title: "Rsyslog Dockerfile 配置"
date: 2019-02-16
categories: 大数据
tags: [docker,rsyslog]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

ryslog 是一个快速处理收集系统日志的程序，提供了高性能、安全功能和模块化设计。rsyslog 是syslog 的升级版，它将多种来源输入输出转换结果到目的地，据官网介绍，现在可以处理100万条信息。

<!-- more -->

### Rsyslog Docker  

```shell
FROM centos:7
ENV ELASTIC_HOSTS=[\"http://10.30.25.40:9200\",\"http://10.30.25.41:9200\",\"http://10.30.25.42:9200\"]

ENV INDEX_TAG=["session","frontend","orm","mesh_proxy","5"]

RUN rpm --import http://rpms.adiscon.com/RPM-GPG-KEY-Adiscon
RUN curl http://rpms.adiscon.com/v8-stable/rsyslog.repo -o /etc/yum.repos.d/rsyslog.repo
RUN yum install -y rsyslog-8.35.0-2.el7.centos.x86_64
RUN yum install -y rsyslog-mmjsonparse-8.35.0-2.el7.centos.x86_64
RUN yum install -y rsyslog-elasticsearch-8.35.0-2.el7.centos.x86_64

COPY rsyslog.conf /etc/rsyslog.conf
RUN mkdir -p /data/log
RUN mkdir -p /data/run
RUN touch /var/log/syslog
RUN touch /var/log/rsyslog.log
RUN chmod 777 /var/log/syslog
RUN chmod 777 /var/log/rsyslog.log

COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["/usr/sbin/rsyslogd", "-n"]

```

```bash docker-entrypoint 用于将conf ELASTIC地址转换为对应环境变量
#!/bin/sh

set -e

sed -i "s#ELASTIC_HOSTS#${ELASTIC_HOSTS}#g" /etc/rsyslog.conf
sed -i "s#INDEX_TAG#${INDEX_TAG}#g" /etc/rsyslog.conf

exec "$@"

```