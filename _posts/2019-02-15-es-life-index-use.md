---
layout: post
title: "Index-Lifecycle-Policies 使用策略"
date: 2019-02-15
categories: 随笔
tags: [ElasticSearch]
image: http://gastonsanchez.com/https://github.com/daijiangtian/NoteBook/blob/master/机器学习/时间序列/https://github.com/daijiangtian/NoteBook/blob/master/机器学习/时间序列/https://github.com/daijiangtian/NoteBook/blob/master/机器学习/时间序列/images/blog/mathjax_logo.png?raw=true?raw=true?raw=true
---

随时间自动化管理索引 管理范围包括　分片大小，性能要求

根据template 的order 对模板进行优先级排序  order 越高 优先级越大

<!-- more -->

###  普通模板

若无index没有匹配上默认使用

#### 配置

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1d",
        "actions": {
          "shrink": {
           "number_of_shards": 1
           },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
       "min_age": "7d",
       "actions": {
       }
      },
      "delete": {
        "min_age": "15d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}


PUT _template/my_template
{
  "index_patterns": ["*-*.*.*"],
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "index.lifecycle.name": "my_policy"
  }

}

```

#### 说明

1天后不在写入 分片由5至1  文段合并为1个
7天后删除副本数据
15天后删除索引

### monitor 模板

对于监控生成的索引使用  索引的大小与集群内总共的索引数有关

#### 配置

```json
PUT _ilm/policy/monitor_policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1d",
        "actions": {
        }
      },

      "delete": {
        "min_age": "3d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
PUT _template/monitor_template
{
  "order":1,
  "index_patterns": ["*monitor*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "monitor_policy"
  }
}

```

#### 说明

1天后不可再写入 由于改索引默认为1个分片文段数也较小 因此无需做处理
3天后删除索引

### 根据要求自定义规则

原则上每个shard最大超过20G,每个 segment 大小不超过5G


### 数据量在 500M 至5G

#### 配置
```json
PUT _ilm/policy/mid_policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
       "min_age": "7d",
       "actions": {
       }
      },
      "delete": {
        "min_age": "15d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}


PUT _template/my_template
{
  "index_patterns": [
    "index_name-*",
    "index_name2-*"
  ],
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "index.lifecycle.name": "mid_policy"
  }

}

```

#### 说明

对于该数据量的索引 合并分片不能提升查询效率 因此只做 merge segment 操作


### 数据量在 5G以上

#### 配置
```json
PUT _ilm/policy/big_policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1d",    # 对于该类索引都有提前建立索引 具体时间需要安装提前几天建立索引判断
        "actions": {
        }
      },
      "cold": {
       "min_age": "3d",
       "actions": {
       }
      },
      "delete": {
        "min_age": "7d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}


PUT _template/my_template
{
  "index_patterns": [
    "index_name-*",
    "index_name2-*"
  ],
  "settings": {
    "number_of_shards": 5,  # 具体数量为集群节点值
    "number_of_replicas": 1,
    "index.lifecycle.name": "big_policy"
  }

}

```

#### 说明

对于数据量太大对索引做合并操作太影响集群整体性能



