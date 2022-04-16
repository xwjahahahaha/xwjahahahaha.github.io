---
title: Operator
tags:
  - null
categories:
  - null
toc: true
date: 2022-04-16 16:20:46
---

# 一、Operator

## 1. 什么是Operator？

* Operator是一个感知应用状态的控制器（通过etcd的watch事件）
* CoreOS推出的旨在简化复杂的，有状态的应用管理的框架
* Operator是一个**有状态应用**的管理框架
* 通过拓展K8S的API来自动创建、管理和配置应用实例
* Operator就是一个管理自定义资源类型CRD的自定义的Controller，与K8S本身的Controller遵循同样的运行模式

<!-- more -->

## 2. k8s中controller的运行模式

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220222115157678.png" alt="image-20220222115157678"  />

* 整体是一个循环调用的过程：
  * 不断的从API Server中监听资源变换
  * 第一次Watch到新的资源时写入到Local Store缓存中，当第二次监听时会比较与自己本地缓存的差异
  * 如果有差异那么就会调用Callbacks函数（处理添加、更新删除操作），会将需要变化的任务放入到work队列中
  * Worker会从工作队列中获取任务，读取本地存储（注意是只读），然后交给clients再通过API Server处理

一个例子：

![image-20220222152236046](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220222152236046.png)
