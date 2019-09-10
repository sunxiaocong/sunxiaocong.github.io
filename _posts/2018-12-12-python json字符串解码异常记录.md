---
layout: post
title: "python json字符串解码异常记录"
date: 2018-12-12
categories: Python
tags: [Python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
python 
<!-- more -->
### 问题描述
在文本文件中，存储的json格式如下
    
    {'role_id':'xx','num':0}

读出来后进行json.loads出现异常。
    
    ValueError: Expecting property name: line 1 column 2 (char 1)
    
google之后发现是就是由于JSON中，标准语法中，不支持单引号，属性或者属性值，都必须是双引号括起来的。

    {'role_id':'xx','num':0}.replace("'", '"')
    
处理之后成功解码！！！

### 总结
解码异常类型比较多，还有u前缀的字符串也会出现异常，处理方式同样暴力。具体原来还不清楚~
