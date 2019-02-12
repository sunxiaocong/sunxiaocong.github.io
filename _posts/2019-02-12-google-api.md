---
layout: post
title: "Google API 使用"
date: 2019-02-12
categories: 随笔
tags: [随笔]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---

当应用在Google发布后 可以通过google 给的api 获取应用的各种信息
本文以应用评论为例

<!-- more -->


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->


## 谷歌应用商店评论接口

目标：从谷歌应用商店获取评论

限制：只能获取一周数据，且只能获取有评论的数据（有评分无评论默认不获取）

### 评论获取接口

#### 使用 - OAuth 2.0

地址：[https://developers.google.com/android-publisher/reply-to-reviews](https://developers.google.com/android-publisher/reply-to-reviews)

使用：

``` txt
GET https://www.googleapis.com/androidpublisher/v2/applications/your_package_name/reviews?
access_token=your_auth_token
```

**your_package_name**: 包名，如：`com.wsy.google.wansuiye`

**access_token**：请求必须携带的`token`

关于`access_token`的获取：

- 进入谷歌应用[控制台](https://play.google.com/apps/publish/?account=8049288854550016816#AppListPlace)
- 点击：设置->API权限
- 进入后点击创建OAuth客户端->在Google Developers Console中查看
- 跳转到GoogleAPIs后下载对应JSON文件

文件内容如下：

``` json
{
    "installed":{
        "client_id":"176094307803-s1u405ta6k3jihpnldf8vu8vb6ra1k1k.apps.googleusercontent.com",
        "project_id":"api-6032512620173712842-333003",
        "auth_uri":"https://accounts.google.com/o/oauth2/auth",
        "token_uri":"https://www.googleapis.com/oauth2/v3/token",
        "auth_provider_x509_cert_url":"https://www.googleapis.com/oauth2/v1/certs",
        "client_secret":"AG9c_gZB6vLtb9LbX4f0wNAi",
        "redirect_uris":["urn:ietf:wg:oauth:2.0:oob","http://localhost"]
    }
}
```

根据这些参数，可构建URL生成code，示例：

``` txt
https://accounts.google.com/o/oauth2/auth?scope=https://www.googleapis.com/auth/androidpublisher&response_type=code&access_type=offline&redirect_uri=http://localhost/&client_id=176094307803-s1u405ta6k3jihpnldf8vu8vb6ra1k1k.apps.googleusercontent.com
```

`redirect_uri`以及`client_id`需要替换，访问后浏览器会返回：

```txt
http://localhost/?code=4/NQCF6qUyoK047iFTnDkgCn2S5BbaYC0E2OBAcCxArYvkUEtkzkhnVLd8kc6gxCzfb1PtA5BS86RyYSyzsTmsxpQ#
```

`4/NQCF6qUyoK047iFTnDkgCn2S5BbaYC0E2OBAcCxArYvkUEtkzkhnVLd8kc6gxCzfb1PtA5BS86RyYSyzsTmsxpQ`获取后才可以下一步获取[`access_token`](https://developers.google.com/android-publisher/authorization)，Python代码如下：

``` python
import requests

data = {
    "grant_type": "authorization_code",
    "code": "4/NQCF6qUyoK047iFTnDkgCn2S5BbaYC0E2OBAcCxArYvkUEtkzkhnVLd8kc6gxCzfb1PtA5BS86RyYSyzsTmsxpQ",
    "client_id": "176094307803-s1u405ta6k3jihpnldf8vu8vb6ra1k1k.apps.googleusercontent.com",
    "client_secret": "AG9c_gZB6vLtb9LbX4f0wNAi",
    "redirect_uri": "http://localhost/"
}

proxies = {
    "http": "http://127.0.0.1:8118",
    "https": "http://127.0.0.1:8118"
}
res = requests.post('https://accounts.google.com/o/oauth2/token', data=data, proxies=proxies)
print(res.json())
```

结果：

``` json
{
    'access_token': 'ya29.GlvzBfkLyKvEGohcT9XK_bZTj9txXIApwAek79JQ-sZNgkLJmTnHzWDXZikVqML1I968sASgTSV36tdgvfqIbpfhMrQi9-VYm5pqhyt6G5dZ8BiUMnh5qHlAixEJ',
    'expires_in': 3600,
    'refresh_token': '1/g90HKEk6_07Wsoc0M4GRxPxJ5wQOjEAvMNTxhUfy1EI',
    'scope': 'https://www.googleapis.com/auth/androidpublisher',
    'token_type': 'Bearer'}
```

请求[评论获取接口](https://www.googleapis.com/androidpublisher/v2/applications/com.wsy.google.wansuiye/reviews?access_token=ya29.GlvzBfkLyKvEGohcT9XK_bZTj9txXIApwAek79JQ-sZNgkLJmTnHzWDXZikVqML1I968sASgTSV36tdgvfqIbpfhMrQi9-VYm5pqhyt6G5dZ8BiUMnh5qHlAixEJ)：

```
{
 "tokenPagination": {
  "nextPageToken": "AGkSnOzxAv9o1yzs-SRt6uZGX176oseAnEO_UlBC_c2na-DLkpKUdHqDLvzpcH1-_v7I96pM-feRXJAE735dQEiCNqd-5emfxVi-dD3IFDJEqg5sI6WD2CHp96Kq53iarwsZqSTj6YYTz2-aEMHh25omkB50qQFp32Q_HIqbHDI010bZRJ8v0XNQZtbCKp1Hebdcq4OuB-hHzBWpQOdVOcW8gdjOFUniTw"
 },
 "reviews": [
  {
   "reviewId": "gp:AOqpTOHuBWkHxXjy1yYfvFwghdiPP555FdBLQwyFO6fnkVfGGvi_IRhp8YZ2erg1DjqMUsKtyF2TnBWDJBHkFac",
   "authorName": "",
   "comments": [
    {
     "userComment": {
      "text": "\t會跳出來",
      "lastModified": {
       "seconds": "1533601272",
       "nanos": 883000000
      },
      "starRating": 3,
      "reviewerLanguage": "zh-Hant_TW",
      "device": "ASUS_X007D",
      "androidOsVersion": 23,
      "thumbsUpCount": 0,
      "thumbsDownCount": 0,
      "deviceMetadata": {
       "productName": "ASUS_X007D (ZenFone Go (ZB552KL))",
       "manufacturer": "Asus",
       "deviceClass": "phone",
       "screenWidthPx": 720,
       "screenHeightPx": 1280,
       "nativePlatform": "armeabi-v7a,armeabi,arm64-v8a",
       "screenDensityDpi": 320,
       "glEsVersion": 196608,
       "cpuModel": "MSM8916",
       "cpuMake": "Qualcomm",
       "ramMb": 2048
      }
     }
    }
   ]
  }]
 }
// 省略了数据
```

#### 使用 - 服务帐号

此方案比较简单：

- 进入谷歌应用[控制台](https://play.google.com/apps/publish/?account=8049288854550016816#AppListPlace)
- 建立服务帐号
- 下载JSON文件

代码如下：

```python
from httplib2 import Http
from oauth2client.service_account import ServiceAccountCredentials
from apiclient.discovery import build

credentials = ServiceAccountCredentials.from_json_keyfile_name(
    'api.json',
    scopes=['https://www.googleapis.com/auth/androidpublisher'])
service = build('androidpublisher', 'v2', http=credentials.authorize(Http()))

package_name = "com.wsy.google.wansuiye"
reviews_resource = service.reviews()
reviews_page = reviews_resource.list(packageName=package_name, maxResults=100).execute()
reviews_list = reviews_page["reviews"]
print(reviews_list)
```

#### reviews_resource.list 参数说明

translationLanguage参数 更多参考 [点击](http://www.lingoes.cn/zh/translator/langcode.htm)

|        名称         |                描述                 |
| :-----------------: | :---------------------------------: |
|     packageName     |        所需查找的游戏的包名         |
|     maxResults      |            返回最大长度             |
| translationLanguage | 返回的语言类型 en ：英文，zh ：中文 |




#### 字段

|            字段名             |                        描述                        |
| :---------------------------: | :------------------------------------------------: |
| tokenPagination.nextPageToken | 用于标识下一次起始位置（当没有下一页时没有改字段） |
|           reviewId            |                   GooglePlay分配                   |
|          authorName           |                       用户名                       |
|             text              |                        内容                        |
|         lastModified          |                      上次修改                      |
|          starRating           |                        评分                        |
|       reviewerLanguage        |                      评论语种                      |
|            device             |                        设备                        |
|       androidOsVersion        |                      系统类型                      |
|        appVersionCode         |                    应用版本编码                    |
|        appVersionName         |                    应用版本名称                    |
|         thumbsUpCount         |                        支持                        |
|        thumbsDownCount        |                        反对                        |
|        deviceMetadata         |                     设备元信息                     |



