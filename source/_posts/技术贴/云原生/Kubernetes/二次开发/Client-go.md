---
title: Client-go
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 16:16:44
---

# 一、client-go的基本使用

<!-- more -->

## 1. 打印当前k8s集群的所有namespace、Pod信息

* 本地下载访问的配置证书

  * 在master节点的`~/.kube/config`文件中，将其下载到本地环境，并放置在本地用户的根目录此文件夹`~/.kube/`下

* 初始化创建项目目录`gin-client-go`, 并且`get clent-go`sdk包

  * `go get k8s.io/client-go   `

* 编写逻辑：

  ```go
  package main
  
  import (
  	"context"
  	"flag"
  	"fmt"
  	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  	"k8s.io/client-go/kubernetes"
  	"k8s.io/client-go/tools/clientcmd"
  	"k8s.io/client-go/util/homedir"
  	"k8s.io/klog"
  	"path/filepath"
  )
  
  func main() {
  	var kubeConfig *string
  	ctx := context.Background()
  	// 读取用户的k8s配置文件
  	// HomeDir返回当前本地环境用户的根目录
  	if home := homedir.HomeDir(); home != "" {
  		kubeConfig = flag.String("kubeConfig", filepath.Join(home, ".kube", "config"), "absolute path to the k8s config file")
  	} else {
  		kubeConfig = flag.String("kubeConfig", "", "")
  	}
  	flag.Parse()
  	// 构建client-go访问需要的config结构，BuildConfigFromFlags可以通过master Url也可以通过config文件路径
  	config, err := clientcmd.BuildConfigFromFlags("", *kubeConfig)
  	if err != nil {
  		klog.Fatal(err)
  		return
  	}
  	// 根据配置创建客户端set实体
  	clientset, err := kubernetes.NewForConfig(config)
  	if err != nil {
  		klog.Fatal(err)
  		return
  	}
  	// 获取所有Namespace
  	// 这里List的第二个参数就是筛选的选择,在ListOptions中添加（这里不需要就不添加）
  	nameSpaceList, err := clientset.CoreV1().Namespaces().List(ctx, metav1.ListOptions{})
  	if err != nil {
  		klog.Fatal(err)
  		return
  	}
  	namespaces := nameSpaceList.Items
  	for _, ns := range namespaces {
  		fmt.Println(ns.Name)
  	}
  	// 获取Pod
  	podList, err := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
  	if err != nil {
  		klog.Fatal(err)
  		return
  	}
  	fmt.Println("===all pods===")
  	for _, pod := range podList.Items {
  		fmt.Println(pod.Name)
  	}
  }
  ```

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220320095836757.png" alt="image-20220320095836757" style="zoom:50%;" />

