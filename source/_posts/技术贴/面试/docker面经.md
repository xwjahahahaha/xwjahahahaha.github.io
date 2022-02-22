---
title: docker面经
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-11-21 09:33:26
---

[TOC]

<!-- more -->

# 一、基本知识点

## 0. 什么是kernel内核

内核是常驻于内存的一套软件，是操作系统的核心，负责调度硬件资源，CPU执行调度等操作

内核向上提供了系统访问接口，提供给用户态的应用程序调用访问使用硬件资源，例如I/O操作等

## 1. 虚拟机与容器的区别

* 虚拟机本质上是一种模拟系统，模拟硬件的输入输出。每个虚拟机都需要使用一个单独的内核，开机前需要检查，开机较慢，占用内存大
* 虚拟机需要包含一整套完整的虚拟环境，包括操作系统、用户程序函数库等，体积比较大，可移植性差

---

* docker本质上是一个进程，所有启动的容器与宿主机共享一个内核，只需要模拟内核的输入输出而不需要模拟硬件的输入输出，是操作系统级别的虚拟化，各个容器运行在用户态。所以docker启动很快，而且占用内存小。
* docker 通过资源隔离和资源限制按需构建镜像并通过联合文件系统保证了文件的隔离性，所以docker的体积较小，移植性很好（`docker hub`）

虚拟机与docker架构对比：

虚拟机：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211028205122292.png" alt="image-20211028205122292" style="zoom: 50%;" />

docker容器：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211028205147983.png" alt="image-20211028205147983" style="zoom: 50%;" />

## 2. 容器的启动过程

docker的启动比一般的linux进程启动多了几步：

1. 复制父进程为一个新的子进程，父子进程上下文完全一致
2. 然后自定义子进程的rootfs，将子进程的根目录变为这个根目录（pivot-root）
3. 使用Namespace隔离，PID NS让当前进程号变为0，HostName NS让容器重新命名一个新主机名，还有用户名隔离User NS、进程间通讯隔离IPC NS、网络隔离Network NS等，这些都是高版本的linux内核提供的

因为docker的第一个进程已经实现了隔离，所以容器后续的子进程再复制就不用再做隔离了，天然继承了父进程的上下文环境。

## 3. Namespace-资源隔离

### 1. 基本

内核实现了将同一个命名空间下的进程的全局系统资源隔离封装，让每个命名空间之间的资源不同且不可见，同一个命名空间下的进程资源共享

| Namespace 名称                   | 作用                                      | 内核版本 |
| :------------------------------- | :---------------------------------------- | :------- |
| Mount（mnt）                     | 隔离挂载点                                | 2.4.19   |
| Process ID (pid)                 | 隔离进程 ID                               | 2.6.24   |
| Network (net)                    | 隔离网络设备，端口号等                    | 2.6.29   |
| Interprocess Communication (ipc) | 隔离 System V IPC 和 POSIX message queues | 2.6.19   |
| UTS Namespace(uts)               | 隔离主机名和域名                          | 2.6.19   |
| User Namespace (user)            | 隔离用户和用户组                          | 3.8      |
| Control group (cgroup) Namespace | 隔离 Cgroups 根目录                       | 4.6      |
| Time Namespace                   | 隔离系统时间                              | 5.6      |

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211028230603400.png" alt="image-20211028230603400" style="zoom: 33%;" />

### 2. 命名空间常见使用的API

| API        | 官方解释                                                     | 简单解释                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| clone(2)   | 系统调用创建一个新进程。如果调用的flags参数指定了上面列出的一个或多个`CLONE_NEW*`标志，则新的命名空间是为每个标志创建，并且子进程被创建为               这些命名空间的成员。 | 创建一个新进程，可以通过`CLONE_NEW*`参数指定哪些命名空间被创建，并且他们的子进程也会被包含在这些Namespace中 |
| setns(2)   | 系统调用允许调用进程加入现有的命名空间。                     | 当前进程切到指定的命名空间                                   |
| unshare(2) | 系统调用的调用进程移动到               新的命名空间。如果调用的标志参数 指定列出的一个或多个`CLONE_NEW*`标志上面，然后为每个标志创建新的命名空间，并且调用进程成为这些命名空间的成员。（这个系统调用还实现了许多功能与命名空间无关。） | 调用进程移动到**新的**命名空间                               |
| ioctl(2)   | 发现有关命名空间的信息。                                     | 输出命名空间的信息                                           |

### 3. 项目中如何实现Namespace

通过`cmd.SysProcAttr`中的`Cloneflags`指定需要隔离的`flag`例如`syscall.CLONE_NEWUTS`,这样新创建的命令进程就会实现隔离

### 4. IPC Namespace

进程间通讯的方式：

* 随文件系统
* 随Kernel内核
* 随共享内存

`System V IPC`与`Posix IPC`是两种IPC的标准，后者在前者之上进行了改进，但是基本的概念都是差不多,`System V IPC`是`UNIX`系统上广泛使用的三种进程间通信机制的名称:**消息队列、信号量和共享内存**。是随内核持续的，直到内核被重启或者对象被显性关闭为止。

### 5. Mount Namespace

挂载点隔离，隔离各个进程看到的挂载点视图。

挂载：就是将Linux多个文件夹叠加到一个目标文件夹下，只要访问这个文件夹就可以看到挂载叠加的全部文件了

