---
layout: post
title: "rancher 部署ipython-notebook.md"
date: 2019-06-15
categories: Docker
tags: [Docker,Python]
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

### 问题
由于镜像安装的python版本是2.7.5 在安装numpy会遇到异常
       
      Collecting numpy
      Using cached https://files.pythonhosted.org/packages/ac/36/325b27ef698684c38b1fe2e546e2e7ef9cecd7037bcdb35c87efec4356af/numpy-1.17.2.zip
        Complete output from command python setup.py egg_info:
        Traceback (most recent call last):
          File "<string>", line 1, in <module>
          File "/tmp/pip-build-ipyhqlik/numpy/setup.py", line 31, in <module>
            raise RuntimeError("Python version >= 3.5 required.")
        RuntimeError: Python version >= 3.5 required.
解决方案
    
    numpy以及scipy的新版本已不再支持python2，但有很多库装的时候依赖numpy和scipy，
    会默认装最新的版本，但最新版本又不支持python2，所以记录下支持python2的版本号：
    pip install numpy==1.16.1
    pip install scipy==1.2.2

### 总结
一些常用的工具通过docker部署之后共享给公司其他人用非常方便。