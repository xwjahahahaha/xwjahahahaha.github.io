---
title: Docker的学习
tags:
  - docker
categories:
  - technical
  - docker
toc: true
declare: true
date: 2020-10-05 20:23:39
---

# 一、Docker介绍

## 1.1 用来解决的问题：

1. 环境不一致问题
2. 多用户环境下的互相影响
3. 运维成本过高
4. 安装软件成功过高

## 1.2 Docker的思想

<!-- more -->

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005203818.png)

1.  集装箱

   会将所有需要的内容放进不同的集装箱中，需要这些环境就直接取即可

2. 标准化

   1. 运输的标准化：Docker有一个码头。所有上传的集装箱都放在码头上，谁需要就可以指派大海豚取搬运
   2. 命令的标准化：Docker提供了一系列的命令，帮助我们获取和上传集装箱
   3. 提供了REST的API：衍生出了很多的图形化界面，Rancher

3. 隔离性

   Docker在运行集装箱的内容时，会在Linux内核中单独开辟一块空间，不影响其他



* 注册中心：超级码头，上面放的就是集装箱

* 镜像：集装箱

* 容器：运行起来的镜像

  

