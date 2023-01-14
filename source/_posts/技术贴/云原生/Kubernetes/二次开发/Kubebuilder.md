---
title: Kubebuilder
tags:
  - null
categories:
  - null
toc: true
date: 2022-04-16 16:22:31
---

[TOC]

> 参考:
>
> * 官网文档: https://book.kubebuilder.io/quick-start.html

<!-- more -->

# 一、kubebuilder的基本使用

很多信息包括基本的安装和使用官网文档很完整，这里只记录一些使用的细节

## 1. 初始化

可以通过`init`初始化一个项目，例如：

```shell
kubebuilder init --domain my.domain --repo my.domain/guestbook
```

但是注意： 

> If your project is initialized within [`GOPATH`](https://golang.org/doc/code.html#GOPATH), the implicitly called `go mod init` will interpolate the module path for you. Otherwise `--repo=<module path>` must be set.
>
> Read the [Go modules blogpost](https://blog.golang.org/using-go-modules) if unfamiliar with the module system.

`init`会隐式调用`go mod init`所以最好指定模块名`--repo=<module path>`

## 2. 基础架构

通过kubebuilder的脚手架我们能够快速的创建一个项目的基础架构

* `go.mod`: A new Go module matching our project, with basic dependencies

* `Makefile`: Make targets for building and deploying your controller

* `PROJECT`: Kubebuilder metadata for scaffolding new components 

  ```yaml
  domain: tutorial.kubebuilder.io
  layout:
  - go.kubebuilder.io/v3
  projectName: project
  repo: tutorial.kubebuilder.io/project
  resources:
  - api:
      crdVersion: v1
      namespaced: true
    controller: true
    domain: tutorial.kubebuilder.io
    group: batch
    kind: CronJob
    path: tutorial.kubebuilder.io/project/api/v1
    version: v1
    webhooks:
      defaulting: true
      validation: true
      webhookVersion: v1
  version: "3"
  ```

* 配置路径`/config`:
  * [`config/default`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/default) : contains a [Kustomize base](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/config/default/kustomization.yaml) for launching the controller in a standard configuration.
  * [`config/manager`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/manager): launch your controllers as pods in the cluster
  * [`config/rbac`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/rbac): permissions required to run your controllers under their own service account

*  [*Scheme*](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#err-but-whats-that-scheme-thing): Every set of controllers needs a [*Scheme*](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#err-but-whats-that-scheme-thing), which provides **mappings between Kinds and their corresponding Go types.** 提供了`kinds`和`go types`的映射

* 对于`main.go`:

  其中的`Manager`可以通过以下方式限制所有控制器将监视资源的命名空间（也就是将项目的范围改为单个的命名空间）：

  ```go
  mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    Namespace:              namespace,			// 在此处配置
    MetricsBindAddress:     metricsAddr,
    Port:                   9443,
    HealthProbeBindAddress: probeAddr,
    LeaderElection:         enableLeaderElection,
    LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
  })
  ```

  或者是这样一组的命名空间：

  ```go
  var namespaces []string // List of Namespaces
  
  mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    NewCache:               cache.MultiNamespacedCacheBuilder(namespaces),
    MetricsBindAddress:     fmt.Sprintf("%s:%d", metricsHost, metricsPort),
    Port:                   9443,
    HealthProbeBindAddress: probeAddr,
    LeaderElection:         enableLeaderElection,
    LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
  })
  ```

## 3. API

### Groups、Versions、Kinds、Resources 

When we talk about APIs in Kubernetes, we often use 4 terms: ***groups*, *versions*, *kinds*, and *resources*.**

基本术语概念解释：

* Group和Version：组只是一系列功能的集合，每个组/功能有多个版本，所以对应不同的`Version`

* Kinds: 每个API group-version包含了一系列 API types, 这就是称之为 *Kinds*. 每个kind的版本应该要向前兼容（有充足的字段描述旧版本）

* Resources：也就是Kinds的实例/对象，一般一个资源一对一一个类型（For instance, the `pods` resource corresponds to the `Pod` Kind.），但是有时一个类型可能有不同的资源( For instance, the `Scale` Kind is returned by all scale subresources, like `deployments/scale` or `replicasets/scale`. )，这也是HPA允许与不同资源交互的原因。然而，对于 CRD，每个种类将对应一个资源。

  > 请注意，资源总是小写的，并且按照惯例是小写形式的 Kind

* GVK: 当我们在特定的**组版本**中引用一种**类型**时，我们将其称为 `GroupVersionKind`，或简称` GVK`，资源就是`GVR`

  **每个 GVK 对应于包中给定的root Go 类型**

### Adding a new API

创建API的命令：`kubebuilder create api --group batch --version v1 --kind CronJob`

Press `y` for “Create Resource” and “Create Controller”. 不同的组版本脚手架会根据版本创建不同的文件夹

In this case, the [`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) directory is created, corresponding to the `batch.tutorial.kubebuilder.io/v1`

此命令的目标是为我们的 Kind(s) 创建自定义资源 (CR) 和自定义资源定义 (CRD), 自动化生成的Go 结构体用于生成 CRD（the CRDs are a definition of our customized Objects, and the CRs are an instance of it.），其中包括我们的数据架构以及跟踪数据，例如我们的新类型被称为什么

生成的`api/v1`中的go文件，其中的注释：

That little `+kubebuilder:object:root` comment is called a marker. We’ll see more of them in a bit, but know that they act as extra metadata, telling [controller-tools](https://github.com/kubernetes-sigs/controller-tools) (our code and YAML generator) extra information.

注释是用于在`controller-tools`工具进行生成时的额外元数据

### API的设计规则

* 所有的`Field`名称都应该使用小驼峰命名法，所以我们使用`json tag`来指定

* 可以使用`omitempty tag`来让空字段不被序列化

* 字段可以使用大多数的基础类型，但是数字类型要注意只能使用`int32`或者`int64`表示一个整型，`resource.Quantity` 表示小数/浮点型

  > `Quantity`是`k8s`中表示计算机存储单位中浮点型的一种特殊的方式，其实大同小异，我们在`yaml`定义资源限制的时候就已经使用过了，例如：
  >
  > For instance, the value `2m` means `0.002` in decimal notation. `2Ki` means `2048` in decimal, while `2K` means `2000` in decimal. If we want to specify fractions, we switch to a suffix that lets us use a whole number: `2.5` is `2500m`.

* 使用`meta.time`代替`time.Time`, 两者类型相同，只是`meta.time`具有固定的、可移植的序列化格式

使用`// +comment`改造你的`CRD`文件:

详细的所有规则见：

* https://book.kubebuilder.io/reference/generating-crd.html
* https://book.kubebuilder.io/reference/markers.html

# 二、实战案例

## 1. VirtualMachine

初始化项目:

```shell
kubebuilder init --plugins go/v3 --domain waibizi.com --owner "waibizi" --skip-go-version-check
```

对应上面的CRD的yaml文件创建api

```yml
# virtualmachines-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: virtualmachines.cloud.waibizi.com
spec:
  group: cloud.waibizi.com	# 因为已经在init的时候指定了域名waibizi.com，所以创建api的时候可以省略
  ...
  names:
    kind: VirtualMachine 		# kind对应api的kind	   
    plural: virtualmachines
    singular: virtualmachines
    shortNames:
    - vm
```

创建API：

```shell
kubebuilder create api --group cloud --version v1 --kind VirtualMachine
```

生成的文件目录结构：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416165656154.png" alt="image-20220416165656154" style="zoom:50%;" />

设置生成CRD的数据结构：

`api/v1/virtualmachine_type.go`

```go
// VirtualMachineSpec defines the desired state of VirtualMachine
type VirtualMachineSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of VirtualMachine. Edit virtualmachine_types.go to remove/update
	UUID string `json:"uuid,omitempty"`
	Name string `json:"name,omitempty"`
	Image string `json:"image,omitempty"`
	Memory int `json:"memory,omitempty"`
	Disk int `json:"disk,omitempty"`
}
```

可以使用的自动化命令：

![image-20220416170055498](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416170055498.png)

自动化生成清单:

```shell
make manifests generate
```

![image-20220416170347342](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416170347342.png)

自动化apply到集群：

因为之前手动创建过这个CRD，所以先手动删除（如果没创建过则无需这一步）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416170613974.png" alt="image-20220416170613974" style="zoom:50%;" />

`kubectl delete crd virtualmachines.cloud.waibizi.com`

```shell
make install   # 会默认使用~/.kube/config目录的配置文件
```

创建成功：

![image-20220416170839235](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416170839235.png)

启动controller：

启动监听之前，首先编写`controller`的逻辑

`controller/virtualmachine_controller.go`

```go
func (r *VirtualMachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	klog.Infoln(req.Name) 				// 简单的输出 

	return ctrl.Result{}, nil
}
```

启动

```shell
make run
```

根据CRD创建一个实例

```yaml
# public-wx.yaml
apiVersion: "cloud.waibizi.com/v1"
kind: VirtualMachine
metadata:
  name: public-wx
spec:
  uuid: "2c4789b2-30f2-4d31-ab71-ca115ea8c199"
  name: "waibizi-wx-virtual-machine"
  image: "Centos-7.9"
  memory: 4096
  disk: 500
```

```shell
k create -f public-wx.yaml
```

监听到了变化（调和）：

![image-20220416171951359](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416171951359.png)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220416172044193.png" alt="image-20220416172044193" style="zoom:50%;" />

# Troubles

## 1、kubebuild create api 报错

错误描述：

```shell
/Users/xwj/projects/k8s_second_dev/oprator_test/vm-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
bash: /Users/xwj/projects/k8s_second_dev/oprator_test/vm-operator/bin/controller-gen: No such file or directory
make: *** [generate] Error 127
```

原因与解决：

https://stackoverflow.com/questions/71608442/generating-controller-gen-with-kubebuilder

要使用go 1.17的版本，我之前使用的1.18就会导致此错误



