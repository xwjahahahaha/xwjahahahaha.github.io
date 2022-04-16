---
title: UnionFS
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 16:08:50
---

# 一、什么是Union File System

Union File System简称UnionFS, 是docker存储驱动功能的技术实现，是一种为Linux、FreeBSD和NetBSD操作系统设计的，**把其他文件系统联合到一个联合挂载点的文件系统服务**。

使用不同的分支(branch)把不同文件系统的文件和目录“透明的”覆盖，形成一个单一的文件系统。这些branch都是只读(read-only)或者读写(read-write)的。

Union FileSystem的核心逻辑是Union Mount，**它支持把一个目录A叠加到另一个目录B之上；用户对目录B的读取就是A加上B的内容，而对B目录里文件写入和改写则会保存在目录A上，因为A在上一层。**这个类似差分VHD的效果，但是是以文件为单位的。因为是Union FS主要负责叠加访问的逻辑，因此对叠加的目录的原始文件系统适应性比较好。

例如：Dockerfile：

```dockerfile
FROM ubuntu:14.04
ADD run.sh /
VOLUME /data
CMD ["./run.sh"]
```

![kQZvVw](http://xwjpics.gumptlu.work/qinniu_uPic/kQZvVw-20220416161230616.png)

<font color='#e54d42'>联合文件系统是docker镜像和容器的基础，它可以**使Docker把镜像做成分层的结构，使镜像的每一层可以被共享。**</font>例如两个业务镜像都是基于 CentOS 7 镜像构建的，那么这两个业务镜像在物理机上只需要存储一次 CentOS 7 这个基础镜像即可，从而节省大量存储空间。

## 1. 写时复制

虚拟后的联合文件系统采用一项资源管理技术来防止改变原来的文件—**写时复制(copy-on-write)/隐式共享**

简单来说就是一个资源被重复使用时，没有任何修改就不会复制一个新的文件，这个资源可以被新旧实例共享。

但是**在一个实例第一次对其执行写操作时，就会复制一个文件**

> <font color='#39b54a'>简单的来说，就是在第一次需要改动的时候创建一个副本，这样就不会影响之前的文件</font>

不同版本的UnionFS的版本不同，可以通过本机的`docker info`命令查看

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211102220030177-20220416161230711.png" alt="image-20211102220030177" style="zoom: 43%;" />

## 2. UnionFS实现分类

UnionFS 常见的实现有：

- **UnionFS**，很早就开始的实现，看名字就很霸道，目前使用较少。
- **AUFS**，全称是Advanced Multi Layered Unification Filesystem。创建于2006年，是对UnionFS的重构，因此初期也叫Another UnionFS。在 Docker 早期，OverlayFS 和 Devicemapper 相对不够成熟，AUFS 是最早也是最稳定的文件系统驱动。AuFS被众多Linux发行版所使用（多用于 Ubuntu 和 Debian 系统中），主要的场景就是LiveCD。目前最新的版本是AUFS4。
- **Devicemapper**：应用于Red Hat 或 CentOS 系统中，Devicemapper 一直作为 Docker 默认的联合文件系统驱动，为 Docker 在 Red Hat 或 CentOS 稳定运行提供强有力的保障。
- **OverlayFS**，近几年的Ubuntu发行版就使用的这个实现。OverlayFS的最大优势是在Linux 3.18时合并到内核，成为了Linux内建支持的文件系统了。

Docker 中最常用的联合文件系统有三种：AUFS、 Devicemapper和 OverlayFS。

# 二、AUFS

全称：`Advanced Multi-Layered Unification Filesystem`, 其完全重写了早期的`UnionFS 1.x`，主要的原因还是提高可靠性和性能，<u>并引入一些新的功能，例如分支的负载均衡等</u>。AUFS的一些实现已经被纳入`UnionFS 2.x `版本。

AUFS 是 Docker 最早使用的文件系统驱动，多用于 Ubuntu 和 Debian 系统中。在 Docker 早期，OverlayFS 和 Devicemapper 相对不够成熟，AUFS 是最早也是最稳定的文件系统驱动。

AUFS 目前并未被合并到 Linux 内核主线，因此只有 Ubuntu 和 Debian 等少数操作系统支持 AUFS。你可以使用以下命令查看你的系统是否支持 AUFS：

```shell
$ grep aufs /proc/filesystems
nodev   aufs
```

执行以上命令后，如果输出结果包含`aufs`，则代表当前操作系统支持 AUFS。AUFS 推荐在 Ubuntu 或 Debian 操作系统下使用，如果你想要在 CentOS 等操作系统下使用 AUFS，需要单独安装 AUFS 模块（生产环境不推荐在 CentOS 下使用 AUFS，如果你想在 CentOS 下安装 AUFS 用于研究和测试，可以参考这个[链接](https://github.com/bnied/kernel-ml-aufs)），安装完成后使用上述命令输出结果中有`aufs`即可。 当确认完操作系统支持 AUFS 后，你就可以配置 Docker 的启动参数了。

## 1. Docker配置AUFS

**更改Docker存储驱动为AUFS**

如果你的系统支持AUFS，那么可以通过以下方式实现修改Docker的启动参数配置

先在 `/etc/docker `下新建 daemon.json 文件，并写入以下内容：

```shell
{
  "storage-driver": "aufs"
}
```

然后重启docker

```shell
$ sudo systemctl restart docker
```

重启成功后查看`docker`的文件系统是否改变：`docker info`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Xqai2E-20220416161230809.png" alt="docker信息" style="zoom:50%;" />

## 2. AUFS原理

AUFS 是联合文件系统，意味着它在主机上使用多层目录存储，**每一个目录在 AUFS 中都叫作分支，而在 Docker 中则称之为层（layer），但最终呈现给用户的则是一个普通单层的文件系统，我们把多层以单一层的方式呈现出来的过程叫作联合挂载。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/rhWyCa-20220416161230902.png" alt="rhWyCa" style="zoom: 50%;" />

![NO0FRW](http://xwjpics.gumptlu.work/qinniu_uPic/NO0FRW-20220416161231015.png)

如图 1 所示，每一个镜像层和容器层都是` /var/lib/docker` 下的一个子目录，**镜像层和容器层都在 `aufs/diff `目录下**，每一层的目录名称是镜像或容器的` ID `值(**需要说明的是自从docker 1.10之后使用基于内容的寻址，diff目录下存储镜像layer文件夹不在与ID相同**))，联合挂载点在` aufs/mnt `目录下，**`mnt `目录是真正的容器工作目录。**

当一个镜像未生成容器时，AUFS 的存储结构如下。

- diff 文件夹：存储镜像内容，每一层都存储在以镜像层 ID 命名（不一定了）的子文件夹中。
- layers 文件夹：存储镜像层关系的元数据，在 diif 文件夹下的每个镜像层在这里都会有一个文件，**文件的内容为该层镜像的父级镜像的 ID。**
- mnt 文件夹：联合挂载点目录，<u>未生成容器时，该目录为空。</u>

当一个镜像已经生成容器时，AUFS 存储结构会发生如下变化。

- diff 文件夹：当容器运行时，会在 diff 目录下生成容器层。
- layers 文件夹：增加容器层相关的元数据。
- mnt 文件夹：容器的联合挂载点，这和容器中看到的文件内容一致。

**读取文件(容器优先策略)**

当我们在容器中读取文件时，可能会有以下场景。

- 文件在容器层中存在时：当文件存在于容器层时，直接从容器层读取。
- 当文件在容器层中不存在时：当容器运行时需要读取某个文件，如果容器层中不存在时，则从镜像层查找该文件，然后读取文件内容。
- 文件既存在于镜像层，又存在于容器层：当我们读取的文件既存在于镜像层，又存在于容器层时，将会从容器层读取该文件。

**修改文件或目录：**

AUFS 对文件的修改采用的是**写时复制**的工作机制，这种工作机制可以最大程度节省存储空间。

具体的文件操作机制如下。

- 第一次修改文件：当我们第一次在容器中修改某个文件时，AUFS 会触发写时复制操作，AUFS 首先从镜像层复制文件到容器层，然后再执行对应的修改操作。

> AUFS 写时复制的操作将会复制整个文件，如果文件过大，将会大大降低文件系统的性能，因此当我们有大量文件需要被修改时，AUFS 可能会出现明显的延迟。好在，**写时复制操作只在第一次修改文件时触发**，对日常使用没有太大影响。

- 删除文件或目录：当文件或目录被删除时，**AUFS 并不会真正从镜像中删除它**，因为镜像层是只读的，**AUFS 会创建一个特殊的文件或文件夹，这种特殊的文件或文件夹会阻止容器的访问。**

## 3. AUFS实践-镜像层

在清空了所有的镜像与容器后：

`tree /var/lib/docker/aufs/`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/2yVRwz-20220416161231067.png" alt="2yVRwz" style="zoom: 50%;" />

可以发现三个文件夹，但是没有任何的内容。

拉取一个镜像：ubuntu:15.04

`docker pull ubuntu:15.04`

![ENA0g5](http://xwjpics.gumptlu.work/qinniu_uPic/ENA0g5-20220416161231145.png)

Docker pull 显示的结果中有四个Layer

查看：

`tree /var/lib/docker/aufs/ -L 2`

![DOoP9g](http://xwjpics.gumptlu.work/qinniu_uPic/DOoP9g-20220416161231339.png)

在执行完命令后发现结果中也对应了四个存储文件夹，layers下面是对应的四个文件

查看**layers下的元数据**

`cat /var/lib/docker/aufs/layers/2bb1a15acde9ea1f49312678e3fa587dfa35d4bdbd91dc33851002c904739875 `

![Qb1yOs](http://xwjpics.gumptlu.work/qinniu_uPic/Qb1yOs-20220416161231392.png)

显示结果为其父级的ID

所以，最终的层级结构如图：

![L9Nk1l](http://xwjpics.gumptlu.work/qinniu_uPic/L9Nk1l-20220416161231451.png)

接下来我们使用`Dockerfile`来实现以ubuntu:15.04为基础镜像创建一个changed-ubuntu的镜像，唯一的区别是在`/tmp`文件夹下创建了一个`hello world`的文件.`Dockerfile`文件内容如下：

> <font color='#39b54a'>Dockerfile 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明。`From`表示要使用的基础镜像，每个`RUN`命令后面都会跟着具体的操作并且会在基础镜像之上创建新的一层</font>

```dockerfile
FROM ubuntu:15.04
RUN echo "hello world" > /tmp/newfile
```

运行：

```shell
# 编译并重命名为changed-ubuntu
$ docker build -t changed-ubuntu .
```

![n7okb9](http://xwjpics.gumptlu.work/qinniu_uPic/n7okb9-20220416161231515.png)

```shell
# 查看当前镜像：
$ docker images

REPOSITORY       TAG       IMAGE ID       CREATED         SIZE
changed-ubuntu   latest    5eec16ceae30   3 minutes ago   131MB
ubuntu           15.04     d1b55fd07600   5 years ago     131MB
```

```shell
# 查看changed-ubuntu使用了哪些image layer
$ docker history changed-ubuntu

IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
5eec16ceae30   5 minutes ago   /bin/sh -c echo "hello world" > /tmp/newfile    12B       
d1b55fd07600   5 years ago     /bin/sh -c #(nop) CMD ["/bin/bash"]             0B        
<missing>      5 years ago     /bin/sh -c sed -i 's/^#\s*\(deb.*universe\)$…   1.88kB    
<missing>      5 years ago     /bin/sh -c echo '#!/bin/sh' > /usr/sbin/poli…   701B      
<missing>      5 years ago     /bin/sh -c #(nop) ADD file:3f4708cf445dc1b53…   131MB  
```

可以看到`5eec16ceae30`镜像位于最上层，只有12B的大小，同时也可以观察到`5eec16ceae30`只用了12B的空间大小！这也证明了AUFS高效的使用了磁盘空间。下面的四层则是共享地构成ubuntu:15.04镜像的4个image layer

> <font color='#39b54a'>missing标记的layer是自docker 1.10之后，一个镜像的image layer的镜像历史数据都存储在一个文件中导致的，并不是什么异常</font>

再次查看layer的存储信息：

![QYHK0Y](http://xwjpics.gumptlu.work/qinniu_uPic/QYHK0Y-20220416161231595.png)

可以观察到，每个目录下都多出了`2b8a9...`这样的文件/文件夹

```shell
# 查看changed-ubuntu中创建的文件
$ cat diff/2b8a9b74f3c04b46702d2e2b3e3ecf558b51e042efb2e37d6d043b653fd68b0e/tmp/newfile

hello world

# 查看元数据文件
$ cat layers/2b8a9b74f3c04b46702d2e2b3e3ecf558b51e042efb2e37d6d043b653fd68b0e 

ee5a42077cd3580a57a2fb6b8e9d8e43ea0c6c8b10fa41911c881972ef45ca0a
eaa5375a0e40af681796c3aa48d822e99b89f48d4dd2484aa012b2b760a5456b
2bb1a15acde9ea1f49312678e3fa587dfa35d4bdbd91dc33851002c904739875
7cb95dc4362ff5d14e931b542153a93ed708c37ce5b9d34e5547766638dd1d14
```

元数据文件的输出也可以看出层级就是在之前的四层上加了一层

## 4. AUFS实践-容器层

docker使用AUFS的写时复制(CoW)技术来实现image layer共享和减少磁盘的空间占用。CoW机制意味着一旦某一个文件只有很小部分的改动，AUFS也需要复制整个文件，这样的设计会对容器的性能产生一定影响，尤其是在文件比较大的时候或者位于很多image layer的下方又或者AUFS需要深度搜索目录结构树的时候。不过，这样的复制也只是一次，后续就不再需要了。

启动一个**容器**的时候，**Docker会为其创建一个read-only的`init layer`，用于存储与这个容器内环境相关的内容；同时Docker还会创建一个read-write的layer来执行所有写操作。**

**存放位置：**

* **容器的mount挂载目录是`/var/lib/docker/aufs/mnt`**
* **元数据和配置文件都在`/var/lib/docker/containers/<container-id>`下**
* **read-write层存储在`/var/lib/docker/aufs/diff`下**

**即使容器停止后，读写层仍然存在，所以容器的重启不会丢失数据。只有当一个容器被删除的时候，这个读写层才会被删除**

接下来，通过实验验证这些结论：

首先，清空所有容器

```shell
# 查看所有容器，空
$ docker ps -aq
# 查看/var/lib/docker/containers/文件夹，空
$ ls /var/lib/docker/containers/
# 查看系统的aufs mount情况, 只有一个config文件夹
$ ls /sys/fs/aufs/

config
```

启动一个changed-ubuntu容器

```shell
$ docker run -dit changed-ubuntu bash
9eb5eb21b523982b561ae6fe1c67aaed2289bc5ec5801de8eea7428c1779988e
$ docker ps -a
CONTAINER ID   IMAGE            COMMAND   CREATED          STATUS          PORTS     NAMES
9eb5eb21b523   changed-ubuntu   "bash"    35 seconds ago   Up 34 seconds             gracious_sutherland
```

查看目录：

`ls /var/lib/docker/aufs/diff/`

![ZL64KQ](http://xwjpics.gumptlu.work/qinniu_uPic/ZL64KQ-20220416161231934.png)

多出了两个文件夹：带有init的是init层只读文件夹，而另一个没有的是read-write层文件夹

同理: `ls /var/lib/docker/aufs/mnt/`

![Xb2UG5](http://xwjpics.gumptlu.work/qinniu_uPic/Xb2UG5-20220416161231998.png)

`ls /var/lib/docker/aufs/layers/`

![dgOkJK](http://xwjpics.gumptlu.work/qinniu_uPic/dgOkJK-20220416161232071.png)

```shell
# 查看文件内容：layer依赖
$ cat /var/lib/docker/aufs/layers/e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121

e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121-init
2b8a9b74f3c04b46702d2e2b3e3ecf558b51e042efb2e37d6d043b653fd68b0e
ee5a42077cd3580a57a2fb6b8e9d8e43ea0c6c8b10fa41911c881972ef45ca0a
eaa5375a0e40af681796c3aa48d822e99b89f48d4dd2484aa012b2b760a5456b
2bb1a15acde9ea1f49312678e3fa587dfa35d4bdbd91dc33851002c904739875
7cb95dc4362ff5d14e931b542153a93ed708c37ce5b9d34e5547766638dd1d14

$ cat /var/lib/docker/aufs/layers/e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121-init 

2b8a9b74f3c04b46702d2e2b3e3ecf558b51e042efb2e37d6d043b653fd68b0e
ee5a42077cd3580a57a2fb6b8e9d8e43ea0c6c8b10fa41911c881972ef45ca0a
eaa5375a0e40af681796c3aa48d822e99b89f48d4dd2484aa012b2b760a5456b
2bb1a15acde9ea1f49312678e3fa587dfa35d4bdbd91dc33851002c904739875
7cb95dc4362ff5d14e931b542153a93ed708c37ce5b9d34e5547766638dd1d14
```

查看container文件夹: `ls `

```shell
$ tree /var/lib/docker/containers/

/var/lib/docker/containers/
└── 9eb5eb21b523982b561ae6fe1c67aaed2289bc5ec5801de8eea7428c1779988e
    ├── 9eb5eb21b523982b561ae6fe1c67aaed2289bc5ec5801de8eea7428c1779988e-json.log
    ├── checkpoints
    ├── config.v2.json
    ├── hostconfig.json
    ├── hostname
    ├── hosts
    ├── mounts
    ├── resolv.conf
    └── resolv.conf.hash
```

新创建了一个与容器ID相同名称的文件夹存放着容器的metadata元数据与config文件

接下来从系统AUFS来看mount的情况：在`/sys/fs/aufs/`下多了一个文件夹：`si_1b321958ae8fb1bb`

```shell
$ cat /sys/fs/aufs/si_1b321958ae8fb1bb/*

/var/lib/docker/aufs/diff/e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121=rw
/var/lib/docker/aufs/diff/e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121-init=ro+wh
/var/lib/docker/aufs/diff/2b8a9b74f3c04b46702d2e2b3e3ecf558b51e042efb2e37d6d043b653fd68b0e=ro+wh
/var/lib/docker/aufs/diff/ee5a42077cd3580a57a2fb6b8e9d8e43ea0c6c8b10fa41911c881972ef45ca0a=ro+wh
/var/lib/docker/aufs/diff/eaa5375a0e40af681796c3aa48d822e99b89f48d4dd2484aa012b2b760a5456b=ro+wh
/var/lib/docker/aufs/diff/2bb1a15acde9ea1f49312678e3fa587dfa35d4bdbd91dc33851002c904739875=ro+wh
/var/lib/docker/aufs/diff/7cb95dc4362ff5d14e931b542153a93ed708c37ce5b9d34e5547766638dd1d14=ro+wh
64
65
66
67
68
69
70
/dev/shm/aufs.xino
```

在这里的输出可以看出diff文件夹下的目录的访问权限情况:

只有最上面的`e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121`有read-write权限，其他都是只读

**删除一个文件：**

想要在容器中删除一个文件(`file1`)，AUFS会在容器的读写层中生成一个`.wh.file1`的文件来隐藏所有只读层的`file1`文件

下面进行测试：

```shell
# 进入容器删除之前的hello world文件然后退出
$ docker exec -it 9eb5eb21b523 /bin/bash
$ rm -rf /tmp/newfile
$ exit
# 在宿主机上检查
$ ls -a /var/lib/docker/aufs/diff/e26931d94a12e4b8bc0b66942d4525ff5aa6974d14ca9ebe847290533f6f0121/tmp/

# 输出结果
.  ..  .wh.newfile

# 删除掉这个 .wh.newfile, 再次进入容器
$ docker exec -it 9eb5eb21b523 /bin/bash
$ cat /tmp/newfile 

hello world
```

## 5. 自定义AUFS文件系统

通过简单的命令创建一个AUFS文件系统，感受如何使用AUFS和CoW实现文件管理

实验环境的创建如下：

（创建对应的文件夹/文件，文件的内容如红字所示）

![jyt6iP](http://xwjpics.gumptlu.work/qinniu_uPic/jyt6iP-20220416161232146.png)

接下来把这些文件夹都用AUFS的方式挂载到`mnt`目录下，需要注意的是，**在mount aufs命令中没有特定的指定权限的命令，默认的行为是dirs最左边的第一个目录是read-write，其他都为read-only**

```shell
$ mount -t aufs -o dirs=./container-layer/:./image-layer1/:image-layer2/:image-layer3/:image-layer4/ none ./mnt
$ tree mnt

mnt/
├── container-layer.txt
├── image-layer1.txt
├── image-layer2.txt
├── image-layer3.txt
└── image-layer4.txt
```

查看文件的读写权限：在`/sys/fs/aufs`文件夹下会默认创建一个`si_`开头的文件夹，我们查看即可

`cat /sys/fs/aufs/si_1b321958e543c9bb/* `    (具体文件夹名称因人而异)

```shell
/root/projects/golang_project/src/myDocker/AUFS-test/container-layer=rw
/root/projects/golang_project/src/myDocker/AUFS-test/image-layer1=ro
/root/projects/golang_project/src/myDocker/AUFS-test/image-layer2=ro
/root/projects/golang_project/src/myDocker/AUFS-test/image-layer3=ro
/root/projects/golang_project/src/myDocker/AUFS-test/image-layer4=ro
64
65
66
67
68
/root/projects/golang_project/src/myDocker/AUFS-test/container-layer/.aufs.xino
```

可以看到，满足了我们的需求

接下来向`mnt/image-layer4.txt`文件中添加一些文字:

```shell
$ echo -e "\nwrite to mnt's image-layer1.txt" >> ./mnt/image-layer4.txt
$ cat mnt/image-layer4.txt

# 输出
I am image layer-4

write to mnt's image-layer1.txt
```

这时`mnt`只是一个虚拟挂载点，并不能真实的反应文件的存储，所以我们需要寻找文件修改到底在什么位置

查看镜像层的文件：

```shell
$ cat image-layer4/image-layer4.txt 
I am image layer-4
```

并未修改，但是当我们查看`container-layer`目录的时候，发现多了一个`image-layer4.txt`的文件夹, 查看：

```shell
$ cat container-layer/image-layer4.txt

# 输出
I am image layer-4

write to mnt's image-layer1.txt
```

发现，文件的修改是在`container-layer`下实现的。**此时就体现了写时复制CoW的过程：**

**当在`mnt`容器的虚拟挂载目录下第一次修改文件（`image-layer4.txt`）时**

* 在`mnt`目录下查找`image-layer4.txt`文件，将其拷贝到读写层的`container-layer`目录中
* 然后，修改是在`container-layer`下的复制的`image-layer4.txt`文件中进行
* 最后在虚拟挂载点`mnt`目录下查看确实修改了文件

<font color='#e54d42'>**总结：**</font>

* **容器的mount挂载目录是`/var/lib/docker/aufs/mnt`, 这个目录是虚拟的，是多个`diff`目录叠加的效果**
* **使用AUFS挂载的目录都会在系统`/sys/fs/aufs/`下创建一个描述文件夹，其中描述了各个层的读写权限**
* **元数据和配置文件(主机配置、域名解析配置等)都在`/var/lib/docker/containers/<container-id>`下**
* **容器的创建会创建两个层只读`init`层与读写层，都存放在`/var/lib/docker/aufs/diff`下**
* **镜像层的文件也存储在`/var/lib/docker/aufs/diff`，容器的文件改变不会影响镜像层文件**
* **通过`Dockerfile`新打包的镜像会在基础镜像的基础上增加层（一个`RUN`一个层）**

# 三、Devicemapper

我们知道 AUFS 并不在 Linux 内核主干中，所以如果你的操作系统是 CentOS，就不推荐使用 AUFS 作为 Docker 的联合文件系统了。**通常使用 Devicemapper 作为 Docker 的联合文件系统。**

## 1. 什么是Devicemapper

**Devicemapper 是 <u>Linux 内核</u>提供的框架**，从 Linux 内核 2.6.9 版本开始引入，Devicemapper 与 AUFS 不同，AUFS 是一种文件系统，而**Devicemapper 是一种映射块设备的技术框架。**

Devicemapper 提供了<u>一种将物理块设备映射到虚拟块设备的机制</u>，目前 Linux 下比较流行的 LVM （Logical Volume Manager 是 Linux 下对磁盘分区进行管理的一种机制）和软件磁盘阵列（将多个较小的磁盘整合成为一个较大的磁盘设备用于扩大磁盘存储和提供数据可用性）都是基于 Devicemapper 机制实现的。

## 2. 关键技术

Devicemapper 将主要的工作部分分为用户空间和内核空间。

- 用户空间负责配置<u>**具体的设备映射策略与相关的内核空间控制逻辑**</u>，例如逻辑设备 dm-a 如何与物理设备 sda 相关联，怎么建立逻辑设备和物理设备的映射关系等。
- 内核空间则负责<u>**用户空间配置的关联关系实现**</u>，例如当 IO 请求到达虚拟设备 dm-a 时，内核空间负责接管 IO 请求，然后处理和过滤这些 IO 请求并转发到具体的物理设备 sda 上。

这个架构类似于 C/S （客户端/服务区）架构的工作模式，客户端负责具体的规则定义和配置下发，服务端根据客户端配置的规则来执行具体的处理任务。

Devicemapper 的工作机制主要围绕三个核心概念。

- **映射设备（mapped device）**：即对外提供的逻辑设备，它是由 Devicemapper 模拟的一个虚拟设备，并不是真正存在于宿主机上的物理设备。
- **目标设备（target device）**：目标设备是映射设备对应的物理设备或者物理设备的某一个逻辑分段，是真正存在于物理机上的设备。
- **映射表（map table）**：映射表记录了映射设备到目标设备的映射关系，它记录了<u>映射设备在目标设备的起始地址、范围和目标设备的类型等变量</u>。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/UPeWpH-20220416161232226.png" alt="UPeWpH" style="zoom: 33%;" />

Devicemapper 三个核心概念之间的关系如图 1，**映射设备通过映射表关联到具体的物理目标设备。事实上，映射设备不仅可以通过映射表关联到物理目标设备，也可以关联到虚拟目标设备，然后虚拟目标设备再通过映射表关联到物理目标设备（多层映射）**

Devicemapper 在内核中<u>通过很多模块化的映射驱动（target driver）插件实现了对真正 IO 请求的拦截、过滤和转发工作，</u>比如 Raid、软件加密、瘦供给（Thin Provisioning）等。其中**瘦供给模块是 Docker 使用 Devicemapper 技术框架中非常重要的模块，**下面我们来详细了解下瘦供给（Thin Provisioning）。

**瘦供给（Thin Provisioning）**

瘦供给的意思是**动态分配**，这跟传统的固定分配不一样。传统的固定分配是无论我们用多少都一次性分配一个较大的空间，这样可能导致空间浪费。而瘦供给是**我们需要多少磁盘空间，存储驱动就帮我们分配多少磁盘空间。**

这种分配机制就好比我们一群人围着一个大锅吃饭，负责分配食物的人每次都给你一点分量，当你感觉食物不够时再去申请食物，而当你吃饱了就不需要再去申请食物了，从而避免了食物的浪费，节约的食物可以分配给更多需要的人。

那么，你知道 Docker 是如何使用瘦供给来做到像 AUFS 那样分层存储文件的吗？答案就是： **Docker 使用了瘦供给的快照（snapshot）技术。**

什么是快照（snapshot）技术？这是全球网络存储工业协会 SNIA（StorageNetworking Industry Association）对快照（Snapshot）的定义：

> 关于指定数据集合的一个完全可用拷贝，该拷贝包括相应数据在某个时间点（拷贝开始的时间点）的映像。快照可以是其所表示的数据的一个副本，也可以是数据的一个复制品。

简单来说，<font color='#e54d42'><u>快照是数据在某一个时间点的存储状态</u></font>**。快照的主要作用是对数据进行备份，当存储设备发生故障时，可以使用已经备份的快照将数据恢复到某一个时间点，而 Docker 中的数据分层存储也是基于快照实现的。**

以上便是实现 Devicemapper 的关键技术，那 Docker 究竟是如何使用 Devicemapper 实现存储数据和镜像分层共享的呢？

## 3. Devicemapper 数据存储

当 Docker 使用 Devicemapper 作为文件存储驱动时，**Docker 将镜像和容器的文件存储在瘦供给池（thinpool）中，并将这些内容挂载在 `/var/lib/docker/devicemapper/` 目录下。**

这些目录储存 Docker 的容器和镜像相关数据，目录的数据内容和功能说明如下。

- devicemapper 目录（`/var/lib/docker/devicemapper/devicemapper/`）：存储镜像和容器实际内容，该目录由一个或多个块设备构成。
- metadata 目录（`/var/lib/docker/devicemapper/metadata/`）： 包含 Devicemapper 本身配置的元数据信息, 以 json 的形式配置，这些元数据<u>记录了镜像层和容器层之间的关联信息</u>。
- mnt 目录（ `/var/lib/docker/devicemapper/mnt/`）：是容器的联合挂载点目录，未生成容器时，该目录为空，而容器存在时，该目录下的内容跟容器中一致。

## 4. Devicemapper 镜像分层与共享

Devicemapper 使用专用的块设备实现镜像的存储，并且**像 AUFS 一样使用了<u>写时复制</u>的技术来保障最大程度节省存储空间**，所以 **Devicemapper 的镜像分层也是依赖快照来是实现的。**

**Devicemapper 的每一镜像层都是其下一层的快照，最底层的镜像层是我们的瘦供给池**，通过这种方式实现镜像分层有以下优点。

- 相同的镜像层，仅在磁盘上存储一次。例如，我有 10 个运行中的 busybox 容器，底层都使用了 busybox 镜像，那么 busybox 镜像只需要在磁盘上存储一次即可。
- 快照是写时复制策略的实现，也就是说，当我们需要对文件进行修改时，文件才会被复制到读写层。
- 相比对文件系统加锁的机制，<u>Devicemapper 工作在块级别，因此可以实现同时修改和读写层中的多个块设备，比文件系统效率更高。</u>

**当我们需要读取数据时，如果数据存在底层快照中，则向底层快照查询数据并读取。**

**当我们需要写数据时，则向瘦供给池动态申请存储空间生成读写层，然后把数据复制到读写层进行修改**。

Devicemapper 默认每次申请的大小是 64K 或者 64K 的倍数，因此每次新生成的读写层的大小都是 64K 或者 64K 的倍数。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/lDzftz-20220416161232307.png" alt="lDzftz" style="zoom:33%;" />

这个 Ubuntu 镜像一共有四层，每一层镜像都是下一层的快照，镜像的最底层是基础设备的快照。当容器运行时，容器是基于镜像的快照。综上，**Devicemapper 实现镜像分层的根本原理就是快照。**

## 5. Docker配置Devicemapper

*（因为本机没有是Unbuntu系统，所以此部分并没有实验，仅记录）*

Docker 的 Devicemapper 模式有两种：第一种是` loop-lvm `模式，该模式主要用来开发和测试使用；第二种是 `direct-lvm `模式，该模式推荐在生产环境中使用。

### loop-lvm

使用以下命令停止已经运行的 Docker：

```shell
$ sudo systemctl stop docker
```

编辑` /etc/docker/daemon.json `文件，如果该文件不存在，则创建该文件，并添加以下配置：

```shell
{
  "storage-driver": "devicemapper"
}
```

启动 Docker：

```shell
$ sudo systemctl start docker
```

验证 Docker 的文件驱动模式：

```shell
$ docker info

# 输出
...
Storage Driver: devicemapper
  Pool Name: docker-253:1-423624832-pool
  Pool Blocksize: 65.54kB
  Base Device Size: 10.74GB
  Backing Filesystem: xfs
  Udev Sync Supported: true
  Data file: /dev/loop0
  Metadata file: /dev/loop1
  Data loop file: /var/lib/docker/devicemapper/devicemapper/data
  Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
...
```

可以看到 Storage Driver 为 devicemapper，这表示 Docker 已经被配置为 Devicemapper 模式。

但是这里输出的 Data file 为 `/dev/loop0`，这表示我们目前在使用的模式为 loop-lvm。但是由于 loop-lvm 性能比较差，因此不推荐在生产环境中使用 loop-lvm 模式。下面我们看下生产环境中应该如何配置 Devicemapper 的 direct-lvm 模式。

### direct-lvm 

同样的做法，只是` /etc/docker/daemon.json `文件修改为：

```json
{
  "storage-driver": "devicemapper",
  "storage-opts": [
    "dm.directlvm_device=/dev/xdf",
    "dm.thinp_percent=95",
    "dm.thinp_metapercent=1",
    "dm.thinp_autoextend_threshold=80",
    "dm.thinp_autoextend_percent=20",
    "dm.directlvm_device_force=false"
  ]
}
```

其中 `directlvm_device `指定需要用作 Docker 存储的磁盘路径，Docker 会动态为我们创建对应的存储池。例如这里我想把 `/dev/xdf `设备作为我的 Docker 存储盘，`directlvm_device `则配置为 /dev/xdf。

```shell
$ docker info

...
Storage Driver: devicemapper
  Pool Name: docker-thinpool
  Pool Blocksize: 65.54kB
  Base Device Size: 10.74GB
  Backing Filesystem: xfs
  Udev Sync Supported: true
  Data file:
  Metadata file:
  Data loop file: /var/lib/docker/devicemapper/devicemapper/data
  Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
...
```

当我们看到 Storage Driver 为 devicemapper，并且 Pool Name 为 `docker-thinpool `时，这表示 Devicemapper 的 direct-lvm 模式已经配置成功

**总结：**

**Devicemapper 使用块设备来存储文件，运行速度会比直接操作文件系统更快，因此很长一段时间内在 Red Hat 或 CentOS 系统中，Devicemapper 一直作为 Docker 默认的联合文件系统驱动，为 Docker 在 Red Hat 或 CentOS 稳定运行提供强有力的保障。**

# 四、OverlayFS

OverlayFS 的发展分为两个阶段。2014 年，OverlayFS 第一个版本被合并到 Linux 内核 3.18 版本中，此时的 OverlayFS 在 Docker 中被称为`overlay`文件驱动。

`overlay`文件驱动模型：

![9mxQXM](http://xwjpics.gumptlu.work/qinniu_uPic/9mxQXM-20211108191055867-20220416161232399.png)

由于第一版的`overlay`文件系统存在很多弊端（例如运行一段时间后Docker 会报 "too many links problem" 的错误）， **Linux 内核在 4.0 版本对`overlay`做了很多必要的改进，此时的 OverlayFS 被称之为`overlay2`。**目前较新版本的docker都是默认使用`overlay2`

因此，在 Docker 中 OverlayFS 文件驱动被分为了两种，<u>一种是早期的`overlay`，不推荐在生产环境中使用，另一种是更新和更稳定的`overlay2`，推荐在生产环境中使用。</u>下面的内容我们主要围绕`overlay2`展开。

## 1. 使用 overlay2 的先决条件 

*（因为本机没有采用xfs的文件系统，所以此部分并没有实验，仅记录）*

`overlay2`虽然很好，但是它的使用是有一定条件限制的。

- 要想使用`overlay2`，**Docker 版本必须高于 17.06.02。**
- 如果你的操作系统是 RHEL 或 CentOS，Linux 内核版本必须使用 3.10.0-514 或者更高版本，其他 Linux 发行版的内核版本必须高于 4.0（例如 Ubuntu 或 Debian），你可以使用`uname -a`查看当前系统的内核版本。
- `overlay2`最好搭配 xfs 文件系统使用，并且使用 xfs 作为底层文件系统时，**d_type必须开启**，可以使用以下命令验证 d_type 是否开启：

```shell
$ apt install xfsprogs
$ xfs_info /var/lib/docker | grep ftype
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
```

当输出结果中有 ftype=1 时，表示 d_type 已经开启。如果你的输出结果为 ftype=0，则**需要重新格式化磁盘目录**，命令如下：

```shell
$ sudo mkfs.xfs -f -n ftype=1 /path/to/disk
```

另外，在生产环境中，推荐挂载 `/var/lib/docker `目录到单独的磁盘或者磁盘分区，这样可以避免该目录写满影响主机的文件写入，并且把挂载信息写入到 `/etc/fstab`，防止机器重启后挂载信息丢失。

挂载配置中推荐开启` pquota`，这样可以防止某个容器写文件溢出导致整个容器目录空间被占满。写入到 `/etc/fstab` 中的内容如下：

```shell
$UUID /var/lib/docker xfs defaults,pquota 0 0
```

其中 UUID 为 `/var/lib/docker `所在磁盘或者分区的 UUID 或者磁盘路径。 如果你的操作系统无法满足上面的任何一个条件，那我推荐你使用 AUFS 或者 Devicemapper 作为你的 Docker 文件系统驱动。

> 通常情况下， overlay2 会比 AUFS 和 Devicemapper 性能更好，而且更加稳定，因为 overlay2 在 inode 优化上更加高效。因此在生产环境中推荐使用 overlay2 作为 Docker 的文件驱动。

下面通过实例来教你如何初始化` /var/lib/docker`目录，为后面配置 Docker 的`overlay2`文件驱动做准备。

**准备 `/var/lib/docker `目录**

1.使用 lsblk（Linux 查看磁盘和块设备信息命令）命令查看本机磁盘信息：

```shell
$ lsblk

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0  500G  0 disk
`-vda1 253:1    0  500G  0 part /
vdb    253:16   0  500G  0 disk
`-vdb1 253:17   0    8G  0 part
```

可以看到，我的机器有两块磁盘，一块是 vda，一块是 vdb。其中 vda 已经被用来挂载系统根目录，这里我想把 /var/lib/docker 挂载到 vdb1 分区上。

2.使用 mkfs 命令格式化磁盘 vdb1：

```shell
$ sudo mkfs.xfs -f -n ftype=1 /dev/vdb1
```

3.将挂载信息写入到 /etc/fstab，保证机器重启挂载目录不丢失：

```shell
$ sudo echo "/dev/vdb1 /var/lib/docker xfs defaults,pquota 0 0" >> /etc/fstab
```

4.使用 mount 命令使得挂载目录生效：

```shell
$ sudo mount -a
```

5.查看挂载信息：

```shell
$ lsblk

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0  500G  0 disk
`-vda1 253:1    0  500G  0 part /
vdb    253:16   0  500G  0 disk
`-vdb1 253:17   0    8G  0 part /var/lib/docker
```

可以看到此时 /var/lib/docker 目录已经被挂载到了 vdb1 这个磁盘分区上。我们使用 xfs_info 命令验证下 d_type 是否已经成功开启：

```shell
$ xfs_info /var/lib/docker | grep ftype

naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
```

可以看到输出结果为 ftype=1，证明 d_type 已经被成功开启。

准备好 /var/lib/docker 目录后，我们就可以配置 Docker 的文件驱动为 overlay2，并且启动 Docker 了。

## 2. Docker配置Overlay2

同样的修改`/etc/docker/daemon.json`文件

```json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.size=20G",							// 按需修改
    "overlay2.override_kernel_check=true"
  ]
}
```

其中 storage-driver 参数指定使用 overlay2 文件驱动，overlay2.size 参数表示限制每个容器根目录大小为 20G。限制每个容器的磁盘空间大小是通过 xfs 的 pquota 特性实现，overlay2.size 可以根据不同的生产环境来设置这个值的大小。我推荐你在生产环境中开启此参数，防止某个容器写入文件过大，导致整个 Docker 目录空间溢出。

重启`sudo systemctl restart docker`再查看`docker info`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/MFU3kK-20220416161232597.png" alt="MFU3kK" style="zoom: 43%;" />

## 3. Overlay2工作原理

### 文件存储/挂载结构

overlay2 和 AUFS 类似，**它将所有目录称之为层（layer）**，overlay2 的目录是镜像和容器分层的基础，而把这些层统一展现到同一的目录下的过程称为**联合挂载（union mount）**。**overlay2把目录的下一层叫作`lowerdir`，上一层叫作`upperdir`，联合挂载后的结果叫作`merged`。**

> overlay支持两层结构overlay2 文件系统最多支持 128 个层数叠加，也就是说你的 Dockerfile 最多只能写 128 行，不过这在日常使用中足够了。

下面我们通过拉取一个 Ubuntu 操作系统的镜像来看下 overlay2 是如何存放镜像文件的。

首先，我们通过以下命令拉取 Ubuntu 镜像：

```shell
$ docker pull ubuntu:16.04

16.04: Pulling from library/ubuntu
58690f9b18fc: Pull complete 
b51569e7c507: Pull complete 
da8ef40b9eca: Pull complete 
fb15d46c38dc: Pull complete 
Digest: sha256:0f71fa8d4d2d4292c3c617fda2b36f6dabe5c8b6e34c3dc5b0d17d4e704bd39c
Status: Downloaded newer image for ubuntu:16.04
docker.io/library/ubuntu:16.04
```

可以看到有四层的镜像层被提取、构建

可以看到镜像一共被分为四层拉取，拉取完镜像后我们查看一下 overlay2 的目录：

```shell
$ sudo ls -l /var/lib/docker/overlay2/

total 20
drwx--x--- 4 root root 4096 Nov  8 14:41 0585e2ae16f2898a091715bb2422a307a4b1df8739e1144f6513697689533ce1
drwx--x--- 4 root root 4096 Nov  8 14:41 4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca
drwx--x--- 3 root root 4096 Nov  8 14:41 6d04c5b6ac6ac44571f3644b89ddfadae8d91bb2b6ddf57fa9165ff3979f6b4a
drwx--x--- 4 root root 4096 Nov  8 14:41 d3ab29df8d4c66ba86e7aab3114ce7be6d76d1a17d73371fc62c35123679701b
drwx------ 2 root root 4096 Nov  8 14:41 l
```

可以看到 overlay2 目录下出现了四个镜像层目录和**一个`l`目录**，我们首先来查看一下`l`目录的内容：

```shell
$ sudo ls -l /var/lib/docker/overlay2/l

lrwxrwxrwx 1 root root 72 Nov  8 14:41 3WWCAJ23M5IBV6SNVFEG3S447Q -> ../6d04c5b6ac6ac44571f3644b89ddfadae8d91bb2b6ddf57fa9165ff3979f6b4a/diff
lrwxrwxrwx 1 root root 72 Nov  8 14:41 AVVSOD5YLZOM447U2GRZRSA6ZY -> ../d3ab29df8d4c66ba86e7aab3114ce7be6d76d1a17d73371fc62c35123679701b/diff
lrwxrwxrwx 1 root root 72 Nov  8 14:41 D3BPPGSXXULN4ZVYG4CHMCDF37 -> ../0585e2ae16f2898a091715bb2422a307a4b1df8739e1144f6513697689533ce1/diff
lrwxrwxrwx 1 root root 72 Nov  8 14:41 ZBKV2DMDHFHGOLRZYHCL4VCJ4R -> ../4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca/diff
```

可以看到<font color='#e54d42'>**`l`目录是一堆软连接**，**把一些较短的随机串软连到镜像层的 diff 文件夹下，这样做是为了避免达到`mount`命令参数的长度限制**。</font> 下面我们查看任意一个镜像层下的文件内容：

```shell
$ sudo ls -l /var/lib/docker/overlay2/0585e2ae16f2898a091715bb2422a307a4b1df8739e1144f6513697689533ce1/

total 16
-rw------- 1 root root    0 Nov  8 14:41 committed
drwxr-xr-x 6 root root 4096 Nov  8 14:41 diff
-rw-r--r-- 1 root root   26 Nov  8 14:41 link
-rw-r--r-- 1 root root   28 Nov  8 14:41 lower
drwx------ 2 root root 4096 Nov  8 14:41 work
```

**镜像层的 link 文件内容为该镜像层的短 ID，diff 文件夹为该镜像层的改动内容，lower 文件为该层的所有父层镜像的短 ID。** 我们可以通过`docker image inspect`命令来查看某个镜像的层级关系，例如我想查看刚刚下载的 Ubuntu 镜像之间的层级关系，可以使用以下命令：

```shell
$ docker image inspect ubuntu:16.04

....
"GraphDriver": {    
            "Data": {                                                                
                "LowerDir": "/var/lib/docker/overlay2/d3ab29df8d4c66ba86e7aab3114ce7be6d76d1a17d73371fc62c35123679701b/diff:/var/lib/docker/overlay2/0585e2ae16f2898a091715bb2422a307a4b1df8739e1144f6513697689533ce1/diff:/var/lib/docker/overlay2/6d04c5b6ac6ac44571f3644b89ddfadae8d91bb2b6ddf57fa9165ff3979f6b4a/diff", 
                "MergedDir": "/var/lib/docker/overlay2/4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca/merged",                     
                "UpperDir": "/var/lib/docker/overlay2/4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca/diff",             
                "WorkDir": "/var/lib/docker/overlay2/4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca/work"                    
            },    
...
```

其中 MergedDir 代表当前镜像层在 overlay2 存储下的目录，LowerDir 代表当前镜像的父层关系，使用冒号分隔，冒号最后代表该镜像的最底层。

下面我们将镜像运行起来成为容器：`docker run --name=ubuntu -d ubuntu:16.04 sleep 3600`

我们使用`docker inspect`命令来查看一下容器的工作目录：

```shell
...
"GraphDriver": {                    
            "Data": {                        
                "LowerDir": "/var/lib/docker/overlay2/fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26f
e36f-init/diff:/var/lib/docker/overlay2/4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca/diff:/var
/lib/docker/overlay2/d3ab29df8d4c66ba86e7aab3114ce7be6d76d1a17d73371fc62c35123679701b/diff:/var/lib/docker/overlay
2/0585e2ae16f2898a091715bb2422a307a4b1df8739e1144f6513697689533ce1/diff:/var/lib/docker/overlay2/6d04c5b6ac6ac4457
1f3644b89ddfadae8d91bb2b6ddf57fa9165ff3979f6b4a/diff",
                "MergedDir": "/var/lib/docker/overlay2/fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26
fe36f/merged",                                                                              
                "UpperDir": "/var/lib/docker/overlay2/fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26f
e36f/diff",
                "WorkDir": "/var/lib/docker/overlay2/fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26fe
36f/work"
            },
...
```

**MergedDir 后面的内容即为容器层的工作目录，LowerDir 为容器所依赖的镜像层目录。** 然后我们查看下 overlay2 目录下的内容：

```shell
$ sudo ls /var/lib/docker/overlay2/

0585e2ae16f2898a091715bb2422a307a4b1df8739e1144f6513697689533ce1
4af68b84dfe0d439f31d66fe8246f851ff87d72b09a531fd5517688ceaf876ca
6d04c5b6ac6ac44571f3644b89ddfadae8d91bb2b6ddf57fa9165ff3979f6b4a
d3ab29df8d4c66ba86e7aab3114ce7be6d76d1a17d73371fc62c35123679701b
fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26fe36f
fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26fe36f-init
l
```

可以看到 overlay2 目录下增加了容器层相关的目录，我们再来查看一下容器层下的内容：

```shell
$ sudo tree /var/lib/docker/overlay2/fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26fe36f -L 2

/var/lib/docker/overlay2/fc45c1a77027a24a029361bd29bc701ce9e00c45b4cba3b949d89fb2a26fe36f
├── diff
├── link
├── lower
├── merged
│   ├── bin
│   ├── boot
│   ├── dev
│   ....
└── work
    └── work
```

link 和 lower 文件与镜像层的功能一致，link 文件内容为该容器层的短 ID，lower 文件为该层的所有父层镜像的短 ID 。**diff 目录为容器的读写层，容器内修改的文件都会在 diff 中出现，merged 目录为分层文件联合挂载后的结果，也是容器内的工作目录。**

总体来说，**overlay2 是这样储存文件的：<u>`overlay2`将镜像层和容器层都放在单独的目录</u>，并且有唯一 ID，每一层仅存储发生变化的文件，最终使用联合挂载技术将容器层和镜像层的所有文件统一挂载到容器中，使得容器中看到完整的系统文件。**

<font color='#e54d42'>**overlayer文件存储结构总结：**</font>

* 不同于AUFS集中放置，以镜像、容器单独目录为主，每一个镜像或者容器都是单独的目录，其中都具有`diff`等文件夹
* 在`/var/lib/docker/overlay2/`下有一个`l`目录，是一堆软连接**，**把一些较短的随机串软连到镜像层的 diff 文件夹下，这样做是为了避免达到`mount`命令参数的长度限制
* 镜像文件夹：
  * link：文件中记录了该镜像的短ID
  * diff：该镜像层的改动内容, 按照层级关系不断向上叠加, 每一层都在下一层的基础上只保存新的文件
  * lower：所有父层的短ID （最底层镜像没有lower），可以索引出整个层次结构
  * work：用来完成如copy-on-write（CoW写时复制）的操作

* 容器文件夹：
  * 容器文件夹与AUFS一样有两个，一个init、一个读写
  * link：同镜像
  * diff：同镜像，容器的读写层，改动都会放在里面
  * lower：同镜像
  * merge：文件的联合挂载点。容器最终的运行环境（虚拟的）（init层没有，只在读写层）

### 读取、修改文件

overlay2 的工作过程中对文件的操作分为读取文件和修改文件。

**读取文件（容器层优先）**

容器内进程读取文件分为以下三种情况。

- 文件在容器层中存在：当文件存在于容器层并且不存在于镜像层时，直接从容器层读取文件；
- 当文件在容器层中不存在：当容器中的进程需要读取某个文件时，如果容器层中不存在该文件，则从镜像层查找该文件，然后读取文件内容；
- 文件既存在于镜像层，又存在于容器层：当我们读取的文件既存在于镜像层，又存在于容器层时，将会从容器层读取该文件。

**修改文件或目录（与AUFS类似）**

overlay2 对文件的修改采用的是**写时复制**的工作机制，这种工作机制可以最大程度节省存储空间。具体的文件操作机制如下。

- 第一次修改文件：当我们**第一次**在容器中修改某个文件时，overlay2 会触发写时复制操作，**overlay2 首先从镜像层复制文件到容器层，然后在容器层执行对应的文件修改操作。**

- 删除文件或目录：当文件或目录被删除时，overlay2 并不会真正从镜像中删除它，因为镜像层是只读的，overlay2 会创建一个特殊的文件或目录，这种特殊的文件或目录会阻止容器的访问。

## 4. 总结

overlay2 目前已经是 Docker 官方推荐的文件系统了，也是目前安装 Docker 时默认的文件系统，因为 overlay2 在生产环境中不仅有着较高的性能，它的稳定性也极其突出。但是 overlay2 的使用还是有一些限制条件的，例如要求 Docker 版本必须高于 17.06.02，内核版本必须高于 4.0 等。因此，在生产环境中，如果你的环境满足使用 overlay2 的条件，请尽量使用 overlay2 作为 Docker 的联合文件系统。

# 五、各个UnionFS对比与总结

|              |            适用系统            | 是否合并内核 |                           版本要求                           | 优点/特点                                                    | 缺点                                        |
| :----------- | :----------------------------: | :----------: | :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------- |
| AUFS         | 多用于 Ubuntu 和 Debian 系统中 |      否      |                >=3.1, 内核>=4.0请使用Overlay2                | Docker 最早使用的文件系统驱动，稳定                          | 代码可读性差（被拒绝合并到内核的原因）      |
| Devicemapper |       Red Hat 或 CentOS        |      是      |                      Linux 内核 >=2.6.9                      | 使用映射块设备的技术框架, 比文件系统效率高，稳定，docker默认的联合文件系统 | 只用于CentOS或Red hat系列                   |
| Overlay      |     Ubuntu等等大多数linux      |      是      |                       Linux内核>=3.18                        | AUFS的继承，合并到了Linux内核                                | 只工作在两层，并且运行时可能有一些意外的bug |
| Overlay2     |     Ubuntu等等大多数linux      |      是      | Docker >=17.06.02,  Red Hat 或 CentOS 内核>=3.10.0-514 其他linux发行版内核>=4.0 | overlay2在inode优化上更加高效，最高128层，已合并到内核，速度更快，主流 | 仍然年轻，不稳定，生产环境使用要慎重        |

