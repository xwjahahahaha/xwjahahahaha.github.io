---
title: gVisor项目理解
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-09 16:35:33
---

[TOC]

> * 官方文档：https://gvisor.dev/docs/

# 零、基础补全

## 1. 容器逃逸

https://www.dockone.io/article/9803

首先，攻击者通过劫持容器化业务逻辑，或直接控制（CaaS等合法获得容器控制权的场景）等方式，已经获得了容器内某种权限下的命令执行能力；攻击者利用这种命令执行能力，**借助一些手段进一步获得该容器所在直接宿主机**（经常见到“物理机运行虚拟机，虚拟机再运行容器”的场景，该场景下的直接宿主机指容器外层的虚拟机）**上某种权限下的命令执行能力**。

## 2.Filesystem bundles

`OCI`规定容器文件系统包必须包含如下两个部分：

* `Config.json`: 包含了容器的所有配置

* `rootFS`: 容器的根文件系统

[OCI runtime spec](https://github.com/opencontainers/runtime-spec) 

https://github.com/opencontainers/runtime-spec/blob/main/bundle.md

## 3. ptrace沙箱

ptrace是一个系统调用，也可以通过其实现沙箱

https://qianfei11.github.io/2020/04/18/Linux-Sandbox-Ptrace/

<!-- more -->

# 一、什么是gVisor？

* `gVisor`是一个go编写的应用程序内核，实现了大部分的linux内核接口，实现了应用程序与操作系统的隔离层

* `gVisor`实现了一个`OCI runtime`叫`runsc`, 已经适配了docker和k8s

# 二、gVisor特点是什么？

* 通过为每个启动的容器/进程设置容器沙盒，避免容器泄漏风险，同时还保证最小化开销

* 对比不同的容器安全解决方案：

  * 虚拟内核完全隔离 （牺牲效率保证完全隔离，强适配）
  * `seccomp`等过滤系统调用规则 （牺牲效率保证部分隔离，适配性差）

  `gVisor`采用了折中的方案：

  * 拦截系统调用并充当内核(但是并不需要像虚拟化硬件一样转换)

  * 又更加灵活的资源调用（即基于线程和内存映射，而不是固定的客户物理资源），不会像传统虚拟机方案占用固定大量资源

  * 但是牺牲了兼容性和每个系统调用的开销（高繁重的系统调用效率可能会很差）

  * 通过规则过滤执行(rule-based)来实现深度防御

  * gVisor类似用户模式Linux(UML)但是与UML不同的是，不是固定占用资源（UML是虚拟化硬件固定占用资源）

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220509202546287.png" alt="image-20220509202546287" style="zoom:50%;" />

# 三、架构/框架是什么？

一个`gVisor`由多个进程组成从而构建出其环境

每个沙箱都有独自的实例：

* `Sentry`：运行容器并拦截和响应应用程序系统调用的内核（应用程序内核）

每个运行在沙箱中的容器都有独自的实例：

* `Gofer`: 负责容器的文件访问

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220509205417832.png" alt="image-20220509205417832" style="zoom:50%;" />

# 四、runsc的作用？

`runsc`实现了 [Open Container Initiative (OCI)](https://www.opencontainers.org/) `runtime`标准，能够使用在Docker和k8s上，这意味着runsc可以运行和OCI兼容文件系统包(Filesystem bundles)容器

# 五、Sentry的作用？

`Sentry`主要功能?

* 用户态/应用程序内核，能够实现如下功能：系统调用、信号传递、内存管理和页面错误逻辑、线程模型等等
* `gVisor`还实现了自己的网络堆栈`netstack`(用户态堆栈), 网络堆栈的所有功能都由`Sentry`内部处理，具体配置[网络](https://gvisor.dev/docs/user_guide/networking/)
* 每个sentry对应一个sandbox，在一个sandbox中可以运行多个container(类似于Pod)

容器进程系统调用过程？

* 当容器进程进行系统调用时，将会重定向到`Sentry`，`Sentry`处理系统调用并返回，需要注意的是`Sentry`并不会传递给系统内核

* 深度防御：首先，应用程序与主机系统 API 的直接交互被 Sentry 拦截，而后者实现了 `System API`。其次，`Sentry `本身可访问的系统 `API `被最小化为更安全、受限制的集合

  第一个原则最大限度地减少应用程序直接利用主机系统 API 的可能性，第二个原则最大限度地减少间接可利用性，即被利用或有缺陷的 Sentry 的利用（例如链接利用）

本身依赖？

* 作为用户态的内核，`Sentry`需要依靠一些宿主机的系统调用来支持其工作，但是不允许容器进程直接通过其进行系统调用，一些文件操作相关的操作（不是内部`/proc`、管道等文件）会转发给`Gofer`操作，其自身不会直接操作

与VM的主要区别？

* 与 VM 的一个重要区别是 `Sentry` 直接基于主机系统 `API` 原语实现系统 `API`，而不是依赖于虚拟化硬件和客户操作系统

`gVisor`是一个`ptrace`沙箱吗？

* 不是，`ptrace`实现的沙箱都是通过限制容器进程系统调用实现，而`Sentry`却是一个用户态沙箱直接对容器进程的所有系统调用负责，然后再通过对`Sentry`本身依赖的系统调用进行限制从而实现沙箱，两者有本质区别

# 六、Goper的作用？

`Gofer`的定义？如何和`Sentry`通信？

* `Gofer`是一个标准的主机进程，它从每个容器开始，并通过  [9P protocol](https://en.wikipedia.org/wiki/9P_(protocol)) 通过套接字或共享内存通道与 `Sentry` 通信

为什么要有`Goper`?

* `Sentry`是在受限的`seccomp`容器中启动, 无法访问文件系统，所以需要`Goper`配合	
* Goper是每个容器对应一个

# 七、如何拦截容器中的系统调用信号？

采用了两种不同的方式：ptrace和KVM

* 如果在VM中，那么只能用ptrace
* 如果在宿主机下，那么两个都可以使用

![Platforms](http://xwjpics.gumptlu.work/qinniu_uPic/platforms.png)
