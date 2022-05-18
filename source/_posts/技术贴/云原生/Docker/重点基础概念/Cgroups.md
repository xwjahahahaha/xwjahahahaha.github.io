---
title: Cgroups
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-02-28 00:30:23
---

> * https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/

# 一、Cgroups的基本概念

## 1.1 基本功能

Linux Cgroups(Control Groups)提供了对一组进程以及将来子进程的资源限制，包括CPU、内存、存储与网络等

使用Cgroups可以方便的限制某个资源的占用并且可以实时的监控和统计信息

> Control Groups，通常被称为cgroups，是Linux内核的一个特性，它允许进程被组织成分层的组，这些组可以被限制和监视各种类型的资源的使用。**内核的cgroup接口是通过一个名为cgroupfs的伪文件系统提供的。分组是在核心cgroup内核代码中实现的，而资源跟踪和限制是在一组每个资源类型的子系统(内存、CPU等)中实现的。**

最初发布的cgroups是在linux 2.6.24中运行的，但是随后的不协调的更新导致了cgroups分为了两个版本维护：v1和v2 

v2发布在linux 3.10之后，v1成为了老版本，因为兼容性没有被移除

cgroups 主要提供了如下功能：

- 资源限制： 限制资源的使用量，例如我们可以通过限制某个业务的内存上限，从而保护主机其他业务的安全运行。
- 优先级控制：不同的组可以有不同的资源（ CPU 、磁盘 IO 等）使用优先级。
- 审计：计算控制组的资源使用情况。
- 控制：控制进程的挂起或恢复。

详细内容见：[cgroups(7) — Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html)

<!-- more -->

## 1.2 三大组件

Cgroups构成主要是三个组件：Cgroup、Subsystem、Hierarchy

### 控制组Cgroup

cgroup用于对进程进行分组管理的机制，一个cgroup包含一组进程，然后可以以cgroup为单位来进行子系统subsystem的各种参数配置。**表示一组进程和一组带有参数的子系统的关联关系**。例如，一个进程使用了 CPU 子系统来限制 CPU 的使用时间，则这个进程和 CPU 子系统的关联关系称为控制组。

查看一下当前系统已经挂载的cgroups信息：

```shell
$ mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup-test on /root/projects/golang_project/src/myDocker/cgroup-test type cgroup (rw,relatime,name=cgroup-test)
```

可以看到当前系统已经挂载了我们常用的cgroups子系统

### 子系统Subsystem

#### 子系统配置

subsystem是一组资源控制的模块，是一个内核的组件，一个子系统代表一类资源调度控制器。包含如下几项：

##### 1、CPU子系统

==cpu==

![image-20220325000617348](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220325000617348.png)

cpu的控制可以划分为两种策略：

* 完全公平的调度策略CFS

  * 按限额/使用上限

    * `cpu.cfs_period_us`：用于设定周期时间，必须与`cfs_quota_us`配合使用
    * `cpu.cfs_quota_us` ：用于设定周期内<font color='#e54d42'>最多</font>可使用的时间。这里的配置指任务对单个cpu的使用上限，`cfs_quota_us`是`cfs_period_us`的两倍，就表示在两个核上完全使用。数值范围为1000-1000,000（微秒）
    * `cpu.stat`：用于统计信息，包含`nr_periods`（表示经历了几个`cfs_period_us`周期）、`nr_throttled`（表示任务被限制的次数）及`throttled_time`（表示任务被限制的总时长）

  * 按比例分配

    * 使用`cpu.shares`设定一个整数（必须大于或等于2）表示相对权重，最后除以权重总和，算出相对比例，按比例分配CPU时间。

      > 例如，cgroup A设置为100，cgroup B设置为300，那么cgroup A中的任务运行25%的CPU时间。对于一个四核CPU的系统来说，cgroup A 中的任务可以100%占有某一个CPU，这个比例是相对整体的一个值

* 实时调度

  * 针对实时任务按周期分配固定的运行时间（以微秒为单位）(与CFS类似，也是在周期内分配一个固定运行时间)
  * `cpu.rt_period_us`：用于设定周期时间
  * `cpu.rt_runtime_us`：用于设定周期中的运行时间

==cpuacct==

这个子系统的配置是cpu子系统的补充，<font color='#e54d42'>提供CPU资源用量的统计</font>，时间单位都是纳秒（ns）

