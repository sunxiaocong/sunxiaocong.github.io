---
layout: post
title: "ElaticSearch查询脚本"
date: 2019-01-22
categories: 大数据
tags: [ElasticSearch,Python]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

在对代码中经常会需要对ElasticSearch的查询，或者需要从ES中下拉数据时，用Kibana的查询就不能够
满足业务，此时可以使用HTTP协议访问ElasticSearch对外暴露的9200端口访问集群。本文主要介绍使用异步
查询和使用scrolAPI完成大量结果的查询
<!-- more -->

## 1. Scoll API 使用

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import json
import rfc3339

from datetime import datetime


import requests
import time
start_time = rfc3339.rfc3339(datetime.strptime("2018-11-02 12:05:00", "%Y-%m-%d %H:%M:%S"))
end_time = rfc3339.rfc3339(datetime.strptime("2018-11-02 12:35:00", "%Y-%m-%d %H:%M:%S"))
print(start_time)
print(end_time)
def make_query():
    # query = '"query":{"bool": {"must":{"match":{"message":"百度OCPC渠道 - 100011 - 激活回调成功"}}}}'
    q1 = {
        "query": {
            "bool": {
                "filter": {
                    "bool": {
                        "must": [{
                            "range": {
                                "@timestamp": {
                                    "gte": start_time,
                                    "lte": end_time
                                }
                            }
                        },
                        ]
                    }
                },
                "must":[{
                    "term":{
                        "act_id":"ad_trace"
                    },
                    "term":{
                        "stat":"297"
                    }}]
            }
        },
        "size": 1000
    }
    # return query
    return json.dumps(q1)


def main(host, tag):
    url = "http://" + host + "/" + tag + "/_search?scroll=10m"
    print(url)
    query_data = make_query()
    print(query_data)
    headers = {'Content-Type': 'application/json'}
    file = open("bak.txt", "w+")
    loop_times = 1
    loop_data_times = 0
    try:
        response = requests.get(url, data=query_data.encode('utf-8'), headers=headers)
        response_data = json.loads(response.text)
        print(response_data)
        total_num = response_data["hits"]["total"]
        datas = response_data["hits"]["hits"]
        scroll_id = response_data["_scroll_id"]
        for d in datas:
            str_d = json.dumps(d)
            file.write(str_d)
            file.write("\n")
            loop_data_times += 1
            print(loop_data_times)
        loop = True
        while loop:
            loop_times += 1
            url2 = "http://" + host  + "/_search/scroll"
            query = {
                "scroll": "10m",
                "scroll_id": '{scroll_id}'.format(scroll_id=scroll_id)
            }
            print("scroll_id ", scroll_id)
            query_data2 = json.dumps(query)
            response = requests.get(url2, data=query_data2.encode('utf-8'), headers=headers)
            print("response.text ", response.text)
            response_data = json.loads(response.text)
            scroll_data = response_data["hits"]["hits"]
            if len(scroll_data) == 0:
                break
            for d in scroll_data:
                str_d = json.dumps(d)
                file.write(str_d)
                file.write("\n")
                loop_data_times += 1
                print(loop_data_times)

        print("total_num ", total_num)
        print("loop_times ", loop_times)
    except Exception as e:
        print(111)
        print(e)


if __name__ == "__main__":
    master_host = "10.50.175.147:9200"
    tag_name = "hwsjus_frontend-2018.11.02"
    main(master_host, tag_name)

```

## 2. aiohttp 异步查询

```python
#!/usr/bin/env python
"""
 Created by Dai at 19-1-14.
"""
import asyncio
import json
from datetime import datetime ,timedelta

import aiohttp
import rfc3339

timetime = datetime.strptime("2019-01-13 08:00:00", "%Y-%m-%d %H:%M:%S")


async def requests(url, query, tag):
    async with aiohttp.ClientSession() as session:
        async with session.get(url=url, data=json.dumps(query), headers={'content-type': 'application/json'}) as resp:
            res = await resp.json()
            print(resp)
            try:
                datas = res.get("hits").get("hits")
                file = open(f"{tag}.txt", "a+")
                for data in datas:
                    source = data.get("_source")
                    message = source.get("message")
                    req = source.get("req")

                    res = {
                        "message": message,
                        "req": req
                    }
                    file.write(json.dumps(res))
                    file.write("\n")

                file.close()
            except Exception as e :
                print(e)
                pass


if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    end = timetime
    tasks = []
    for i in range(1440):
        start = end
        end =  start + timedelta(minutes=1)

        start_time = rfc3339.rfc3339(start)
        end_time = rfc3339.rfc3339(end)

        if i % 100 == 1:
            loop.run_until_complete(asyncio.wait(tasks))
            tasks = []

        query = {
            "query": {
                "bool": {
                    "filter": {
                        "bool": {
                            "must": [{
                                "range": {
                                    "@timestamp": {
                                        "gte": start_time,
                                        "lte": end_time
                                    }
                                }
                            },
                            ]
                        }
                    }
                }
            },
            "size": 167
        }

        tag = "hwsj_frontend-2019.01.13"
        tasks.append(asyncio.ensure_future(requests(url=f"http://10.70.148.254:9200/{tag}/_search",
                                                    query=query, tag=tag)))

```

