---
title: Prometheus官方文档学习纪要
tags:
  - hide
categories:
  - technical
  - Prometheus
toc: true
declare: true
date: 2022-01-10 16:42:05
---

> 学习自：
>
> * 官网文档https://prometheus.io/docs/introduction/first_steps/

记录学习与使用prometheus的重点，建议先过一遍官方文档：

# 一、基本概念

Prometheus（Go语言开发）是由SoundCloud开发的开源监控告警系统和时序列数据库。从字面上理解，**Prometheus由两个部分组成，一个是监控告警系统，另一个是自带的时序数据库（TSDB）**。

<!-- more -->

![image-20220110193658682](http://xwjpics.gumptlu.work/image-20220110193658682.png)

Prometheus server从各个exporter拉取数据，或者间接地通过网关Pushgateway拉取数据，它默认本地存储抓取的所有数据。采集到的数据有两个去向，一个是可视化，另一个是告警。PromQL和其他API可视化地展示收集的数据，通过Alertmanager提供告警能力。

对比其他监控方案：

* Ceilometer：OpenStack中用来做计量计费功能的一个组件，后来又逐步发展增加了部分监控采集、告警的功能。
* Nightingale：（夜莺，简称n9e，Go语言开发）是滴滴开源的一个企业级监控解决方案。能达到企业级别的需求

**Prometheus的优缺点：**

优点：

* 简单轻量、自带时序服务器
* 处理能力强：指标都存储在Server本地数据库中，单个实例可以处理数百万的 Metrics
* 灵活的数据模型：引入了 Tag，给不同的接收端打好标识，属于多维数据模型，聚合统计更方便
* 强大的查询语句：PromQL 允许在同一个查询语句中，对多个 Metrics 进行加法、连接等操作。
* 生态与社区：很好兼容k8s，社区有很多常见应用的exporter可以使用

缺点：

* 自带的时序服务器不支持**降采样**

  > 即通过降低数据精度，来达到存储更长时间历史数据的目的。比如，12个5秒精度的点，通过计算均值，合并成1个1分钟精度的点。

* 中心化server不支持拓展，容灾问题（社区目前已经有一些高可用的方案）

# 二、export





