---
layout: post
title: "Rancher 搭建基础设施nfs"
date: 2019-06-22
categories: nfs
tags: [nfs]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

本文主要记录rancher 搭建nfs基础服务的过程！


<!-- more -->


### 为何要搭建nfs基础服务？
~~~
在此之前，要在容器中使用nfs服务我的操作过程是这样的！

1.进入容器所在的主机，安装nfs所需的包。
    apt-get install nfs-common
    或者
    yum install nfs-utils

2. 挂载命令
    sudo mount -t nfs nfs地址:/nfs上的目录 /你的本地目录

3. 在rancher 上挂载主机的目录.
    这样就实现了将容器中的目录挂载到nfs的目录上！是不是很麻烦！
    非常讨厌这样的重复工作！！！！！！！！！！
    这应该是所有程序员的通病吧！！！！！讨厌重复！！！！！！
    ok，直接将如何部署nfs基础服务！！！！以及如何使用！！！！
~~~

### 在rancher上搭建nfs基础服务步骤，直接上图！
ps：必须是环境的管理员才有权限创建基础设施  
第一步：点击应用商店，搜索nfs。如下图所示
![图一](https://raw.githubusercontent.com/sunxiaocong/sunxiaocong.github.io/master/images/nfs/1.PNG)

第二步：在nfs server 填nfs地址  
        在export base dir 配置nfs的文件夹路径  
        在mount option 配置参数 
        完事，点击启动
![图二](https://raw.githubusercontent.com/sunxiaocong/sunxiaocong.github.io/master/images/nfs/2.PNG)

nfs基础设施搭建完。容器挂载nfs如下.   
nfs路径:/容器路径  
/主机路径:/容器路径  
两种方式的区别在于开始是否有/ 
![图三](https://raw.githubusercontent.com/sunxiaocong/sunxiaocong.github.io/master/images/nfs/3.PNG)

### 总结
这样的挂载方式，直接将nfs挂载到容器内部的目录下。十分便于添加新节点无需繁琐的操作！！！！

