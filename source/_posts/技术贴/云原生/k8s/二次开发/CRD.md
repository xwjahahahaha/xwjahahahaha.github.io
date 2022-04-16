---
title: CRD
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 16:19:33
---

# 一、CRD基础

* CRD （Custom Resource Definition）自定义的k8s资源类型
* k8s通过Api Server，在etcd中注册一种新的资源类型，通过实现Custom Controller来监听资源对象的事件变化
* 自定义的CRD仅仅只是将自定义资源的yaml清单插入到了etcd，如果想像使用k8s自带的资源一样控制他，就需要自定义控制器

<!-- more -->

自定义一个CRD：

```yaml
# virtualmachines-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name 必须匹配下面的spec字段：<plural>.<group>  
  name: virtualmachines.cloud.waibizi.com
spec:
  # group 名用于 REST API 中的定义：/apis/<group>/<version>  
  group: cloud.waibizi.com
  versions:
  - name: v1       # 版本名称，比如 v1、v2beta1 等等    
    served: true   # 版本名称，比如 v1、v2beta1 等等    
    storage: true  # 是否开启通过 REST APIs 访问 `/apis/<group>/<version>/...`    
    schema:        # 定义自定义对象的声明规范 
      openAPIV3Schema:
        description: Define virtualMachine YAML Spec
        type: object
        properties:
          # 自定义CRD的字段类型
          spec:
            type: object
            properties:
              uuid:
                type: string
              name:
                type: string
              image:
                type: string
              memory:
                type: integer
              disk:
                type: integer
  # 定义作用范围：Namespaced（命名空间级别）或者 Cluster（整个集群）              
  scope: Namespaced
  names:
    # 定义作用范围：Namespaced（命名空间级别）或者 Cluster（整个集群）
    kind: VirtualMachine
     # plural 名字用于 REST API 中的定义：/apis/<group>/<version>/<plural>    
    plural: virtualmachines
     # singular 名称用于 CLI 操作或显示的一个别名 
    singular: virtualmachines
# 这个地方就是平时使用kubectl get po 当中这个 po 是 pod的缩写的定义，我们可以直接使用kubectl get vm查看
    shortNames:
    - vm
```

apply创建进etcd

![image-20220326144508235](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220326144508235.png)

查看自己创建的crd：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220326144812304.png" alt="image-20220326144812304" style="zoom:50%;" />

创建一个CRD对象：

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

apply然后查看：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220326145919071.png" alt="image-20220326145919071" style="zoom:50%;" />

<font color='#e54d42'>注意：这里没有controller为我们的自定义资源负责，所以这里没有任何意义，就像是在操作数据库一样，建立了一个表，然后又插入了一条数据，最后将其读出来</font>

