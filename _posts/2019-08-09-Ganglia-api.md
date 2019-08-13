---
layout: post
title: "Ganglia-api"
date: 2019-08-09
categories: docker
tags: [docker,flume]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
ganglia-api添加到ganglia镜像中
<!-- more -->
### ganglia-api
参考github项目readme.md,要添加ganglia监控api大概要实现如下几个步骤。  
1. 创建ganglia_api.py 和 settings.py 两个文件,并将代码复制进去。  
2. 创建如下三个文件并将对应内容复制进去
    ~~~
    gmetad-PROD.conf   # => xml_port 8651, interactive_port 8652
    gmetad-STAGE.conf  # => xml_port 8751, interactive_port 8752
    gmetad-DEV.conf    # => xml_port 8851, interactive_port 8852
    ~~~
3. 安装tornado  
    pip install --no-cache-dir --trusted-host mirrors.aliyun.com -i http://mirrors.aliyun.com/pypi/simple/ tornado==3.2.1
    建议指定源和版本，最新版本在2.7.15并不兼容。不要问我为什么知道。
4. 执行ganglia_api.py  
    python ganglia.py  

经过测试，ganglia_api可以正常工作。  
http://localhost:8080/ganglia/api/v2/metrics  
ok，一切非常完美。那么如何集成到镜像中呢？

### python2.6 升级到 python2.7
有看过之前ganglia image rebuild的肯定知道，dockerfile使用的基础镜像是centos6里面自带了python2.6.由于我们ganglia-api服务使用到了tornado，必须使用的是python2.7 or python3.5。所以要升级到2.7的话，我想到的方式一开始是直接替换基础镜像为centos7，但是这样可能对后续的服务有影响，没深入研究过可不可行。反正我是使用原有基础镜像的基础上更新到python2.7。

更新步骤:
1.  yum -y install gcc openssl-devel bzip2-devel (安装相关依赖，一定要做！！！)
2.  cd /opt  
    wget https://www.python.org/ftp/python/2.7.15/Python-2.7.15.tgz  
3.  tar xvzf Python-2.7.15.tgz
4.  cd Python-2.7.15  
    ./configure --enable-optimizations  
    make && make altinstall
5.  mv /usr/bin/python /usr/bin/python2.6.6  
    ln -s /usr/local/bin/python2.7 /usr/bin/python  
6.  sed -i 's/python/python2.6.6/' /usr/bin/yum  
7.  python -m ensurepip  
8.  pip install --no-cache-dir --trusted-host mirrors.aliyun.com -i http://mirrors.aliyun.com/pypi/simple/ tornado==3.2.1

写成dockerfile就是这个样子！！！
~~~
    # add python2.7.15 for ganglia-api
    RUN cd /opt && \
    wget https://www.python.org/ftp/python/2.7.15/Python-2.7.15.tgz && \
    yum -y install gcc openssl-devel bzip2-devel && \
    tar xvzf Python-2.7.15.tgz && \
    cd Python-2.7.15  && \
    ./configure --enable-optimizations  && \
    make && make altinstall  && \
    mv /usr/bin/python /usr/bin/python2.6.6 && \
    ln -s /usr/local/bin/python2.7 /usr/bin/python  && \
    sed -i 's/python/python2.6.6/' /usr/bin/yum   && \
    python -m ensurepip  && \
    rm -f /opt/Python-2.7.15.tgz && \
    pip install --no-cache-dir --trusted-host mirrors.aliyun.com -i http://mirrors.aliyun.com/pypi/simple/ tornado==3.2.1

~~~


### 集成到ganglia镜像
集成到ganglia并不难，没编写过同时运行多个进程的dockerfile，稍微研究了一下发现并不难。  
启动多个服务时，还可以用ENTRYPOINT执行一个脚本，在脚本中启动多个服务  
~~~
ENTRYPOINT ["/opt/hrms/run/entrypoint.sh"]
~~~
entrypoint.sh脚本文件
~~~
    #!/bin/bash
    
    python /etc/ganglia/ganglia_api.py  & 
    # 启动其他服务or脚本
    server xx start &
    # 启动其他服务or脚本
    server xx start 
~~~
注意启动不同进程间用&分别运行。  
至此，dockerfile的编写也就不难了,具体的内容就不贴出来。

### api路由整理
这个暂时不做

### 相关链接
[github 项目地址](https://github.com/guardian/ganglia-api)  
[docker 利用CMD或者ENTRYPOINT命令同时启动多个服务](https://blog.csdn.net/bocai_xiaodaidai/article/details/92641534)
[centos6.5安装/升级到python2.7](https://blog.csdn.net/bang152101/article/details/88539640)