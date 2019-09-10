---
layout: post
title: "http tcp/ip socket区别"
date: 2018-08-30
categories: Socket
tags: [Socket]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
进一步加深理解7层网络模式。
<!-- more -->
### 7层网络模型
有网络基础的都知道网络由上到下有物理层、数据链路层、网络层、传输层、会话层、表示层和应用层。ip协议属于网络层，tcp协议属于传输层，http协议属于应用层。socket就是ip/tcp协议的封装（对于程序员来说）。
言简意赅，TPC/IP协议是传输层协议，主要解决数据如何在网络中传输，而HTTP是应用层协议，主要解决如何包装数据。

用我自己理解，数据传输好比寄一封加密（http协议）信，tcp/ip协议想当于信上的地址。邮差（网络上的路由）才能知道送到什么地址。而接受信件的人按照我们预定的协议将信的内容解读出来。

### socket的本质
socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议。
 

### tcp 三次握手（面试被问烂大街的题）
第一次握手: 客户端发送syn包到服务端，并进入SYN_SEND状态。
第二次握手：服务端接受到客户端的syn包，同时自己也发送一个syn包。即SYN+ASK包。服务端进去SYN_REVE状态
第三次握手：客户端接受到syn+ask包，再次向服务端发送syn确认包。服务端客户端都进去ESTABLISHED状态

### HTTP链接的特点
http链接特点是每次请求都需要服务端response。请求结束后主动释放链接。（短连接）post get等请求也称为http请求，使用http协议。

### tcp 和 udp 的区别
tcp有建立链接通讯，upd是无状态的不建立链接。udp不确认数据是否发送成功，因此udp容易丢包。但是udp传输效率高（由于缺少三次握手和数据验证）


### 总结
整理完这些对这些协议有更深一点的了解。。。