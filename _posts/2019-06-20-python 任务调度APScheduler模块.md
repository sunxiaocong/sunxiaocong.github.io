---
layout: post
title: "python 任务调度APScheduler模块"
date: 2019-06-20
categories: python
tags: [python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
APScheduler模块使用总结
<!-- more -->
### 安装
    pip install apscheduler
    
### 三种模式
* cron 根据参数匹配上的时间执行     
* interval 每隔一定间隔执行一次
* data 再某一时间点执行一次
    
#### date 在某个时间执行一次  
示例:

    scheduler.add_job(dateJob, 'date', run_date=date(2009, 11, 6), args=['text'])
    scheduler.add_job(dateJob, 'date', run_date=datetime(2019, 1, 25, 16, 47, 0), args=['text'])
     # 立即执行，'date', run_date=datetime.now()，是默认的  
    scheduler.add_job(dateJob, args=['text']) 

#### interval
如下参数可选:

    weeks (int) – number of weeks to wait
    days (int) – number of days to wait
    hours (int) – number of hours to wait
    minutes (int) – number of minutes to wait
    seconds (int) – number of seconds to wait
    start_date (datetime|str) – starting point for the interval calculation
    end_date (datetime|str) – latest possible date/time to trigger on
    timezone (datetime.tzinfo|str) – time zone to use for the date/time calculations 

示例:

    # start_date 的时刻启动，并没两个小时执行一次
    scheduler.add_job(intervalJob, 'interval', hours=2, start_date='2019-01-10 09:30:00', end_date='2019-06-15 11:00:00)
    # 立刻执行，并在之后每两个小时执行一次
    scheduler.add_job(intervalJob, 'interval', hours=2, next_run_time=datetime.datetime.now())    
    
#### cron
如下参数可选:

    year (int|str) – 4-digit year
    month (int|str) – month (1-12)
    day (int|str) – day of the (1-31)
    week (int|str) – ISO week (1-53)
    day_of_week (int|str) – number or name of weekday (0-6 or mon,tue,wed,thu,fri,sat,sun)
    hour (int|str) – hour (0-23)
    minute (int|str) – minute (0-59)
    second (int|str) – second (0-59)
    start_date (datetime|str) – earliest possible date/time to trigger on (inclusive)
    end_date (datetime|str) – latest possible date/time to trigger on (inclusive)
    timezone (datetime.tzinfo|str) – time zone to use for the date/time calculations (defaults to scheduler timezone)   
    
    参数配置的值说明：eg  day= '*' ,hour=x,y,z 
    ---------------------------------------------------------------------
    *	    任意	任意值
    */a	    任意	从最小值开始，触发每个值
    a-b	    任意	触发a-b范围内的任何值（必须小于b）
    a-b/c	任意	触发a-b范围内的每个c值
    xth y	day	    在该月内的工作日y的第x次出现时触发
    last x	day	    在工作日x的最后一次出现在月内
    last	day	    在这个月的最后一天
    x,y,z	任意	触发任何匹配的表达式；可以组合任意数量的任何上述表达式
    ---------------------------------------------------------------------

示例:

    # 在6-8,11-12月的第三个星期五的 00:00, 01:00, 02:00, 03:00 运行
    scheduler.add_job(job_function, 'cron', month='6-8,11-12', day='3rd fri', hour='0-3')
    # 周一至周五上午5:30 至 2014-05-30 00:00:00
    sched.add_job(job_function, 'cron', day_of_week='mon-fri', hour=5, minute=30, end_date='2014-05-30')

### 完整代码
    
    from apscheduler.schedulers.blocking import BlockingScheduler

    scheduler = BlockingScheduler()
    
    # 装饰器的方式
    @scheduler.scheduled_job("interval", seconds=5,id='test1')
    def test():
        print ('running')
    
    # 常规方式
    def test2():
        print('running test2')
    
    scheduler.scheduled_job(test2,"interval", seconds=5, id='test2')    
    
    # 移除任务
    scheduler.remove(id='test1')
    
### 高级用法
    #todo
    补充事件监听等   

### 相关文档
[官方文档](https://apscheduler.readthedocs.io/en/v3.3.0/index.html)  
[GitHub](https://github.com/agronholm/apscheduler)