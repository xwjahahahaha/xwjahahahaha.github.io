---
title: k8s安装问题大全
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-09-29 23:28:30
---

[TOC]


<!-- more -->

# 1. docker pull 拉取k8s.gcr.io镜像失败

## 错误描述

```shell
Failed to create sandbox for pod" err="rpc error: code = Unknown desc = failed to get sandbox image "k8s.gcr.io/pause:3.6": failed to pull image...
```

导致了`kubelet`一直报错：

![image-20220930093752002](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220930093752002.png)

apiServer访问失败：

```log
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.447330   22478 kubelet.go:2448] "Error getting node" err="node \"master\" not found"
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.548541   22478 kubelet.go:2448] "Error getting node" err="node \"master\" not found"
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606504   22478 remote_runtime.go:222] "RunPodSandbox from runtime service failed" err="rpc error: code = Un>
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606576   22478 kuberuntime_sandbox.go:71] "Failed to create sandbox for pod" err="rpc error: code = Unknown>
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606594   22478 kuberuntime_manager.go:772] "CreatePodSandbox for pod failed" err="rpc error: code = Unknown>
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606635   22478 pod_workers.go:965] "Error syncing pod, skipping" err="failed to \"CreatePodSandbox\" for \">
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606703   22478 remote_runtime.go:222] "RunPodSandbox from runtime service failed" err="rpc error: code = Un>
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606731   22478 kuberuntime_sandbox.go:71] "Failed to create sandbox for pod" err="rpc error: code = Unknown>
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606746   22478 kuberuntime_manager.go:772] "CreatePodSandbox for pod failed" err="rpc error: code = Unknown>
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.606789   22478 pod_workers.go:965] "Error syncing pod, skipping" err="failed to \"CreatePodSandbox\" for \">
Sep 30 01:34:33 master kubelet[22478]: E0930 01:34:33.649030   22478 kubelet.go:2448] "Error getting node" err="node \"master\" not found"
```

## 分析与解决

> 参考：
>
> * https://www.365seal.com/y/qevqlMb0nX.html
> * 类似问题的issue：https://github.com/kubernetes/minikube/issues/14641

一开始以为是docker的镜像安装`k8s.gcr.io`外网环境的问题，但是在`kubeadm init`的时候是加了`image`的国内仓库镜像的

```shell
sudo kubeadm init --apiserver-advertise-address=192.168.88.129 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version v1.25.2 --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=master --v=5
```

然后还自己手动做了一下docker的tag，防止因为外网的原因超时无法启动apiserver

```shell
sudo docker pull registry.cn-hangzhou.aliyuncs.com/kuberimages/pause:3.6
sudo docker tag registry.cn-hangzhou.aliyuncs.com/kuberimages/pause:3.6 k8s.gcr.io/pause:3.6
```

但是随后发现这与这个问题无关，因为现在k8s使用`contained`而不是`docker`，所以即使删除掉所有docker镜像都没关系

所以检查contained，使用`crictl`命令行工具：`sudo crictl info`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220930103810274.png" alt="image-20220930103810274" style="zoom:50%;" />

从输出中可以看到，这个错误的源头在于安装k8s组件的时候（使用静态pod），contained默认使用的sandbox镜像是`k8s.gcr.io/pause:3.6`,但是这个镜像地址是外网，所以同样要做一个tag：

```shell
sudo crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6
sudo ctr -n k8s.io i tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
```

同理，其他所有的wroknode也需要执行上述的操作

`kubeadm reset`然后重新init，至此解决了这个问题

> * kubelet报错"Error getting node" err="node \"master\" not found"是很常见的问题，因为连接不上api-server无法注册节点，问题应该是发生在此错误之前也就是创建api-server的时候，所以这个错误是一个很宽泛的错误，要去前面查更加详细的错误log

# 2. coredns一直卡在容器创建

## 错误描述

![image-20220930115301555](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220930115301555.png)

## 分析与解决

一开始以为是`containerd`镜像代理的问题，后来查看`kubelet`的日志发现问题是：

` error: /run/flannel/subnet.env: no such file or directory`

具体类似的解决方案：https://github.com/kubernetes/kubernetes/issues/70202

手动添加如下文件：

`/run/flannel/subnet.env`

```json
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

其他建议的方式：https://github.com/kubernetes/kubernetes/issues/70202#issuecomment-1042506375

