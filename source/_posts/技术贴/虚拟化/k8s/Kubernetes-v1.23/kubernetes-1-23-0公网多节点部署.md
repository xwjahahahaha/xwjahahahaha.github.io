---
title: kubernetes__v_1_23_0公网多节点部署
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-15 15:47:13
---

记录部署**k8s v1.23.0**版本的重点

> * https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
> * https://blog.csdn.net/qq_39382182/article/details/121915330
> * https://blog.lanweihong.com/posts/56314/

<!-- more -->

# 一、集群配置与建立

|          |     master     |     Node1      |     Node2      |
| :------: | :------------: | :------------: | :------------: |
| IP(公网) | 211.159.224.96 | 124.223.56.208 | 47.103.203.133 |
| IP(私有) |   10.0.16.15   |   10.0.16.14   |  172.17.59.2   |
|   系统   |  Ubuntu20.04   |  Ubuntu18.04   |  Ubuntu18.04   |

安装kubernetes的注意点：

* 安装需要的前置工作

* 安装指定版本：

  ```shell
  apt-get install -y kubectl=1.23.0-00 kubeadm=1.23.0-00 kubelet=1.23.0-00
  apt-mark hold kubelet kubeadm kubectl  # 锁定版本
  ```

* 不管使用centos还是Ubuntu，使用自带的yum或apt下载docker、kubeadm、kubectl、kubelet等组件的时候都需要先配置仓库，在国内配置国内的代理仓库防止过慢（官网是外网的，下载很慢）(自行搜索yum/apt的docker、k8s库)

  https://developer.aliyun.com/mirror/kubernetes

  ```shell
  # 切换到root，否则公钥设置不上
  apt-get update && apt-get install -y apt-transport-https
  curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
  deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
  EOF  
  apt-get update
  apt-get install -y kubelet kubeadm kubectl
  ```

