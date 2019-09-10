---
layout: post
title: "java 定时任务timer"
date: 2019-07-24
categories: Java
tags: [Java]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
java timer 定时任务模块
<!-- more -->
### 前言
昨天自己整了一个定时任务，通过sleep的方式，在每个整5分钟的时刻执行任务。简直不能太水，今天发现了timer异步定时任务，感觉实在好用。 

### java timer 简介
java.util.Timer定时器，实际上是个线程，定时调度所拥有的TimerTasks。 
一个TimerTask实际上就是一个拥有run方法的类，需要定时执行的代码放到run方法体内，TimerTask一般是以匿名类的方式创建。 

### java timer 几种调度方式
~~~
    java.util.Timer timer = new java.util.Timer(true);   
    // true 说明这个timer以daemon方式运行（优先级低，   
    // 程序结束timer也自动结束），注意，javax.swing   
    // 包中也有一个Timer类，如果import中用到swing包，   
    // 要注意名字的冲突。   
      
    TimerTask task = new TimerTask() {   
    public void run() {   
    ... //每次需要执行的代码放到这里面。   
    }   
    };   
      
    //以下是几种调度task的方法：   
      
    timer.schedule(task, time);   
    // time为Date类型：在指定时间执行一次。   
      
    timer.schedule(task, firstTime, period);   
    // firstTime为Date类型,period为long   
    // 从firstTime时刻开始，每隔period毫秒执行一次。   
      
    timer.schedule(task, delay)   
    // delay 为long类型：从现在起过delay毫秒执行一次   
      
    timer.schedule(task, delay, period)   
    // delay为long,period为long：从现在起过delay毫秒以后，每隔period   
    // 毫秒执行一次。
~~~

### 我的实现
实现每整五分钟定时任务，我是用的是这个接口。  
timer.schedule(task, delay, period)    
// delay为long,period为long：从现在起过delay毫秒以后，每隔period   
// 毫秒执行一次。

~~~
        Timer timer = new Timer();
        timer.schedule(new java.util.TimerTask() {
            @Override
            public void run() {
                // do sometime
            }
        }, delay_time, 1000 * 60 * 5);
        
~~~
delay_time 是计算服启动时间到下一个整5分钟时间的间隔时间。
这样就实现了一个异步定时任务。每个整点5分钟的时刻执行任务

### 总结
通过timer包实现的定时任务比昨天的实现方式要高大上许多。  
生命不息，迭代不止》》》》》》》