* `cpuacct.usage`：用于统计cgroup中所有任务的CPU使用时长
* `cpuacct.stat`：用于统计cgroup中所有任务的用户态和内核态分别使用CPU的时长
* `cpuacct.usage_percpu`：用于统计cgroup中所有任务使用每个CPU的时长

==cpuset==

主要用于<font color='#e54d42'>CPU绑定</font>，它为任务分配独立CPU资源的子系统，参数较多，这里只讲解两个必须配置的参数，目前在Docker中也只用到这两个参数。

* `cpuset.cpus`：在这个文件中填写cgroup可使用的CPU编号，如0-2, 16代表 0、1、2和16这4个CPU
* `cpuset.mems`：与CPU类似，表示cgroup可使用的memory node（NUMA架构），格式同上

##### 2、Memory子系统

* 限额类
  * `memory.limit_in_bytes`：表示强制限制最大内存使用量(硬限制)，单位有k、m、g这3种，填-1则代表无限制。
  * `memory.soft_limit_in_bytes`：表示软限制，只有比强制限制设置的值小时才有意义。填写格式同上。<font color='#e54d42'>当整体内存紧张的情况下，任务获取的内存就被限制在软限制额度之内</font>，以保证不会有太多任务因内存挨饿。可以看到，加入了内存的资源限制并不代表没有资源竞争
  * `memory.memsw.limit_in_bytes`：用于设定最大内存与swap区内存之和的用量限制，填写格式同上。
* 报警与自动控制
  * `memory.oom_control`，该参数填0或1， 0表示开启，当cgroup中的任务使用资源超过界限时立即杀死任务；1表示不启用。默认情况下，包含memory子系统的cgroup都启用。当`oom_control`不启用且实际使用内存超过界限时，任务会被暂停直到有空闲的内存资源。
* 统计与监控类
  * `memory.usage_in_bytes`：报告该cgroup中进程当前使用的总内存用量，单位为字
  * `memory.max_usage_in_bytes`：报告该cgroup中进程使用的最大内存用量
  * `memory.failcnt`：报告内存达到`memory.limit_in_bytes`设定的限制值的次数
  * `memory.stat`：包含大量的内存统计数据
    * `cache`：表示页缓存，包括 tmpfs（shmem），单位为字节
    * `rss`：表示匿名和swap缓存，不包括 tmpfs（shmem），单位为字节
    * `mapped_file`：表示memory-mapped映射的文件大小，包括 tmpfs（shmem），单位为字节
    * `pgpgin`：表示存入内存中的页数
    * `pgpgout`：表示从内存中读出的页数
    * `swap`：表示swap用量，单位为字节
    * `active_anon`：表示在活跃的最近最少使用的列表（least-recently-used，LRU）中的匿名和swap缓存，包括tmpfs（shmem），单位为字节
    * `inactive_anon`：表示不活跃的LRU列表中的匿名和swap缓存，包括tmpfs（shmem），单位为字节。
    * `active_file`：表示活跃的LRU列表中的file-backed内存，单位为字节。
    * `inactive_file`：表示不活跃的LRU列表中的file-backed内存，单位为字节。
    * `unevictable`：表示无法再生的内存，单位为字节。
    * `hierarchical_memory_limit`：包含memory cgroup层级的内存限制，单位为字节。
    * `hierarchical_memsw_limit`：包含memory cgroup层级的内存加 swap限制，单位为字节。

每个subsystem关联到相应的cgroup上并对其中的进程进行资源的限制与控制。这些subsystem都是逐步合并到内核中的,查看自己linux内核支持的subsystem：

```shell
# 安装cgroup命令工具
$ apt-get install cgroup-bin
# 查看
$ lssubsys
```

