---
layout: post
title: "ElaticSearch Xpack 破解"
date: 2019-01-22
categories: 大数据
tags: [ElasticSearch,Xpack]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

 X-pack 安装与使用  es版本:6.2.3  该文详解如何进行破解
 [破解.jar](https://github.com/daijiangtian/NoteBook/blob/master/%E5%A4%A7%E6%95%B0%E6%8D%AE/xpack/x-pack-core-6.2.3.jar)
该为破解jar 直接放上替换即可 有效不忘打赏哦
<!-- more -->

## 1. elasticsearch X-pack 安装
```shell
cd /usr/sharel/elasticsearch
bin/elasticsearch-plugin install x-pack

安装过程全部输入y
bin/x-pack/setup-passwords interactive  # 设置密码
重启 elasticsearch
```

### 1.1 功能启用与禁用

/etc/elasticsearch/elasticsearch.yml  中配置

```yml
#设置                     #描述
xpack.graph.enabled       #设置为false以禁用X-Pack图形功能。
xpack.ml.enabled          #设置为false以禁用X-Pack机器学习功能。
xpack.monitoring.enabled  #设置为false以禁用X-Pack监视功能。
xpack.reporting.enabled   #设置为false以禁用X-Pack报告功能。
xpack.security.enabled    #设置为false以禁用X-Pack安全功能。 访问控制 一切9200都需认证才可访问
xpack.watcher.enabled     #设置为false以禁用Watcher。
```

## 2. kibana X-pack 安装
```shell
cd /usr/share/kibana
bin/kibana-plugin install x-pack
```
若在es上配置了security 则在 /etc/kibana/kibana.yml 下添加
```yml
elasticsearch.username: "kibana"
elasticsearch.password: "kibanapassword"
 ```

## 3. X-pack 破解

### 3.1 解压x-pack-core-6.2.3.jar

```shell
/usr/share/elasticsearch/plugins/x-pack/x-pack-core/x-pack-core-6.2.3.jar /tmp/x-pack
cd /tmp/x-pack
jar -xvf x-pack-core-6.2.3.jar

找到 org.elasticsearch.license.LicenseVerifier.class 
用luyten反编译保存为java文件
```

### 3.2 修改LicenseVerifier

LicenseVerifier 中有两个静态方法，这就是验证授权文件是否有效的方法，我们把它修改为全部返回true.
```Script

package org.elasticsearch.license;
import java.nio.*;
import java.util.*;
import java.security.*;
import org.elasticsearch.common.xcontent.*;
import org.apache.lucene.util.*;
import org.elasticsearch.common.io.*;
import java.io.*;

public class LicenseVerifier
{
    public static boolean verifyLicense(final License license, final byte[] encryptedPublicKeyData) {
        return true;
    }

    public static boolean verifyLicense(final License license) {
        return true;
    }
}
```

### 3.3 修改XPackBuild

XPackBuild 中 最后一个静态代码块中 try的部分全部删除，这部分会验证jar包是否被修改.
```
package org.elasticsearch.xpack.core;

import org.elasticsearch.common.io.*;
import java.net.*;
import org.elasticsearch.common.*;
import java.nio.file.*;
import java.io.*;
import java.util.jar.*;

public class XPackBuild
{
    public static final XPackBuild CURRENT;
    private String shortHash;
    private String date;

    @SuppressForbidden(reason = "looks up path of xpack.jar directly")
    static Path getElasticsearchCodebase() {
        final URL url = XPackBuild.class.getProtectionDomain().getCodeSource().getLocation();
        try {
            return PathUtils.get(url.toURI());
        }
        catch (URISyntaxException bogus) {
            throw new RuntimeException(bogus);
        }
    }

    XPackBuild(final String shortHash, final String date) {
        this.shortHash = shortHash;
        this.date = date;
    }

    public String shortHash() {
        return this.shortHash;
    }

    public String date() {
        return this.date;
    }

    static {
        final Path path = getElasticsearchCodebase();
        String shortHash = null;
        String date = null;
        Label_0157: {
            shortHash = "Unknown";
            date = "Unknown";
        }
        CURRENT = new XPackBuild(shortHash, date);
    }
}
```

### 3.4 重新编译生成class文件 并放入temp目录覆盖之前的class文件

```
javac -cp "/usr/share/elasticsearch/lib/elasticsearch-6.2.3.jar:/usr/share/elasticsearch/lib/lucene-core-7.2.1.jar:/usr/share/elasticsearch/plugins/x-pack/x-pack-core/x-pack-core-6.2.3.jar:/usr/share/elasticsearch/plugins/x-pack/x-pack-core/netty-common-4.1.16.Final.jar:/usr/share/elasticsearch/lib/elasticsearch-core-6.2.3.jar" XPackBuild.java
```
```
javac -cp "/usr/share/elasticsearch/lib/elasticsearch-6.2.3.jar:/usr/share/elasticsearch/lib/lucene-core-7.2.1.jar:/usr/share/elasticsearch/plugins/x-pack/x-pack-core/x-pack-core-6.2.3.jar" LicenseVerifier.java 
```
```
cp LicenseVerifier.class /root/temp/org/elasticsearch/license/LicenseVerifier.class
```
```
cp XPackBuild.class /root/temp/org/elasticsearch/xpack/core/XPackBuild.class 
```
### 3.5 打包生成x-pack-core-6.2.3.jar 并替换
```
cd /root/temp
jar -cvf x-pack-core-6.2.3.jar ./*
cp x-pack-core-6.2.3.jar /usr/share/elasticsearch/plugins/x-pack/x-pack-core/x-pack-core-6.2.3.jar 
```
### 3.6 重启elasticsearch kibana 并导入修改过的授权文件
若授权文件过期 则在Kibana上获取 base的授权文件 然后将文件的 type改为platinum , expiry_date_in_millis修改为期望到期的时间
```
{"license":{"uid":"de49ea79-bcbc-4480-a58f-01e5d8eb0dd5","type":"platinum","issue_date_in_millis":1542672000000,"expiry_date_in_millis":2552543123999,"max_nodes":100,"issued_to":"dai jiangtian (xiamendianchu)","issuer":"Web Form","signature":"AAAAAwAAAA2lPvNcqINIsMQ5jPnmAAABmC9ZN0hjZDBGYnVyRXpCOW5Bb3FjZDAxOWpSbTVoMVZwUzRxVk1PSmkxaktJRVl5MUYvUWh3bHZVUTllbXNPbzBUemtnbWpBbmlWRmRZb25KNFlBR2x0TXc2K2p1Y1VtMG1UQU9TRGZVSGRwaEJGUjE3bXd3LzRqZ05iLzRteWFNekdxRGpIYlFwYkJiNUs0U1hTVlJKNVlXekMrSlVUdFIvV0FNeWdOYnlESDc3MWhlY3hSQmdKSjJ2ZTcvYlBFOHhPQlV3ZHdDQ0tHcG5uOElCaDJ4K1hob29xSG85N0kvTWV3THhlQk9NL01VMFRjNDZpZEVXeUtUMXIyMlIveFpJUkk2WUdveEZaME9XWitGUi9WNTZVQW1FMG1DenhZU0ZmeXlZakVEMjZFT2NvOWxpZGlqVmlHNC8rWVVUYzMwRGVySHpIdURzKzFiRDl4TmM1TUp2VTBOUlJZUlAyV0ZVL2kvVk10L0NsbXNFYVZwT3NSU082dFNNa2prQ0ZsclZ4NTltbU1CVE5lR09Bck93V2J1Y3c9PQAAAQCMxbBqGIXMIFzTrAdYGsm7NJe1vfABxskNQFDjDKXAosvybvPGt8/VLFqMD+YV14Qr60BotU8Etss/llAvZCgzy6zMc+U9WnmatftboGhbXw+/RTp9QtUXsxo2LH1eApKYMWYy32KDvS0kFHbp9oiovvAl9IDYp9+ZMca+ttI380V4DocFaKquBq/THp5XeHlVaOt0xIUrbsbU37j3JlZ4nVUFspvNOlY0XyFTTrJ0PQ9OQBlOT+SbutLG6Bdm7YozOzoVVG0yyns2U80gzojudNq53DSP32Ktuqt3CMDHDb72b1JJZ/gm9goJ22KTXj6bJizIE/4uXOxiFKEciGez","start_date_in_millis":1542672000000}}
```

### 3.7 end  X-pack 破解为 白金版 到期时间 2050年

