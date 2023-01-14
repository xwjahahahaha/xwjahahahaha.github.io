---
title: eBPF-3-现状调研
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-18 15:53:06
---

[TOC]

> 参考资料：
>
> * 

<!-- more -->

# 一、概览

## 1. 项目介绍与基本原理

* `bumblebee`:  https://bumblebee.io/EN
*  [IOVisor/Hover](https://github.com/iovisor/iomodules)
* [Pixie](https://pixielabs.ai/)

## 2. 不同方向项目对比

活跃度、生产环境可用性、兼容性、性能、缺陷和隐患

![image-20220527142033594](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220527142033594.png)

![image-20220527142056168](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220527142056168.png)

# 二. Pixie项目-Profile

## 1. 基本功能

* 通过火焰图观察`Pod`中应用程序正在做什么CPU性能占比使用情况？热点情况？运行时间占比？
* 无需重新部署应用实现在分布式集群下的可视化CPU分析
* 持久化采样（占用很低资源）

## 2. 细节

* `Pixie` 每`10s`执行一次采样

* 采样对象粒度（两种，可选）：

  * 以`Pod`为统计的基础单位(或者说针对应用程序做基本单位), 火焰图的底部是一些`pod`的元数据信息

    ![image-20220525171310102](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220525171310102.png)

  * 以节点为统计基本单位：

    ![image-20220525174505424](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220525174505424.png)

* 发给`UI`多长时间的火焰图？

  * 堆栈跟踪样本批处理为30秒的窗口（将此30秒数据生成火焰图，然后发送给`UI`），然后可以根据需要将其组合为更大的时间跨度

* 开销？开销会随Pod负载越多越来越大吗？

  * （官方）大部分时间<0.5%
  * （官方）并不会，因为是基于`CPU`时钟中断实现的定时采样，对于当前环境进程量以及负载都能控制在0.5%以下

* 所有采集的数据都可以通过api去获取到，格式为折叠堆栈信息(分号间隔)，可以转为其他格式

  ![image-20220525174930129](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220525174930129.png)

## 3. 限制

![image-20220525172748759](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220525172748759.png)

### 基本限制

* 需要`debug symbols`实现出现的图中的文字可读, 如果识别不出来就放地址

### 支持的语言

* 完全支持：
  * `go`、`C`、`c++`、`rust`
* 测试支持中：
  * `java`
    * `unkown`问题解决方案：`For best results, run Java applications with -XX:+PreserveFramePointer.`
