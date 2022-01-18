---
title: go-zero项目开发重点记录
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-18 14:29:46
---

> * https://go-zero.dev/cn

[TOC]

记录学习官方文档中项目开发的重点知识

<!-- more -->

# 一、go-zero的CI/CD的生态

![image-20220118143444909](http://xwjpics.gumptlu.work/image-20220118143444909.png)

# 二、一般的开发流程

![image-20220118141013688](http://xwjpics.gumptlu.work/image-20220118141013688.png)

https://go-zero.dev/cn/dev-flow.html

官方给出了整体的开发流程：

- goctl环境准备
- 数据库设计
- 业务开发
- 新建工程
- 创建服务目录
- 创建服务类型（api/rpc/rmq/job/script）
- 编写api、proto文件
- 代码生成
- 生成数据库访问层代码model
- 配置config，yaml变更
- 资源依赖填充（ServiceContext）
- 添加中间件
- 业务代码填充
- 错误处理

# 三、业务分解重点

* **微服务的拆分**：对于一个业务横向的拆分成多个子系统，每个子系统有独立的缓存和数据库机制，每一个子系统都是一个服务，对外提供了两种访问该系统的方式：**api和rpc**

* **rpc调用链**：服务之间的调用是单向的，不会出现循环调用；因为互相依赖调用会互相影响认为对方服务不可用，进入死循环；如果有大量服务之间互相调用，此时应该考虑是否是业务拆分是否合理

* **常见服务类型的目录结构**：

  一个服务不单单只有rpc和api，还可能有如rmq（消息处理系统），cron（定时任务系统），script（脚本）等，所以一个服务的结构可能如下所示：

  ```shell
  user
      ├── api //  http访问服务，业务需求实现
      ├── cronjob // 定时任务，定时数据更新业务
      ├── rmq // 消息处理系统：mq和dq，处理一些高并发和延时消息业务
      ├── rpc // rpc服务，给其他子系统提供基础数据访问
      └── script // 脚本，处理一些临时运营需求，临时数据修复
  ```

* 一个常见的工程目录： 

  ```shell
  mall // 工程名称
  ├── common // 通用库
  │   ├── randx
  │   └── stringx
  ├── go.mod
  ├── go.sum
  └── service // 服务存放目录
      ├── afterSale
      │   ├── api
      │   └── model
      │   └── rpc
      ├── cart
      │   ├── api
      │   └── model
      │   └── rpc
      ├── order
      │   ├── api
      │   └── model
      │   └── rpc
      ├── pay
      │   ├── api
      │   └── model
      │   └── rpc
      ├── product
      │   ├── api
      │   └── model
      │   └── rpc
      └── user
          ├── api
          ├── cronjob
          ├── model
          ├── rmq
          ├── rpc
          └── script
  ```

# 四、实现jwt鉴权功能

https://go-zero.dev/cn/jwt.html















# bug记录

## 1.生成的model层代码的数据库model实例没有缓存选项

错误描述：

编写向model层的服务依赖的时候，`UserModel: model.NewUserModel(sqlx.NewMysql(c.Mysql.DataSource), c.CacheRedis),`没有第二个选项即cache

原因：

检查发现生成的代码model对象的方法就是没有缓存选项，所以检查生成的语句是否正确，发现最后忘记加上参数`-c`

解决：

`goctl model mysql ddl -src user.sql -dir . -c`

