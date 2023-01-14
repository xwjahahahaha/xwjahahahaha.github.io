---
title: apiServer故障报告
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-08-19 11:14:27
---

[TOC]

<!-- more -->

# 一、故障现象

* 大量慢查询日志：etcd告警overload，etcd节点存在大量的慢查询1200+
* 自定义的api-Server停止响应，日志报了一些自定义metrics apiserver错误`connect timeout`

# 二、原因与处理方式

## Etcd大量慢查询日志

查询etcd官方的建议：

* 建议节点都使用SSD：因为基于raft协议，所以要等多数节点dist commit之后才算是存储完成达成一致，所以磁盘的写入性能很重要，不然可能会发生水桶效应
* 对于当前中型的集群规模应该使用4核16G的配置限制

## 自定义Metric ApiServer错误原因

自定义的ApiServer接口会被`AvailableConditionController`这样的对象扫描以此来确定当前的状态是否正常

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220819113937380.png" alt="image-20220819113937380" style="zoom: 50%;" />

在检查函数检查完毕之后会进行apiservice资源的`updateStatus`（更新其状态），这一步会去更新ETCD，如果此时ETCD不可用，那么就会报出`connect timeout`的错误，但是这个错误不是客户端与apiService的连接错误

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220819114709822.png" alt="image-20220819114709822" style="zoom:50%;" />

然后对节点进行磁盘和CPU的分析：发现一个节点的负载很高（就是一直在使用一个节点），etcd的负载均衡有问题

然后看了etcd client连接etcd server的源码，其中关键点在于：

* Apiservice会根据资源的名称和版本定义来申明etcd client，这个和etcd的连接数量正相关k8s的资源名称和版本数量（v1, v1/apps/, v1extension …）
* 我们集群使用的是clientv3-grpc1.7版本，此版本只会维护**一个**到`endpoint`的连接，默认使用第一个`endpoint`连接
* 只要第一个连接不是服务中断的状态，etcdClient就会一直连接重试，而不是新建连接

对应的操作：

* 升级etcd server的版本，针对apiserver 每个节点启动时修改etcd_servers的参数，将多个host调换顺序，实现etcd server的负载均衡

# 三、结果

## etcd

将CPU提升到限额4核16G之后，etcd的慢查询日志从1200+到20+

CPU的使用情况，5min基本在180%～250%之间

结论：没有按照官方的配置建议及时提升配置，主要的瓶颈在于CPU，磁盘并不是主要的瓶颈

## Api Server

负载均衡
