---
layout: post
title: "Rsyslog Dockerfile 配置"
date: 2019-02-16
categories: 大数据
tags: [docker,rsyslog]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，
将Rsyslog打成一个镜像将会非常方便的进行部署，只需要在配置时将Elasticsearch地址写在对应环境变量即可
<!-- more -->

### Dockerfile  

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

### docker-entrypoint 
用于将conf ELASTIC地址转换为对应环境变量
```bash 
    #!/bin/sh
    
    set -e
    
    sed -i "s#ELASTIC_HOSTS#${ELASTIC_HOSTS}#g" /etc/rsyslog.conf
    sed -i "s#INDEX_TAG#${INDEX_TAG}#g" /etc/rsyslog.conf
    
    exec "$@"

```