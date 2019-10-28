---
layout: post
title: "2019-10-28-flink kafka获取数据实时写入hdfs.md"
date: 2019-10-28
categories: Flink
tags: [Flink]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
Flink 写入hdfs
<!-- more -->
### pom.xml 文件
        <properties>
            <encoding>UTF-8</encoding>
            <scala.version>2.11.12</scala.version>
            <scala.binary.version>2.11</scala.binary.version>
            <flink.version>1.6.0</flink.version>
        </properties>
    
        <!--flink java-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>compile</scope>
        </dependency>
        <!--flink kafka connector-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.11_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>


        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-filesystem_2.11</artifactId>
            <version>${flink.version}</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.7.0</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.7.0</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.7.0</version>
            <scope>compile</scope>
        </dependency>
### kafka source hdfs sink
    import org.apache.flink.api.common.serialization.SimpleStringSchema;
    import org.apache.flink.streaming.api.datastream.DataStream;
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.streaming.connectors.fs.StringWriter;
    import org.apache.flink.streaming.connectors.fs.bucketing.BucketingSink;
    import org.apache.flink.streaming.connectors.fs.bucketing.DateTimeBucketer;
    import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer011;
    
    import java.time.ZoneId;
    import java.util.Properties;
    
    public class Main {
        public static void main(String[] args) throws Exception {
    
            final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
            
            Properties props = new Properties();
            props.put("bootstrap.servers", "10.46.0.27:9092, 10.46.17.114:9092, 10.46.44.35:9092");
            props.put("zookeeper.connect", "10.46.187.169:2181 10.46.233.219:2181 10.46.85.85:2181");
            props.put("group.id", "metric-group2");
            props.put("enable.auto.commit", false);
            props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");      //key 反序列化
            props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");    //value 反序列化
            props.put("auto.offset.reset", "earliest");
            props.put("max.poll.records", 1000);
            SingleOutputStreamOperator<String> dataStreamSource = env.addSource(new FlinkKafkaConsumer011<>(
                    "topic",  //kafka topic
                    new SimpleStringSchema(),  // String 序列化
                    props)).setParallelism(1); 
                    
             
            BucketingSink<String> hadoopSink = new BucketingSink<>("/flink/test");
            // hadoopSink.setFSConfig(conf);
            // 使用东八区时间格式"yyyy-MM-dd--HH"命名存储区
            hadoopSink.setBucketer(new DateTimeBucketer("yyyy-MM-dd--HH"));
        
            // 下述两种条件满足其一时，创建新的块文件
            // 条件1.设置块大小为100MB
            hadoopSink.setBatchSize(1024 * 1024 * 100);
            // 条件2.设置时间间隔20min
            hadoopSink.setBatchRolloverInterval(20 * 60 * 1000);
            // 设置块文件前缀
            hadoopSink.setPendingPrefix("");
            // 设置块文件后缀
            hadoopSink.setPendingSuffix("");
            // 设置运行中的文件前缀
            hadoopSink.setInProgressPrefix(".");   
            
            //新增sink，写入hdfs
            dataStreamSource.addSink(hadoopSink);
        }
    }
    
### 总结
   kafka 和 hdfs 可以理解为数据湖。dataStreamSource 作为河流（数据流）连接两个湖泊，将水资源（数据）从kafka接入hdfs。当然也可以将dataStreamSource分流或者合并到新的河流之中（数据流）