---
layout: post
title: "Flume自定义拦截器"
date: 2019-02-02
categories: 大数据
tags: [Flume,数据采集]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

Flume 内置了许多 source channel sink 拦截器 等等
同时提供了 抽象类让我们可以编写自定义的对应类
本文主要记录 自定义拦截器 对event header 中的字段做处理

<!-- more -->

### 创建maven项目并打包成jar

#### pom.xml
创建时选择groupId com.us  artifactId flumeInterceptor version 1.0-SNAPSHOT
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.us</groupId>
    <artifactId>flumeInterceptor</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>

        <maven.compiler.target>1.8</maven.compiler.target>
        <maven.compiler.source>1.8</maven.compiler.source>
        <version.flume>1.7.0</version.flume>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.flume</groupId>
            <artifactId>flume-ng-core</artifactId>
            <version>${version.flume}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flume</groupId>
            <artifactId>flume-ng-configuration</artifactId>
            <version>${version.flume}</version>
        </dependency>

    </dependencies>


</project>
```

#### 编写自定义类

在src/main/java/com.us 下创建文件 RegexExtractorHeaderInterceptor

```java
package com.us;
import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang.StringUtils;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;
import org.apache.flume.interceptor.RegexExtractorInterceptorPassThroughSerializer;
import org.apache.flume.interceptor.RegexExtractorInterceptorSerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.common.base.Charsets;
import com.google.common.base.Preconditions;
import com.google.common.base.Throwables;
import com.google.common.collect.Lists;

public class RegexExtractorHeaderInterceptor implements Interceptor {

    static final String REGEX = "regex";
    static final String SERIALIZERS = "serializers";


    static final String EXTRACTOR_HEADER = "extractorHeader";
    static final boolean DEFAULT_EXTRACTOR_HEADER = false;
    static final String EXTRACTOR_HEADER_KEY = "extractorHeaderKey";

    private static final Logger logger = LoggerFactory
            .getLogger(RegexExtractorHeaderInterceptor.class);

    private final Pattern regex;
    private final List<NameAndSerializer> serializers;

    private final boolean extractorHeader;
    private final String extractorHeaderKey;

    private RegexExtractorHeaderInterceptor(Pattern regex,
                                            List<NameAndSerializer> serializers,boolean extractorHeader, String extractorHeaderKey) {
        this.regex = regex;
        this.serializers = serializers;

        this.extractorHeader = extractorHeader;
        this.extractorHeaderKey = extractorHeaderKey;

    }

    @Override
    public void initialize() {
        // NO-OP...
    }

    @Override
    public void close() {
        // NO-OP...
    }

    @Override
    public Event intercept(Event event) {
        String extractorHeaderVal;
        if (extractorHeader){

            extractorHeaderVal = event.getHeaders().get(extractorHeaderKey);

        }else{

            extractorHeaderVal = new String(event.getBody(),Charsets.UTF_8);

        }

        Matcher matcher = regex.matcher(extractorHeaderVal);
        Map<String, String> headers = event.getHeaders();
        if (matcher.find()) {
            for (int group = 0, count = matcher.groupCount(); group < count; group++) {
                int groupIndex = group + 1;
                if (groupIndex > serializers.size()) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Skipping group {} to {} due to missing serializer",
                                group, count);
                    }
                    break;
                }
                NameAndSerializer serializer = serializers.get(group);
                if (logger.isDebugEnabled()) {
                    logger.debug("Serializing {} using {}", serializer.headerName,
                            serializer.serializer);
                }
                headers.put(serializer.headerName,
                        serializer.serializer.serialize(matcher.group(groupIndex)));
            }
        }
        return event;
    }

    @Override
    public List<Event> intercept(List<Event> events) {
        List<Event> intercepted = Lists.newArrayListWithCapacity(events.size());
        for (Event event : events) {
            Event interceptedEvent = intercept(event);
            if (interceptedEvent != null) {
                intercepted.add(interceptedEvent);
            }
        }
        return intercepted;
    }

    public static class Builder implements Interceptor.Builder {

        private Pattern regex;
        private List<NameAndSerializer> serializerList;

        private boolean extractorHeader;
        private String extractorHeaderKey;

        private final RegexExtractorInterceptorPassThroughSerializer defaultSerializer = new RegexExtractorInterceptorPassThroughSerializer();


        @Override
        public void configure(Context context) {
            String regexString = context.getString(REGEX);
            Preconditions.checkArgument(!StringUtils.isEmpty(regexString),
                    "Must supply a valid regex string");
            regex = Pattern.compile(regexString);
            regex.pattern();
            regex.matcher("").groupCount();
            configureSerializers(context);

            extractorHeader = context.getBoolean(EXTRACTOR_HEADER,DEFAULT_EXTRACTOR_HEADER);

            if (extractorHeader){

                extractorHeaderKey = context.getString(EXTRACTOR_HEADER_KEY);
                Preconditions.checkArgument(!StringUtils.isEmpty(extractorHeaderKey),"header key must");

            }

        }

        private void configureSerializers(Context context) {
            String serializerListStr = context.getString(SERIALIZERS);
            Preconditions.checkArgument(!StringUtils.isEmpty(serializerListStr),
                    "Must supply at least one name and serializer");

            String[] serializerNames = serializerListStr.split("\\s+");

            Context serializerContexts =
                    new Context(context.getSubProperties(SERIALIZERS + "."));

            serializerList = Lists.newArrayListWithCapacity(serializerNames.length);
            for(String serializerName : serializerNames) {
                Context serializerContext = new Context(
                        serializerContexts.getSubProperties(serializerName + "."));
                String type = serializerContext.getString("type", "DEFAULT");
                String name = serializerContext.getString("name");
                Preconditions.checkArgument(!StringUtils.isEmpty(name),
                        "Supplied name cannot be empty.");

                if("DEFAULT".equals(type)) {
                    serializerList.add(new NameAndSerializer(name, defaultSerializer));
                } else {
                    serializerList.add(new NameAndSerializer(name, getCustomSerializer(
                            type, serializerContext)));
                }
            }
        }

        private RegexExtractorInterceptorSerializer getCustomSerializer(
                String clazzName, Context context) {
            try {
                RegexExtractorInterceptorSerializer serializer = (RegexExtractorInterceptorSerializer) Class
                        .forName(clazzName).newInstance();
                serializer.configure(context);
                return serializer;
            } catch (Exception e) {
                logger.error("Could not instantiate event serializer.", e);
                Throwables.propagate(e);
            }
            return defaultSerializer;
        }

        @Override
        public Interceptor build() {
            Preconditions.checkArgument(regex != null,
                    "Regex pattern was misconfigured");
            Preconditions.checkArgument(serializerList.size() > 0,
                    "Must supply a valid group match id list");
            return new RegexExtractorHeaderInterceptor(regex, serializerList, extractorHeader, extractorHeaderKey);
        }
    }

    static class NameAndSerializer {
        private final String headerName;
        private final RegexExtractorInterceptorSerializer serializer;

        public NameAndSerializer(String headerName,
                                 RegexExtractorInterceptorSerializer serializer) {
            this.headerName = headerName;
            this.serializer = serializer;
        }
    }
}

