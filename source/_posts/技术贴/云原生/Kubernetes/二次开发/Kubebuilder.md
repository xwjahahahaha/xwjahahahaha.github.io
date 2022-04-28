---
title: Kubebuilder
tags:
  - null
categories:
  - null
toc: true
date: 2022-04-16 16:22:31
---

# 一、kubebuilder的基本使用

Mac下直接下载

```shell
brew install kubebuilder 
brew install kustomize
```

<!-- more -->

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



