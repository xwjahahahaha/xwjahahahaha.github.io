---
title: k8s_Operator
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



# 一、operator概述

## 1.1 CRD

* CRD （Custom Resource Definition）自定义的k8s资源类型
* k8s通过Api Server，在etcd中注册一种新的资源类型，通过实现Custom Controller来监听资源对象的事件变化

## 1.2 什么是Operator？

* Operator是一个感知应用状态的控制器（通过etcd的watch事件）
* CoreOS推出的旨在简化复杂的，有状态的应用管理的框架

* Operator是一个**有状态应用**的管理框架

* 通过拓展K8S的API来自动创建、管理和配置应用实例

* Operator就是一个管理自定义资源类型CRD的自定义的Controller，与K8S本身的Controller遵循同样的运行模式

## 1.3 k8s中controller的运行模式

![image-20220222115157678](http://xwjpics.gumptlu.work/image-20220222115157678.png)

* 整体是一个循环调用的过程：
  * 不断的从API Server中监听资源变换
  * 第一次Watch到新的资源时写入到Local Store缓存中，当第二次监听时会比较与自己本地缓存的差异
  * 如果有差异那么就会调用Callbacks函数（处理添加、更新删除操作），会将需要变化的任务放入到work队列中
  * Worker会从工作队列中获取任务，读取本地存储（注意是只读），然后交给clients再通过API Server处理

一个例子：

![image-20220222152236046](http://xwjpics.gumptlu.work/image-20220222152236046.png)

# 二、安装operator-sdk和搭建docker registry

## 2.1 安装operator-sdk

https://github.com/operator-framework/operator-sdk

```shell
git clone https://github.com/operator-framework/operator-sdk
make install  # 安装二进制工具
```

## 2.2 搭建docker registry

目的：镜像部署到本地仓库

实现：在docker hub上创建一个私有化仓库（其实和git类似）

```shell
docker run -d -p 5000:5000 --restart always --name registry -v ~/docker-data/docker_registry:/var/lib/registry registry:2
```

> 作用：创建一个registry的容器，挂载宿主机~/docker-data/docker_registry到容器存储镜像，持久化存储防止容器重启后镜像文件丢失，并映射本地端口5000到容器

> 报错：
>
> ```shell
> docker: Error response from daemon: driver failed programming external connectivity on endpoint registry (4f23b8c50972fc97017ec8dc8ebf61b0bad08ec6c9eaa22b4930c72396f04fc8):  (iptables failed: iptables --wait -t filter -A DOCKER ! -i docker0 -o docker0 -p tcp -d 192.168.100.2 --dport 5000 -j ACCEPT: iptables: No chain/target/match by that name.
> ```
>
> 解决：https://blog.csdn.net/ystyaoshengting/article/details/102553889
>
> 向filter表中`DOCKER`链中添加一条规则的时候出错，filter表中没有名字为`DOCKER`的规则链，我们在filter表中创建该规则链
>
> `iptables -t filter -N DOCKER`

在`/etc/docker/daemon.json`中添加:

```shell
"insecure-registries": ["mock.com:5000"]
```

> 注意：mock.com是本地配置的假域名，在`/etc/hosts`中配置`127.0.0.1 mock.com`

查看仓库镜像：

`curl -i http://mock.com:5000/v2/_catalog`

目前是空的镜像：

![image-20220222161059530](http://xwjpics.gumptlu.work/image-20220222161059530.png)

# 三、开发自定义operator

## 3.1 开发的流程

![image-20220222161248009](http://xwjpics.gumptlu.work/image-20220222161248009.png)



开发前的准备：

* 复制需要的k8s.io、controller-runtime的包

  ```shell
  git clone https://github.com/kubernetes/kubernetes.git
  cp -R kubernetes/staging/src/k8s.io/ $GOPATH/src/
  
  git clone https://github.com/kubernetes-sigs/controller-runtime.git
  mkdir $GOPATH/src/sigs.k8s.io
  mv controller-runtime/ $GOPATH/src/sigs.k8s.io/
  ```

使用脚手架初始化operator项目：

```shell
mkdir mock
go mod init mock.com/v1
operator-sdk init imockpod-operator --repo=github.com/mock.com/v1
```







# 四、探索operator在etcd中的存储