* 因为三台主机都是公网IP不在同一局域网段，现在的云服务器自己的网卡是看不到自己公网IP的，都是外部网卡给其配置的公网IP，所以直接设置公网IP会连自己都无法访问自己(如果使用ifconfig可以看到自己的公网IP，那么就不需要此步骤)。目前的三种解决方案：

  * **利用iptables转发**：在Master主机上将公网IP转发到内网IP，在work-node节点上将Master的内网IP转换为其公网IP（本文使用此方法，最方便）

  * 虚拟网卡，在本地虚拟一个绑定公网IP的网卡，把内网的流量复制过来

  * 看能不能设置etcd监听的IP。参考[k8s——部署基于公网的k8s集群 - 知乎](https://zhuanlan.zhihu.com/p/74134318)

## 1. kubeadm的配置与启动（master）

首先看一下官网对于配置的详细说明：v1.23.0使用的是[v1beta3](https://pkg.go.dev/k8s.io/kubernetes@v1.23.1/cmd/kubeadm/app/apis/kubeadm/v1beta3#hdr-Kubeadm_init_configuration_types)

关键点：

* 通过`--config`选项 + 一个`YAML`配置文件配置，当然也可以直接使用命令后接一些命令行flag的方式启动（但这种方式只适合简单的部署）

* `kubeadm`的配置文件有多种类型，每种类型之间用`---`分割开

  ```yaml
  # init Confiuration types 
  
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: InitConfiguration      	# 主要用于配置kubeadm运行时的一些配置
  
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: ClusterConfiguration   	# 主要用于配置整个集群广泛的配置
  
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration		# 更改将传递给集群中部署的所有kubelet实例的配置。
  
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration 	# 更改传递给集群中部署的kube-proxy实例的配置。
  
  # join Confiuration types 
  
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: JoinConfiguration 			# 主要用于配置运行时的一些设置，主要关于运行时kubeadm的节点发现
  ```

* 查看默认的两项配置命令如下，如果一些配置项没有配置就会使用默认，有一些配置在集群中要**高度一致**（e.g. cluster-cidr flag on controller manager and clusterCIDR on kube-proxy）

  ```shell
  kubeadm config print init-defaults
  kubeadm config print join-defaults
  ```

### 1. init Confiuration

#### kind: InitConfiguration

主要用于配置`kubeadm`运行时的一些配置

* `NodeRegistration`：包含将新节点注册到集群相关的字段；使用它来自定义节点名称、要使用的 CRI (容器运行时接口)套接字或仅适用于该节点的任何其他设置;
* `LocalAPIEndpoint`表示要在此节点上部署的 API Server实例的端点；

#### kind: ClusterConfiguration

主要用于配置整个集群广泛的配置

* `Networking`：保存集群网络拓扑的配置；例如自定义 pod 子网或服务子网。

* `Etcd`：Etcd集群配置，自定义本地 etcd 或配置 API 服务器以使用外部 etcd 集群。

* `kube-apiserver, kube-scheduler, kube-controller-manager`

  使用它通过添加自定义设置或覆盖 kubeadm 默认设置来自定义控制平面组件。

### 2. join configuration

主要用于配置运行时的一些设置，主要关于运行时kubeadm的节点发现

#### kind:JoinConfiguration

* `NodeRegistration` ：包含将新节点注册到集群相关的字段；使用它来自定义节点名称、要使用的 CRI 套接字或应仅适用于该节点的任何其他设置（例如节点 ip）。
* `APIEndpoint`：表示最终将部署在此节点上的 API Server实例的端点。

### 3. 修改init的默认配置启动

给`kubeadm init`命令编写一个启动配置：

* 将默认配置倒出到一个`yaml`文件

  `kubeadm config print init-defaults > init.yaml`

* 修改：

  ```yaml
  apiVersion: kubeadm.k8s.io/v1beta3
  bootstrapTokens:
  - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
  kind: InitConfiguration
  localAPIEndpoint:
    advertiseAddress: 0.0.0.0  # 这里就设置云服务器的内网IP，不能设置公网IP
    bindPort: 6443
  nodeRegistration:
    criSocket: /var/run/dockershim.sock
    imagePullPolicy: IfNotPresent
    name: master
    taints: null
  ---
  apiServer:
    timeoutForControlPlane: 4m0s
  apiVersion: kubeadm.k8s.io/v1beta3
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns: {}
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: registry.aliyuncs.com/google_containers # 核心：替换k8s拉取镜像的源为国内的阿里源,默认拉取镜像地址k8s.gcr.io国内无法访问
  kind: ClusterConfiguration
  kubernetesVersion: 1.23.0
  networking:
    dnsDomain: cluster.local
    serviceSubnet: 10.96.0.0/12	# 集群内部分配IP的网段
    podSubnet: 10.244.0.0/16		# pod子网分配IP的网段，与网络插件密切相关
  scheduler: {}
  ```

启动`kubeadm init --config ./kubeadm-init.yaml --v=5`

(`--v=5`可以看到详细输出以及错误)

成功就可以看到如下输出：

![image-20220116101208828](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220116101208828.png)

要使Master中非 root 用户可以运行 `kubectl`，请运行以下命令， 它们也是 `kubeadm init` 输出的一部分：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

或者，如果你是 `root` 用户，则可以运行：

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

注意上面的命令必须执行后才可以使用`kubectl get nodes`这样的命令

查看master目前的状况:

`kubectl get nodes`

```shell
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   13m   v1.23.0
```

下面继续配置

### 4. 配置网络插件CNI（这一步在将所有work node加入之后操作）

[CNI文档](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni)

> 通过给 Kubelet 传递 `--network-plugin=cni` 命令行选项可以选择 CNI 插件。 Kubelet 从 `--cni-conf-dir` （默认是 `/etc/cni/net.d`） 读取文件并使用 该文件中的 CNI 配置来设置各个 Pod 的网络。 CNI 配置文件必须与 [CNI 规约](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration) 匹配，并且配置所引用的所有所需的 CNI 插件都应存在于 `--cni-bin-dir`（默认是 `/opt/cni/bin`）下。

**网络插件要与`init`配置中的`podSubnet`对应或者与未使用配置文件直接指定的flag: `--pod-network-cidr`对应**

三种选其一（我使用的是flannel）：

1）Calico插件安装：

wget https://docs.projectcalico.org/manifests/calico.yaml

`Vim calico.yaml `修改 CALICO_IPV4POOL_CIDR参数和pod-network-cidr一致

`kubectl apply -f calico.yaml`

2）flannel插件安装：

`flannel` Github地址：https://github.com/flannel-io/flannel

`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

3）weave插件安装：

`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

### 5. 设置Iptables转发

可以看到，节点加入集群命令为内网IP，但几个云服务器内网不互通，所以我们需要使用 `iptables` 进行 IP 转发，**将主节点公网IP转发至内网IP**，由于node节点加入集群的命令是内网IP，因此还需要**配置 node 节点将主节点的内网IP转发至主节点的公网IP**。

```shell
# 在主节点 master
sudo iptables -t nat -A OUTPUT -d <主节点公网IP> -j DNAT --to-destination <主节点私有IP>
# sudo iptables -t nat -A OUTPUT -d 211.159.224.96 -j DNAT --to-destination 10.0.16.15 
# sudo iptables -t nat -A OUTPUT -d 172.17.59.2 -j DNAT --to-destination 47.103.203.133
# sudo iptables -t nat -A OUTPUT -d 10.0.16.14 -j DNAT --to-destination 124.223.56.208


# 在 work node 节点上
sudo iptables -t nat -A OUTPUT -d <主节点私有IP> -j DNAT --to-destination <主节点公网IP>
# sudo iptables -t nat -A OUTPUT -d 10.0.16.15 -j DNAT --to-destination 211.159.224.96
```

## 3. Work节点加入（work node）

work节点不需要`kubeadm init`，只需要按照上面的步骤安装好`kubectl kubeadm kubelet`即可

首先要在工作节点上设置上面的iptables转发，然后根据master的提示输出加入（用的是master的私网）：

```shell
kubeadm join 10.0.16.15:6443 --token 3vjisz.whvzscikyzd01z8d --discovery-token-ca-cert-hash sha256:d49fd6e0669836f28908a3b4ad40a642e77fcdc3ebcf3ed660eb104ecc6c3b80 
```

如果上面的忘记，可以在master节点上生成：

```shell
kubeadm token create --print-join-command
```

加入成功后显示：

![image-20220116104818497](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220116104818497.png)

在主节点

![image-20220116104936647](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220116104936647.png)

# 二、错误整理

## 1. 多次init前需要重置一下

[卸载集群](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down)

删除之前的环境：

```shell
sudo kubeadm reset				# 只有master节点
rm -rf /root/.kube/
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/cni/net.d
apt-get --purge remove kubectl kubelet kubeadm 
```

注意，即使这样也没有删除掉k8s对本机网卡iptables转发的配置，完全的删除还需要执行:

```shell
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm -C
```

## 2. kubelet与docker驱动不同启动失败

* 问题描述：

  kubelet启动失败，`journalctl -xeu kubelet`查看错误信息：

  `failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs\""`

* 原因：

  kubelet与docker的cgroups驱动程序不同

* 解决：

  [配置 cgroup 驱动程序](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configure-cgroup-driver)

  官方推荐kubelet使用systemd驱动，那么我们修改docker的驱动, 修改`/etc/docker/daemon.json`文件:

  ```json
  {
      "exec-opts": ["native.cgroupdriver=systemd"]
  }
  ```

  ```shell
  # 重启docker\kubelet
  systemctl restart docker
  systemctl restart kubelet
  ```

## 3. kubelet找不到配置文件

`kubelet`启动报错：

问题描述：`failed to load Kubelet config file /var/lib/kubelet/ config.yaml, error failed to read kubelet config file "/var`

原因与解决：新版本需要先`kubeadmin init`生成对应的配置文件，`kubelet`才会正常

## 4.使用公网IP配置init启动导致 Initial timeout of 40s passed.

问题描述：

master init启动卡在:

```shell
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
```

原因：

因为阿里云主机网络是VPC，公网IP只能在控制台上看到，在系统里面看到的是内部网卡的IP，阿里云采用NAT方式将公网IP映射到ECS的位于私网的网卡上，所以在网卡上看不到公网IP，使用 `ifconfig` 查看到的也是私有网卡的IP，导致 `etcd` 无法启动。

解决：

使用`iptables`转发（见上方）

## 5.使用kubectl get nodes提示8080端口未开启

问题描述：

`kubectl get nodes`

`The connection to the server localhost:8080 was refused - did you specify the right host or port?`

原因：可能是api-server启动问题，但很大可能是没有创建`.kube`文件，导致`kubectl`还不能正常使用

解决：https://discuss.kubernetes.io/t/the-connection-to-the-server-localhost-8080-was-refused-did-you-specify-the-right-host-or-port/1464

创建`.kube`文件见上文

## 6. kube-flannel插件pod无法启动，一直是CrashLoopBackOff状态

错误描述：

![image-20220116111921116](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220116111921116.png)

都是worknode上的未启动，并且在worknode的kubelet上可以看到报错：

`pod_workers.go:918] "Error syncing pod, skipping" err="...`

原因：顺序错误，应该先join连接好节点，然后再使用CNI插件，这样才会正常安装

解决：所有worknode加入后再次尝试
