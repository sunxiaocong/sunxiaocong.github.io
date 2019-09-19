---
layout: post
title: "python psycopg2安装问题记录"
date: 2019-01-02
categories: Python
tags: [Python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
记录windows和ubuntu下安装使用psycopg2
<!-- more -->
### 环境
windows: python3  
ubuntu: python2
### 问题
安装后使用出现如下问题:  

    ImportError: No module named 'psycopg2._psycopg'   
    
在windows上安装时直接使用pip install psycopg出现如上的错误。  
尝试了pip uninstall psycopg 卸载安装然后重试安装，依旧是同样的错误。查了很久发现安装时使用-U:  

    pip install -U psycopg 

安装时更新最新版。That is OK！！！！  


那么在ubuntu上安装呢。尝试了pip install psycopy2同样出现问题，安装报错！！！当然经过查询发现:  

    apt-get install python-psycopg2 

就ok了，使用也不出现问题。

### 总结
环境问题很头疼
