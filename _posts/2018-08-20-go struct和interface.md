---
layout: post
title: "go struct和interface"
date: 2018-08-10
categories: go
tags: [go]
image: http://gastonsanchez.com/images/blog/mathjax_logo.png
---
go struct和interface。结构体和接口。
<!-- more -->

### struck 结构体
* 结构体定义
~~~
type struct_variable_type struct {
   member definition;
   member definition;
   ...
   member definition;
}
~~~
* 结构体成员变量访问
~~~
struct 类似于 java 中的类，可以在struct中定义成员变量。
要访问成员变量，可以有两种方式:
1.通过struct变量.成员变量来访问。
2.通过struct指针.成员变量来访问。
type Rect struct{   //定义矩形类
    x,y float64       //类型只包含属性，并没有方法
    width,height float64
}

// 不可改变成员变量
func (r Rect) changeWidth1(){    
    r.width := 1.0    
}

//可以改变成员变量
func (r *Rect) changeWidth2(){    
    r.width := 1.0    
}
~~~


### interface 接口

* 接口的定义
~~~
Go 语言提供了另外一种数据类型即接口，
它把所有的具有共性的方法定义在一起，
任何其他类型只要实现了这些方法就是实现了这个接口。

type name interface{
    method()
    method2() string
}

type test1 struck {}
type test2 struck {}

func (t *test1) method(){
    # do something
}

func (t *test2) method() {
    # do something
}
~~~
