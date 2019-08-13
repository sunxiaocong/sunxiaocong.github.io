---
layout: post
title: "flume transaction"
date: 2019-07-01
categories: flume
tags: [flume]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
flume transaction事务是flume非常重要的实现。了解transaction对深入学习flume有非常大的帮助。本文主要分析transaction原理，transaction原理其实与数据库的事务类似。但是也有很大的区别，也是flume不完美的地方。
<!-- more -->

### transaction接口类
    public void begin();
    
    public void commit();
    
    public void rollback();
    
    public void close();
与数据库的接口基本一样。事务的主要作用有两部分,从source去数据写入channel和sink从channel取数据的过程都要使用。接下来介绍一下FileBackedTransaction，也就是file channel事务个个方法的实现原理。


### 获取事务的方法
    @Override
    public Transaction getTransaction() {
        if (!initialized) {
          synchronized (this) {
            if (!initialized) {
              initialize();
              initialized = true;
            }
          }
        }
        // currentTransaction是一个ThreadLocal对象
        BasicTransactionSemantics transaction = currentTransaction.get();
        // 如果是第一次获取事务或者当前事务的已经close。那么会重新create一个新的事务
        if (transaction == null || transaction.getState().equals(
                BasicTransactionSemantics.State.CLOSED)) {
          transaction = createTransaction();
          currentTransaction.set(transaction);
        }
        return transaction;
    }
第一次拿事务或者事务关闭之后，才会重新去构造一个新的事务。各个线程之间的事务都是独立的,source和sink支持的线程数都是可配置的。

### doPut方法
    @Override
    protected void doPut(Event event) throws InterruptedException {
        // 增加channel.event.put.attempt的值,监控中记录尝试写入的数据量
        channelCounter.incrementEventPutAttemptCount();
        // 写入队列可用容量为0，抛出异常。
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
        // 获取共享锁，个个线程之间会争夺这个锁。
        log.lockShared();

        try {
            // 将日志(事务ID，event)写入磁盘缓冲区（还未强制刷盘）
            FlumeEventPointer ptr = log.put(transactionID, event);
            # 将指针写入putList队列中
            Preconditions.checkState(putList.offer(ptr), "putList offer failed "
                    + channelNameDescriptor);
            // 写入内存队列，此时sink还没办法取，因为还未提交事务。
            queue.addWithoutCommit(ptr, transactionID);
            success = true;
        } catch (IOException e) {
            channelCounter.incrementEventPutErrorCount();
            throw new ChannelException("Put failed due to IO error "
                    + channelNameDescriptor, e);
        } finally {
            // 释放共享锁
            log.unlockShared();
            if (!success) {
                // release slot obtained in the case
                // the put fails for any reason
                queueRemaining.release();
            }
        }
    }
    
    
    
queue是FlumeEventQueue的一个实例，对于FileChannel来说，采用的是本地文件作为存储介质，服务器的宕机重启不会带来数据丢失（没有刷盘的数据或磁盘损坏除外），同时为了高效索引文件数据，我们需要在内存中保存写入到文件的Event的位置信息，前面说到Event的位置信息可由FlumeEventPointer对象来存储，FlumeEventPointer是由FlumeEventQueue来管理的，FlumeEventQueue有两个InflightEventWrapper类型的属性：inflightPuts和inflightTakes，InflightEventWrapper是FlumeEventQueue的内部类，专门用来存储未提交Event（位置指针），这种未提交的Event数据称为飞行中的Event（in flight events）。因为这是向Channel的写操作，所以调用InflightEventWrapper的addEvent(Long transactionID,Long pointer)方法将未提交的数据保存在inflightPuts中（inflightEvents.put(transactionID, pointer);）


### doCommit方法
事务的commit，当一个事务提交后，Channel需要告知Source Event已经成功保存了，而Source需要通知写入源已经写入成功，即成功ACK.那么。File Channel的事务提交过程由FileBackedTransaction.doCommit方法来完成，对于Source往Channel的写操作（putList的size大于0）

    protected void doCommit() throws InterruptedException {
            int puts = putList.size();
            int takes = takeList.size();
            if (puts > 0) {
                Preconditions.checkState(takes == 0, "nonzero puts and takes "
                        + channelNameDescriptor);
                log.lockShared();
                try {
                    log.commitPut(transactionID);
                    channelCounter.addToEventPutSuccessCount(puts);
                    synchronized (queue) {
                        while (!putList.isEmpty()) {
                            if (!queue.addTail(putList.removeFirst())) {
                                StringBuilder msg = new StringBuilder();
                                msg.append("Queue add failed, this shouldn't be able to ");
                                msg.append("happen. A portion of the transaction has been ");
                                msg.append("added to the queue but the remaining portion ");
                                msg.append("cannot be added. Those messages will be consumed ");
                                msg.append("despite this transaction failing. Please report.");
                                msg.append(channelNameDescriptor);
                                LOG.error(msg.toString());
                                Preconditions.checkState(false, msg.toString());
                            }
                        }
                        queue.completeTransaction(transactionID);
                    }
                } catch (IOException e) {
                    throw new ChannelException("Commit failed due to IO error "
                            + channelNameDescriptor, e);
                } finally {
                    log.unlockShared();
                }

            } else if (takes > 0) {
                log.lockShared();
                try {
                    log.commitTake(transactionID);
                    queue.completeTransaction(transactionID);
                    channelCounter.addToEventTakeSuccessCount(takes);
                } catch (IOException e) {
                    throw new ChannelException("Commit failed due to IO error "
                            + channelNameDescriptor, e);
                } finally {
                    log.unlockShared();
                }
                queueRemaining.release(takes);
            }
            putList.clear();
            takeList.clear();
            channelCounter.setChannelSize(queue.getSize());
        }
先通过Log对象拿到检查点的共享锁（log.lockShared()），拿到共享锁后，将调用Log对象的commit方法，commit方法先将事务ID、新生成的写顺序ID和操作类型（这里是put）来构建一个Commit对象，该对象和上面步骤4提到过的Put对象类似，然后再将该Commit写入数据文件，写入目录下的哪个文件，和写Put对象是一致的，由相同的事务ID来决定，这将意味着在一次事务操作中，一个Commit对象和对应的Put（如果是批次提交，则有多少个Event就有多少个Put）会被写入同一个数据目录下。当然，写入Commit对象到文件后并不需要保留其位置指针。


做完这一步，将putList中的Event文件位置指针FlumeEventPointer对象依次移除并放入内存队列queue（FlumeEventQueue）的队尾部。

这样Sink才有机会从内存队列queue中索引要取的Event在数据目录中的位置，同时也保证了这批次的Event谁第一个写入的Event也将第一个被放入内存队列也将会第一个被Sink消费。当FlumeEventPointer被全部写入内存队列queue后，则需要将保存在queue.inflightPuts中的该事务ID移除（queue.completeTransaction(transactionID);）,因为该事务不再处于“飞行中（in flight events）”了，于是，一个事务真正的被commit了。


### flume 异常恢复（reply）
这个模块暂时不写。

### 参考链接
[flume -- fileChannel简要分析其过程](https://www.noblogs.cn/blog/27079.html)  
[Flume Transaction介绍](https://fangjian0423.github.io/2016/01/03/flume-transaction/)