* Mount的传播机制

  `Mount NS`的一个**坑**：即使实现了隔离，但是在容器中挂载`/proc`文件夹会导致宿主机的`/proc`文件夹失去挂载

  因为Mount的传播机制，传播机制的出现是为了解决Mount NS实现中操作不方便的问题，例如宿主机有新的设备挂载需要让所有其他Mount NS都知道，如果一个个改的话会比较麻烦，所以就出现了传播机制

  主要有以下几个类型：

  * 共享挂载
  * 从属挂载
  * 共享/从属挂载
  * 私有挂载
  * 不可绑定挂载

  默认情况下是共享挂载，所以复制的子进程也是共享挂载，子进程中挂载会影响到父进程，所以需要设置为私有挂载`mount --make-rprivate mountpoint		`

  在实现中，我们直接将根目录设置为私有挂载，这样所有的文件挂载就不会影响到宿主机了

## 4. Cgroups-资源限制

### 1. 基本概念

主要同于限制容器的资源，也就是对进程以及子进程的资源限制，包括CPU、内存、存储和网络等

除了资源限制，Cgroups还提供了不同资源使用**优先级控制，审计**等功能

### 2. 三大组件：

* 控制组cgroup：控制组包含一组进程和带有参数的子系统的关联关系，在文件层面上理解就是一个子系统文件夹，其中包含了描述组内所有进程文件`tasks`还有一些资源限制文件
* 子系统：内核的组件，一个子系统代表一**类**资源控制，例如cpu、cpuacct、cpuset、devices、memory等
* 层级树：一系列控制组按照树状结构排列下来的关系表示，文件系统中就是上下层级目录的关系。主要的作用就是实现控制组子系统关系的继承

层级树与子系统一对多

### 3. 项目中如何实现Cgroups

项目通过在系统初始化就会创建的层级树目录下对应的资源限制子系统创建子文件夹，然后改动其中设置的值

### 4. cpu和cpuset

cpu用于对cpu使用率的划分；

cpuset用于设置cpu的亲和性等，主要用于numa架构的os；

cpuacct记录了cpu的部分信息。

对cpu资源的设置可以从2个维度考察：cpu使用百分比和cpu核数目。前者使用cpu subsystem进行配置，后者使用cpuset subsystem进行配置。

### 5. CgroupsV2与V1的不同

https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/

* 结构不同

  * 不同于 v1 版本， cgroup v2 版本只有一个层级 Hierarchy(层级).

  * cgroup2 文件系统有一个根 Cgroup ，以 `0x63677270`数字来标识，**所有支持v2版本的子系统控制器会自动绑定到 v2的唯一层级上并绑定到根 Cgroup.**

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220220211259940.png" alt="image-20220220211259940" style="zoom:50%;" />

    在V2 版本中，因为只有一个层级，所有进程只绑定到cgroup的叶子节点。

    - 父节点开启的子系统控制器控制到儿子节点，比如 A节点开启了memory controller，那么 C节点cgroup就可以控制进程的memory.
    - 叶子节点不能控制开启哪些子系统的controller,因为叶子节点关联进程Id.所以非叶子节点不能控制进程的使用资源。

  * v1 版本为了灵活一个进程可能绑定多个层级(Hierarchy)，但是通常是每个层级对应一个子系统，多层级就显得没有必要。所以**一个层级包含所有的子系统就比较简单容易管理**

* 线程模式

  * `Cgroup` v2 版本支持线程模式，将 `threaded` 写入到 cgroup.type 就会开启 Thread模式。**当开始线程模式后，一个进程的所有线程属于同一个cgroup**,会采用Tree结构进行管理。

## 5.Union File System 联合文件系统 - 文件存储引擎

### 1. 基本概念

联合文件系统就是将其他文件系统联合挂载的一个文件系统服务，它通过对目录层层累加到一个目标目录，然后用户使用时看起来像一个文件本身就有的内容一样的虚拟目录的作用（其实也就是挂载的作用）

在docker中联合文件系统是文件存储引擎，它可以让docker把镜像或容器做成分层的结构，多个容器共享一个镜像却不会影响镜像本身。

### 2. 写时复制/隐式共享

如果一个资源被重复使用的时候，如果在容器内没有修改它，那么就不会复制一个新的文件。当第一次修改/执行写操作的时候就会复制一个文件，然后在复制的文件中操作

对应到项目中docker就是会在容器的读写层复制一个修改的文件，因为容器的读取是从容器层往下读取，所以可以看到修改过的文件，但是底层的镜像文件是没有被修改的。在容器内删除一个文件的时候，会在读写层创建一个文件`.wh.file`来屏蔽底层文件的读，这样就相当于删除

### 3. 分类

|              |            适用系统            | 是否合并内核 |                           版本要求                           | 优点/特点                                                    | 缺点                                        |
| :----------- | :----------------------------: | :----------: | :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------- |
| AUFS         | 多用于 Ubuntu 和 Debian 系统中 |      否      |                >=3.1, 内核>=4.0请使用Overlay2                | Docker 最早使用的文件系统驱动，稳定                          | 代码可读性差（被拒绝合并到内核的原因）      |
| Devicemapper |       Red Hat 或 CentOS        |      是      |                      Linux 内核 >=2.6.9                      | 使用映射块设备的技术框架, 比文件系统效率高，稳定，docker默认的联合文件系统 | 只用于CentOS或Red hat系列                   |
| Overlay      |     Ubuntu等等大多数linux      |      是      |                       Linux内核>=3.18                        | AUFS的继承，合并到了Linux内核                                | 只工作在两层，并且运行时可能有一些意外的bug |
| Overlay2     |     Ubuntu等等大多数linux      |      是      | Docker >=17.06.02,  Red Hat 或 CentOS 内核>=3.10.0-514 其他linux发行版内核>=4.0 | overlay2在inode优化上更加高效，最高128层，已合并到内核，速度更快，主流 | 仍然年轻，不稳定，生产环境使用要慎重        |

### 4. AUFS

