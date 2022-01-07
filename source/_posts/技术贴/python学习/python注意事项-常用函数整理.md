---
title: python注意事项/常用函数整理
tags:
  - python
categories:
  - technical
toc: true
date: 2020-07-01 19:06:40
---

# 1. python中通过循环修改列表中的值，却达不到效果

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200607213207.png)
<!-- more -->

结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200607212542.png)

## 原因

修改后的值保存到每次循环中的变量magician中了，但是并没有重新赋值到列表中的元素

## 解决方法

使用**索引**修改，才有效！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200607213228.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200607213242.png)

# 数学相关

1.pow( x, y )方法返回x的y次方的值。


## python没有自增自减，需注意

可使用`x += 1`代替相关的功能。

## python常用的进制转换函数

10进制转其他进制：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200701191008.png)

其他进制转10进制：

int（"x进制数", x）

例：

2进制 -> 10进制 : int("1101", 2) = 13

8进制 -> 10进制 : int('0o226', 8) = 150