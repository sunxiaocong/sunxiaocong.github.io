---
layout: post
title: "go goroutine 和 channel"
date: 2018-08-22
categories: go
tags: [go]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
go goroutine 和 channel 并发和通道是go语言的两个特点。
<!-- more -->
### goroutine
* go func() 实现并发  
```
    Go允许使用go语句开启一个新的运行期线程， 
    即goroutine，以一个不同的、新创建的 goroutine来执行一个函数。 
    同一个程序中的所有goroutine共享同一个地址空间。
    go func() { 
        // do something in one new goroutine
    }()
```

### channel
* channel通道作用  
```
    通道(channel)是用来传递数据的一个数据结构。
    通道可用于两个goroutine之间通过传递一个指定类型的值来同步运行和通讯。
    操作符<-用于指定通道的方向，发送或接收。如果未指定方向，则为双向通道。
    
    声明通道:
    ch := make(chan int, size)
    
    发送数据和取数据
    ch <- value
    int v := <-ch
```