![mJmFWQ](http://xwjpics.gumptlu.work/qinniu_uPic/mJmFWQ.png)

### 层级树Hierarchy

层级树是**由一系列的控制组按照树状结构排列组成的。**

**cgroup是一组进程和其子系统关联的集合， hierarchy是一系列cgroup的集合**

通过这样的树结构，cgroups可以实现**继承**

例如：将多个进程设置为一组cgroup，并在其中设置了限制cpu的使用率。但是其中一个特殊的进程活动需要限制磁盘的I/O，如果单独设置就可能会影响其他同cgroup的进程。此时就可以将这个进程单独化为一个新的cgroup2，让其继承于cgroup1，这样单独对cgroup2设置限制就不会影响到cgroup1的其他进程。

### 三者关系

* 系统创建hierarchy之后，默认创建一个cgroup根节点，系统会将所有进程加入到此cgroup中
* 一个subsystem只能附加到一个hierarchy上（一对一的关系）
* 一个hierarchy可以附加多个subsystem（一对多的关系）
* 一个进程可以作为多个cgroup的成员，但是这些cgroup不能在一个hierarchy上（hierarchy与进程一对一）
* 一个进程fork出的子进程与父进程在同一个cgroup上，当然也可以通过操作移动

三者的关系如下图所示：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220228004307141.png" alt="image-20220228004307141" style="zoom:50%;" />

每个层级都会有一个根节点, 子节点是根节点的比重划分。

## 1.3 其他基础知识

### 1. NUMA架构

* RAM：易失性内存，关机没有电流后就会消失
* ROM：非易失性内存，持久存储在芯片上

NUMA 指的是针对某个 CPU，内存访问的距离和时间是不一样的。是为了解决多 CPU 系统下共享 BUS 带来的性能问题

如果多个CPU（不是单个多核CPU）共享一个BUS，那么BUS总线的压力过大影响性能：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220401211700325.png" alt="image-20220401211700325" style="zoom:50%;" />

NUMA架构的解决：

通过把临近的RAM当作一个Node，CPU会优先访问临近的node，同时CPU之间还有一个快速通道连接，所以每个CPU还是能够访问到各个RAM，只是距离带来的时间差异问题

在linux中查看：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220401212342915.png" alt="image-20220401212342915" style="zoom:50%;" />

### 2. VFS文件系统

VFS 是一个内核抽象层，能够**隐藏具体文件系统的实现细节，从而给用户态进程提供一套统一的API接口**。

VFS 使用了一种通用文件系统的设计，具体的文件系统只要实现了 VFS 的设计接口，就能够注册到 VFS 中，从而**使内核可以读写这种文件系统**。 这很像面向对象设计中的抽象类与子类之间的关系，抽象类负责对外接口的设计，子类负责具体的实现。其实，VFS本身就是用 c 语言实现的一套面向对象的接口。

# 二、操作与使用Cgroups

## 2.1 配置与操作Cgroups

kernel为了让cgroups的配置更加直观，通过一个虚拟的树状文件系统配置Cgroups，通过层级的目录虚拟出Cgroup树(hierarchy)。

配置案例：

### 1. 初始化

```shell
$ mkdir cgroup-test		# 创建挂载点
# 挂载一个hierarchy， -o表示添加可选项
$ sudo mount -t cgroup -o none,name=cgroup-test cgroup-test ./cgroup-test
# 查看挂载点信息
$ cat /proc/self/mountinfo
```

![ocpDDK](http://xwjpics.gumptlu.work/qinniu_uPic/ocpDDK.png)

可以看到刚刚设置的挂载

```shell
# 挂载后可以在目录下看到生成了许多默认文件
$ tree ./cgroup-test
```

默认文件以及其解释如下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Jng8oK.png" alt="Jng8oK" style="zoom:150%;" />

### 2. 创建子cgroup

创建两个子cgroup

```shell
$ sudo mkdir cgroup-1
$ sudo mkdir cgroup-2
$ tree
```

![P7lzMt](http://xwjpics.gumptlu.work/qinniu_uPic/P7lzMt.png)

可以看到在一个cgroup的目录下创建文件夹时，内核会把这个文件夹标记为子cgroup创建必要文件并**继承父cgroup的属性**

### 3. 移动进程

一个进程在一个hierarchy上只能存在于一个cgroup，移动进程cgroup修改task文件即可

```shell
$ echo $$
# 将我的终端进程移动到cgroup-1中
$ cd cgroup-1
$ sudo sh -c "echo $$ >> tasks"
# 查看
$ cat /proc/17795/cgroup 
```

![](http://xwjpics.gumptlu.work/qinniu_uPic/aJnSgh-20211111190813207.png)

### 4. subsystem限制进程资源

上面在创建层级树的之后并没有指定关联到任何的子系统，但是**系统默认给每个子系统创建了一个默认的层级树**（见控制组部分，使用`mount -t cgroup`可以查看到），例如memory的层级树，在子系统的根目录下的memory目录，这是一个层级树

```shell
# 查看
$ mount | grep memory
$ ls /sys/fs/cgroup/
```

下面就通过在这个层级树中创建cgroup限制如下进程占用的内存资源

```shell
$ cd /sys/fs/cgroup/memory/
# 在没有限制的情况下，启动一个占用内存的stress进程
$ apt install stress
$ stress --vm-bytes 200m --vm-keep -m 1
# 创建一个cgroup
$ sudo mkdir limit-memory-test && cd limit-memory-test
# 设置最大占用内存为100MB
$ sudo sh -c "echo  "100m" > memory.limit_in_bytes"
$ cat memory.limit_in_bytes
# 将当前进程移动到此group
$ sudo sh -c "echo $$ > tasks"
# 再次运行占用内存200MB的stress进程
$ stress --vm-bytes 200m --vm-keep -m 1
```

发现200m的压力测试启动失败，只有小于这个限制的stress才可以跑起来, 因为做了限制，当stress使用的内存大于限制后，cgroup就会将其进程杀死。

```shell
stress: info: [17728] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: FAIL: [17728] (415) <-- worker 17729 got signal 9
stress: WARN: [17728] (417) now reaping child worker processes
stress: FAIL: [17728] (451) failed run completed in 0s
```

```shell
# 删除创建的测试文件
$ rmdir limit-memory-test/
```

## 2.2 Docker如何使用Cgroups

通过实例查看一个docker容器怎样配置Cgroups

```shell
# 设置内存限制启动容器
$ sudo docker run -itd -m 128m ubuntu
```

docker会为每个容器在系统的hierarchy中创建cgroup

```shell
$ cd /sys/fs/cgroup/memory/docker/b496830a00213173ec4d43aee2ae01be6c524cda8e6f71e66e1defb90e10c32e
# 查看cgroup中进程使用的内存大小
$ cat memory.usage_in_bytes 
```

docker为每个容器创建cgroup并通过cgroup配置资源限制与监控

## 2.3 Go简单实现通过cgroup限制容器资源

```go
package main

import (
	"fmt"
	"io/ioutil"
	"os"
	"os/exec"
	"path"
	"strconv"
	"syscall"
)

// 挂载了memory subsystem的hierarchy的根目录位置
const cgroupMemoryHierarchyMount = "/sys/fs/cgroup/memory"

func main() {
	fmt.Printf("os.Args[0] is %s\n", os.Args[0])
	// /proc/self/exe 它代表当前程序
	if os.Args[0] == "/proc/self/exe" {
		// 容器进程执行函数
		fmt.Printf("current pid %d\n", syscall.Getegid())	// 获得自己Namespace下的pid
		cmd := exec.Command("sh", "-c", `stress --vm-bytes 60m --vm-keep -m 1`)
		cmd.SysProcAttr=&syscall.SysProcAttr{}
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
		if err := cmd.Run(); err != nil {
			fmt.Errorf("run err : %s \n", err.Error())
			os.Exit(1)
		}
	}
	// 执行当前程序
	cmd := exec.Command("/proc/self/exe")
	// fork一个子进程并设置Namespace
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	// start启动一个命令，但是不会等待其运行结束
	if err := cmd.Start(); err != nil {
		fmt.Errorf("start err : %s\n", err.Error())
		os.Exit(1)
	}
	// 得到fork出来的进程在外部命名空间的pid
	fmt.Printf("cmd pid is %v\n", cmd.Process.Pid)
	// 在系统默认创建挂载了memory subsystem的Hierarchy上创建新的cgroup
	os.Mkdir(path.Join(cgroupMemoryHierarchyMount, "testmemorylimit"), 0755)
	// 将容器进程(cmd进程)放到这个cgroup中(在task文件中写入进程id)
	ioutil.WriteFile(path.Join(cgroupMemoryHierarchyMount, "testmemorylimit", "tasks"), []byte(strconv.Itoa(cmd.Process.Pid)), 0644)
	// 限制cgroup进程使用
	ioutil.WriteFile(path.Join(cgroupMemoryHierarchyMount, "testmemorylimit", "memory.limit_in_bytes"), []byte("50m"), 0644)
	// 等待进程结束
	cmd.Process.Wait()
}

```

运行结果：

![cOIHnY](http://xwjpics.gumptlu.work/qinniu_uPic/cOIHnY.png)

`/proc/self/exe `它代表当前执行的程序，是一个软链接. `os.Args[0]`就是当前执行文件路径

运行`go run 2-cgroups.go `：

首先，当前执行路径是go编译器在tmp里面临时编译的可执行文件路径（见输出第一行），所以`if os.Args[0] == "/proc/self/exe" {...}`不会进入

而是使用cmd创建隔离Namespace环境再次执行当前程序（创建一个新的进程），因为采用的是`cmd.Start()`并不会等待此次执行结束（第二次执行当前程序），所以先输出了底下的`cmd pid is 31669`，然后一系列操作将这个新的进程设置了cgroup的内存限制为`50m`。

随后已经第二次执行也同步开始，符合`if`条件进入if体内，输出当前进程的`pid`为`0`（因为`PID Namespace`的隔离），值得注意的是这个进程与上面输出的`31669`进程是同一个进程，只是在外部`Namespace`中看此进程是`31669`,而在新的Namespace中其被隔离为`0`. 所以上面的进程限制就是设置在此进程。

随后在子进程中启动`stress`压力测试程序，内存占用要求为`60m`，因为收到了限制，所以无法启动，发生了报错。

可以尝试修改启动的限制，当在限制下启动是没有问题的。所以说明我们的内存限制起了作用。

# 三、cgroups v1与cgroups v2的区别

* 结构不同

  * 不同于 v1 版本， cgroup v2 版本只有一个层级树Hierarchy(层级).

  * cgroup2 文件系统有一个根 Cgroup ，以 `0x63677270`数字来标识，**所有支持v2版本的子系统控制器会自动绑定到 v2的唯一层级上并绑定到根 Cgroup.**

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220220211259940.png" alt="image-20220220211259940" style="zoom:50%;" />

    在V2 版本中，因为只有一个层级，所有进程只绑定到cgroup的叶子节点。

    - 父节点开启的子系统控制器控制到儿子节点，比如 A节点开启了memory controller，那么 C节点cgroup就可以控制进程的memory. （父节点不关联进程）
    - 叶子节点不能控制开启哪些子系统的controller,因为**叶子节点关联进程Id**.所以非叶子节点不能控制进程的使用资源。

  * v1 版本为了灵活一个进程可能绑定多个层级(Hierarchy)，但是通常是每个层级对应一个子系统，多层级就显得没有必要。所以**一个层级包含所有的子系统就比较简单容易管理**

* 线程模式

  * `Cgroup` v2 版本支持线程模式，将 `threaded` 写入到 cgroup.type 就会开启 Thread模式。**当开始线程模式后，一个进程的所有线程属于同一个cgroup**,会采用Tree结构进行管理。

# 四、进程与cgroups的关系

一个进程限制内存和CPU资源，就会绑定到CPU Cgroup和Memory Cgroup的节点上，Cpu cgroup 节点和Memory cgroup节点属于两个不同的Hierarchy层级（v1版本）

进程和 cgroup 节点是多对多的关系，因为一个进程涉及多个子系统，每个子系统可能属于不同的层次结构(Hierarchy)。

<img src="https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/%E8%BF%9B%E7%A8%8B%E5%92%8Ccgroup%E7%9A%84%E5%85%B3%E7%B3%BB.png" alt="img" style="zoom:50%;" />

上图 P 代表进程，因为多个进程可能共享相同的资源，所以会抽象出一个 `CSS_SET`, **每个进程会属于一个CSS_SET 链表中，同一个 `CSS_SET` 下的进程都被其管理。**一个 `CSS_SET` 关联多个 Cgroup节点，也就是关联多个子系统的资源控制，那么 `CSS_SET`和 `Cgroup`节点就是多对多的关系。

参考下进程结构中指向的 `CSS_SET` 结构定义:

```c++
struct task_struct { //进程的数据结构
...
#ifdef CONFIG_CGROUPS
	/* Control Group info protected by css_set_lock */
	struct css_set __rcu *cgroups; 	//关联的cgroup 节点
	/* cg_list protected by css_set_lXock and tsk->alloc_lock */
	struct list_head cg_list; 			// 关联所有的进程的链表头节点
#endif
...
}
```

# 五、Cgroups是如何实现的

## 5.1 数据结构

1. 每个进程都会指向一个`CSS_SET`数据结构（如上面进程结构代码所示）
2. 一个 `CSS_SET` 关联多个 `cgroup_subsys_state` 对象，`cgroup_subsys_state` 指向一个 cgroup 子系统。**所以进程和 cgroup 是不直接关联的，需要通过 `cgroup_subsys_state` 对象确定属于哪个层级，属于哪个 `Cgroup` 节点。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/css_set.png" alt="img" style="zoom:50%;" />

3. <font color='#e54d42'>**一个 `Cgroup Hierarchy(`层次）其实是一个文件系统, 可以挂载在用户空间查看和操作**</font>
4. 我们可以查看绑定任何一个cgroup节点下的所有进程Id(PID) （查看对应的文件系统）
   - 实现原理: 通过进程的fork和退出，从 `CSS_SET` attach 或者 detach 进程

## 5.2 用户空间的文件系统

Cgroup 通过 `VFS `文件系统将功能暴露给用户，**用户创建一些文件，写入一些参数即可使用**，那么用户使用Crgoup功能会创建哪些文件？

文件如下:

- `tasks` 文件: 列举绑定到某个 cgroup的 所有进程ID（PID）.
- `cgroup.procs` 文件: 列举 一个Cgroup节点下的所有线程组Id.
- `notify_on_release` flag 文件: ：填 0 或 1，表示是否在 cgroup 中最后一个 task 退出时通知运行release agent，默认情况下是 0，表示不运行。
- `release_agent` 文件： 指定 release agent 执行脚本的文件路径（该文件在最顶层 cgroup 目录中存在），在这个脚本通常用于自动化umount无用的 cgroup
- `clone_children flag`文件: 这个标志只会影响 cpuset 子系统，如果这个标志在 cgroup 中开启，一个新的 cpuset 子系统 cgroup节点 的配置会继承它的父级cgroup节点配置。
- 每个子系统也会创建一些特有的文件。

# 六、cgroups作用范围

对于系统进程调度的方式中，cgrous只适用于普通调度，而不适用于实时调度

* 普通调度：关注周转时间
* 实时调度：关注响应时间，实时性要求非常高

适用的进程调度算法是：CFS（Completely Fair [Scheduler](https://so.csdn.net/so/search?q=Scheduler&spm=1001.2101.3001.7020)的缩写，即完全公平调度器，负责进程调度）

https://blog.csdn.net/XD_hebuters/article/details/79623130

现在一般的linux都是CFS的进程调度算法，其他较老的算法，cgroups也不会生效

* FIFO：先进先出
* RR: 时间片轮转
* CFS：公平调度

# 七、cgroups的软限制和硬限制

可参考内存分配的软硬限制:

* **硬限制**
  进程可以使用的CPU资源最多只能是配置的参数，超过参数就会触发分组的OOM-kill杀死进程(如果设置`oom_controller`的话)
* **软限制**
  在系统资源闲的情况下，进程使用的CPU资源可以超过配置的参数，但是当系统整体资源紧张时，就会触发资源隔离机制，将进程可以使用的资源限制在软限制配置的参数内部

# 八、CFS完全公平调度策略

https://blog.csdn.net/XD_hebuters/article/details/79623130

CFS是英文`Completely Fair Scheduler`的缩写，即完全公平调度器，负责进程调度。在`Linux Kernel 2.6.23`之后采用，它负责将CPU资源，分配给正在执行的进程，目标在于最大化程式互动效能，最小化整体CPU的运用。使用**红黑树**来实现，算法效率为`O(log(n))`

cfs定义一种新的模型，它给cfs_rq（cfs的run queue）中的每一个进程安排一个虚拟时钟，`vruntime`。如果一个进程得以执行，随着时间的增长（即一个个tick的到来），其`vruntime`将不断增大。没有得到执行的进程`vruntime`不变。**调度器总是选择`vruntime`值最低的进程执行**。这就是所谓的“完全公平”。对于不同进程，优先级高的进程vruntime增长慢，以至于它能得到更多的运行时间。

用红黑树的架构保存多个调度实体，每个调度实体就对应一个进程，通过此结构快速找到`vruntime`最低的进程执行

