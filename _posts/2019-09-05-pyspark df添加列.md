---
layout: post
title: "pyspark df添加列"
date: 2019-09-05
categories: BigData
tags: [spark]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
编写pyspark任务,对df添加新的列。  
使用自定义函数和自带的函数。
<!-- more -->
### pyspark.sql.functions (自带函数)
* split （将一行分成列表 eg:"90 100" --> [90,100]）
> df_split = df.withColumn("s", split(df['score'], " "))
* explode （将一行分成多行 eg:"90 100" --> "90","100"）
> df_explode = df.withColumn("e", explode(split(df['score'], " ")))
* concat(合并多个字段不带分隔符 eg:"column1column2")
> df_concat = df.withColumn("s", concat(df["column1"],df["column2"]))
* concat_ws (合并多个字段带分隔符 eg:"column1-column2")
> df_concat_ws = df.withColumn("s", concat_ws("-",df["column1"],df["column2"]))


### 实现自定义函数（基本的代码结构如下）
    
    # 1.创建普通的python函数
    def toDate(s):
        return str(s)+'-'
    
    # 2.注册自定义函数
    from pyspark.sql.functions import udf
    from pyspark.sql.types import StringType
    
    # 根据python的返回值类型定义好spark对应的数据类型
    # python函数中返回的是string，对应的pyspark是StringType
    toDateUDF=udf(toDate, StringType())  
    
    # 使用自定义函数
    df1.withColumn('color',toDateUDF('color')).show()
    
    ----------------------------------------------------------------------------------------
    
    # 创建udf自定义函数
    from pyspark.sql import functions
    concat_func = functions.udf(lambda name,age:name+'_'+str(age))  # 简单的连接两个字符串
    
    # 应用自定义函数
    concat_df = spark_df.withColumn("name_age",concat_func(final_data.name, final_data.age))
    concat_df.show()
    

### 自定义函数返回值类型
自定义函数的重点在于定义返回值类型的数据格式，其数据类型基本都是从from pyspark.sql.types import * 导入，常用的包括： 
- StructType()：结构体 
- StructField()：结构体中的元素 
- LongType()：长整型 
- StringType()：字符串 
- IntegerType()：一般整型 
- FloatType()：浮点型


