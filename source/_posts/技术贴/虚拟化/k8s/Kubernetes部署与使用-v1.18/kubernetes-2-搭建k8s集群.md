---
title: kubernetes-2-搭建k8s集群
tags:
  - kubernetes
categories:
  - technical
  - kubernetes
toc: true
declare: true
date: 2021-05-28 16:58:00
---

# 二、搭建k8s集群

## 2.1 平台规划

### 1.单master集群

![HF3wO7](http://xwjpics.gumptlu.work/qinniu_uPic/HF3wO7.png)

### 2.多master集群(高可用集群)

![wXVClp](http://xwjpics.gumptlu.work/qinniu_uPic/wXVClp.png)

<!-- more -->

## 2.2 服务器硬件要求

![irGHD7](http://xwjpics.gumptlu.work/qinniu_uPic/irGHD7.png)

## 2.3 部署方式

### 2.3.1. Kubeadm工具安装

> 官方的部署k8s工具, 用于快速部署

第一、创建一个 Master 节点 kubeadm init

第二、将 Node 节点加入到当前集群中 `$ kubeadm join <Master 节点的 IP 和端口 >`

#### 1. 前置条件

* 一台或多台机器，操作系统 CentOS7.x-86_x64

* 硬件配置:2GB 或更多 RAM，2 个 CPU 或更多 CPU，硬盘 30GB 或更多 - 集群中所有机器之间网络互通

* 可以访问外网，需要拉取镜像

* 禁止 swap 分区

  [步骤](https://blog.csdn.net/yefun/article/details/102772368?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control)

  1. `vim /etc/fstab`

     注释swap那一行

  2. `	echo vm.swappiness=0 >> /etc/sysctl.conf`

  3. 重启: `	sudo reboot`

  4. 验证 `free -m`

     ![RNdXg4](http://xwjpics.gumptlu.work/qinniu_uPic/RNdXg4.png)

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```shell
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

#### 2. 准备环境

| 角色   | IP            |
| ------ | ------------- |
| master | 192.168.8.146 |
| node1  | 192.168.8.147 |
| node2  | 192.168.8.148 |

```shell
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts (换成自己的IP)
cat >> /etc/hosts << EOF
192.168.8.146 k8smaster
192.168.8.147 k8snode1
192.168.8.148 k8snode2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```

#### 3. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

##### 3.1 安装Docker

```shell
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
```

```shell
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

##### 3.2 添加阿里云YUM软件源

```shell
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

##### 3.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
$ systemctl enable kubelet
```

#### 4. 部署Kubernetes Master

在192.168.8.146（Master）执行。

```shell
# 注意第一个参数换成自己的IP
$ kubeadm init \
  --apiserver-advertise-address=192.168.8.146 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

> 这一步主要是拉取Master要使用的镜像
>
> ![XennPZ](http://xwjpics.gumptlu.work/qinniu_uPic/XennPZ.png)

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定**阿里云镜像仓库地址。**

使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

#### 5. 加入Kubernetes Node

在其他（Node工作节点）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：(如果没找到可以用下面的命令再创建)

```shell
$ kubeadm join 192.168.1.11:6443 --token esce21.q6hetwm8si29qxwn \
    --discovery-token-ca-cert-hash sha256:00603a05805807501d7181c3d60b478788408cfe6cedefedb1f97569708be9c5
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```shell
kubeadm token create --print-join-command
```

> 错误: <font color='#e54d42'>ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?</font>
> errors pretty printing info, error: exit status 1
> [ERROR Service-Docker]: docker service is not active, please run 'systemctl start docker.service'
> [ERROR IsDockerSystemdCheck]: cannot execute 'docker info -f …...
>
> 原因: docker服务未启动
>
> 解决: `systemctl start docker.service`

添加成功后再次在master中检查node:

![rIIL0n](http://xwjpics.gumptlu.work/qinniu_uPic/rIIL0n.png)

#### 6. 部署CNI网络插件

> NotReady状态是不行的, 变成运行状态需要配置网络插件

```shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

默认镜像地址无法访问，sed命令修改为docker hub镜像仓库。

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl get pods -n kube-system

# 运行结果:
NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-2pc95   1/1     Running   0          72s
.....
```

![tmG4ju](http://xwjpics.gumptlu.work/qinniu_uPic/tmG4ju.png)

![SyoZDI](http://xwjpics.gumptlu.work/qinniu_uPic/SyoZDI.png)

#### 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```shell
$ kubectl create deployment nginx --image=nginx		# 创建一个nginx pod
$ kubectl get pods																# 查看pod状态
$ kubectl expose deployment nginx --port=80 --type=NodePort	# 暴露端口
$ kubectl get pod,svc															# 再次查看
```

访问地址格式：http://[NodeIP]:[Port]  

例如:

![aL0ypU](http://xwjpics.gumptlu.work/qinniu_uPic/aL0ypU.png)

访问: 192.168.8.146:31798

![DN71Sm](http://xwjpics.gumptlu.work/qinniu_uPic/DN71Sm.png)

#### 总结步骤

![8d9Thc](http://xwjpics.gumptlu.work/qinniu_uPic/8d9Thc.png)

### 2.3.2  二进制包安装

> 手动的部署过程,可以了解整个过程

#### 1. 总步骤

![ec1FCz](http://xwjpics.gumptlu.work/qinniu_uPic/ec1FCz.png)

> **前两步同之前, 自行操作**

| 角色   | IP            | hostname      |
| ------ | ------------- | ------------- |
| master | 192.168.8.122 | 122-k8smaster |
| node1  | 192.168.8.121 | 121-k8snode1  |

#### 2. 自签证书和部署

##### 1. cfssl证书生成工具

cfssl 是一个开源的证书管理工具，使用 json 文件生成证书，相比 openssl 更方便使用。 找任意一台服务器操作，这里用 Master 节点。

```shell
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64 
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

##### 2. 生成Etcd证书

* **自签证书颁发机构(CA)**

  创建工作目录:

  ```shell
  mkdir -p ~/TLS/{etcd, k8s}
  cd TLS/etcd
  ```

  自签CA:(创建两个json文件)

  ```shell
  cat > ca-config.json << EOF
  {
      "signing": {
          "default": {
              "expiry": "87660h"
          },
          "profiles": {
              "www": {
                  "expiry": "87660h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  EOF
  
  
  cat > ca-csr.json << EOF
  {
      "CN": "etcd CA",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing"
          }
      ]
  }
  EOF
  ```

  生成证书:

  ```shell
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
  ls *pem	  # 查看
  ```

  ![](http://xwjpics.gumptlu.work/qinniu_uPic/FNrclb.png)

* **使用自签CA签发Etcd HTTPS证书**

  创建证书申请文件:

  > 注意: 这里的Hosts字段改成你自己的集群IP,  hosts 字段中 IP 为所有 etcd 节点的集群内部通信 IP，一个都不能少!为了方便后期扩容可以多写几个预留的 IP。

  ```shell
  cat > server-csr.json <<EOF
  {
      "CN": "etcd",
      "hosts": [
          "192.168.8.121",
          "192.168.8.122"
      ],
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
  ```

  生成证书

  ```shell
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
  # 查看
  ls server*pem
  ```

  ![4YDcal](http://xwjpics.gumptlu.work/qinniu_uPic/4YDcal.png)

##### 3. 部署Etcd集群

* 在github上下载etcd二进制文件压缩包:

```shell
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
```

​	以下在单个节点上操作,然后复制给其他节点

* 创建工作目录并解压二进制包

  ```shell
  mkdir /opt/etcd/{bin,cfg,ssl} –p
  tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
  mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
  ```

* 创建etcd配置文件

  > 注意: 改成你自己的IP

  ```shell
  cat > /opt/etcd/cfg/etcd.conf << EOF
  #[Member]
  ETCD_NAME="etcd-1"
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="https://192.168.8.122:2380"
  ETCD_LISTEN_CLIENT_URLS="https://192.168.8.122:2379"
  
  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.8.122:2380"
  ETCD_ADVERTISE_CLIENT_URLS="https://192.168.8.122:2379"
  ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.8.122:2380,etcd-2=https://192.168.8.121:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
  ETCD_INITIAL_CLUSTER_STATE="new"
  EOF
  ```

  ETCD_NAME:节点名称，集群中唯一

  ETCD_DATA_DIR:数据目录

  ETCD_LISTEN_PEER_URLS:集群通信监听地址

  ETCD_LISTEN_CLIENT_URLS:客户端访问监听地址

  ETCD_INITIAL_ADVERTISE_PEER_URLS:集群通告地址

  ETCD_ADVERTISE_CLIENT_URLS:客户端通告地址

  ETCD_INITIAL_CLUSTER:集群节点地址

  ETCD_INITIAL_CLUSTER_TOKEN:集群 Token

  ETCD_INITIAL_CLUSTER_STATE:加入集群的当前状态，new 是新集群，existing 表示加入有集群

* systemd管理etcd

  ```shell
  cat > /usr/lib/systemd/system/etcd.service << EOF
  [Unit]
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target
  [Service]
  Type=notify
  EnvironmentFile=/opt/etcd/cfg/etcd.conf
  ExecStart=/opt/etcd/bin/etcd \
  --cert-file=/opt/etcd/ssl/server.pem \
  --key-file=/opt/etcd/ssl/server-key.pem \
  --peer-cert-file=/opt/etcd/ssl/server.pem \
  --peer-key-file=/opt/etcd/ssl/server-key.pem \
  --trusted-ca-file=/opt/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \ 
  --logger=zap
  Restart=on-failure
  LimitNOFILE=65536
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 拷贝刚才的证书

  把刚才的证书拷贝到配置文件中的路径

  ```shell
  cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/
  ```

* 复制文件到其他节点(换自己的IP)

  ```shell
  scp -r /opt/etcd/ root@192.168.8.121:/opt/
  scp /usr/lib/systemd/system/etcd.service root@192.168.8.121:/usr/lib/systemd/system/
  ```

* 修改其他节点的配置文件, 分别修改 etcd.conf 配置文件中的节点名称和当前服务器 IP

  以第二台为例:

  ![X5CYPF](http://xwjpics.gumptlu.work/qinniu_uPic/X5CYPF.png)

* 主节点、从节点分别启动并设置开机自启

  ```shell
  systemctl daemon-reload
  systemctl start etcd
  systemctl enable etcd
  ```

* 查看日志

  ```shell
  jorunalctl -u etcd
  ```

##### 4. 生成Kube-apiserver证书

![xq3hxf](http://xwjpics.gumptlu.work/qinniu_uPic/xq3hxf.png)

* 自签证书颁发机构(CA)

  在`~/TLS/k8s/`下

  ```shell
  cat > ca-config.json << EOF
  {
      "signing": {
          "default": {
              "expiry": "87660h"
          },
          "profiles": {
              "kubernetes": {
                  "expiry": "87660h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  EOF
  
  
  cat > ca-csr.json << EOF
  {
      "CN": "kubernetes",
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "Beijing",
              "ST": "Beijing",
              "O": "k8s",
              "OU": "System"
          }
      ]
  }
  EOF
  ```

  生成证书

  ```shell
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
  ```

* 使用自签CA签发kube-apiserver HTTPS证书

  创建证书申请文件:

  > <font color='#e54d42'>这里的hosts就是信任的节点IP</font>

  ```shell
  cat > server-csr.json << EOF
  {
      "CN": "kubernetes",
      "hosts": [
          "10.0.0.1",
          "127.0.0.1",
          "192.168.8.121",
          "192.168.8.122",
          "kubernetes",
          "kubernetes.default",
          "kubernetes.default.svc",
          "kubernetes.default.svc.cluster",
          "kubernetes.default.svc.cluster.local"
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [{
          "C": "CN",
          "L": "BeiJing",
          "ST": "BeiJing",
          "O": "k8s",
          "OU": "System"
      }]
  }
  EOF
  ```

  生成证书

  ```shell
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
  ```
  
  复制证书:
  
  ```shell
  cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
  ```

##### 5.部署Master-Node 

在Master主机上操作, 我这里是192.168.8.122节点

> <font color='#39b54a'>**部署MasterNode一共要部署三个组件: kube-apiserver、 kube-controller-manager、 kube-scheduler, 这些组件启动需要特定的配置文件, 并且将其加入到system管理中,方便日后管理.**</font>

下载源文件

下载地址: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG- 	1.18.md#v1183

> 貌似已经失效了, 找到的可用的新的连接:
>
> https://dl.k8s.io/v1.19.0/kubernetes-server-linux-amd64.tar.gz

只需要压缩包中的server文件夹就够了

解压压缩包:

```shell
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin 
cp kubectl /usr/bin/
```

###### 1.部署kube-apiserver

* 创建配置文件

  ```shell
  cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
  KUBE_APISERVER_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --etcd-servers=https://192.168.8.121:2379,https://192.168.8.122:2379 \\
  --bind-address=192.168.8.122 \\
  --secure-port=6443 \\
  --advertise-address=192.168.8.122 \\
  --allow-privileged=true \\
  --service-cluster-ip-range=10.0.0.0/24 \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
  --authorization-mode=RBAC,Node \\
  --enable-bootstrap-token-auth=true \\
  --token-auth-file=/opt/kubernetes/cfg/token.csv \\
  --service-node-port-range=30000-32767 \\
  --kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
  --kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
  --tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
  --tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --etcd-cafile=/opt/etcd/ssl/ca.pem \\
  --etcd-certfile=/opt/etcd/ssl/server.pem \\
  --etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
  EOF
  ```

  注：上面两个\ \ 第一个是转义符，第二个是换行符，使用转义符是为了使用 EOF 保留换行符。
  –logtostderr：启用日志
  —v：日志等级
  –log-dir：日志目录
  –etcd-servers：etcd 集群地址 (使用自己的etcd集群地址)
  –bind-address：监听地址
  –secure-port：https 安全端口
  –advertise-address：集群通告地址
  –allow-privileged：启用授权
  –service-cluster-ip-range：Service 虚拟 IP 地址段
  –enable-admission-plugins：准入控制模块
  –authorization-mode：认证授权，启用 RBAC 授权和节点自管理
  –enable-bootstrap-token-auth：启用 TLS bootstrap 机制 (新加入节点就自动授权,不用手动签署证书)
  –token-auth-file：bootstrap token 文件
  –service-node-port-range：Service nodeport 类型默认分配端口范围
  –kubelet-client-xxx：apiserver 访问 kubelet 客户端证书
  –tls-xxx-file：apiserver https 证书
  –etcd-xxxfile：连接 Etcd 集群证书
  –audit-log-xxx：审计日志

* 启用TLS Bootstrapping机制

  TLS Bootstraping:Master apiserver 启用 TLS 认证后，Work Node 节点 kubelet 和 kube- proxy 要与 kube-apiserver 进行通信，必须使用 CA 签发的有效证书才可以，当 Node 节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程, kubernetes引入了TLS bootstraping机制来自动颁发客户端证书,kubelet会以一个低权限用户向apiserver申请证书, kubelet证书由apiserver动态签署.

  ![gEHQLj](http://xwjpics.gumptlu.work/qinniu_uPic/gEHQLj.png)

  创建上述描述文件:

  ```shell
  cat > /opt/kubernetes/cfg/token.csv << EOF
  c47ffb939f5ca36231d9e3121a252940,kubelet-bootstrap,10001,"system:node-bootstrapper"
  EOF
  ```

  格式: `token，用户名，UID，用户组 token` 也可自行生成替换:

  `head -c 16 /dev/urandom | od -An -t x | tr -d ''`

* system管理apiserver配置:

  ```shell
  cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
  [Unit]
  Description=Kubernetes API Server
  Documentation=https://github.com/kubernetes/kubernetes
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
  ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
  Restart=on-failure
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 启动并设置开机自启

  ```shell
  systemctl daemon-reload 
  systemctl start kube-apiserver 
  # 检查启动状态
  systemctl status kube-apiserver		
  # 查看错误log输出
  cat /var/log/messages|grep kube-apiserver|grep -i error
  systemctl enable kube-apiserver		# 设置开机自启
  ```

  > **错误:** service-account-issuer is a required flag, --service-account-signing-key-file and --service-account-issuer are required flags
  >
  > **原因:** 下载的kubernetes-server源文件版本过高(1.20)导致kube-apiserver等执行命令的版本都是1.20, 高版本需要在/opt/kubernetes/cfg/kube-apiserver.conf配置文件中配置以上几个参数
  >
  > **解决:** 重新下载低版本
  >
  > https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/626

  正常运行:

  ![O1APRA](http://xwjpics.gumptlu.work/qinniu_uPic/O1APRA.png)

* 授权kubelet-bootstrap用户允许请求证书

  ```shell
  kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
  ```

  提示: `clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created`代表成功

###### 2.部署kube-controller-manager

* 创建配置文件

  ```shell
  cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
  KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --leader-elect=true \\
  --master=127.0.0.1:8080 \\
  --bind-address=127.0.0.1 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.244.0.0/16 \\
  --service-cluster-ip-range=10.0.0.0/24 \\
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --experimental-cluster-signing-duration=87600h0m0s"
  EOF
  ```

  –master：通过本地非安全本地端口 8080 连接 apiserver。

  –leader-elect：当该组件启动多个时，自动选举（HA）

  –cluster-signing-cert-file/–cluster-signing-key-file：自动为 kubelet 颁发证书的 CA，与apiserver保持一致/相同

* 使用Systemd管理controller-manager

  ```shell
  cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
  [Unit]
  Description=Kubernetes Controller Manager
  Documentation=https://github.com/kubernetes/kubernetes
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
  ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
  Restart=on-failure
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 启动并设置开机自启

  ```shell
  systemctl daemon-reload
  systemctl start kube-controller-manager
  systemctl enable kube-controller-manager
  systemctl status kube-controller-manager
  ```

###### 3.部署kube-scheduler

* 创建配置文件

  ```shell
  cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
  KUBE_SCHEDULER_OPTS="--logtostderr=false \
  --v=2 \
  --log-dir=/opt/kubernetes/logs \
  --leader-elect \
  --master=127.0.0.1:8080 \
  --bind-address=127.0.0.1"
  EOF
  ```

	–master：通过本地非安全本地端口 8080 连接 apiserver。

	–leader-elect：当该组件启动多个时，自动选举（HA）

* systemctl管理

  ```shell
  cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
  [Unit]
  Description=Kubernetes Scheduler
  Documentation=https://github.com/kubernetes/kubernetes
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
  ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
  Restart=on-failure
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 启动与开机自启

  ```shell
  systemctl daemon-reload
  systemctl start kube-scheduler
  systemctl enable kube-scheduler
  systemctl status kube-scheduler
  ```

###### 4.查看集群状态

`kubectl get cs`

![acdmko](http://xwjpics.gumptlu.work/qinniu_uPic/acdmko.png)

##### 6. 部署work-Node

在工作节点上操作,我这里是192.168.8.121节点

> <font color='#39b54a'>**工作节点需要安装的组件有: kubelet、 kube-proxy, 同样的使用systemctl进行管理**</font>

下载kubelet、 kube-proxy源文件

```shell
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kubelet kube-proxy /opt/kubernetes/bin
cp kubectl /usr/bin/
```

或者直接从Master节点发送过来

```shell
# work节点
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
# master节点
scp ~/k8s/kubernetes/server/bin/kubelet root@192.168.8.121:/opt/kubernetes/bin
scp ~/k8s/kubernetes/server/bin/kube-proxy root@192.168.8.121:/opt/kubernetes/bin
scp ~/k8s/kubernetes/server/bin/kubectl root@192.168.8.121:/usr/bin
```

拷贝Master上的一些配置文件到node节点:

```shell
scp -r /opt/kubernetes/ssl root@192.168.8.121:/opt/kubernetes
```

###### 1.部署kubelet

* 配置文件

  <font color='#e54d42'>**注意, 这里换成你的worknode的主机名**</font>

  ```shell
  cat > /opt/kubernetes/cfg/kubelet.conf << EOF
  KUBELET_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --hostname-override=121-k8snode1 \\			
  --network-plugin=cni \\
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
  --bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
  --config=/opt/kubernetes/cfg/kubelet-config.yml \\
  --cert-dir=/opt/kubernetes/ssl \\
  --pod-infra-container-image=lizhenliang/pause-amd64:3.0"
  EOF
  ```

  –hostname-override：显示名称，集群中唯一
  –network-plugin：启用CNI
  –kubeconfig：空路径，会自动生成，后面用于连接apiserver
  –bootstrap-kubeconfig：首次启动向apiserver申请证书
  –config：配置参数文件
  –cert-dir：kubelet证书生成目录
  –pod-infra-container-image：管理Pod网络容器的镜像

  kubelet-config.yml文件:

  ```shell
  cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
  kind: KubeletConfiguration
  apiVersion: kubelet.config.k8s.io/v1beta1
  address: 0.0.0.0
  port: 10250
  readOnlyPort: 10255
  cgroupDriver: cgroupfs
  clusterDNS:
  - 10.0.0.2
  clusterDomain: cluster.local 
  failSwapOn: false
  authentication:
    anonymous:
      enabled: false
    webhook:
      cacheTTL: 2m0s
      enabled: true
    x509:
      clientCAFile: /opt/kubernetes/ssl/ca.pem 
  authorization:
    mode: Webhook
    webhook:
      cacheAuthorizedTTL: 5m0s
      cacheUnauthorizedTTL: 30s
  evictionHard:
    imagefs.available: 15%
    memory.available: 100Mi
    nodefs.available: 10%
    nodefs.inodesFree: 5%
  maxOpenFiles: 1000000
  maxPods: 110
  EOF
  ```

* 生成bootstrap.kubeconfig文件

  ```shell
  KUBE_APISERVER="https://192.168.8.122:6443" # apiserver IP:PORT
  TOKEN="c47ffb939f5ca36231d9e3121a252940" # 与token.csv里保持一致
  kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=bootstrap.kubeconfig
  kubectl config set-credentials "kubelet-bootstrap" \
    --token=${TOKEN} \
    --kubeconfig=bootstrap.kubeconfig
  kubectl config set-context default \
    --cluster=kubernetes \
    --user="kubelet-bootstrap" \
    --kubeconfig=bootstrap.kubeconfig
  kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
  
  mv bootstrap.kubeconfig /opt/kubernetes/cfg
  ```

  生成的文件类似如下:

  ![d2uzo0](http://xwjpics.gumptlu.work/qinniu_uPic/d2uzo0.png)

* systemctl管理kunelet

  ```shell
  cat > /usr/lib/systemd/system/kubelet.service << EOF
  [Unit]
  Description=Kubernetes Kubelet
  After=docker.service
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
  ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
  Restart=on-failure
  LimitNOFILE=65536
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 启动并设置开机自启

  ```shell
  systemctl daemon-reload
  systemctl start kubelet
  systemctl enable kubelet
  systemctl status kubelet
  systemctl restart kubelet		#  重启
  ```

* 批准kubelet证书申请并加入集群(Master操作)

  ```shell
  # 查看kubelet证书请求
  kubectl get csr
  NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
  node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A   6m3s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
  
  # 批准申请(这里后面的Name是上面的Name,每个节点都不同)
  kubectl certificate approve node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A
  
  # 查看节点
  kubectl get node
  ```

  ![XPDOqE](http://xwjpics.gumptlu.work/qinniu_uPic/XPDOqE.png)

  注:由于网络插件还没有部署，节点会没有准备就绪 NotReady

###### 2.部署kube-proxy

* 配置文件

  ```shell
  cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
  KUBE_PROXY_OPTS="--logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --config=/opt/kubernetes/cfg/kube-proxy-config.yml"
  EOF
  ```
	<font color='#e54d42'>**注意, 这里换成你的worknode的主机名**</font>

  ```shell
  cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
  kind: KubeProxyConfiguration
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  bindAddress: 0.0.0.0
  metricsBindAddress: 0.0.0.0:10249
  clientConnection:
    kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
  hostnameOverride: 121-k8snode1			
  clusterCIDR: 10.0.0.0/24
  EOF
  ```

* 生成kube-proxy.kubeconfig文件(master生成在传到node)

  master

  ```shell
  # 切换工作目录
  cd ~/TLS/k8s
  # 创建证书请求文件
  cat > kube-proxy-csr.json << EOF
  {
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "L": "BeiJing",
        "ST": "BeiJing",
        "O": "k8s",
        "OU": "System"
      }
    ]
  }
  EOF
  # 生成证书
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
  
  # 发送
  scp -r /root/TLS/k8s root@192.168.8.121:/opt/TLS/
  ```

* 生成kubeconfig文件

  ```shell
  KUBE_APISERVER="https://192.168.8.122:6443"
  kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=kube-proxy.kubeconfig
  kubectl config set-credentials kube-proxy \
    --client-certificate=./kube-proxy.pem \
    --client-key=./kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig
  kubectl config set-context default \
    --cluster=kubernetes \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig
  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
  
  
  cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
  ```

* systemd管理kube-proxy

  ```shell
  cat > /usr/lib/systemd/system/kube-proxy.service << EOF
  [Unit]
  Description=Kubernetes Proxy
  After=network.target
  [Service]
  EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
  ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
  Restart=on-failure
  LimitNOFILE=65536
  [Install]
  WantedBy=multi-user.target
  EOF
  ```

* 启动并设置开机自启

  ```shell
  systemctl daemon-reload
  
  systemctl start kube-proxy
  
  systemctl enable kube-proxy
  
  systemctl status kube-proxy
  ```

###### 3.部署CNI网络

```shell
wget https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz
```

node节点操作

```shell
mkdir /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin
```

master节点操作:

> kube-flannel.yml文件下载
>
> 链接：https://pan.baidu.com/s/1abu6OwzAgcRdpbEpPPpDnw
> 提取码：hvf3

```shell
kubectl apply -f kube-flannel.yml
```

![Wh1ZnK](http://xwjpics.gumptlu.work/qinniu_uPic/Wh1ZnK.png)

###### 4. 查看集群状态

再次检查node节点状态:

```shell
kubectl get nodes
```

![Fdlx4R](http://xwjpics.gumptlu.work/qinniu_uPic/Fdlx4R.png)

#### 3. RBAC授权(master节点)

给用户授予RBAC权限

没有权限执行查看资源会报错` unable to upgrade connection: Forbidden (user=kubernetes, verb=create, resource=nodes, subresource=proxy)`

```shell
cat > apiserver-to-kubelet.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kubernetes-to-kubelet
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
kind: ClusterRoleBinding
metadata:
  name: system:kubernetes
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubernetes-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

创建授权:

`kubectl create -f apiserver-to-kubelet.yaml `

# 错误栈

所有错误首先要做的就是去看对应组件的日志

```shell
systemctl status [组件名] -l
```

## 1.容器无法部署,一直在创建creating

我的问题是配置kubelet时主机名称设置错误, 重装即可

## 2.查看容器、进入容器出现错误

> **错误描述:** 
>
> unable to upgrade connection: Forbidden (user=kubernetes, verb=create, resource=nodes, subresource=proxy)
>
> **原因:**
>
> 没有给用户RBAC权限
>
> **解决:**
>
> 见上方RBAC授权



