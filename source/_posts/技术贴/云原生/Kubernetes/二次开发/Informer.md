---
title: Informer
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 16:18:03
---

# 1. 为什么需要Informers？

因为如果希望每次都查询到新的数据就需要不断的调用API Server接口的请求，这样的方式会给服务器带来巨大的压力

所以期望采用watch机制去监听资源的变化

<!-- more -->

# 2. 什么是Informers？

`Informers`是这个事件接口和带索引查找功能的内存缓存的组合

第一次调用时会全量查询（调用`List`来获取所有对象集合），然后就会通过`Watch`机制来获取增量的对象更新缓存

# 3. 工作原理

![image-20220322140501767](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220322140501767.png)

一个简单的实现样例：

```go
package main

import (
	"fmt"
	v1 "k8s.io/api/apps/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/klog/v2"
	"sigs.k8s.io/controller-runtime/pkg/client/config"
	"time"
)

func main() {
	// 获取用户根目录下的k8s配置文件，与之前的方式一样，只不过这里封装了
	config, err := config.GetConfig()
	if err != nil {
		panic(err)
		return
	}
	// 创建clientSet
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
		return
	}
	// 创建Informer
	// 第二个参数表示周期性调用lister方法获取数据
	informerFactory := informers.NewSharedInformerFactory(clientSet, 30*time.Second)
	// 生成deployment的Informer
	deploymentInformer := informerFactory.Apps().V1().Deployments()
	Informer := deploymentInformer.Informer()
	deployList := deploymentInformer.Lister()
	// 给Informer添加事件
	Informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    onAdd,
		UpdateFunc: onUpdate,
		DeleteFunc: onDelete,
	})
	stoper := make(chan struct{})
	defer close(stoper)
	// 开启list & watch
	informerFactory.Start(stoper)
	// 等待所有Informer缓存同步
	informerFactory.WaitForCacheSync(stoper)
	// 获取所有的deployments
	deployments, err := deployList.Deployments("kube-system").List(labels.Everything())
	if err != nil {
		panic(err)
		return
	}
	for idx, dp := range deployments {
		fmt.Printf("%d => %s\n", idx, dp.Name)
	}
	<-stoper
}

// 添加对象时的事件
func onAdd(obj interface{}) {
	deployment := obj.(*v1.Deployment)
	klog.Infof("add a new deployment: %s\n", deployment.Name)
}

// 更新对象事件
func onUpdate(oldObj interface{}, newObj interface{}) {
	oldDeployment := oldObj.(*v1.Deployment)
	newDeployment := newObj.(*v1.Deployment)
	klog.Infoln("update deploy: ", oldDeployment.Status.Replicas, newDeployment.Status.Replicas)
}

// 删除对象事件
func onDelete(obj interface{}) {
	deployment := obj.(*v1.Deployment)
	klog.Infoln("delete a deploy:", deployment.Name)
}
```

