---
title: Namespace
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 15:42:24
---

# 一、 基础介绍

Linux实现隔离的方式： [namespaces(7)](http://link.zhihu.com/?target=http%3A//man7.org/linux/man-pages/man7/namespaces.7.html)

命名空间将全局系统资源包装在一个抽象中 使名称空间内的进程看起来拥有自己的全局资源的独立实例。全局资源的修改对同一个命名空间下的其他进程是可见的，对于其他命名空间是不可见的。命名空间的一个用途是实现容器。

Linux Namespace各种命名空间：

<!-- more -->

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/ItNpxb.png" alt="ItNpxb" style="zoom:50%;" />

最新的 Linux 5.6 内核中提供了 8 种类型的 Namespace：

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

虽然 Linux 内核提供了8种 Namespace，但是最新版本的 Docker 只使用了其中的前6 种，分别为Mount Namespace、PID Namespace、Net Namespace、IPC Namespace、UTS Namespace、User Namespace。

第二列显示了用于在各种api中指定名称空间类型的标志值（系统调用参数），第三列标识了手册页中关于命名空间的详细信息，最后一列标识命名空间对应隔离的资源

Namespace能够实现在同一台主机下UID级别的隔离，给每一个UID的用户虚拟化出一个Namespace，这样多个用户之间可以访问系统资源的同时互相之间还实现了隔离。

从每个用户的角度看，每个命名空间都像一台单独的linux一样，有自己的init进程，并且pid不断递增

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211028230603400.png" alt="image-20211028230603400" style="zoom:67%;" />

例如上图：

用户A在命名空间A中认为自己的1号进程就是init进程，同理B如此。但是实际上都是分别映射到主进程3，4进程，从host的角度看只是3、4号进程虚拟化出来的一个空间而已。

## 查看进程所有命名空间

查看当前进程的所有命名空间：`ls -l /proc/$$/ns | awk '{print $1, $9, $10, $11}'`

```shell
total   
lrwxrwxrwx cgroup -> cgroup:[4026531835]
lrwxrwxrwx ipc -> ipc:[4026531839]
lrwxrwxrwx mnt -> mnt:[4026531840]
lrwxrwxrwx net -> net:[4026531993]
lrwxrwxrwx pid -> pid:[4026531836]
lrwxrwxrwx pid_for_children -> pid:[4026531836]
lrwxrwxrwx user -> user:[4026531837]
lrwxrwxrwx uts -> uts:[4026532271]
```

## 单独查看命名空间

例如查看UTS： `readlink /proc/$$/ns/uts`

```shell
# readlink /proc/$$/ns/uts
uts:[4026532271]
```

# 二、命名空间API

目前Namespace的API的系统调用：

| API        | 官方解释                                                     | 简单解释                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| clone(2)   | 系统调用创建一个新进程。如果调用的flags参数指定了上面列出的一个或多个`CLONE_NEW*`标志，则新的命名空间是为每个标志创建，并且子进程被创建为               这些命名空间的成员。 | 创建一个新进程，可以通过`CLONE_NEW*`参数指定哪些命名空间被创建，并且他们的子进程也会被包含在这些Namespace中 |
| setns(2)   | 系统调用允许调用进程加入现有的命名空间。                     | 当前线程切到指定的命名空间                                   |
| unshare(2) | 系统调用的调用进程移动到               新的命名空间。如果调用的标志参数 指定列出的一个或多个`CLONE_NEW*`标志上面，然后为每个标志创建新的命名空间，并且调用进程成为这些命名空间的成员。（这个系统调用还实现了许多功能与命名空间无关。） | 调用进程移动到新的命名空间                                   |
| ioctl(2)   | 发现有关命名空间的信息。                                     | 输出命名空间的信息                                           |

# 三、UTS Namespace

主要用于隔离Hostname、Domainname系统标识（主机与域名）

go实现UTS命名空间的调用

```go
// +build linux
package main

import (
	"log"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	cmd := exec.Command("sh")		// 被复制出来的新进程的初始命令，我们使用sh执行
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,			// 使用CLONE标志创建一个UTS Namespace
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

这段代码执行后我们会进入一个`sh`运行环境中，在这个环境中我们就实现了一个新的UTS Namespace

验证效果：

* 查看当前宿主机进程之间的关系：`pstree -pl` (我的运行文件为test)

  ![image-20211029165014390](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211029165014390.png)

* 查看当前`sh`的pid: `echo $$`

  ```shell
  $ echo $$  
  6372
  ```

* 验证父进程与子进程是否在同一个UTS Namespace: `/proc/xxx/ns/uts` （`xxx`是进程pid）

  ```shell
  $ readlink /proc/6369/ns/uts				// 父进程就是go可执行文件test
  uts:[4026531838]
  $ readlink /proc/6372/ns/uts                    
  uts:[4026532271]
  ```

* 测试修改Hostname（正常情况下修改此环境内的hostname是不会影响外部主机的）

  ```shell
  $ hostname -b bird				// 修改主机名为bird
  $ hostname
  bird
  ```

  在另一个终端中查看宿主机的`hostname`是否改变

  ```shell
  $ hostname
  wenjie
  ```

  并没有因此改变，所以实现了主机名隔离

# 四、IPC Namespace

IPC Namespace用来隔离`System V IPC` 和`POSIX message queues`，每一个IPC Namespace都拥有自己唯一的`System V IPC` 和`POSIX message queues`

## 1. 进程间通讯IPC基本概念

IPC(Inter-Process Communication)进程间通信有三种信息共享方式：1.随文件系统 2.随kernel内核 3.随共享内存

  相对的IPC的持续性（Persistence of IPC Object）也有三种：

1. **随进程持续的（Process-Persistent IPC）**

   IPC对象一直存在，直到最后拥有他的进程被关闭为止，典型的IPC有pipes（管道）和FIFOs（先进先出对象） 

2. **随内核持续的（Kernel-persistent IPC）**

   IPC对象一直存在直到内核被重启或者对象被显式关闭为止，在Unix中这种对象有***System v 消息队列，信号量，共享内存*。**（注意***Posix消息队列，信号量和共享内存***被要求为至少是内核持续的，但是也有可能是文件持续的，这样看系统的具体实现）。

3. **随文件系统持续的（FileSystem-persistent IPC）**

   除非IPC对象被显式删除，否则IPC对象会一直保持（即使内核才重启了也是会留着的）。如果Posix消息队列，信号量，和共享内存都是用内存映射文件的方法，那么这些IPC都有着这样的属性。

不同的Unix IPC的持续性：

1. **随进程**

   `Pipe, FIFO, Posix的mutex（互斥锁）, condition variable（条件变量）, read-write lock（读写锁）,memory-based semaphore（基于内存的信号量） 以及 fcntl record lock，TCP和UDP套接字，Unix domain socket`

2. **随内核**

   `Posix的message queue（消息队列）, named semaphore（命名信号量）, System V Message queue, semaphore, shared memory。`

要注意的是，虽然上面所列的IPC并没有随文件系统的，但是我们就像我们刚才所说的那样，Posix IPC可能会跟着系统具体实现而不同（具有不同的持续性），举个例子，写入文件肯定是一个文件系统持续性的操作，但是通常来说IPC不会这样实现。很少有IPC会实现文件系统持续，因为这会降低性能，不符合IPC的设计初衷。

**System V IPC与Posix IPC是两种IPC的标准，后者在前者之上进行了改进，但是基本的概念都是差不多**

具体的差别可见：[系统V IPC与POSIX IPC](https://blog.csdn.net/guiwin/article/details/82782151)

System V IPC是UNIX系统上广泛使用的三种进程间通信机制的名称:消息队列、信号量和共享内存。是随内核持续的，直到内核被重启或者对象被显性关闭为止。

1. System V 消息队列：

   System V 消息队列允许**数据以称为消息的单位交换**，每个消息都可以有一个关联的优先级。POSIX消息队列提供了实现相同结果的替代API

2. System V 信号量：

   System V信号量允许**进程同步它们的动作**。系统V的信号量被分配到称为集合的组中;集合中的每个信号量都是一个计数信号量。POSIX信号量提供了实现相同结果的替代API。

3. System V 共享内存:

   系统V共享内存允许**进程共享一个区域一个内存**(一个scegment)。POSIX共享内存是实现相同结果的另一种API

## 2. 实践

实践其实很简单，在刚刚的程序上增加一个flag即可：

```go
Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC,
```

再次编译启动：

查看现有的ipc消息队列: `ipcs -q`

```shell
$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
```

创建一个消息队列

```shell
$ ipcmk -Q							// 创建一个消息队列
Message queue id: 0
$ ipcs -q								// 查看

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xcc4f9f77 0          root       644        0            0         
```

使用另一个终端查看消息队列：

```shell
$ ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
```

无法查看到，说明实现了消息队列的隔离

# 五、PID Namespace

pid Namespace就是用于隔离进程ID的，同样一个进程在不同的PID Namespace可以拥有不同的进程ID

同样的修改一下代码中的flag：

```go
Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID,
```

编译启动后，首先查看自己真实的PID

![GHqSyk](http://xwjpics.gumptlu.work/qinniu_uPic/GHqSyk.png)

当前的Pid是10166

然后查看自己当前隔离之后的ID:

```shell
$ echo $$
1
```

（注意，这里不能使用ps、top等命令查看，因为会调用/proc内容，后面会解决这个问题）

# 六、Mount Namespace

[mount_namespaces(7)](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)

负责隔离各个进程看到的挂载点视图，让每一个进程看到的文件系统层次是不同的。这也是Linux第一个实现的Namespace类型，所以注意他的flag是`CLONE_NEWNS`（new Namespace的缩写）

## 1. Mount命令

`Mount`可以将分区挂接到Linux的一个文件夹下，从而将分区和该目录联系起来，因此我们只要访问这个文件夹，就相当于访问该分区了。

在挂载了`/proc`的进程下可以使用`Mount`命令

`mount`命令主要用于挂载linux中的文件系统等操作

关于此命令的详细使用可以查看：[mount(8) — Linux manual page](https://man7.org/linux/man-pages/man8/mount.8.html) 

这里列出常用的几个并解释：

* `mount` : 输出当前所有的设备、挂载目录以及类型

* `mount -t type device dir`
  **device**：指定要挂载的设备，比如磁盘、光驱等。
  **dir**：指定把文件系统挂载到哪个目录。
  **type**：指定挂载的文件系统类型，一般不用指定，mount 命令能够自行判断。
  **options**：指定挂载参数，比如 ro 表示以只读方式挂载文件系统。

  告诉内核将在device上(类型为type)上找到的文件系统附加到目录dir。选项`-t type`是可选的, 不写的话通常会自动检测。 

* `mount --bind olddir newdir `

  将文件系统层次结构的一部分重新挂载到其他地方，调用之后相同的内容可以在两个地方访问。重要的是要理解“bind”不会在内核VFS中创建任何二类或特殊的节点。“绑定”只是附加文件系统的另一个操作。

### shared subtree

Mount有一个重要的地方，就是它的**shared subtree**机制：

该机制的出现主要是为了解决Mount Namespace操作不方便的问题：当宿主机有新的磁盘挂载后，我们希望能够通知所有的其他Namespace挂载这个磁盘，但是如果是完全隔离的状态，那么是需要每个都手动操作，非常麻烦。所以就出现了shared subtree机制，这个机制主要有两个关键点：

1. peer group

   表示一个或者多个挂载点的集合，有下面两种情况会分到一个集合（group）：

   * 通过`--bind`操作挂载的源挂载点（必须是一个挂载点）与目的挂载点
   * 新生成mount ns复制过去的挂载点在同一个集合

2. **propagate type 传播属性**

   上述的问题依靠这个属性解决。传播属性重新定义了挂载对象/点之间的关系，定义的关系包括：

   * 共享关系： 一个挂载对象的挂载事件会传播到另一个挂载对象，反之亦然 （双向）
   * 从属关系：一个挂载对象的挂载事件会传播给另一个，反之不会 （单向）

   一个挂载的状态可以有以下几种：

   ![8T4RPY](http://xwjpics.gumptlu.work/qinniu_uPic/8T4RPY.png)

   ![image-20211030233301325](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211030233301325.png)

   ​	默认情况下，所有挂载状态都是私有的，改变状态的方法如下所示：

```shell
mount --make-shared mountpoint					// 共享
mount --make-slave mountpoint						// 从属
mount --make-private mountpoint					// 私有
mount --make-unbindable mountpoint			// 设置不可被绑定
// 有r前缀的表示递归的修改挂载点以及其子目录
mount --make-rshared mountpoint
mount --make-rslave mountpoint
mount --make-rprivate mountpoint
mount --make-runbindable mountpoint
```

## 2. 实践

修改代码:

```go
Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
```

编译启动：

查看一下`/proc`的文件内容

> `/proc`是一个文件系统，可以通过特殊的机制将内核和内核信息发送给进程

```shell
$ ls /proc
1      19510  21767  21800  21833  21866  21900  21942  21975  22006  22039  22072  22105  22138  22172  22206  24     5299       driver        softirqs
10     19570  21768  21801  21834  21867  21901  21943  21976  22007  22040  22073  22106  22139  22173  22207  24819  5300       execdomains   stat
10143  19615  21769  21802  21835  21868  21902  21944  21977  22008  22041  22074  22107  22140  22174  22208  249    584        fb            swaps
11     2      21770  21803  21836  21869  21903  21945  21978  22009  22042  22075  22108  22141  22175  22209  25     589        filesystems   sys
115    20     21771  21804  21837  21870  21904  21946  21979  22010  22043  22076  22109  22142  22176  22210  25538  6          fs            sysrq-trigger
12     20444  21772  21805  21838  21871  21905  21947  21980  22011  22044  22077  22110  22143  22177  22211  258    6021       interrupts    sysvipc
13     20474  21773  21806  21839  21872  21906  21948  21981  22012  22045  22078  22111  22144  22178  22212  26     672        iomem         thread-self
....
```

这是宿主机的`/proc`，我们还需要手动的将`/proc` mount(挂载)到我们自己的Namespace下

使用命令`mount -t proc proc /proc`将宿主机的`proc`文件系统挂载到自己的Namespace下的`/proc`目录上

再次查看`/proc`文件系统：

```shell
$ ls /proc
1          cgroups   devices      fb           ioports   key-users    loadavg  modules       partitions   slabinfo  sysrq-trigger  uptime             zoneinfo
5          cmdline   diskstats    filesystems  irq       kmsg         locks    mounts        sched_debug  softirqs  sysvipc        version
acpi       consoles  dma          fs           kallsyms  kpagecgroup  mdstat   mtrr          schedstat    stat      thread-self    version_signature
buddyinfo  cpuinfo   driver       interrupts   kcore     kpagecount   meminfo  net           scsi         swaps     timer_list     vmallocinfo
bus        crypto    execdomains  iomem        keys      kpageflags   misc     pagetypeinfo  self         sys       tty            vmstat
```

相比宿主机的`/proc`已经少了很多内容

使用`ps -ef`查看进程：

```shell
$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 22:21 pts/1    00:00:00 sh
root         8     1  0 22:34 pts/1    00:00:00 ps -ef
```

可以看到这时候就只能看到自己Namespace下的进程了

我们回到宿主机，发现宿主机的`/proc`不能使用了，这就是因为宿主机的Namespace文件中本来对挂载点`/proc`设置的就是共享挂载，所以我们使用clone复制的时候，新的Mount Namespace也是共享挂载，两者是共享和传播的方式。

回到宿主机通过查看`cat /proc/self/mountinfo`当前宿主机NS的挂载点信息：

![image-20211031092033846](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211031092033846.png)

`/proc`等系统的挂载点都是`shared`的状态

为了让新建的Mount Namespace不影响其他的Mount Namespace，所以我们需要设置为私有挂载模式：

一样的编译启动新的Mount Namespace，但是这次挂载的操作不同：

```shell
# 将/proc目录设置为私有挂载
$ mount --make-rprivate /proc
# 挂载/proc
$ mount -t proc proc /proc
# 查看
$ ls /proc
```

![u6ByLy](http://xwjpics.gumptlu.work/qinniu_uPic/u6ByLy.png)

通过运行可以看到在新的NS中我们实现了隔离

现在返回原来的NS，直接查看`/proc`看看是否受到影响

![x0UWFs](http://xwjpics.gumptlu.work/qinniu_uPic/x0UWFs.png)

可以看到并没有收到影响。我们不需要再重新挂载了，实现了隔离。

mount实现了和外部空间的隔离，在本Namespace下挂载的文件系统不会影响外部，<font color='#e54d42'>**所以这也是docker数据卷的特性原因之一**</font>

# 七、User Namespace

User Namespace主要隔离用户的用户组ID。比较常见的做法是将宿主机上的一个非root用户在新建的User Namespace中映射成一个root用户，这意味着这个进程在User Namespace内部是有root权限的。

**在Linux Kernel 3.8开始非root进程也可以创建User Namespace了，并且实现了在新创的User Namespace中拥有root权限**

修改代码如下：

```go
func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS | syscall.CLONE_NEWUSER,
	}
	// 设置凭证
	cmd.SysProcAttr.Credential = &syscall.Credential{
		Uid: uint32(1),
		Gid: uint32(1),
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

首先查看当前宿主机的用户与用户组: `id`

```shell
$ id
uid=0(root) gid=0(root) groups=0(root)
```

接下来运行程序再同样执行：

> 报错：`fork/exec /bin/sh: operation not permitted`
>
> 原因：https://github.com/xianlubird/mydocker/issues/3
>
> Linux kernel 在 3.19 以上的版本中对 `user namespace `做了些修改
>
> 解决：删除掉`cmd.SysProcAttr.Credential`
>
> *注意：centos不支持user namspace需要额外的设置，见原因链接*

再次运行：

```shell
$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

可以看到UID已经不同了，因此User Namespace生效了

# 八、Network Namespace

用于隔离网络设备、IP地址端口等网络栈的Namespace。可以让每个容器拥有自己独立的（虚拟的）网络设备，并且容器可以绑定到自己端口，每个Namespace内的端口都不会互相冲突。

在宿主机上搭建网桥后就可以很方便的实现容器之间的通信，并且不同的容器可以使用相同的端口！

同样的添加flag:

```go
Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS | syscall.CLONE_NEWUSER | syscall.CLONE_NEWNET,
```

首先查看宿主机的网路设备情况：

```shell
$ ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 172.18.255.255
        ether 02:42:df:81:27:f6  txqueuelen 0  (Ethernet)
        RX packets 585  bytes 140849 (140.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 611  bytes 987099 (987.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.59.2  netmask 255.255.192.0  broadcast 172.17.63.255
        ether 00:16:3e:0e:2d:b8  txqueuelen 1000  (Ethernet)
        RX packets 1168249932  bytes 1131213773283 (1.1 TB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 683891978  bytes 296842597629 (296.8 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 67191390  bytes 8889749352 (8.8 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 67191390  bytes 8889749352 (8.8 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

运行程序，在新的Network Namespace中查看网络：

运行的结果是：啥也没有。所以已经处于隔离状态了。