```

#### 打包

打包结束后将该jar文件放入 apache-flume-1.7.0-bin/lib 目录下

然后编写Dockerfile

```
FROM centos:7
RUN yum install -y net-tools wget java
ENV APP_ROOT=/data\
    NAME=a1\
    ZK_HOSTS=10.19.52.143:2181,10.19.52.249:2181,10.19.195.164:2181,10.19.85.119:2181,10.19.144.93:2181\
    ZK_PATH=/tran_and_ad/tran_and_ad_flume\
    TIME_ZONE=Asia/Shanghai
COPY . ${APP_ROOT}/
WORKDIR ${APP_ROOT}/apache-flume-1.7.0-bin
EXPOSE 57888
RUN echo "${TIME_ZONE}" > /etc/timezone \
&& ln -sf /usr/share/zoneinfo/${TIME_ZONE} /etc/localtime
VOLUME ["/data/agent"]

RUN chmod 777 bin/flume-ng

#CMD ["bin/flume-ng","agent","--conf conf","-z ${ZK_HOSTS}","-p ${ZK_PATH}","--name ${NAME}","-Dflume.root.logger=INFO,console"]
CMD bin/flume-ng agent --conf conf -z ${ZK_HOSTS} -p ${ZK_PATH} --name ${NAME} -Dflume.root.logger=INFO,console
```

### 测试flume

#### 编写配置文件 放入zookeeper 对应路径下 本文为 /behaviors_log/hdfs/a6

```yml
a6.sources = r1
a6.channels = c1 c2
a6.sinks = k1 k2


a6.sources.r1.type = thrift
a6.sources.r1.channels = c1 c2
a6.sources.r1.bind = 0.0.0.0
a6.sources.r1.port = 57888
a6.sources.r1.threads = 200

a6.sources.r1.interceptors = i2

a6.sources.r1.interceptors.i2.type = com.us.RegexExtractorHeaderInterceptor$Builder

a6.sources.r1.interceptors.i2.regex =  ([\\w-]*),([\\w-]*)

a6.sources.r1.interceptors.i2.extractorHeader = true

a6.sources.r1.interceptors.i2.extractorHeaderKey = Filename

a6.sources.r1.interceptors.i2.serializers = s1 s2

a6.sources.r1.interceptors.i2.serializers.s1.name = name

a6.sources.r1.interceptors.i2.serializers.s2.name = log_date



channel selector configuration
a6.sources.r1.selector.type = multiplexing
a6.sources.r1.selector.header = log_date
a6.sources.r1.selector.mapping.2018 = c1


a6.sinks.k1.type = logger
a6.sinks.k1.channel = c1


#配置channel(内存做缓存)
a6.channels.c1.type = memory
```

#### 启动flume

```shell
docker run  -it -e ZK_HOSTS=192.168.5.30:2181,192.168.5.30:2182,192.168.5.30:2183 -e ZK_PATH=/behaviors_log/hdfs -e NAME=a6 -p 57888:57888 flume-test:0.0.4

```



