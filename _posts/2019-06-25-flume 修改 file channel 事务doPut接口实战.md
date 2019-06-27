---
layout: post
title: "flume 修改 file channel 事务doPut接口实战"
date: 2019-06-25
categories: flume
tags: [flume]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
flume 自定义channel开发。
<!-- more -->

### 需求
~~~
修改的初衷：
由于业务需求,在海外服务的日志通过flume spooldir 的方式收集到flume上，并通过flume到flume的方式对传回到国内存储起来提供后续的离线分析。由于日志格式是json格式，并且有日志量大，单条日志大的特点。
给到我的需求是,在source到channel这一过程将event 中不需要的key剔除。
第一想到的是拦截器，but对拦截器一点想法都没有。
灵光一现，实现了如下这个方案。
~~~

### 官方git源码
~~~
git地址 : https://github.com/apache/flume 克隆对应版本flume1.9.0的代码到本地
~~~

### 步骤

#### 克隆 flume-flie-channel
~~~
克隆这个文件夹到自己的项目下.
打开src/java 可以看到如下文件夹目录org.apache.flume.channel.file,修改为自己的项目名称如com.sxc.flume.channel.file。这个可以根据自己的想法修改。
~~~

#### 修改 pom.xml 
~~~
1. 修改生成的包名  
2. 修改依赖的包的版本
3. build 跳过测试用例
(具体参照我修改之后的pom.xml，主要的工作就是添加一些包的版本)
~~~
#### 源码分析
~~~
//FileBackedTransaction继承BasicTransactionSemantics
public abstract class BasicTransactionSemantics implements Transaction {
    private BasicTransactionSemantics.State state;
    private long initialThreadId;

    protected void doBegin() throws InterruptedException {
    }
    protected abstract void doPut(Event var1) throws InterruptedException;

    protected abstract Event doTake() throws InterruptedException;

    protected abstract void doCommit() throws InterruptedException;

    protected abstract void doRollback() throws InterruptedException;

    protected void doClose() {
    }

FileChannel的内部事务类 -- FileBackedTransaction
所以无论get还是put数据都要获取这个事务。source调用doPut将event写入channel，在doPut这个接口实现剔除逻辑即可
~~~

### 添加配置removeKeys
~~~
添加配置的方式非常简单,只需要在FileChannelConfiguration增加如下两行
    public static final String REMOVE_KEYS = "removeKeys";
    public static String DEFAULT_REMOVE_KEYS = null;
配置项removeKeys的结构是json结构，如{"table_name":["removeKey1","removeKey2"]}。首先解释一下table_name的含义，在event的header中有标记这个event是属于哪一张表的字段，通过这个标记来判断是否需要剔除body里面的数据，removeKey1就是需要剔除的key值。
~~~

#### 修改doPut在event写入channel前进行处理
~~~
@Override
protected void doPut(Event event) throws InterruptedException {
    channelCounter.incrementEventPutAttemptCount();
    if (putList.remainingCapacity() == 0) {
        throw new ChannelException("Put queue for FileBackedTransaction " +
                "of capacity " + putList.size() + " full, consider " +
                "committing more frequently, increasing capacity or " +
                "increasing thread count. " + channelNameDescriptor);
    }
    // this does not need to be in the critical section as it does not
    // modify the structure of the log or queue.
    if (!queueRemaining.tryAcquire(keepAlive, TimeUnit.SECONDS)) {
        throw new ChannelFullException("The channel has reached it's capacity. "
                + "This might be the result of a sink on the channel having too "
                + "low of batch size, a downstream system running slower than "
                + "normal, or that the channel capacity is just too low. "
                + channelNameDescriptor);
    }
    boolean success = false;
    log.lockShared();

    // -------------------------获取剔除的表名，以及剔除的字段----------------------------------
    if (this.removeKeys != null) {
        Map<String, String> headersMap = event.getHeaders();
        for (Object key : this.removeKeys.keySet()) {
            if (headersMap.containsKey("splitBaseName0") && headersMap.get("splitBaseName0").contains(key.toString())) {
                try {
                    String actLog = new String(event.getBody());
                    JSONObject jsonObj = JSONObject.parseObject(actLog);
                    for (Object value : this.removeKeys.getJSONArray(key.toString())) {
                        if (jsonObj.containsKey(value.toString())) {
                            jsonObj.remove(value.toString());
                        }
                    }
                    event.setBody(jsonObj.toString().getBytes());
                } catch (JSONException e) {
                    e.printStackTrace();
                    System.out.println("error: " + e.toString());
                }
            }
        }
    }
    //-------------------------------------剔除逻辑添加完毕-----------------------------------
    try {
        FlumeEventPointer ptr = log.put(transactionID, event);
        Preconditions.checkState(putList.offer(ptr), "putList offer failed "
                + channelNameDescriptor);
        queue.addWithoutCommit(ptr, transactionID);
        success = true;
    } catch (IOException e) {
        channelCounter.incrementEventPutErrorCount();
        throw new ChannelException("Put failed due to IO error "
                + channelNameDescriptor, e);
    } finally {
        log.unlockShared();
        if (!success) {
            // release slot obtained in the case
            // the put fails for any reason
            queueRemaining.release();
        }
    }
}
~~~


### 打包流程
~~~
第一步：使用idea打包这个项目生成一个jar包
第二步：进入apache-flume-1.9.0-bin/plugins.d,新建目录sxc_file_channel ,进入新建的目录，创建lib，libext，native三个文件夹，将第一步打好的jar包放入lib目录下
第三步，测试使用。在flume的配置文件中使用刚才打包好的file_channel。还记得在之前改的名字嘛？在配置中使用如下配置获取FileChannel对应的类。我们就可以使用我们先添加的removeKeys配置了。
a1.channels.c1.type = com.sxc.flume.channel.file.FileChannel
a1.channels.c1.removeKeys = {"table_name":["removeKey1","removeKey2"]}
~~~

### 推荐阅读链接
[flume 自定义source，sink，channel，拦截器](https://blog.csdn.net/qq_36864672/article/details/78663718)
[Flume - FileChannel源码详解](https://blog.csdn.net/qianshangding0708/article/details/48133033)


### 总结
解决方案不是很完美，之后研究一下拦截器的用法。