AUFS中每个目录都称为层，最终给用户呈现的就是普通的单层文件系统

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/rhWyCa.png" alt="rhWyCa" style="zoom: 33%;" />



目录作用：

* Diff: 存储创建的镜像层以及容器的init只读层与读写层
* Layer：存储镜像层关系的原数据（关系的表示通过镜像的ID）
* mnt：联合挂载目录，容器最终的运行目录
* 元数据和配置文件(主机配置、域名解析配置等)都在`/var/lib/docker/containers/<container-id>`下

### 5. Overlay2

类似于AUFS，也是将目录称为层，overlay2将目录的下一层作为`lowerdir`，上一层作为`upperdir`，联合挂载的结果叫做`merged`

目录结构：

* `l`目录：软连接，将一些较短的随机串连接到镜像层的diff文件夹下，为了避免参数过长
* 每个镜像层、容器都有一个单独的目录，其下包含了：
  * `diff`：类似于AUFS，镜像的改动按照层级逐层向上叠加，上一层只保存下一层的新的文件。容器的读写层也放在这里
  * `link`：记录了该镜像的短ID
  * `lower`: 所有父层的短ID，表示层级关系
  * `merge`：容器才有的联合挂载目录

### 6. Devicemapper

是一种**映射块设备**的技术框架，由linux内核提供

**关键技术：**

* 用户空间：负责虚拟设备与物理设备的逻辑连接关系
* 内核空间：将逻辑与物理设备的关联关系具体实现。例如I/O请求到达虚拟设备时内核空间负责处理过滤并最终转发到物理设备上

**工作原理：**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/UPeWpH.png" alt="UPeWpH" style="zoom: 33%;" />

* 映射设备：对外提供的逻辑设备，是由Devicemapper模拟的一个虚拟设备
* 目标设备：真实的物理设备或者一个物理设备的逻辑分段
* 映射表：记录了映射设备与目标的映射关系（映射设备在目标设备上的范围、类型等）

**瘦供给：**

是`DeviceMapper`中目标设备**动态分配**策略，每次按照需求分配需要的磁盘空间，避免资源的浪费

**docker中DeviceMapper的实现思路：**

docker使用了瘦供给的快照技术（快照就是数据在某一个时间点的存储状态）

数据的存储结构同样在`/var/lib/docker/devicemapper`目录下生成三个子目录：

* `devicemapper`: 存储镜像和容器的实际内容，由多个块设备构成
* `metadata`:存储元数据信息，记录了镜像层与容器层之间的关联关系
* `mnt`: 联合挂载目录也就是容器的启动目录

`Devicemapper`的分层依赖**快照**实现，每一镜像层都是其下一层的快照，最底层是瘦供给池，快照也同样按照写时复制的策略实现。`Devicemapper`工作在块级别所以效率更加高效

当需要写数据时就从底层瘦供给池中申请空间生成读写层，把数据复制到读写层修改；读取是同样的从上向下读取

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/lDzftz.png" alt="lDzftz" style="zoom:33%;" />

## 6. pivot_root

期望子进程在fork的时候拥有自己的根文件系统，就可以使用系统调用：

*pivot_root（new_root, put_old）*

作用：将原本的`/`存储到*put_old*目录，然后将子进程的`/`设置为*new_root*

有一些约束条件：

1. `new_root`和`put_old`都必须是目录
2. `new_root`和`put_old`不能与当前根目录在同一个挂载上。
3. `put_old`必须是`new_root`，或者是`new_root`的子目录
4. `new_root`必须是一个挂载点，但不能是"/"。还不是挂载点的路径可以通过绑定将路径挂载到自身上转换为挂载点。

使用流程：

1. 首先创建一个new_root的临时子目录作为put_old，然后调用pivot_root实现切换
2. chdir("/")
3. umount put_old and clear

**对比`chroot`、`switch_root`:**

* pivot-root是将整个根文件系统切换到一个新的root目录，所以可以解挂载之前的根文件目录。**完全与之前的根目录脱钩**
* chroot只是针对某一个进程，系统其他部分还是运行在老的root目录下
* pivot-root将当前Mount Namespace的所有进程改变为`/`
* chroot只是改变了当前进程的`/`
* switch_root和chroot类似，但是专门用于初始化系统的时候使用，不经chroot还会删除旧根目录下的所有内容，释放内存。其只能由pid=1的进程使用

## 7. 三个核心概念

* 镜像： 由只读文件和文件夹组合，包含了容器运行指定的基础文件、配置等
  * 构建镜像的两种方式：
    1. `docker commit 容器名称 新镜像名称`: 将当前容器打包成一个镜像
    2. `docker build -t 容器名:tag .`: 使用Dockfile构建

* 容器：容器是镜像运行的实体，在镜像文件的基础上创建上层容器层。
* 仓库：用来存储和分发Docker镜像。镜像仓库分为公有和私有

## 8. Docker架构理解

* docker整体架构师采用C/S模式，由客户端和服务端两部分组成。服务器提供一组REST API，客户端通过REST API于docker服务器进行交互，客户端于服务器通信的方式有多种，可以通过UNIX套接字通信也可以通过网络远程通信

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/3WiKW8.png" alt="3WiKW8" style="zoom:33%;" />

