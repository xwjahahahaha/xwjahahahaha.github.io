---
title: kube-eagel架构设计
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-30 12:26:05
---

[TOC]

<!-- more -->

# 一、项目简介

一个在k8s平台上的cpu性能剖析系统，适配多种编程(`c/c++、go、java、python`)语言实现的业务Pod，提供在线不同纬度(`namespace=>deployment=>pod=>container=>pid`)的综合火焰图快速定位代码性能瓶颈

功能演示见：[四、功能介绍 ](# 四、功能介绍)

# 二、使用前提与快速安装

## 1. 使用前提

### 硬性条件

集群每台主机：

* 每台主机都已经安装了本机的`linux`内核版本头文件`linux-header-xxx`

  以`ubuntu`为例：

  ```shell
  # 检查是否安装,查看是否有此目录
  $ ls -l /usr/src/linux-headers-$(uname -r)
  # $ ls /lib/modules/$(uname -r)/build
  # 如果提示没有则可如下安装:
  $ sudo apt-get install linux-headers-$(uname -r)
  ```

  其他`unix`版本安装方式见：https://github.com/iovisor/bcc/blob/master/INSTALL.md#packages

  > 原因：
  >
  > * `BPF`程序依赖基本的内核头文件，集群中不同主机内核不同无法在容器层面适配，所以会在`DaemonSet`启动时寻找主机此目录并挂载

* 最低`linux`内核版本限制：

  * `Stack trace`  内核版本最低要求>=4.6  [`d5a3b1f69186`](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=d5a3b1f691865be576c2bffa708549b8cdccda19)

* 已安装`docker`

  * 目前还只支持`overlay2`


### 软性条件/最佳实践

这些条件并不会影响程序的运行流程与结果，但是会影响结果的展示效果即火焰图的可读性，为了最好的效果建议有能力而为之

* pod中的应用程序进程未`stripped`即未删除符号表

  ```shell
  # 检查方式(仅限于编译型语言),以二进制文件test-server为例
  $ file test-server    # 查看输出最后一部分是否显示no stripped or strpped
  # 或者检查elf符号表文件
  $ readelf -s test-server  # 输出中.symtab section存在则说明符号表还存在
  ```

  > 原因：
  >
  > * `strip`会删除掉程序编译带的符号表，这样`BPF`进行调用栈回溯的时候就会无法解析出地址所对应的函数符号，导致最终火焰图中显示的用户态调用栈为一系列地址

## 2. 快速安装 TODO

项目提供了方便的`kubectl`工具支持一键部署

```shell
$ git clone https://github.com/xwjahahahaha/kube-eagle
$ pushd perf-agent/kube-eagle/
$ go build -o kubectl-eagle ./
$ sudo cp kubectl-eagle /usr/local/bin
$ popd
# 测试安装是否成功
$ kubectl eagle 
```

关于kubectl工具使用说明见：[kube-eagle命令行工具](###kube-eagle 工具使用介绍)

# 三、架构设计

![image-20220803215529685](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220803215529685.png)

- `eagle-agent`：作为`Daemonset`封装定义了不同编程语言类别的`profiler`（见下方），根据程序种类进行`profile`；分别适配了裸机环境、容器docker环境、k8s环境的进程/容器/pod自动周期性采集
  - `BPF`：主要用于编译型语言的profile
  - `Async-profiler`：用于`java`程序的profile
  - `py-spy`：用于对python程序的profile
- `eagle-server`：接受客户端请求下发任务，聚合多个agent的profile结果，生成火焰图
- 数据存储：redis: 缓存任务队列，检测任务冲突; tsdb: 持久化运行存储profile数据，提供历史时序profile火焰图原始数据
- `kubectl eagle`：基于kubectl插件开发的eagle客户端，用于部署组建与服务交互

# 四、功能介绍

核心功能分为两个方面：

1. 在线profile：使用客户端或者调用接口对未来的一个连续时间段的某个或多个pod进行cpu性能采集生成汇总火焰图
2. 后台持久运行：根据初始配置文件对于每台主机持久化的采集profile信息放入TSDB中格式化存储，再由上层取出格式化展示，更加倾向于历史数据的采集

## 1. 在线Profile

以kubectl客户端工具来介绍在线profile的所有功能

### kube-eagle 工具使用介绍

kube-eagel是一个快速部署、使用分布式profile的kubectl插件（命令行工具）

#### 1. 下载与部署

在你的kubernetes集群中：

##### 下载 TODO

```shell
$ git clone x x x x x x
$ pushd perf-agent/kube-eagle/
$ go build -o kubectl-eagle ./
$ sudo cp kubectl-eagle /usr/local/bin
$ popd
# 测试安装是否成功
$ kubectl eagle 
```

##### 部署分布式profile组件

```shell
$ kubectl eagle profile init
```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/init.gif" alt="init" style="zoom: 67%;" />

#### 2.开始使用

##### Usage

```shell
$ kubectl eagle
```

```
///,        ////                        
 \  /,     /  >.                       
  \  /,  _/  /.    ███████╗ █████╗  ██████╗ ██╗     ███████╗                       
    \__/_   <      ██╔════╝██╔══██╗██╔════╝ ██║     ██╔════╝         
    /<<< \_\_      █████╗  ███████║██║  ███╗██║     █████╗                 
   /,)^>>_._ \     ██╔══╝  ██╔══██║██║   ██║██║     ██╔══╝   
   (/   \\ /\\\    ███████╗██║  ██║╚██████╔╝███████╗███████╗ 
        // '''''   ╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝╚══════╝
 ======((===========================================================
Usage: kubectl eagle [type] 
        -- You can choose any of the following types to continue viewing:
"Type":
        profile                         Sample CPU perf events for a resource in the kubernetes cluster and obtain the flame graph                              
        container                       Monitor container security behavior(support later...)
```

```shell
$ kubectl eagle profile
```

```
Usage: kubectl eagle profile [action] [k8s-source] [source-name]

"Action":
        init                            Initialize the deployment of kube-eagle to the kubernetes cluster
        start                           execution profile
        list                            Displays historical tasks for the past two days

"K8s-Source":
        namespace                       Specify all pods under a namespace of kubernetes
        service                         Specify all pods under a service of kubernetes(label matched)
        deployment                      Specify all pods under a deployment of kubernetes(label matched)
        replicaset                      Specify all pods under a replicaset of kubernetes(label matched)
        pod                             Sample a pod in Kubernetes cluster

"Source-Name":
        The corresponding resource name in kubernetes

"Flags":
        -a, --async              Change synchronization to asynchronous call
        -d, --duration uint16    Duration of a profile, must be set when making a profile (default 30)
        -f, --frequency uint     sample frequency, Hertz (default 999)
        -h, --help               help for start
        -n, --namespace string   Service、Deployment、Replicaset、Pod resource must specify the namespace (default "default")
```

注意：

* 一分钟以下时常的profile任务建议使用同步方式(默认), 大于一分钟(<5min)的profile任务必须使用异步(-a)，等待一段时间后再访问返回的url，可以通过list查看任务状态和描述

##### 查看某一个Pod的CPU消耗

```shell
$ kubectl eagle profile start pod kube-apiserver-cloudgamer-test-01 -n kube-system -d 30
```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/profile_pod-20220803223306445.gif" alt="profile_pod" style="zoom: 67%;" />

对于namespace和duration两个flag，如果不设置默认的值分别是default和30

##### 查看集群中的某个Deployment下所有Pod的CPU消耗

```shell
$ kubectl eagle profile start deployment redis -n test -d 10
```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/profile_deployment-20220803223307341.gif" alt="profile_deployment" style="zoom:67%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/profile_deployment.svg" alt="profile_deployment" style="zoom:50%;" />

##### 查看整个Namespace的CPU消耗（异步）

对于>1min的profile任务使用异步调用:

```shell
$ kubectl eagle profile start namespace test -d 120 -a
```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/async_profile_namespace.svg" alt="async_profile_namespace" style="zoom:50%;" />

##### 使用list命令查看最近两天的profile任务状态和描述

```shell
$ kubectl eagle profile list
```

```text
ID			TimeStamp		Status		Desc
2904c8d098		2022-06-24 14:23:56	Completed	all profile objects results is empty, so not generate flame figure, please try to increase the acquisition time	
9e2017d384		2022-06-24 15:00:14	Completed		
e0a613b6a7		2022-06-24 15:28:49	Running		
6e88709188		2022-06-24 15:19:39	Completed		
```

##### 调整采样的频率

-f参数可以调整采样的频率，不设置默认是999HZ

```shell
$ k eagle profile start namespace test -f 49   # 49HZ
$ k eagle profile start namespace test -f 2000 # 2000HZ
```

同一个Namespace对比两种设置不同的HZ结果:

49HZ:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/49hz.svg" alt="49hz" style="zoom:50%;" />

2000HZ:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/2000hz.svg" alt="2000hz" style="zoom:50%;" />

#### 3. 卸载删除

(目前命令行还未支持)

```shell
$ sudo rm -rf /usr/local/bin/kubectl-eagle
$ kubectl delete ns eagle
```

## 2. 后台持久profile

（待更新）

# 五、RoadMap

