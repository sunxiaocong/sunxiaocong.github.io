---
layout: post
title: "python  atexit函数"
date: 2018-09-14
categories: python
tags: [python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
python  atexit函数妙用
<!-- more -->
### 前言
接手了一个flask web中间件服务,其中嵌入了一个发送日志的sdk，由于sdk用到内存队列暂存日志。在中间件升级（通过rancher部署升级）的时候，通过捕获信号的方式无法获取退出的信号导致无法正常执行清理缓存队列的stop函数。至于为什么无法正常捕获，研究到后面发现了原因，比较复杂。


### atexit简介
atexit是标准库中一个简单的小模块，作用是提供退出的回调。 如果有什么事是在退出时需要做的，就可以使用它。 但仅限于比较简单的程序。 实际上，最好不要使用它。  
atexit其实由来已久，属于上个世纪的设计。 它不仅是Python标准库的一部分，其实最初是Unix系统的一个机制，在C、C++中也存在。  
Python里atexit，和Unix系统是同一个机制。 在程序的任何地方都可以注册一个或多个回调函数，在程序退出时执行。   

### 官方示例
常规用法：

    def goodbye(name, adjective):
    print('Goodbye, %s, it was %s to meet you.' % (name, adjective))
    
    import atexit
    atexit.register(goodbye, 'Donny', 'nice')
    # or:
    atexit.register(goodbye, adjective='nice', name='Donny')

由于register函数返回的是func，亦可以用于装饰器
    
    import atexit

    @atexit.register
    def goodbye():
        print("You are now leaving the Python sector.")

### 注意
以上是Python 3的文档。  
atexit自Python 2.0加入，但在Python 2.x中，不包含unregister()函数。也就是只能注册，无法注销。
        

### 相关链接
[atexit官方文档](https://docs.python.org/zh-cn/3/library/atexit.html)  

### 总结
虽然官方也不推荐使用register函数，但是在某一个特定场合还是非常好用。
在注册stop函数之后，在升级服务时，正常执行了退出清理内存队列的操作。