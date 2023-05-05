---
title: kubernetes-3-kubernetes核心技术
tags:
  - kubernetes
categories:
  - technical
  - kubernetes
toc: true
declare: true
date: 2021-05-30 10:00:16
---

# 三、核心技术

## 3.1 Kubectl命令行工具和Yaml文件

### 1.Kubectl

```shell
# 基本格式
$ kubectl [command] [TYPE] [NAME] [flags]
```

<!-- more -->

* 基础命令

  ![1GWEYg](http://xwjpics.gumptlu.work/qinniu_uPic/1GWEYg.png)

* 部署和集群管理命令

  ![Eqjku5](http://xwjpics.gumptlu.work/qinniu_uPic/Eqjku5.png)

* 故障和调试命令

  ![XV5gsc](http://xwjpics.gumptlu.work/qinniu_uPic/XV5gsc.png)

* 其他命令

![Dlc8Gd](http://xwjpics.gumptlu.work/qinniu_uPic/Dlc8Gd.png)

### 2.Yaml配置

* 生成配置文件模版:

  -o 表示生成yaml类型文件,--dry-run表示尝试运行,而不会真的运行

  `kubectl create deployment web --image=nginx -o yaml --dry-run > my.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    creationTimestamp: null
    labels:
      app: web
    name: web
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: web
    strategy: {}
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: web
      spec:
        containers:
        - image: nginx
          name: nginx
          resources: {}
  status: {}
  ```

* 导出已经部署项目配置

  ```shell
  kubectl get deploy nginx -o=yaml --export > my2.yaml
  ```

## 3.2 核心概念-Pod

![lExlbH](http://xwjpics.gumptlu.work/qinniu_uPic/lExlbH.png)

### 1.基本概念

> Pod 是 k8s 系统中可以创建和管理的最小单元，是资源对象模型中由用户创建或部署的最小资源对象模型，也是在 k8s 上运行容器化应用的资源对象，其他的资源对象都是用来支撑或者扩展 Pod 对象功能的，比如**控制器对象是用来管控 Pod 对象的，Service 或者 Ingress 资源对象是用来暴露 Pod 引用对象的，PersistentVolume 资源对象是用来为 Pod 提供存储等等**，k8s 不会直接处理容器，而是 Pod，Pod 是由一个或多个 container 组成
>
> Pod 是 Kubernetes 的最重要概念，**每一个 Pod 都有一个特殊的被称为”根容器“的 Pause 容器**。Pause 容器对应的镜像属于 Kubernetes 平台的一部分，**除了 Pause 容器，每个 Pod 还包含一个或多个紧密相关的用户业务容器**

![rKgsfD](http://xwjpics.gumptlu.work/qinniu_uPic/rKgsfD.png)

![iPQdvS](http://xwjpics.gumptlu.work/qinniu_uPic/iPQdvS.png)

* Pod是k8s中部署的**最小单元**

* k8s不会直接处理容器而是处理Pod, Pod是由一个或者多个容器container组成

* 一个Pod中的容器**网络共享**
* Pod是短暂存在的

> <font color='#e54d42'>**为什么用Pod管理而不用容器?**</font>
>
> * 创建容器使用docker,但是一个docker对应一个容器,一个容器一般为单进程,所以一般运行一个应用程序
> * Pod是多个容器,一个容器运行一个程序, Pod采用多进程的设计, 更加方便管理
> * Pod的存在为了亲密型应用
>   * 两个应用之间进行交互
>   * 网络之间调用
>   * 两个应用之间频繁的调用

### 2. 实现机制

#### 共享网络

![NFwIBG](http://xwjpics.gumptlu.work/qinniu_uPic/NFwIBG.png)

> 通过Pause/Info容器将所有的其他业务容器加入到其中,从而实现共享网络

#### 共享存储

> 数据卷机制, 防止单容器宕机造成重要数据的丢失,所以容器数据会映射到主机持久化存储, Pod中所有的容器共享存储

![K4aYJI](http://xwjpics.gumptlu.work/qinniu_uPic/K4aYJI.png)

![Xba9FT](http://xwjpics.gumptlu.work/qinniu_uPic/Xba9FT.png)

### 3. 配置策略

#### 镜像拉取策略

拉取的三种配置:

![4htPPj](http://xwjpics.gumptlu.work/qinniu_uPic/4htPPj.png)

#### Pod资源限制

![BquhOZ](http://xwjpics.gumptlu.work/qinniu_uPic/BquhOZ.png)

#### 重启机制

 ![Jr5d6G](http://xwjpics.gumptlu.work/qinniu_uPic/Jr5d6G.png)

#### 健康检查

![DoomGO](http://xwjpics.gumptlu.work/qinniu_uPic/DoomGO.png)

### 4.调度分配

使用`kubectl get pods -o wide`可以获取到容器具体运行分配的节点位置:

![TQZ1EL](http://xwjpics.gumptlu.work/qinniu_uPic/TQZ1EL.png)

那么Pod创建的流程是什么,怎样分配容器的部署的呢?

#### 创建流程

用apiserver作为统一的管理, 使用etcd作为存储

![AdazvN](http://xwjpics.gumptlu.work/qinniu_uPic/AdazvN.png)

#### 调度影响

##### 资源限制

> 对于Pod配置的需求,选择符合要求的节点进行分配

![WirCP8](http://xwjpics.gumptlu.work/qinniu_uPic/WirCP8.png)



##### 节点选择器

> 对当前所有的节点进行标签化处理,然后通过标签选择节点

* 给节点打标签

  `kubectl label node node1 env_role=dev`

  查看节点标签: `kubectl get nodes cosmosibc01 --show-labels`

  ![97Jo16](http://xwjpics.gumptlu.work/qinniu_uPic/97Jo16.png)

  ![h5juFf](http://xwjpics.gumptlu.work/qinniu_uPic/h5juFf.png)

* 在yaml文件中配置节点选择器, 指定环境选择标签节点

  ![tmmQeV](http://xwjpics.gumptlu.work/qinniu_uPic/tmmQeV.png)

![EUr32s](http://xwjpics.gumptlu.work/qinniu_uPic/EUr32s.png)

##### 节点亲和性

![3Wy2ri](http://xwjpics.gumptlu.work/qinniu_uPic/3Wy2ri.png)

NotIn和DoesNotExists可以用于反亲和性

##### 污点以及污点容忍

> * **节点亲和性的角度是从Pod, 在Pod配置文件中实现配置**
> * **污点则是直接从节点本身的角度, 对节点进行配置**

![IZ1Tkl](http://xwjpics.gumptlu.work/qinniu_uPic/IZ1Tkl.png)

* 查看节点污点情况: `kubectl describe node [node] | grep Taints`

* 节点添加污点: `kubectl taint node [node] key=value:污点值`
* 删除节点污点: `kubectl taint node [node] key:污点值- `   (注意最后有一个横杠)

![eNyKwi](http://xwjpics.gumptlu.work/qinniu_uPic/eNyKwi.png)

## 3.3 核心概念-Controller

### 1.基本概念

**Controller是在集群中管理和运行容器的对象**

Pod与Controller之间的关系:

* 建立关系方式: **通过label标签建立关系**

  ![rdtIlF](http://xwjpics.gumptlu.work/qinniu_uPic/rdtIlF.png)

* **Pod通过Controller实现应用的运维**: 例如伸缩、滚动升级等

![DZv6m3](http://xwjpics.gumptlu.work/qinniu_uPic/DZv6m3.png)

### 2. Deployment(部署无状态应用)

Deployment 是 Kubenetes v1.2 引入的新概念，引入的目的是为了更好的**解决Pod的编排问题**，Deployment 内部使用了 Replica Set 来实现.

#### 应用场景

* 部署无状态的应用
* 管理Pod和ReplicaSet
* 部署,滚动升级等功能
* **一般应用于Web服务、微服务**

#### 部署应用(yaml)

* 导出yaml文件,并修改

  通过cli部署命令将yaml文件导出:

   `kubectl create deployment [应用名] --image=[镜像名] --dry-run -o yaml > xxx.yaml `

  (第三个参数表示选择Deployment控制器,  —dry-run表示尝试运行)

  下面以nginx镜像创建的web应用为例:

  ```yaml
  # web.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:  
    creationTimestamp: null
    labels:				
      app: web
    name: web
  spec:                                                                                                                  
    replicas: 1  
    selector:   			# label的应用名与selector相配对 
      matchLabels:
        app: web
    strategy: {}
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: web  	# label的应用名与selector相配对
      spec:
        containers:
        - image: nginx
          name: nginx
          resources: {}
  status: {}
  ```

* 通过配置文件运行:

  `kubectl apply -f web.yaml `

  ![thRHQV](http://xwjpics.gumptlu.work/qinniu_uPic/thRHQV.png)

* 对外发布(暴露对外端口号)

  生成yaml文件

  `kubectl expose deployment web --port=80 --type=NodePort --target-port=80 --name=web1 -o yaml > web-expose.yaml`

  > * port是服务的服务端口
  >
  > * target-port是容器中提供服务的端口
  >
  > * type是服务的类型, 有如下类型: (默认第一种)
  >
  >   ClusterIP, NodePort, LoadBalancer, or ExternalName

  应用:

  `kubectl apply -f web-expose.yaml `

  查看:

  `kubectl get pods,svc`

#### 应用升级回滚与弹性伸缩

* 应用升级

	升级命令: `kubectl set image deployment web nginx=nginx:1.15`

	![Zff0kl](http://xwjpics.gumptlu.work/qinniu_uPic/Zff0kl.png)

	在节点二查看镜像:

	![ivLUTy](http://xwjpics.gumptlu.work/qinniu_uPic/ivLUTy.png)

	查看升级状态:
	
	`kubectl rollout status deployment web`
	
	![kg76TU](http://xwjpics.gumptlu.work/qinniu_uPic/kg76TU.png)

* 应用回滚

  查看历史版本:

  `kubectl rollout history deployment web`

  ![kxnFMe](http://xwjpics.gumptlu.work/qinniu_uPic/kxnFMe.png)

  回滚到上一个版本:

  `kubectl rollout undo deployment web`

  ![Cmv8VQ](http://xwjpics.gumptlu.work/qinniu_uPic/Cmv8VQ.png)

  回滚到指定版本:

  `kubectl rollout undo deployment web --to-revision=2`

  弹性伸缩:

  > 设置冗余副本数量

  `kubectl scale deployment web --replicas=10`

  ![fBJ89S](http://xwjpics.gumptlu.work/qinniu_uPic/fBJ89S.png)

### 3. StatefulSet(部署有状态应用)

#### 1.有状态与无状态对比

![LOlcdH](http://xwjpics.gumptlu.work/qinniu_uPic/LOlcdH.png)

#### 2.部署过程

配置文件关键点:

* 无头Service, ClusterIP: None
* StatefulSet部署有状态应用

配置文件实例 (有两个配置,一部分是无头Service, 一部分是StatefulSet):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None		# 设置为None
  selector:
    app: nginx

---

apiVersion: apps/v1
kind: StatefulSet		# 设置为StatefulSet
metadata:
  name: nginx-statefulset
  namespace: default
spec:
  serviceName: nginx
  replicas: 3			# 3个副本
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

部署应用: `kubectl alpply -f sts.yaml`

![gcmOo1](http://xwjpics.gumptlu.work/qinniu_uPic/gcmOo1.png)

查看创建的无头Service:

![GgzER1](http://xwjpics.gumptlu.work/qinniu_uPic/GgzER1.png)

查看Pod, 每个Pod都有唯一的名称:

![wUZw6g](http://xwjpics.gumptlu.work/qinniu_uPic/wUZw6g.png)

**deployment和statefulset的区别在于: statefulset是有身份的(唯一标识的), 根据主机名和一定的规则生成域名**

* 每个Pod都有唯一的主机名

* 唯一的域名:

  * 格式: `主机名称.service名.名称空间.svc.cluster.local`

    例如: `nginx-statefulset-0.nginx.default.svc.cluster.local`

### 4. DaemonSet(部署守护进程)

守护进程: 确保所有的node运行同一个Pod , 或者说把同一个Pod部署到所有的node中, 新加入的node也同样运行在一个Pod中

* 例: 每个节点都做数据采集的工作:

配置文件:

```yaml
apiVersion: apps/v1
kind: DaemonSet				# DaemonSet类型
metadata:
  name: ds-test 
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: logs
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: varlog
          mountPath: /tmp/log
      volumes:						# 同一数据卷
      - name: varlog
        hostPath:
          path: /var/log
```

应用:`kubectl apply -f ds.yaml`

查看: 

![l0ygBp](http://xwjpics.gumptlu.work/qinniu_uPic/l0ygBp.png)

可以看到每一个worknode节点都运行了该Pod

### 5.一次性任务与定时任务

#### 5.1 job一次性任务

配置创建:

```yaml
apiVersion: batch/v1
kind: Job					# 类型为Job
metadata:
  name: pi				# 创建一个圆周率计算任务
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4		# 最大重启次数
```

应用后查看: `kubectl get pods,jobs`

![ToqImt](http://xwjpics.gumptlu.work/qinniu_uPic/ToqImt.png)

查看日志(运行结果):

![RCb3D6](http://xwjpics.gumptlu.work/qinniu_uPic/RCb3D6.png)

#### 5.2 cronjob定时任务

定时执行的任务,会按定时创建多个

配置文件:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"				# 定时执行表达式
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster			# 定时输出
          restartPolicy: OnFailure
```

应用并查看:

![Cz8kaj](http://xwjpics.gumptlu.work/qinniu_uPic/Cz8kaj.png)

![42gf87](http://xwjpics.gumptlu.work/qinniu_uPic/42gf87.png)

一段时间就会创建一个执行一个job:

![NI9eKf](http://xwjpics.gumptlu.work/qinniu_uPic/NI9eKf.png)

## 3.4 核心概念-Service

我们在向外暴露端口的操作时,使用的配置文件就是Service的配置

### 1. 作用

* 防止Pod失联(服务发现)

  每一个Pod运行的时候, 回滚更新等操作IP都是临时的、变化的.所以对于Pod之间的交互来说,需要一个service来作为服务发现去管理这些IP,实现不断连

  ![ywLEAR](http://xwjpics.gumptlu.work/qinniu_uPic/ywLEAR.png)

  ![2JMhMv](http://xwjpics.gumptlu.work/qinniu_uPic/2JMhMv.png)

* 定义一组Pod访问策略(负载均衡)

  对于多个请求通过service实现请求的负载均衡

  ![1WCTzK](http://xwjpics.gumptlu.work/qinniu_uPic/1WCTzK.png)

### 2. Pod和Service的关系

![ha8idm](http://xwjpics.gumptlu.work/qinniu_uPic/ha8idm.png)

serice使用vip虚拟IP实现对外的服务

### 3. 常用Service类型

![ZnJgDs](http://xwjpics.gumptlu.work/qinniu_uPic/ZnJgDs.png)

* 创建一个ClusterIP集群**内部访问**的服务

  导出service的yaml文件: `kubectl expose deployment web --port=80 --target-port=80 --dry-run -o yaml >service1.yaml `

  修改expose导出的yaml文件: 

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: null
    labels:
      app: web
    name: web2
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 80
    selector:
      app: web
  status:
    loadBalancer: {}
  ```

  默认就是ClusterIP类型, 应用:

  `kubectl apply -f service1.yaml `

  查看与访问:

  `kubectl get svc`

  其他节点访问: `10.0.0.100`

  ![Ihr0AA](http://xwjpics.gumptlu.work/qinniu_uPic/Ihr0AA.png)

## 3.5 核心概念-Secret

作用: **加密数据**存在etcd中, 让Pod容器以挂载Volume的方式进行访问

场景: 凭证

### 1.创建Secret加密数据

配置文件:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

应用:`kubectl apply -f secret.yaml`

查看:`kubectl get secret`

![MJekMa](http://xwjpics.gumptlu.work/qinniu_uPic/MJekMa.png)

### 2.以变量形式挂载到Pod容器中

配置文件:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: nginx
    image: nginx
    env:
      - name: SECRET_USERNAME			# 变量名
        valueFrom:
          secretKeyRef:						# secret中的关联key
            name: mysecret
            key: username
      - name: SECRET_PASSWORD			# 变量名
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
```

创建Pod: 

`kubectl apply -f secret_var.yaml`

查看:

`kubectl get pods`

![OWhQUr](http://xwjpics.gumptlu.work/qinniu_uPic/OWhQUr.png)

进入容器中查询变量:

```shell
kubectl exec -it  mypod bash
echo $SECRET_USERNAME
echo $SECRET_PASSWORD
```

![XWeL10](http://xwjpics.gumptlu.work/qinniu_uPic/XWeL10.png)

### 3.以Volume的形式挂载到Pod容器中

配置文件:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"			# 挂载的目录 
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret			# 这里要与secret对应
```

应用并查看:

```shell
kubectl apply -f secret_vol.yaml 
kubectl exec -it mypod bash
cd /etc/foo/
cat password
cat username
```

![MkM66R](http://xwjpics.gumptlu.work/qinniu_uPic/MkM66R.png)

## 3.6 核心概念-ConfigMap

作用: 存储**不加密数据**到etcd, 让Pod以变量或者Volume挂载到容器中

场景: 配置文件

### 1.创建configMap

redis.properties文件:

```properties
redis.host=127.0.0.1
redis.port=6379
redis.password=123456
```

```shell
# kubectl create [类型] [自定义名称] [flags]
kubectl create configmap redis-config --from-file=redis.properties
kubectl get cm
kubectl describe cm redis-config
```

![Ntrp3m](http://xwjpics.gumptlu.work/qinniu_uPic/Ntrp3m.png)

### 2.以Volume形式进行挂载Pod容器

配置文件:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: busybox
      image: busybox
      command: [ "/bin/sh","-c","cat /etc/config/redis.properties" ]		# 运行的命令
      volumeMounts:
      - name: config-volume			
        mountPath: /etc/config								# 挂载的目录
  volumes:
    - name: config-volume
      configMap:
        name: redis-config										# 创建的configmap的名字
  restartPolicy: Never
```

```shell
kubectl apply -f cm_yol.yaml 
kubectl logs mypod
```

![fyvQAM](http://xwjpics.gumptlu.work/qinniu_uPic/fyvQAM.png)

### 3.以变量形式挂载到Pod容器中

创建configMap的配置文件:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfig						# 名字
  namespace: default
data:
  special.level: info				# 两个变量
  special.type: hello
```

创建Pod的配置文件:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: busybox
      image: busybox
      command: [ "/bin/sh", "-c", "echo $(LEVEL) $(TYPE)" ]		
      env:
        - name: LEVEL
          valueFrom:
            configMapKeyRef:
              name: myconfig			# 对应configmap
              key: special.level	# 对应变量
        - name: TYPE
          valueFrom:
            configMapKeyRef:
              name: myconfig
              key: special.type
  restartPolicy: Never
```

![vWc0Lp](http://xwjpics.gumptlu.work/qinniu_uPic/vWc0Lp.png)

## 3.7 核心概念-NameSpace命名空间

Namespace 在很多情况下用于实现多用户的**资源隔离**，通过将集群内部的资源对象分配到 不同的Namespace中，形成逻辑上的分组，便于不同的分组在共享使用整个集群的资源同时还能被分别管理。**Kubernetes 集群在启动后，会创建一个名为"default"的 Namespace**， 如果不特别指明 Namespace,则用户创建的 Pod，RC，Service 都将被系统创建到这个默认的名为 default 的 Namespace 中。

> 简单来说**NameSpace就是k8s中用于在逻辑上隔离资源的配置工具**

相关操作:

```shell
# 获取所有命名空间
kubectl get ns		
# 新增一个命名空间
kubectl create ns [自定义名称]
# 获取对应命名空间的资源
kubectl get pods,svc --namespace=[目标命名空间名称]
# 使用一个命名空间
flag: -n [目标命名空间名称] 或者 --namespace=[目标命名空间名称]
```

![kMngTv](http://xwjpics.gumptlu.work/qinniu_uPic/kMngTv.png)

使用配置文件创建:

```yaml
apiVersion: v1 
kind: Namespace 								# 创建命名空间
metadata: 
  name: development							# 空间名为development

---

apiVersion: v1 
kind: Pod 
metadata: 
  name: busybox 
  namespace: development 				# 指定Pod的命名空间
spec:
  containers:
    - image: busybox 
      command: [ "/bin/sh", "-c", "sleep 5" ]		
      name: busybox
```

应用并查看:

```shell
kubectl apply -f ns.yaml 
kubectl get ns
kubectl get pods --namespace=development
kubectl get pods   # 默认default命名空间无法找到创建的Pod
```

![Kub4Yt](http://xwjpics.gumptlu.work/qinniu_uPic/Kub4Yt.png)

## 3.8 核心概念-集群安全机制

### 1.概述

![drMTIz](http://xwjpics.gumptlu.work/qinniu_uPic/drMTIz.png)

![CTN4xp](http://xwjpics.gumptlu.work/qinniu_uPic/CTN4xp.png)

![JWLBkP](http://xwjpics.gumptlu.work/qinniu_uPic/JWLBkP.png)

![U5V3KU](http://xwjpics.gumptlu.work/qinniu_uPic/U5V3KU.png)

### 2.鉴权: RBAC基于角色的访问控制

**Role-Based policies Access Control**

![uBkqcN](http://xwjpics.gumptlu.work/qinniu_uPic/uBkqcN.png)

**主体与角色绑定, 规划角色的访问权利, 其关联主体也随之由对应的权利**

* 角色
  * role: 对特定的命名空间访问访问权限控制
  * ClusterRole: 所有的命名空间访问权限控制

* 角色绑定
  * roleBinding: 角色绑定到主体
  * ClusterRoleBinding: 集群角色绑定到主体
* 主体
  * user: 用户
  * group: 用户组
  * serviceaccount: 服务账户

#### 1.创建role角色与角色绑定

创建一个命名空间

`kubectl create ns rbactest`

创建Role配置文件:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role							# 类别是role
metadata:
  namespace: rbactest			# 命名空间(必须已存在/创建)
  name: pod-reader			# 角色名
rules:
- apiGroups: [""] 			# "" 表示核心API组
  resources: ["pods"]		# 访问的资源只有Pod
  verbs: ["get", "watch", "list"]		# 访问方法, ["*"]则表示全部
```

```shell
# 创建角色
kubectl apply -f rbac-role.yaml 
# 查看(注意指定命名空间)
kubectl get role -n rbactest
# 结果输出
NAME         CREATED AT
pod-reader   2021-06-06T12:55:03Z
```

创建RoleBinding配置文件:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding			# 类别是RoleBinding
metadata:
  name: read-pods			# 绑定关系的名称
  namespace: rbactest	# 关联的命名空间
subjects:
- kind: User
  name: mary 					# 名字区分大小写, 这里绑定mary
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role 					#this must be Role or ClusterRole
  name: pod-reader 		# 这必须与创建的Role想匹配
  apiGroup: rbac.authorization.k8s.io
```

```shell
# 应用
kubectl apply -f rbac-rolebinding.yaml 
# 查看
kubectl get rolebinding -n rbactest
# 结果输出
NAME        ROLE              AGE
read-pods   Role/pod-reader   88s
```

#### 2.创建Cluster角色与角色绑定

此部分在二进制搭建k8s集群中已经实现, 其目的是**创建kubernetes访问worknode的kubelet的权限**, 具体过程如下:

创建配置文件`apiserver-to-kubelet.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole											# ClusterRole类别
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kubernetes-to-kubelet		# 名称 system表示系统级别
  # ClusterRole 不受限于命名空间，所以省略了 namespace name 的定义
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding					# ClusterRoleBinding类别
metadata:
  name: system:kubernetes
  namespace: ""										# 默认命名空间
roleRef:													# 标明绑定的角色
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubernetes-to-kubelet
subjects:													# 标明所绑定的角色
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes							# 角色名
```

```shell
# 应用
kubectl apply -f apiserver-to-kubelet.yaml
# 查看
kubectl get clusterrole,clusterrolebinding
# 结果
NAME                                                                   CREATED AT
...
system:kubernetes-to-kubelet                                           2021-06-04T11:37:22Z
...

NAME             ROLE             AGE
...
system:kubernetes             ClusterRole/system:kubernetes-to-kubelet             2d1h
...
```

### 3.认证: 用户证书创建

> 上面的配置都是给k8s集群进行配置, 下面给用户创建证书用于Https访问apiserver

前置工作:

* 创建一个用户目录

  `mkdir mary && cd mary`

* 复制CA证书(二进制集群搭建中就在Master节点的`~/TLS/k8s/`下)

  `cp ~/TLS/k8s/ca* ./`

执行以下步骤进行用户mary证书的创建:

```shell
# 创建csr证书签名请求文件
cat > mary-csr.json <<EOF
{
  "CN": "mary",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

# 使用cfssl工具让CA给mary生成/颁发证书, 生成mary.csr、mary-key.pem、mary.pem三个公私钥文件
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes mary-csr.json | cfssljson -bare mary 

# 给用户mary配置一系列证书配置用于访问api

# 设置/添加集群配置
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.8.122:6443 \
  --kubeconfig=mary-kubeconfig
  
# 设置/添加自身证书配置
kubectl config set-credentials mary \
  --client-key=mary-key.pem \
  --client-certificate=mary.pem \
  --embed-certs=true \
  --kubeconfig=mary-kubeconfig

# 设置/添加上下文配置default
kubectl config set-context default \
  --cluster=kubernetes \
  --user=mary \
  --kubeconfig=mary-kubeconfig
  
# 使用当前环境default
kubectl config use-context default --kubeconfig=mary-kubeconfig
```

正确测试方法：kubectl get pods -n roledemo --kubeconfig=./mary-kubeconfig 切换命名空间看输出结果区别

测试mary用户访问情况:

> 注意在rolebinding配置时mary的权限:
>
> ```yaml
> resources: ["pods"]		# 访问的资源只有Pod
> verbs: ["get", "watch", "list"]		# 访问方法, ["*"]则表示全部
> ```
>
> 所以mary只能访问pod,而不能访问svc

```shell
# 创建一个nginx Pod、 一个configMap
kubectl run nginx --image=nginx -n rbactest
cp ../redis.properties ./ && kubectl create configmap redis-config --from-file=redis.properties -n rbactest
# mary查看Pod
kubectl get pods -n rbactest --kubeconfig=./mary-kubeconfig 
# 结果正常
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          4m2s
# mary查看service
kubectl get svc -n rbactest --kubeconfig=./mary-kubeconfig 

# 错误: Error from server (Forbidden): services is forbidden: User "mary" cannot list resource "services" in API group "" in the namespace "rbactest"

# mary查看configMap
kubectl get cm -n rbactest --kubeconfig=./mary-kubeconfig

# 错误:Error from server (Forbidden): configmaps is forbidden: User "mary" cannot list resource "configmaps" in API group "" in the namespace "rbactest"
```

## 3.9 核心概念-Ingress

![HxLDSw](http://xwjpics.gumptlu.work/qinniu_uPic/HxLDSw.png)

> Ingress是为了弥补k8s自身NodePort的不足, 具体来说就是端口问题

### 1. Ingress与Pod之间的关系

* pod和ingress通过service关联

* ingress作为统一入口,由service关联一组Pod

### 2. Ingress工作流程

ingress不是k8s原生支持的组件, 所以需要**额外单独的安装**

![xKQZs6](http://xwjpics.gumptlu.work/qinniu_uPic/xKQZs6.png)

### 3. 使用ingress

1. 部署ingress Controller
2. 创建ingress规则

----

> 这里选择官方维护的nginx控制器,实现部署流程

#### 创建nginx Pod 以及使用NodePort模式对外暴露端口

```shell
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80 --target-port=80 --type=NodePort
```

#### 部署Ingress Controller

```yaml
# 创建一个工作空间 ingress-nginx
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

# 创建三个ConfigMap
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration			# nginx配置
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services    					# tcp
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services							# udp
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx


# 创建一个ServiceAccount
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

# 创建rbac, clusterRole、Role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

# 两个绑定
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx


# 创建deployment控制器
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      hostNetwork: true				# 这里要设置为true
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      # 容器配置
      containers:
        - name: nginx-ingress-controller
          image: lizhenliang/nginx-ingress-controller:0.30.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 101
            runAsUser: 101
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

# LimitRange配置容器
---

apiVersion: v1
kind: LimitRange
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  limits:
  - min:
      memory: 90Mi
      cpu: 100m
    type: Container
```

```shell
kubectl apply -f ingress-controller.yaml 

namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
limitrange/ingress-nginx created
```

#### 创建Ingress规则

> 作用: 配置ingress域名对应绑定的service

配置文件:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: example.ingressdemo.com				# 域名
    http:
      paths:
      - path: /
        backend:
          serviceName: web						# service名称
          servicePort: 80							# 端口
```

```shell
kubectl apply -f ingress-principel.yaml
# 查询创建的ingress
kubectl get ing
kubectl get pods -n ingress-nginx -o wide  			# 查询Pod对应存在的主机
# 在该主机中查询端口是否开启
netstat -antp | grep 443
netstat -antp | grep 80
```

#### 在访问主机上做域名映射

在对应系统的host文件中添加以下映射规则:

```shell
192.168.8.121 example.ingressdemo.com	
```

![2K4rn9](http://xwjpics.gumptlu.work/qinniu_uPic/2K4rn9.png)

## 3.10 核心概念-Helm

### 1.基本概念

![GmoePe](http://xwjpics.gumptlu.work/qinniu_uPic/GmoePe.png)

K8S 上的应用对象，都是由特定的资源描述组成，包括 deployment、service 等。都保存 各自文件中或者集中写到一个配置文件。然后 kubectl apply –f 部署。如果应用只由一 个或几个这样的服务组成，上面部署方式足够了。而对于一个复杂的应用，会有很多类似 上面的资源描述文件，例如微服务架构应用，组成应用的服务可能多达十个，几十个。如 果有更新或回滚应用的需求，可能要修改和维护所涉及的大量资源文件，而这种组织和管 理应用的方式就显得力不从心了。且由于缺少对发布过的应用版本管理和控制，使 Kubernetes 上的应用维护和更新等面临诸多的挑战，主要面临以下问题:

1. 如何将这些服务作为一个整体管理
2. 这些资源文件如何高效复用
3. 不支持**应用级别的版本管理**

**Helm 是一个 Kubernetes 的包管理工具，**就像 Linux下的包管理器，如 yum/apt 等，可以很方便的将之前打包好的 yaml 文件部署到 kubernetes 上。

Helm 有 3 个重要概念:

1. helm: 一个**命令行客户端工具**，主要用于 Kubernetes 应用 chart 的创建、打包、发布和管理。

2. Chart: 应用描述，一系列用于描述 k8s 资源相关文件的集合。**(把yaml打包,用chart管理)**

3. Release: 基于 Chart 的部署实体，一个chart 被 Helm 运行后将会生成对应的一个release. 将在 k8s 中创建出真实运行的资源对象. **(应用级别的版本管理)**

### 2.V3版本变动

2019 年 11 月 13 日， Helm 团队发布 Helm v3 的第一个稳定版本。 该版本主要变化如下:

架构变化:

1. 最明显的变化是 Tiller 的删除 (架构的变化)
2. release可以在不同的命名空间重用
3. 将chart推送到docker

![WqgqTy](http://xwjpics.gumptlu.work/qinniu_uPic/WqgqTy.png)

### 3.helm的安装

1. 下载压缩包文件,上传到系统中

   https://github.com/helm/helm/releases

   ![qbHMJu](http://xwjpics.gumptlu.work/qinniu_uPic/qbHMJu.png)

2. 解压缩helm压缩文件, 解压后的helm(就是一个可执行文件)复制到/usr/bin目录下

   `cp helm /usr/bin/`

3. 测试安装: `helm version`

4. 配置helm仓库

   * 添加仓库

     ```shell
     helm repo add stable http://mirror.azure.cn/kubernetes/charts
     helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
     helm repo update
     # 查看
     helm repo list 
     ```

   * 删除仓库

     `helm repo remove aliyun`

### 4.使用chart部署一个应用

* 搜索应用:

  `helm search repo [名称]`

* 安装应用

  `helm install [自定义名称] [搜索的应用名称]`

* 查看安装之后的状态

  ```shell
  helm list
  helm status [安装之后的名称]
  ```

* 示例安装weave

  ```shell
  helm search repo weave 
  helm install ui stable/weave-scope
  helm list
  helm status ui
  kubectl get pods,svc
  # 修改service中type为NodePort,对外暴露端口
  kubectl edit svc ui-weave-scope
  # 将ClusterIP改为NodePort
  # 再次访问查看暴露端口
  kubectl get svc
  ```

  ![mTPBgc](http://xwjpics.gumptlu.work/qinniu_uPic/mTPBgc.png)

### 5.自己制作Chart

#### 1.使用命令创建Chart

`helm create chart [名称]`

![tAupAk](http://xwjpics.gumptlu.work/qinniu_uPic/tAupAk.png)

#### 2.写入自己的yaml文件

在templates目录下, 加入自己的各类yaml文件

这里还是用nginx举例:

```shell
cd templates
rm -rf *

# deployment.yaml
cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: nginx:1.14
        name: nginx
EOF

#service.yaml 
cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: web-service
spec:
  ports:
  - nodePort: 31245
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web
  type: NodePort
EOF
```

#### 3.安装chart

注意执行install命令需要在创建的chart文件夹之外

> <font color='#e54d42'>注意: 这里的自定义名称不能大写</font>

```shell
helm install [自定义名称] [目标chart]
# 示例的命令
helm install my-helm-web mychart
### 提示:
AME: my-helm-web
LAST DEPLOYED: Mon Jun  7 09:27:47 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
# 查看创建的Pod和service
kubectl get pods,svc
```

#### 4.应用升级

修改要升级的yaml文件,然后执行:

```shell
helm upgrade [自定义名称] [目标chart]
# 示例
helm upgrade my-helm-web mychart
### 输出
NAME: my-helm-web
LAST DEPLOYED: Mon Jun  7 09:34:41 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2 				## 版本号提升
TEST SUITE: None
```

### 6.实现yaml的高效复用

通过传递参数,动态渲染模版,yaml内容动态传入参数生成

使用value.yaml文件:

1. 在values.yaml定义变量和值
2. 在具体yaml文件中获取定义变量的值

> 把动态的参数对写到values.yaml中

示例:

编辑values.yaml

```shell
cat > values.yaml << EOF
replicaCount: 1
image:
  repository: nginx
  tag: "1.16"
  label: "nginx"
service:
  type: NodePort
  port: 80
  targetPort: 32541
EOF
```

模版yaml使用表达式(注意有个空格): 

```shell
{{ .Values.变量Key}}

# 常用的其他固定变量
{{ .Release.Name}}  # 版本名称也就是自己使用chart安装的名称
```

修改deployment.yaml和service.yaml

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: {{ .Release.Name}}-dep
spec:
  replicas: {{ .Values.replicaCount}}
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: {{ .Values.image.repository}}:{{ .Values.image.tag}}
        name: {{ .Values.image.label}}
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: {{ .Release.Name}}-svc
spec:
  ports:
  - nodePort: {{ .Values.service.targetPort}}
    port: {{ .Values.service.port}}
    protocol: TCP
    targetPort: {{ .Values.service.port}}
  selector:
    app: web
  type: {{ .Values.service.type}}
```

```shell
# 启动前查看配置参数
helm install --dry-run web mychart
# 启动
helm install web mychart
# 查看
kubectl get pods,svc 
```

## 3.11 核心概念-持久化存储

### 1. 基本概念

https://www.cnblogs.com/benjamin77/p/12446765.html

**volume大致可以分为三类:**

- 公有云：[awsElasticBlockStore](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore) [azureDisk](https://kubernetes.io/docs/concepts/storage/volumes/#azuredisk)
- 本地：hostPath  emptydir
- 网络共享：NFS Ceph GlusterFS

#### 本地

* emptydir

  数据卷volume的本地方式(emptydir)是创建一个空卷,挂载到Pod容器中,**当Pod容器被删除,数据卷也随之删除**

  应用场景:  Pod容器之间数据共享

* hostPath

  挂载Node文件系统上文件或者目录到Pod中的容器，**pod删除后宿主机上的目录不会被清除**

  应用场景：Pod中容器需要访问宿主机文件

### 2. NFS

为了更加持久化的存储，还可以采用**网络共享nfs**的方式

=> 使用一台单独的数据服务器提供NFS持久化存储服务 (nfs是一种网络存储, 即使重启Pod数据也还会存在)

搭建流程如下

准备一台Nfs服务器提供服务, 这里选用主机: 192.168.8.147:

```shell
# 安装nfs
yum install -y nfs-utils
# 设置挂载路径
vim /etc/exports
/root/k8s/nfs_data *(rw,no_root_squash)			# 前面的路径需要先创建出来
# 启动nfs服务
systemctl start nfs
systemctl status nfs      
```

在node节点上**都**安装nfs:

```shell
yum install -y nfs-utils
```

部署使用nfs的Pod应用:

* 配置文件:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html		# 容器中挂载的目录
        ports:
        - containerPort: 80
      volumes:
        - name: wwwroot
          nfs:
            server: 192.168.8.147							# nfs服务器IP
            path: /root/k8s/nfs_data					# nfs服务的目录
```

* 应用

```shell
kubectl apply -f nfs-nginx.yaml 
```

* 测试

```shell
# 在nfs服务器上对应的文件夹下创建一个文件
cat > /root/k8s/nfs_data/hello.html << EOF
hello world
EOF

# 切换到k8s Master查看
kubectl exec -it pod/nginx-dep1-586964c5c-28qvn -- bash
ls /usr/share/nginx/html/
cat /usr/share/nginx/html/hello.html 

# 进一步的这是nginx的主页,所以expose外部访问端口可以看到这个html
kubectl expose deployment nginx-dep1 --port=80 --target-port=80 --type=NodePort
kubectl get svc 
# 访问对应端口
```

### 3. PV与PVC

直接配置像

```yaml
 nfs:
 	server: 192.168.8.147							# nfs服务器IP
 	path: /root/k8s/nfs_data					# nfs服务的目录
```

这样是不安全的, 一般会使用PVC和PV来进行配置

> <font color='#39b54a'>其实相比于NFS就是多了一层通过PVC绑定PV(根据匹配模式)</font>

![yyaHnm](http://xwjpics.gumptlu.work/qinniu_uPic/yyaHnm.png)

示例流程:

yaml配置文件:

pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /root/k8s/nfs_data
    server: 192.168.8.147
```

pvc.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: wwwroot
        persistentVolumeClaim:
          claimName: my-pvc						# 与Pvc名字相同绑定

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:													# 资源需求
    requests:
      storage: 5Gi
```

```shell
kubectl apply -f pv.yaml 
kubectl apply -f pvc.yaml 
# 查看
kubectl get pv,pvc		
# 进入容器查看
kubectl exec -it pod/nginx-dep1-69f5bb95b-zw7qv -- bash
cat /usr/share/nginx/html/hello.html 
```

![8yAKpR](http://xwjpics.gumptlu.work/qinniu_uPic/8yAKpR.png)

## 3.12 核心概念-集群监控

### 基本方案

Prometheus + Grafana

![BpBBzs](http://xwjpics.gumptlu.work/qinniu_uPic/BpBBzs.png)

![u9k4j5](http://xwjpics.gumptlu.work/qinniu_uPic/u9k4j5.png)

### 搭建平台

#### Prometheus

安装在集群中, 这里采用yaml安装

* 部署一个守护进程

  以后增加、删除节点每一个都会安装监控器

  node-exporter.yaml:

  ```yaml
  ---
  apiVersion: apps/v1
  kind: DaemonSet 												# 守护进程
  metadata:
    name: node-exporter
    namespace: kube-system								# 在kube系统的命名空间中
    labels:
      k8s-app: node-exporter
  spec:
    selector:
      matchLabels:
        k8s-app: node-exporter
    template:
      metadata:
        labels:
          k8s-app: node-exporter
      spec:
        containers:
        - image: prom/node-exporter
          name: node-exporter
          ports:
          - containerPort: 9100
            protocol: TCP
            name: http
  ---
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      k8s-app: node-exporter
    name: node-exporter
    namespace: kube-system
  spec:
    ports:
    - name: http
      port: 9100
      nodePort: 31672
      protocol: TCP
    type: NodePort
    selector:
      k8s-app: node-exporter
  ```

  ```shell
  # 创建
  kubectl create -f node-exporter.yaml 
  ```

  

* 其他配置文件

  下载地址: 链接: https://pan.baidu.com/s/1Pe19PEFWrDoIeNg1cwBaPw  密码: chs0

  ![zFO2E6](http://xwjpics.gumptlu.work/qinniu_uPic/zFO2E6.png)

  ```shell
  # 部署
  kubectl create -f rbac-setup.yaml
  kubectl create -f prometheus.deploy.yml
  kubectl create -f prometheus.svc.yml 
  # 查看
  
  ```

  > error: unable to recognize "prometheus.deploy.yml": no matches for kind "Deployment" in version "apps/v1beta2”
  >
  > 修改测试版本号, 将beta2删除掉

#### Grafana

链接: https://pan.baidu.com/s/1QY3Ci2I1r26s0GqgtbrpkA  密码: 45qk

> 其中deploy.yaml要做以下改动
>
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: grafana-core
>   namespace: kube-system
>   labels:
>     app: grafana
>     component: core
> spec:
>   replicas: 1
>   selector:					# 添加selector
>      matchLabels:
>        app: grafana
>        component: core
>        .....
> ```

```shell
kubectl create -f grafana-deploy.yaml 
kubectl create -f grafana-svc.yaml 
kubectl create -f grafana-ing.yaml 
# 查看
kubectl get pods,svc -n kube-system
```

打开Grafana配置数据源,导入显示模版

访问方式可以用两种:

* 在本机hosts记录ingress的域名与外暴露的端口映射, 然后直接访问域名
* 直接访问端口号

![i75EPL](http://xwjpics.gumptlu.work/qinniu_uPic/i75EPL.png)

默认的账户密码都是admin

设置数据源:

* 左上角下拉选择Data sources, 点击添加数据源, 进行配置

![HNjxzH](http://xwjpics.gumptlu.work/qinniu_uPic/HNjxzH.png)

设置数据显示模版

![r3dFZ4](http://xwjpics.gumptlu.work/qinniu_uPic/r3dFZ4.png)

https://grafana.com/grafana/dashboards官方可以选择自己想要的模版

这里用这个模版:https://grafana.com/grafana/dashboards/315

如果直接用ID不行的话就就下载对应的json文件,然后用json导入, 315 JSON的下载链接:https://grafana.com/api/dashboards/315/revisions/3/download

![AyRicZ](http://xwjpics.gumptlu.work/qinniu_uPic/AyRicZ.png)

创建好模版之后, 显示出页面

![v3OC9Q](http://xwjpics.gumptlu.work/qinniu_uPic/v3OC9Q.png)

> <font color='#e54d42'>这里使用k8s官方工具部署的集群可以显示, 但是使用二进制部署的集群无法显示,都是N/A. 此外一些新的模版可能无法适配此prometheus和grafana的版本</font>

