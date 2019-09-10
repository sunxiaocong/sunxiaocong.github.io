---
layout: post
title: "Ganglia image rebuild"
date: 2019-08-07
categories: Docker
tags: [Docker,Flume]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
ganglia image rebuild   
feat : add env GANGLIA_HOST 
<!-- more -->

### 主要实现目的
为了部署方便，直接通过配置环境变量GANGLIA_HOST，将部署的主机ip写入配置文件gmetad.conf和gmond.conf
### 目录结构
    ├── Dockerfile
    ├── gmetad.conf
    ├── gmond.conf
    └── start.sh


### DOCKERFILE
DOCKER FILE COME FROM:https://hub.docker.com/r/wookietreiber/ganglia/dockerfile

~~~
    FROM centos:centos6
    
    MAINTAINER wookietreiber
    
    # base system upgrade and system dependencies
    RUN yum upgrade -y && \
        yum install -y \
          gcc-c++ automake make \
          apr-devel expat-devel rrdtool-devel zlib-devel \
          httpd php rsync wget tar && \
        yum clean all
    
    # pcre dependency
    RUN cd /usr/src && \
        wget -q http://sourceforge.net/projects/pcre/files/pcre/8.33/pcre-8.33.tar.gz/download -O pcre-8.33.tar.gz && \
        tar xzf pcre-8.33.tar.gz && \
        cd pcre-8.33 && \
        ./configure --prefix=/usr && \
        make && make install && ldconfig && \
        rm -rf /usr/src/pcre-8.33*
    
    # confuse dependency
    RUN cd /usr/src && \
        wget -q http://savannah.nongnu.org/download/confuse/confuse-2.7.tar.gz && \
        tar xzf confuse-2.7.tar.gz && \
        cd confuse-2.7 && \
        ./configure --prefix=/usr --enable-shared && \
        make && make install && ldconfig && \
        rm -rf /usr/src/confuse-2.7*
    
    # ganglia-core
    RUN cd /usr/src && \
        wget -q http://downloads.sourceforge.net/ganglia/ganglia-3.6.0.tar.gz && \
        tar xzf ganglia-3.6.0.tar.gz && \
        cd /usr/src/ganglia-3.6.0 && \
        ./configure --prefix=/usr --sysconfdir=/etc/ganglia/ --sbindir=/usr/sbin/ --with-gmetad --enable-gexec --enable-status && \
        make && make install && ldconfig && \
        rm -rf /usr/src/ganglia-3.6.0*
    
    # ganglia-web
    RUN cd /usr/src && \
        wget -q http://downloads.sourceforge.net/ganglia/ganglia-web-3.5.10.tar.gz && \
        tar xzf ganglia-web-3.5.10.tar.gz && \
        mv ganglia-web-3.5.10 /var/www/html/ganglia && \
        cd /var/www/html/ganglia && \
        make install && \
        rm -rf /usr/src/ganglia-web-3.5.10*
    
    # add the ganglia user and group
    RUN useradd -r -U -d /var/lib/ganglia -s /bin/false ganglia
    
    # create the default gmond config file, with the default gmetad cluster name
    RUN gmond -t \
        | sed 's/name = "unspecified"/name = "dc-flume"/' \
        > /etc/ganglia/gmond.conf
    
    # add the start script
    ADD start.sh start.sh
    
    copy . /etc/ganglia
    # entrypoint is the start script
    ENTRYPOINT ["bash","start.sh"]
    
    # default is with gmond for seeing something
    CMD ["--with-gmond"]
~~~


### gmetad.conf
    data_source "dc-flume" GANGLIA_HOST


### gmond.conf (只贴出修改的部分)
    cluster {
      name = "dc-flume"
      owner = "unspecified"
      latlong = "unspecified"
      url = "unspecified"
    }
    udp_send_channel {
      # mcast_join = 239.2.11.71
      port = 8649
      ttl = 1
    }
    
    udp_recv_channel {
      port = 8649
      bind = GANGLIA_HOST
      retry_bind = true
    }

### start.sh (只贴出新增的内容)
    sed -i "s#GANGLIA_HOST#${GANGLIA_HOST}#g" /etc/ganglia/gmetad.conf
    sed -i "s#GANGLIA_HOST#${GANGLIA_HOST}#g" /etc/ganglia/gmond.conf
    

### 部署
    运行默认配置
    docker run -p 0.0.0.0:80:80 ganglia
    
    查看运行帮助
    docker run ganglia --help
    
    替换自己的配置文件运行
    docker run -v /path/to/conf:/etc/ganglia -p 0.0.0.0:80:80 ganglia
    
    一般最基础的运行命令如下
    docker run -rm \
      -name ganglia \
      -h my.fqdn.org \
      -v /path/to/conf:/etc/ganglia \
      -v /path/to/ganglia:/var/lib/ganglia \
      -p 0.0.0.0:80:80 \
      ganglia
      --timezone Continent/City
    

### 总结
修改比较顺利，一遍就通过。