![WGTBZa](http://xwjpics.gumptlu.work/qinniu_uPic/WGTBZa.png)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/d74Hbm.png" alt="d74Hbm" style="zoom:33%;" />

## 9. 进入容器命令attach与exec的区别

* `docker attach`可以多个终端同步显示相同的内容，只能使用一个shell实例（共享一个sh进程）
* `docker exec`可以进入容器，会**单独**启动要给sh的进程，并且每个窗口都是互相不干扰的

## 10. docker的上下文是什么

docker通过C/S的模式使客户端与服务端交互，客户端通过API与docker引擎交互。

`docker build`命令构建镜像并不是在本地构建而是在服务端，也就是docker的后台引擎中。客户端会将指定路径(.)下的所有文件压缩为一个`tar`包发送给服务端，然后服务端解压读取其中的`Dockerfile`文件提取镜像、分层构建。

上下文就是客户端发送给服务端的`tar`文件，上下文路径就是(.)所在的路径。因为这个路径在本地，所以服务端无法看到上下文路径以外的任何内容，所以在Dockerfile中不能够通过`COPY`命令别的路径

## 11. -v和卷的区别

* `-v`：可以挂载宿主机的任意目录
* 数据卷：卷是docker对目录的封装，卷不仅支持本地目录，还支持NFS等网络目录

## 12. Dockerfile中`ADD`和`COPY`的区别

虽然 ADD 和 COPY 功能相似（都是拷贝上下文中的文件到容器），推荐 COPY 。

那是因为 COPY 比 ADD 更直观易懂。 COPY 只是将本地文件拷入容器这么简单，而 ADD 有一些其它特性功能(诸如，本地归档解压和支持远程网址访问等)，这些特性在指令本身体现并不明显。因此，有必要使用 ADD 指令的最好例子是**需要在本地自动解压归档文件到容器中的情况**，如 `ADD rootfs.tar.xz`

## 13. docker的运行状态

* 运行中(Running)
* 已暂停(Paused)
* 重启中(Restarting)
* 已退出(Exited) （退出并没有被删除）

## 14. Docker 群Swarm是什么？

Docker Swarm - Docker 群是原生的 Docker 集群服务工具。它将一群 Docker 主机集成为单一一个虚拟 Docker 主机。利用一个 Docker 守护进程， 通过标准的 Docker API 和任何完善的通讯工具，Docker Swarm 提供透明地将 Docker 主机扩散到多台主机上的服务。

## 15. Linux proc文件系统

`/proc`是由内核提供的，不是一个真正的文件系统，它只包含了系统运行时的信息（mount、ns、内存、硬件配置等）。它**以文件系统的形式为访问内核数据提供接口**

## 16. 容器网络

### 1. 网络虚拟化技术

随着虚拟化技术的发展，linux支持虚拟化的设备，可以通过虚拟化的设备组合实现多种多样的功能和网络拓扑

例如常见的虚拟化设备：`Veth`、`Bridge`、`802.1.q`、`VLAN device`、`TAP`等

### 2. veth\bridge\iptables以及一些网络命令

* Veth：

  linux 提供了`veth pair` 。可以把 **`veth pair` 当做是双向的 pipe（管道），从一个方向发送的网络数据，可以直接被另外一端接收到**；或者也可以想象成两个 namespace 直接通过一个特殊的虚拟网卡连接起来，可以直接通信。

* Bridge：

  多个Veth通信的时候，就可以通过Bridge实现，Bridge虚拟设备是用来桥接的网络设备，它相当于现实生活中的**交换机**，可以连接不同的网络设备，当请求到达Bridge设备时，可以通过报文中的Mac地址进行广播或转发。

* Iptables：

  **用来管理包的流动和传送定义了一套链式的处理结构**，在网络包传输的过程中使用不同的策略进行加工

  经常使用两种策略：

  * MASQUERADE

    能够让容器内对外部的请求变为宿主机的网卡的IP请求

    ```shell
    # 对Namespace中发出的包添加网络地址的转换
    $ sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/24 -o eth0 -j MASQUERADE
    ```

  * DNAT：

    将容器内部网络地址的端口映射到外部去

    ```shell
    $ sudo iptables -t nat -A PREROUTING -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.18.0.2:80
    ```

```shell
# ip netns
$ ip netns add ns1
$ ip netns exec ns1 ip addr
$ ip netns exec ns1 ip link set lo up

# 创建Veth
$ ip link add type veth
$ sudo ip link add veth0 type veth peer name veth1
$ sudo ip link set veth0 netns ns1			# 设置Veth NS
# default代表0.0.0.0/0，即在Net Namespace中所有流量都经过veth1的网络设备流出
$ sudo ip netns exec ns1 route add default dev veth0

# 创建一个网桥 br0
$ sudo brctl addbr br0
# 挂载网络设备
$ sudo brctl show
$ sudo brctl addif br0 veth0
```

## 17. docker网络通信机制 、docker底层有哪几种网络通信方式 

* Bridge网桥方式（默认）：网桥模式，在宿主机网卡与多个容器之间构建网桥让多个容器之间互通并且可以通过iptables设置让容器访问外网（默认有一个网桥`docker0`）
* Host主机方式(`--net="host"`)：可以看到主机上的所有网络设备，容器中也具有对这些设备的访问权限。所以这种方式是不安全的，如果在隔离良好的环境中影响不大（虚拟的宿主机中）
* None方式(`--net="none"`)：创建的容器没有网络，将网络创建的责任完全交给用户（可以使用Ip link配置）
* Container复用的方式(`--net="container:name or id"`)：复制一个特定容器的网络

## 18. 虚拟机的hypervisor与Guest OS

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/G9xHG4.png" alt="G9xHG4" style="zoom: 50%;" />

**hypervisor 虚拟机的管理系统:** 利用hypervisor可以在主操作系统中运行多个不同的从操作系统，并且其主要负责物理服务器的CPU、内存、硬盘网卡等硬件资源虚拟化的调度，多个虚拟机在hypervisor的协调下共享这些虚拟化的硬件资源，同时保持独立性。

hypervisor有两种类型：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/xlua3s.png" alt="xlua3s" style="zoom:67%;" />

* **Host OS**操作系统虚拟化：在宿主机已安装的操作系统上安装hypervisor，然后虚拟化出多个Guest OS运行应用程序，虚拟的操作系统就是Guest OS
* **Bare-metal**裸机虚拟化：直接在没有操作系统的逻辑上虚拟化其硬件资源，此时hypervisor就是一个仅对硬件资源虚拟和调度的薄操作系统

## 19. docker的一些操作

* 查看镜像支持的环境变量：`docker run IMAGE env`
* 如何退出一个容器的bash而不终止它：` Ctrl p` 或 `Ctrl q`
* 退出容器时自动删除：`docker run -rm -it ubuntu`

## 20. 打包镜像应该遵循哪些原则

整体上要让镜像尽量的精简：

* 尽量选择合适的基础镜像
* 提前清理一些缓存文件
* 确定依赖的版本号避免不必要的依赖，尽量使用库依赖
* 使用Dockerfile创建镜像要添加`.dockerignore`文件或者使用干净的工作目录

## 21.Stop与Kill的区别

* stop：直接向容器发送`SIGKILL`信号直接退出应用程序

* kill：优雅的退出，先发送`SIGTERM`信号然后10s后再发送`SIGKILL`信号，让容器的应用程序有时间保存状态处理当前请求等操作

## 22.Dockerfile的使用

* `FROM`: 指定基础镜像

  ```dockerfile
  FROM <image>
  FROM <image>:<tag>
  FROM <image>:<digest> 
  ```

* `RUN`: 构建时指定的命令, 每一个`RUN`就会构建一层

  ```dockerfile
  RUN <command>
  RUN ["executable", "param1", "param2"]
  ```

* `CMD`: 容器启动时运行的命令

  ```dockerfile
  CMD ["executable","param1","param2"]
  CMD ["param1","param2"]
  CMD command param1 param2
  ```

  > <font color='#39b54a'>RUN是构件容器时就运行的命令以及提交运行结果，CMD是容器启动时执行的命令，在构件时并不运行，构件时仅仅指定了这个命令到底是个什么样子</font>

* `LABEL`: 给镜像指定标签

  ```dockerfile
  LABEL <key>=<value> <key>=<value> <key>=<value> ...
  ```

* `MAINTAINER`：指定作者

  ```dockerfile
  MAINTAINER <name>
  ```

* `EXPOSE`：暴露容器的端口给外部，不会建立宿主机端口与容器端口的映射关系（需要用-v）

* `ENV`: 设置环境变量

  ```dockerfile
  ENV <key> <value>
  ENV <key>=<value> ...
  ```

* `ADD`:赋值上下文环境中的文件到容器目录中，可以是压缩文件、url会自动解压、解析

  ```dockerfile
  ADD <src>... <dest>
  ADD ["<src>",... "<dest>"]
  ```

* `COPY`: 只是本地文件的复制

  ```dockerfile
  COPY <src>... <dest>
  ```

* `ENTRYPOINT`: 启动时的命令

  ```dockerfile
  ENTRYPOINT ["executable", "param1", "param2"]
  ENTRYPOINT command param1 param2
  ```

  与`CMD`的不同点：

  * `ENTRYPOINT`不会被运行的命令覆盖但是CMD会
  * 如果同时写了`ENTRYPOINT`和CMD：
    * CMD不是一个完整的命令，则其作为`ENTRYPOINT`的参数
    * CMD是一个完整的命令，谁在最后谁生效

* `VOLUME`: 挂载数据卷

  ```dockerfile
  VOLUME ["/data"]
  ```

* `USER`: 指定容器用户

* `WORKDIR`: 设置工作目录，对`RUN,CMD,ENTRYPOINT,COPY,ADD`生效。如果不存在则会创建，也可以设置多次。

  ```dockerfile
  WORKDIR /path/to/workdir
  ```

* `ARG`：指定build时的一个参数

  ```dockerfile
  ARG <name>[=<default value>]
  ```

  设置变量命令，ARG命令定义了一个变量，在docker build创建镜像的时候，使用` --build-arg <varname>=<value>`来指定参数,如果用户在build镜像时指定了一个参数没有定义在Dockerfile种，那么将有一个Warning

## 23. 容器可以运行多个应用进程吗

一般一个容器就运行一个进程，不推荐运行多个进程。如果需要使用多个进程则可以通过一些进程管理工具管理

## 24. Docker默认的文件位置

* 默认配置文件位置

  Ubuntu 系统下 Docker 的配置文件是`/etc/default/docker`，CentOS 系统配置 文件存放在`/etc/sysconfig/docker`

* 默认存储位置

  `/var/lib/docker`

## 25. Docker与LXC有什么不同

`LXC(Linux Container)`利用`Linux`上相关的技术实现容器，Docker则在其上做了改进：

* 移植性：通过抽象容器配置，容器可以实现平台之间的移植
* 镜像系统：镜像系统可以使多个容器共享一个底层镜像，实现高效率的存储
* 版本管理：类似于git的版本管理机制
* 仓库系统：大大降低了镜像分发和管理的成本
* 周边工具/生态：基于docker的Paas、CI等系统、k8s等

## 26. Docker与Vagrant的区别

Vagrant 类似于 Boot2Docker(一款运行 Docker 的最小内核)，是**一套虚拟机的管理环境**，Vagrant 可以在多种系统上和虚拟机软件中运行，可以在 Windows、Mac 等非 Linux 平台上为 Docker 支持，自身具有较好的包装性和移植性。

<u>原生</u> Docker 自身只能运行在 Linux 平台上，但启动和运行的性能都比虚拟机要快， 往往更适合快速开发和部署应用的场景。

Docker 不是虚拟机，而是进程隔离，对于资源的消耗很少，单一开发环境下 Vagrant 是虚拟机上的封装，虚拟机本身会消耗资源。

## 27. xaaS

SaaS（软件即服务），PaaS（平台即服务）和IaaS（基础架构即服务）

Faas(function as a service) 函数即服务

## 28. 容器操作相关

* 退出容器时自动删除这个容器：添加`-rm`参数`docker run --rm -it <镜像名>`
* 查看镜像支持的env：`docker run <镜像名> env`
* 退出一个正在交互的容器终端而不终止它：核心就是不让容器的初始进程结束
  * `ctrl c`
    * 不指定`-t`, 在docker attach时设置`--sig-proxy=false` (默认为`true`)，SIGINT信号不会发送到容器中PID为1的进程，而是被docker attach进程响应
    * 指定`-t`, 此时`--sig-proxy=false`不起作用，`-i`时会将信号发送至初始进程导致退出，不`-i`时只会退出attach的进程
  * `ctrl p` + `ctrl q `(需要同时指定`-it`选项)
  * https://www.cnblogs.com/doujianli/p/10048707.html

## 29.  容器退出后，`docker ps`命令看不到，数据会被删除吗？

不会，当前容器处于exited状态，可以通过`docker ps -a`查看，只有将容器删除数据才会被删除

## 30. docker -i、-t、-d参数

| –detach      | -d   | 在后台运行容器，并且打印容器id。                             |
| ------------ | ---- | ------------------------------------------------------------ |
| –interactive | -i   | 即使没有连接，也要保持标准输入保持打开状态，一般与 -t 连用。 |
| –tty         | -t   | 分配一个伪tty，一般与 -i 连用。                              |

## 31.使用 docker port 命令映射容器的端口时，系统报错 Error: No public port ‘80’ published for ...，是什么意思?

创建镜像时 Dockerfile 要指定正确的 EXPOSE 的端口，容器启动时指定 `PublishAllport=true`

## 32. 了解runV吗？和runC的区别

https://help.aliyun.com/document_detail/160288.html

runV：安全沙箱

区别：runV通过轻量级的VM实现运行的每个容器都是完全隔离的，享受单独的内核，更加安全

![image-20220217172220302](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220217172220302.png)

# 二、runc项目

## 1. 开发过程中遇到问题

### 1. `Mount NS`

`Mount NS`的一个**坑**：即使实现了隔离，但是在容器中挂载`/proc`文件夹会导致宿主机的`/proc`文件夹失去挂载

需要在容器的init进程中实现Mount传播机制的手动改动：

```go
if err := syscall.Mount("/", "/", "", syscall.MS_REC | syscall.MS_PRIVATE, ""); err != nil {
  log.LogErrorFrom("setUpMount", "Mount proc", err)
}
```

### 2. goland编写代码飘红

因为我在mac上编写代码，但是需要在Linux代码编写环境，但是goland默认还是随本机，所以代码编写就会识别报错。

解决：修改goland编译器的编写设置，让`OS`还有架构变为linux和amd64并开启cgo

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211121150950491.png" alt="image-20211121150950491" style="zoom:50%;" />

### 3. `Cgroups`的配置注意事项：

配置cpuset的时候需要先配置`cpuset.mems`即给cpu node分配内存，因为我的机器只有一个node即node 0，所以填写0即可，然后再配置其他例如`cpuset.cpus`，否则会报错：no space left on device

### 4. bridge转发失败

需要注意的是，我们采用Bridge实现容器之间搭建网桥，但是需要我们事先**将宿主机的`iptables FORWARD` 链（转发链）默认策略设置为ACCEPT**，因为我们是在宿主机上本地路由转发，所以一定要设置允许通过才能够在容器之间ping通。

如果是`DROP`那么我们需要通过`iptables -P FORWARD ACCEPT`设置为允许。

## 2. 容器的运行怎么实现的

交互式和后台运行都是一套代码然后有部分的区别，两个参数不能同时设置

首先创建一个容器的前置工作：

* 内置了一个`init`命令用于一个容器的最初的初始化工作

* 首先通过cmd创建一个子命令进程`exec.command("/proc/self/exe", "init")`

  这个命令执行命令的内容就是调用自己，执行内置的`init`命令

* 然后先不执行子命令而是先配置：

  * **资源隔离：**配置它的一系列Namespace

  * **命令传输：**父进程命令的传输：设置了一个管道机制`os.Pipe()`，将读的一端文件句柄配置到这个命令，写的一端返回

  * **标准输出：**如果是`-t`交互式命令，那么就把这个命令的标准输出、输入、错误重新定向到当前；如果是`-d`后台运行就把所有的输出重定位写入到一个log文件中

  * **文件隔离/挂载：**在`/var/fs/aufs/`创建AUFS的文件空间(容器的读写层、镜像层/init只读层、挂载到联合挂载点)如果设置了`volume`还需要在这里挂载

     `cmd := exec.Command("mount", "-t", "aufs", "-o", dirs, "mnt_" + cId[:4], mntURL)`

  * **环境变量：**将宿主机的环境变量添加到此命令，继承宿主机的环境变量

    `cmd.Env = append(os.Environ(), EnvSlice...)`

* `exec.Start()`init子命令进程执行，但主进程不等待其执行结束

* `init`子命令在管道的读取端阻塞等待父进程写入命令，一旦写入命令：

  1. 设置根目录挂载模式为递归私有

     `syscall.Mount("/", "/", "", syscall.MS_REC | syscall.MS_PRIVATE, "")`

  2. `pivot_root:` 将指定的容器运行目录变为根目录

  3. 挂载`/proc`。这样容器里面看到的进程号就是只有自己的init了

  4. 通过`syscall.Exec()`系统调用将管道用户写入的进程命令执行起来并**覆盖掉**本来的`init`命令

* 主进程此时继续执行（此时父进程有子进程的PID`parent.Process.Pid`）：
  * 记录容器信息到文件(为了列表查看)
  * 发送运行附带的容器命令给管道
  * 如果容器有连接网络的需求的话执行连接网络
  * 根据启动的配置设置容器的资源隔离
  * 如果是交互式那么就等待容器的结束并关闭一些资源（AUFS文件、容器信息、资源限制等），如果是后台式的话主进程输出容器的唯一ID结束

## 3. 容器资源限制Cgroups的设计

设计了一个子系统的通用接口，然后分别实现了`cpu`、`cpuset`、`memory`去实现这个接口

```go
type Subsystem interface {
	Name() string                      // 返回子系统的名字
	Set(string, *ResourceConfig) error // 设置某个cgroup在这个子系统中的资源限制（设置子系统限制文件的内容）
	Apply(string, int) error           // 将进程添加到某个cgroup中
	Remove(string) error               // 移除某个cgroup
}
```

然后用一个包含了以上几种子系统的通用接口数组来表示一个设置cgroup的设置链

设置的时候遍历这个数组中的每个元素调用Set或Apply函数即可统一设置

## 4. 容器中管道/pivot-root实现细节

管道：进程之间**通信**就会使用管道的机制， 本质上，管道也是文件的一种，但是它和文件通信的区别在于管道有一个固定大小的缓冲区（一般是4KB）	`cmd.ExtraFiles = []*os.File{readPipe}`

pivot-root的实现细节：

* 为了满足约束: `new_root`必须是一个挂载点，所以先通过`mount bind`自己把自己设置为一个挂载点
* 然后在当前目录下创建一个临时目录`.pivot_root`保存之前的根目录也就是*put_old*
* 然后系统调用pivot-root设置容器当前的启动目录与根目录 `syscall.PivotRoot(root, pivotPath);`
* 修改容器的工作目录为根目录标识`/`, `syscall.Chdir("/")`
* 取消临时文件`.pivot_root`的挂载并删除它

## 5. 容器中AUFS/数据卷实现

AUFS文件名的创建采用每个容器的ID命名

* 根据镜像文件（`tar`）创建容器只读层
* 手动创建读写层
* 然后将这些目录都挂载到mnt文件夹，使之成为容器的启动目录

挂载的类型是`aufs`

![image-20211114155949850](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211114155949850.png)

数据卷实现步骤：

1. 根据数据卷的宿主机目录，创建宿主机文件目录（如果没有的话）
2. 根据数据卷的容器目录，在容器目录中创建挂载点目录
3. 将宿主机的文件目录挂载到容器挂载点

这样我们在进入容器之前就将这样的挂载设置好了，自然在容器启动后可以看到宿主机数据卷中的数据

## 6. 容器镜像打包的实现

在容器的mnt目录下使用tar命令打包成压缩包

## 7. 容器后台运行/查看容器ps的实现

docker的实现是通过containerd以及containerd-shim实现docker daemon和runc容器的解偶

我的项目没有实现那么复杂，借助于runc的`detach`功能实现：

* 核心思路：父进程的结束不需要等待子进程，让子进程变为孤儿进程由宿主机的1号进程托管，从而实现容器不宕机

查看容器的实现就是扫描创建容器时创建的容器json信息文件夹，然后格式化输出出来

## 8. 容器的进入容器exec实现

使用了cgo运行go语言调用c的函数与标准库调用`setns`将当前子命令进程进入到容器的`Namespace`中，因为内核中使用`setns`进入一个指定的`Mount Namespace`对于多线程的进程是不允许的，go每次启动都是多线程的状态，所以go没法做

go的c语言是在申明包之后的注释里面写的，最后加一个`import C`

c的函数前加入`__attribute__((constructor))` 类似于构造函数，用来表示这个函数在这个go包一被调用就调用这个c函数，在底下创建了一个空的go函数用来当作初始化这个包的引子

设置了两个环境变量`PID`和`CMD`， c函数判断当前宿主机是否有环境变量来是否执行`setns`

整体进入容器的思路如下：

1. 第一次调用`exec`此时环境变量`PID`和`CMD`为空，不执行c函数，创建一个子命令进程，命令是调用自己`exec`(`/proc/self/exe`), 在命令启动之前设置好这两个环境变量：`PID`为待进入容器的PID（从保存的容器信息文件中获取），`CMD`为执行`exec`附带的进入容器的命令，例如`sh`
2. 当子命令执行，第二次调用`exec`时，此时有环境变量执行C函数，在函数中实现将当前进程`setns`为容器的所有`Namespace`并执行环境变量的命令。（`setns(fd, 0)`）

这样就在容器中出现了一个新的命令进程，进入到了容器内

## 9. 容器停止的实现

1. 获取容器PID，从容器信息文件
2. 对该PID发送Kill信号（系统调用Kill发送`SIGTERM`信号） （`syscall.Kill(pid, syscall.SIGTERM)`）
3. 修改容器状态信息（`STOP`）
4. 重新写入存储容器信息的文件

## 10.容器删除的实现

本节实现的思路很简单就是删除掉一些文件：

* `/var/run/mydocker/`下的容器信息以及日志文件
* `/var/lib/mydocker/aufs/`目录下的对应容器的读写层目录以及`mnt`挂载目录（之前已经实现了函数，调用即可）

步骤：

1. 获取对应容器的信息，判断是否处于stopping状态
2. 删除容器信息、日志文件
3. 删除容器读写层、挂载目录

## 11. 容器指定环境变量运行实现

之前初始化容器的进程实现了复制父进程/宿主机的环境变量，我们在重新进入容器`exec`时也可以指定环境变量

实现的步骤：

1. 根据exec要进入的容器获取其当下的环境变量

   ```go
   path := fmt.Sprintf("/proc/%s/environ", pid)
   contentBytes, err := ioutil.ReadFile(path)
   ```

2. 然后在执行之前加入即可

   ```go
   // 将容器进程的环境变量都放到exec进程内
   cmd.Env = append(os.Environ(), getEnvsByPid(pid)...)
   ```

## 12. 容器网络怎样设计的

* **网络：** 容器的集合，同一个网络下的容器之间可以在此网络下互相通信。其中记录了网络的名称、网络驱动、和网段
* **网络端点：** 每个容器内的Veth虚拟设备的抽象封装，用于连接容器与网络，保证容器内部与网络的通信。包含veth设备、端口映射、IP、Mac地址等
* **网络驱动：**是项目中网桥的抽象接口，不同的驱动对网络的创建、连接、销毁的策略不同。主要包含了创建网络、删除网络、连接容器Veth网络端点到网络，移除容器的网络端点方法

* **IPAM：**使用位图算法分配和释放网络的IP
  * IPAM.Allocate(subnet *net.IPNet): 从指定的subnet网段中分配IP地址
  * IPAM.Release(subnet net.IPNet, padder net.IP): 从指定的subnet网段中释放指定的IP地址

## 13. 容器的Bridge网络怎样实现的

go的`netlink`包提供了一些操作网络接口、路由表等配置的工具

### 1. 创建bridge网络

容器网络的创建也是持久化将网络、网络驱动的信息保存到文件，如果需要列出当前网络也是扫描这些文件

网络操作首先都会将这些读取到内存

我们定义了bridge对象实现了网络驱动实例，**创建一个网络**方法的实现流程如下：

1. 创建一个网络Network对象

2. 初始化网桥：

   1. 创建Bridge虚拟设备 `&netlink.Bridge{ LinkAttrs: la }`

   2. 设置Bridge设备的地址和路由

      ```go
      // 通过netlink.AddrAdd给网络接口配置地址，等价于 ip addr add xxxx命令
      // 同时如果配置了地址所在的网段信息，例如192.168.0.0/24, 还会配置路由表192.168.0.0/24转发到这个bridge上
      addr := &netlink.Addr{
        IPNet:       ipNet,
        Label:       "",
        Flags:       0,
        Scope:       0,
      }
      return netlink.AddrAdd(iface, addr)
      ```

   3. 启动Bridge设备 ` netlink.LinkSetUp(iface)`

   4. 设置`iptables`的SNAT规则，让容器能够访问外网 `fmt.Sprintf("-t nat -A POSTROUTING -s %s ! -o %s -j MASQUERADE", subnet.String(), bridgeName)`

### 2. 容器连接网络

容器连接网络的实现过程如下：

1. 判断提供的网络名是否有效

2. 调用IPAM获取容器自己的一个可用IP

3. 创建网络端点veth

4. **调用网络驱动连接和配置网络端点**

   1. 创建Veth对，容器的一端命名为`cif-{endpoint ID前5位}`而另一端Bridge需要连接的一端设置为`{endpoint ID前5位}`

      ```go
      endpoint.Device = netlink.Veth{
        LinkAttrs: la,
        PeerName: "cif-" + endpoint.ID[:5],
      }
      ```

   2. 让Bridge连接一端（将Veth的一端挂载在网桥上）` netlink.LinkAdd(&endpoint.Device)`，并让Veth启动`netlink.LinkSetUp(&endpoint.Device)`

5. **进入容器配置容器网络设备的IP地址和路由**（需要进入网络的Namespace，通过访问`"/proc/%s/ns/net"`实现` netns.Set(netns.NsHandle(nsFD))`）

   设置路由：

   ```go
   _, cidr, _ := net.ParseCIDR("0.0.0.0/0")
   // 构建要添加的路由数据，包括网络设备，网关IP以及目的网段
   defaultRoute := &netlink.Route{
     LinkIndex: peerLink.Attrs().Index,
     Gw:        ep.Network.IpRange.IP,
     Dst:       cidr,
   }
   if err := netlink.RouteAdd(defaultRoute); err != nil {
     return err
   }
   ```

6. **配置端口映射, 暴露容器端口**

   ```go
   fmt.Sprintf("-t nat -A PREROUTING -p tcp -m tcp --dport %s -j DNAT --to-destination %s:%s", portMapping[0], ep.IpAddress.String(), portMapping[1])
   ```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211117163124138.png" alt="image-20211117163124138" style="zoom: 33%;" />

### 3. IPAM的位图算法

利用了**偏移量可以迅速定位数据所在的位置（ip地址）**, 然后分配信息

转换的算法：

* 偏移量-> ip地址：从网段的最后一段开始分别**除**0、8、16、24
* ip地址 -> 偏移量：从网段的最后一段开始分别**乘**0、8、16、24

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211117160056527.png" alt="image-20211117160056527" style="zoom:33%;" />

结构：

```go
type IPAM struct {
	SubnetAllocatorPath string             // 分配文件存放的位置
	Subnets             map[string]string // 网段和位图算法的数组map：key是网段，value是分配的位图字符串（使用字符串的一个字符标识一个状态位）
}
```

并存储在文件中

## 14. 项目还可以优化的地方

* 适配`Dockerfile`
* 多机环境下IPAM位图文件的一致性：存储到中心化的一致性KV-store
* 跨主机容器间的通信：
  * 封包：中间过程用宿主机封包变为宿主机间通信，到了另一端宿主机然后再解包发送给容器
  * 路由：在路由器上设置路由，让其知道从容器发出的包下一跳为另一个宿主机
* 实现多种Union FS（`Overlay2`）系统
