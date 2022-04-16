---
title: k8s二次开发-入门
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-02-22 11:26:21
---

> * https://www.bilibili.com/video/BV1zE411j7ky

[TOC]

# 一、k8s的拓展点

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220319234241256.png" alt="image-20220319234241256" style="zoom:50%;" />

一般拓展的地方：

* 客户端：使用client-go封装
* APIServer：引入了聚合层，在主流控制中拓展
* 内置资源：自定义CRD
* 自定义Controller
* Schedule：WebHook二次扩展

二开的API分类：

![image-20220319234944144](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220319234944144.png)

# 二、Kubernetes的API

无论哪一种语言的客户端开发包，最终都是访问k8s的API，k8s的API规范符合Open API规范: https://www.openapis.org/

API不同版本的不同之处:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220325220340645.png" alt="image-20220325220340645" style="zoom:50%;" /> 

API的规则：

* 资源：`GVR`也就是“分组、版本、资源”
* 实体：`GVK`，这里的`K`就是yaml清单中的`Kind`，也就是client-go需要请求返回的内容

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220325221222470.png" alt="image-20220325221222470" style="zoom:50%;" />

































