---
title: Prometheus改造优化
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-12 11:37:10
---

# 1、总体说明

开源组件prometheus，用于数据采集和监控告警

---

## 源码说明

- 官方网站: https://prometheus.io/
- 当前版本: 2.27.1
- 发布日期: 2021-05-18
- 发布说明: https://github.com/prometheus/prometheus/releases/tag/v2.27.1
- 开源协议: Apache License 2.0
- 下载地址: https://github.com/prometheus/prometheus/archive/v2.27.1.tar.gz

---

<!-- more -->

## 依赖包

相关的依赖已在`make.sh`中处理。部分依赖可以通过内网镜像源在线安装，不方便在线安装的就归档到`deps`目录

- `promu` 
    使用go get在线安装即可：`go get github.com/prometheus/promu`
- `node.js`
    编译环境上node.js版本是v0.12.0（2015年的版本），yarn需要node.js版本高于4.0.0
- `yarn`
    采用npm在线安装即可：`npm install -g yarn`

---

## 端口分配
服务使用端口：19090

## Patch补丁文件说明
- **`0001-remove-yarn-lock.patch`**
    - 处理内网编译依赖问题，删除yarn.lock文件，避免yarn从外网下包

- **`0002-remove-wal-segment-size-restriction.patch`**
    - [WAL](https://baike.baidu.com/item/WAL/6324359?fr=aladdin): Write-Ahead Logging 预写日志系统, 加快了磁盘I/O操作的效率
    - 去掉对WALSegmentSize的限制，以便支持传入负值，禁用WAL功能。
    - prometheus中WAL存在的目的是在prometheus实例挂掉时，用来恢复其内存热数据。
    - WAL是<u>没有经过压缩的原始数据</u>，磁盘空间占用，以及IO开销都很高。并且<u>prometheus每次启动时，都会从磁盘中的WAL将数据加载到内存，WAL中的数据越多，加载越耗时，这一点对于HCI上的采集程序来说，不可接受。</u>
    - <u>数据的持久化方面，我们有设计RRD来做，所以prometheus原生的这部分wal可以禁用掉。</u>
    - 禁用WAL传参示例`--storage.tsdb.wal-segment-size=-1B`
    - WALSegmentSize = 0, segment size is default size.
    - WALSegmentSize > 0, segment size is WALSegmentSize.
    - WALSegmentSize < 0, wal is disabled.
    
- **`0003-send-data-to-statuscenterd.patch`**
    - 新增特性，<u>使prometheus支持将采集到的全部数据发送给HCI状态中心</u>
    - prometheus采集到的数据，默认都会保存在其内置的TSDB中，对于不希望存储到其TSDB的数据，在数据采集阶段加上_**status**_标签即可
    
- **`0004-log-to-file.patch`**
    - prometheus启动时，增加--log.file选项，<u>使prometheus支持指定输出的日志文件路径</u>
    
- **`0005-scrape-offset.patch`**
    - 原有特性，通过向prometheus进程发送SIGHUP或向/-/reload端点发送HTTP POST请求（启用–web.enable-lifecycle标志时）来触发配置重新加载
    - 原有特性，prometheus接收重新加载配置的请求时，会比较每一个scrapepool前后的配置，配置不相等时才会触发reload函数，并且会等待一段时间才会采集
    - 新增特性，由于HCI状态中心服务重启时，需要触发prometheus立即采集一次数据，所以不需要比较每一个scrapepool前后的配置，<u>全部都需要触发reload函数</u>
    - 新增特性，prometheus.yml配置文件中的全局配置global_config新增scrape_offset标签，scrape_config新增scrape_offset标签，这个标签来控制<u>向prometheus进程发送SIGHUP后，prometheus下次采集的时间偏移量</u>
    - scrape_offset: model.Duration类型，默认为-1，若不填写，则默认为源码的方式
    - eg: 1、scrape_offset: -1，表示无效的状态，默认为源码的方式，等待的时间为(0,scrape_interval]s
          2、scrape_offset: 0s，表示立即采集一次数据，等待的时间0s
          3、scrape_offset: 3s，表示等待的时间为(0,3]s
    
- **`0006-async-send.patch`**
    - prometheus将数据转发给状态中心的这部分流程，是嵌在原生的采集落盘流程中的，串行方式
    - <u>当状态中心故障时，prometheus原生的流程本可以正常工作，但在当前新增的串行转发流程中会出错退出本次采集</u>
    - 改动方案：prometheus将数据转发给状态中心的这部分流程，<u>单独起一个协程来处理</u>，避免新增流程对原生流程造成影响
---

## 构建方法
因为prometheus关联common项目，所以应手动先编译common项目，再编译prometheus项目，否则会报错

1.编译common项目

```shell
cd ./common && make
```

2.编译prometheus项目

```shell
cd ./prometheus && make
```

---

## 维护规范

#### 源码修改

- 源码路径(`prometheus-2.27.1`)保存开源项目原始代码，不在其上做直接修改，便于版本更新
- 需要修改的部分，**按特性/问题/Bug独立成patch按顺序存放于patch目录**

#### 版本更新

- 添加新版本目录，删除旧版本目录
- 检查原有patch是否继续适应于新版本
    - 新代码仍需要此patch
        - 打patch时无冲突，不用更新patch
        - 打patch时有冲突或失败，以新版本源码为基础更新patch
    - 新代码不需要此patch（新版本已合入或不存在此问题）
        - 打patch时提示已经applied，不用更新patch
        - 打patch时有冲突或失败，则可以删除patch

# 2、优化总结

* 去掉
