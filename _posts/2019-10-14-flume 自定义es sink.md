---
layout: post
title: "flume 自定义es sink"
date: 2019-10-14
categories: Flume
tags: [Flume]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
自定义flume sink
<!-- more -->
### 概述
本项目主要实现自定义flume sink输出日志到elasticsearch集群。通过基于HTTP协议，以JSON为数据交互格式的RESTful API。其他所有程序语言都可以使用RESTful API，通过9200端口的与Elasticsearch进行通信。

### 自定义sink demo
最简单的输出sink demo，主要是继承了AbstractSink,部分继承Configurable为了获取flume配置文件中配置的内容。在主要的处理逻辑中，先获取channel，以及事务getTransaction，根据处理的情况选择回滚还是提交事务，以及返回本次处理的状态。

    public class PrintSink extends AbstractSink implements Configurable {
        private static final Logger LOG = Logger.getLogger(PrintSink.class);
        private int batchSize;
        private SinkCounter sinkCounter;  //监控计数
    
        //获取flume配置信息
        @Override
        public void configure(Context context) {
                this.batchSize = context.getInteger("batchSize", 100);
                if (sinkCounter == null) {
                    sinkCounter = new SinkCounter(getName());
                }
            }
        
            // sink 启动准备工作
        @Override
        public void start() {
            sinkCounter.incrementConnectionCreatedCount();
            sinkCounter.start();
        }
    
        // sink 关闭清理资源操作
        @Override
        public void stop() {
            sinkCounter.stop();
        }
    
        // 处理逻辑
        @Override
        public Status process() {
            Status status;
            // Start transaction
            Channel ch = getChannel();
            Transaction txn = ch.getTransaction();
            txn.begin();
            try {
                Event event;
                int count;
                sinkCounter.incrementEventDrainAttemptCount();
                for (count = 0; count <= batchSize; ++count) {
                    event = ch.take();
                    if (event == null) {
                        break;
                    }
                    System.out.println(event.getBody());  // 显示event body内容
                    sinkCounter.incrementConnectionCreatedCount();
                }
                return Status.READY;
            } catch (Throwable t) {
                txn.rollback();
                LOG.info(t.getMessage());
                status = Status.BACKOFF;
                if (t instanceof Error) {
                    throw (Error) t;
                }
            } finally {
                txn.close();
            }
            return status;
        }
    }

### es restful api
其他所有程序语言都可以使用RESTful API，通过9200端口的与Elasticsearch进行通信。  
向Elasticsearch发出的请求的组成部分与其它普通的HTTP请求是一样的:  
    
    curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'


* VERB HTTP方法：GET, POST, PUT, HEAD, DELETE
* PROTOCOL http或者https协议（只有在Elasticsearch前面有https代理的时候可用）
* HOST Elasticsearch集群中的任何一个节点的主机名，如果是在本地的节点，那么就叫localhost
* PORT Elasticsearch HTTP服务所在的端口，默认为9200
* PATH API路径（例如_count将返回集群中文档的数量），PATH可以包含多个组件，例如_cluster/stats或者_nodes/stats/jvm
* QUERY_STRING 一些可选的查询请求参数，例如?pretty参数将使请求返回更加美观易读的JSON数据
* BODY 一个JSON格式的请求主体（如果请求需要的话）   



举例说明，为了计算集群中的文档数量，我们可以这样做

    curl -XGET 'http://localhost:9200/_count?pretty' -d '
    {
        "query": {
            "match_all": {}
        }
    }'
 
因此可以通过post请求的方式将日志输出到es集群之中，并且不受版本的限制。

### http post 模块
    
    /**
     * 利用HttpClient进行post请求的工具类
     */
    public class HttpClientUtil {
        @SuppressWarnings("resource")
        public static Boolean httpPostWithJSON(String url, String json) {
            try {
                HttpPost httpPost = new HttpPost(url);
                CloseableHttpClient client = HttpClients.createDefault();
                StringEntity entity = new StringEntity(json, "utf-8");//解决中文乱码问题
                entity.setContentEncoding("UTF-8");
                entity.setContentType("application/json");
                httpPost.setEntity(entity);
                HttpResponse resp = client.execute(httpPost);
                if (resp.getStatusLine().getStatusCode() == 200) {
                    HttpEntity httpEntity = resp.getEntity();
                    String body = EntityUtils.toString(httpEntity, "UTF-8");
                    JSONObject bodyObj = JSONObject.parseObject(body);
                    if (!bodyObj.getBoolean("errors")) {
                        return true;
                    } else {
                        System.out.println(bodyObj);
                        // todo send alarm to let me know
                    }
                }
            } catch (Exception e) {
                System.out.println(e);
            }
            return false;
        }
    }
    
### body封装
post 批发数据,数据结构如下。第一天json是数据index信息，第二条是数据实体内容。

     private String addEventToBatch(Event event, String indexName, String timeStamp) {
            String str = "{ \"index\" : { \"_index\" : \"" + event.getHeaders().get(indexName) + "-" + timeStamp +
                    "\", \"_type\" : \"fluentd\" } }";
            String jsonObj = new String(event.getBody());
            return str + "\n" + jsonObj + "\n";
        }

### 请求路径封装
根据配置的master节点地址,随机获取发送节点，拼接出请求路径。

    private String getHostByRandom() {
        int index = (int) (Math.random() * this.esHosts.size());
        return "http://" + this.esHosts.get(index) + "/_bulk";
    }
### 总结
基础模块基本是介绍完了。