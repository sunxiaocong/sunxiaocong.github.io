---
layout: post
title: "rancher 部署ipython-notebook.md"
date: 2019-06-15
categories: docker
tags: [docker,python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
对于数据工程师来说ipython/notebook是常用的编写python的工具
<!-- more -->
### 如何快速部署
部署ipython-notebook当然不难，不过为了办公方便，在家和在公司都能使用同一个环境。研究了一下在rancher上部署ipyhon。
在docker hub 上查找了一下ipython/notebook很快就找到了镜像。看来一下overview基本上就清楚如何部署。在服务器上直接如下命令即可。
    
    # 拉去镜像
    docker pull ipython/notebook
    
    # 启动镜像
    docker run -d -p 80:8888 -e "PASSWORD=MakeAPassword" -e "USE_HTTP=1" ipython/notebook
    
当然在rancher上部署也容器，在启动容器的时候注意配置上PASSWORD=，USE_HTTP=这两个环境变量即可。

### 访问
    
    https://ip:80
    
### 总结
一些常用的工具通过docker部署之后共享给公司其他人用非常方便。