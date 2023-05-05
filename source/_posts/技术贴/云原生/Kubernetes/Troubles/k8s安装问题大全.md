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

# 3. 下载 Google Cloud 公开签名秘钥错误

## 错误描述

> 环境：
>
> * Linux 02 5.19.0-0.deb11.2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 5.19.11-1~bpo11+1 (2022-10-03) x86_64 GNU/Linux

按照官网下载kubeadm的流程中（https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/）

其中的第二步：

下载 Google Cloud 公开签名秘钥：

```shell
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

出现错误：

`curl: (23) Failure writing output to destination`

![image-20230419214832115](http://xwjpics.gumptlu.work/image-20230419214832115.png)

## 解决

根据下面的提示：

![image-20230419214906918](http://xwjpics.gumptlu.work/image-20230419214906918.png)

我是debian11 所以需要先创建`/etc/apt/keyrings`文件夹并给权限才可，这样这个问题就解决了

命令如下：

```shell
mkdir /etc/apt/keyrings
chmod a+r /etc/apt/keyrings/
chmod +w /etc/apt/keyrings/
```

# 4. 更新apt包索引提示公钥验证错误

> 环境：
>
> * Linux 02 5.19.0-0.deb11.2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 5.19.11-1~bpo11+1 (2022-10-03) x86_64 GNU/Linux

## 错误描述

（2023-4-20）根据官网https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#troubleshooting的步骤下载Google Cloud 公开签名秘钥后`apt update`出现如下错误：

 `The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B53DC80D13EDEF05`

![image-20230420105416611](http://xwjpics.gumptlu.work/image-20230420105416611.png)

## 解决

社区相关的解决方案有很多：

> * https://github.com/kubernetes/release/issues/2862
> * https://unix.stackexchange.com/questions/361642/keyserver-receive-failed-on-every-keyserver-available
> * https://github.com/kubernetes/release/issues/2860

操作包括：

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# or
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv B53DC80D13EDEF05  # 这个keyserver.ubuntu.com域名貌似不可用了
# or
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys B53DC80D13EDEF05
```

但是这些对于我来说一直都不生效，这个报错始终提示，考虑到可能官网给的密钥也过期了（没更新文档），所以就做了如下操作，解决了问题

```shell
# 删除source.list中指定的keyring
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
# 再次验证提示错误的公钥，注意这里的recv-keys填写你对应的NO_PUBKEY
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys B53DC80D13EDEF05
# 更新
sudo apt-get update
```

![image-20230420144515497](http://xwjpics.gumptlu.work/image-20230420144515497.png)

# 5. debian11整体安装k8s(v1.27.1)流程

## 1. 系统初始化

```shell
# 安装时间同步
apt install ntpdate
ntpdate ntp1.aliyun.com iburst
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' > /etc/timezone

#关闭防火墙
ufw disable

## 关闭swap
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

## 2. 转发 IPv4 并让 iptables 看到桥接流量

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter


# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system

# 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载：
lsmod | grep br_netfilter
lsmod | grep overlay

# 通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1：
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## 3. 安装Ipvs

```shell
###系统依赖包
apt install -y conntrack ipvsadm ipset jq iptables curl sysstat libseccomp-dev wget vim net-tools git
### 开启ipvs 转发
modprobe br_netfilter 

mkdir -p /etc/sysconfig/modules
cat > /etc/sysconfig/modules/ipvs.modules << EOF 
#!/bin/bash 
modprobe -- ip_vs 
modprobe -- ip_vs_rr 
modprobe -- ip_vs_wrr 
modprobe -- ip_vs_sh 
modprobe -- nf_conntrack
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules 
bash /etc/sysconfig/modules/ipvs.modules 
lsmod | grep -e ip_vs -e nf_conntrack
```

<img src="http://xwjpics.gumptlu.work/image-20230420153324880.png" alt="image-20230420153324880" style="zoom: 67%;" />

## 4. 安装containerd

注意containerd的版本不能太旧，所以我们采用去官网二进制安装（debian自带的1.4版本太老了）

```shell
# 下载
wget https://github.com/containerd/containerd/releases/download/v1.7.0/containerd-1.7.0-linux-amd64.tar.gz
tar xvf containerd-1.7.0-linux-amd64.tar.gz
cp ./bin/* /usr/local/bin/
# 创建containerd systemd service启动管理文件
touch /etc/systemd/system/containerd.service && vim /etc/systemd/system/containerd.service
----
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target
 
[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
 
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999
 
[Install]
WantedBy=multi-user.target
----
# 重新加载系统管理服务文件
systemctl daemon-reload

# 生成配置文件
containerd config default > /etc/containerd/config.toml
# 编辑配置文件
vim /etc/containerd/config.toml
-----
systemd_cgroup = false 改为 systemd_cgroup = true
# 修改runtimetype否则会报错
[plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
	...
	runtime_type = "io.containerd.runtime.v1.linux"
# 修改镜像代理
sandbox_image = "k8s.gcr.io/pause:3.8"
改为：
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.8"
# vim下搜索/mirrors，添加镜像加速，使用docker镜像源即可
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://dxc7f1d6.mirror.aliyuncs.com"]
------
# 应用
systemctl enable containerd
systemctl start containerd
systemctl status containerd


# 测试拉取一个redirs并启动容器
ctr i pull docker.io/library/redis:latest
ctr run -d -t docker.io/library/redis:latest redis
ctr c ls
```

## 4. 安装kubelet kubeadm kubectl

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys B53DC80D13EDEF05
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 5. cgroup驱动匹配

为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性(使用官方推荐的systemd)，建议修改如下文件内容

```shell
vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"

# 设置kubelet为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动
systemctl enable kubelet
```

## 6. 集群初始化(主节点)

```shell
sudo kubeadm init \
--apiserver-advertise-address=10.0.86.216 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.27.0 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--v=5
```

解释如下：

```shell
sudo kubeadm init \
--apiserver-advertise-address=10.0.86.216 \    # master的ip 集群通告地址
--image-repository registry.aliyuncs.com/google_containers \ # 由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址
--kubernetes-version v1.27.1 \ # 下载的版本
--service-cidr=10.96.0.0/12 \ # 集群内部虚拟网络，Pod统一访问入口
--pod-network-cidr=10.244.0.0/16 # Pod网络，，与下面部署的CNI网络组件yaml中保持一致
--v=5
```

使非 root 用户可以运行 kubectl:

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 7. 其他节点加入

```shell
kubeadm join 10.0.86.216:6443 --token aveiu8.i3y19l7i1p5wndve \
        --discovery-token-ca-cert-hash sha256:ec5adf91c6fefd36f356e98950107d1a1f9e638d181e77f2ae649fbf9c0b8d1b 
```

## 8. 集群部署网络插件

网络组件有很多种，只需要部署其中一个即可，这里用Calico

```shell
wget https://docs.tigera.io/archive/v3.24/manifests/calico.yaml

vim calico.yaml
...
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
...

kubectl apply -f calico.yaml
kubectl get pod -n kube-system
```



# 6. kubeadm init报错

> * https://zhuanlan.zhihu.com/p/618551600

## 错误描述

在执行kubeadm init的时候报错

```shell
[preflight] Some fatal errors occurred:
	[ERROR CRI]: container runtime is not running: output: time="2023-04-20T17:03:33+08:00" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
```

此时containerd已经启动，仔细观察日志发现也有一个错误：

```shell
error="invalid plugin config: systemd_cgroup only works for runtime io.containerd.runtime.v1.linux"
```

## 解决

在containerd的配置文件修改：

`vim /etc/containerd/config.toml`

`runtime_type = "io.containerd.runtime.v1.linux"`
