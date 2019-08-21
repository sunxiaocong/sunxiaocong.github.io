---
layout: post
title: "go 缓冲channel和非缓冲channel的区别"
date: 2018-09-26
categories: go
tags: [go]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
在Go中我们make一个channel有两种方式，分别是有缓冲的和没缓冲的
<!-- more -->
### 非缓冲和缓冲
* 缓冲channel即buffer channel创建方式为make(chan TYPE,SIZE)  
> ch := make(chan int, 10)   // 缓冲通道
* 非缓冲channel即unbuffer channel创建方式为make(chan TYPE)
> ch2 := make(chan int, 0) //非缓冲通道  
> ch := make(chan int)   //非缓冲通道

### 非缓冲和缓冲
* 非缓冲channel，channel发送和接收动作是同时发生的
* 缓冲channel类似一个队列，只有队列满了才可能发送阻塞


### 代码示例
非缓存chan

    func main() {
        ch := make(chan int)
        ch <- 1
        go func(){
            i := <-ch
            println(i)
        }
    }
这样看似没毛病！but运行之后引发异常fatal error: all goroutines are asleep - deadlock! 就是因为 ch<-1 发送了，但是同时没有接收者，所以就发生了阻塞。  
但如果我们把 ch <- 1 放到go func()下面，程序就会正常运行。  

缓冲chan
    
    func main() {
        ch := make(chan int,1)
        ch <- 1
        ch <- 2
        go func(){
            i := <-ch
            println(i)
        }
        time.Sleep(1 * time.Millisecond)
    }

同样引发异常。what！！！channel的大小为1，而我们要往里面塞2个数据，所以就会阻塞住。  
可以增加chan容量，或者在go func()之后运行写入数据。

### 总结
显式调用close(ch),可以阻塞数据写入，但是还可以取数据。