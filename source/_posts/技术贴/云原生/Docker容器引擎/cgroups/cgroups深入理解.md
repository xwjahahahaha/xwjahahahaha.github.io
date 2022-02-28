---
title: cgroups深入理解
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

Linux Cgroups(Control Groups)提供了对一组进程以及将来子进程的资源限制/细粒度的控制资源使用，包括CPU、内存、存储与网络等

使用Cgroups可以方便的限制某个资源的占用并且可以实时的监控和统计信息

最初发布的cgroups是在linux 2.6.24中运行的，但是随后的不协调的更新导致了cgroups分为了两个版本维护：v1和v2 

v2发布在linux 3.10之后，v1成为了老版本，因为兼容性没有被移除

cgroups 主要提供了如下功能：

- 资源限制： 限制资源的使用量，例如我们可以通过限制某个业务的内存上限，从而保护主机其他业务的安全运行。
- 优先级控制：不同的组可以有不同的资源（ CPU 、磁盘 IO 等）使用优先级。
- 审计：计算控制组的资源使用情况。
- 控制：控制进程的挂起或恢复。

<!-- more -->

## 1.2 三大组件

Cgroups构成主要是三个组件：Cgroup、Subsystem、Hierarchy

### **控制组Cgroup**

cgroup用于对进程进行分组管理的机制，一个cgroup包含一组进程，然后可以以cgroup为单位来进行子系统subsystem的各种参数配置。**表示一组进程和一组带有参数的子系统的关联关系**。例如，一个进程使用了 CPU 子系统来限制 CPU 的使用时间，则这个进程和 CPU 子系统的关联关系称为控制组。

### **子系统Subsystem**

subsystem是一组资源控制的模块，是一个内核的组件，一个子系统代表一类资源调度控制器。包含如下几项：

![image-20211101193901382](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211101193901382.png)

每个subsystem关联到相应的cgroup上并对其中的进程进行资源的限制与控制。这些subsystem都是逐步合并到内核中的。

### **层级树Hierarchy**

层级树是**由一系列的控制组按照树状结构排列组成的。**

**cgroup是一组进程和其子系统关联的集合， hierarchy是一系列cgroup的集合**

通过这样的树结构，cgroups可以实现**继承**

### 总结

三者的关系如下图所示：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220228004307141.png" alt="image-20220228004307141" style="zoom:50%;" />

每个层级都会有一个根节点, 子节点是根节点的比重划分。

# 二、cgroups v1与cgroups v2的区别

* 结构不同

  * 不同于 v1 版本， cgroup v2 版本只有一个层级树Hierarchy(层级).

  * cgroup2 文件系统有一个根 Cgroup ，以 `0x63677270`数字来标识，**所有支持v2版本的子系统控制器会自动绑定到 v2的唯一层级上并绑定到根 Cgroup.**

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220220211259940.png" alt="image-20220220211259940" style="zoom:50%;" />

    在V2 版本中，因为只有一个层级，所有进程只绑定到cgroup的叶子节点。

    - 父节点开启的子系统控制器控制到儿子节点，比如 A节点开启了memory controller，那么 C节点cgroup就可以控制进程的memory.
    - 叶子节点不能控制开启哪些子系统的controller,因为叶子节点关联进程Id.所以非叶子节点不能控制进程的使用资源。

  * v1 版本为了灵活一个进程可能绑定多个层级(Hierarchy)，但是通常是每个层级对应一个子系统，多层级就显得没有必要。所以**一个层级包含所有的子系统就比较简单容易管理**

* 线程模式

  * `Cgroup` v2 版本支持线程模式，将 `threaded` 写入到 cgroup.type 就会开启 Thread模式。**当开始线程模式后，一个进程的所有线程属于同一个cgroup**,会采用Tree结构进行管理。

# 三、进程与cgroups的关系

一个进程限制内存和CPU资源，就会绑定到CPU Cgroup和Memory Cgroup的节点上，Cpu cgroup 节点和Memory cgroup节点 属于两个不同的Hierarchy层级（v1版本）。

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

# 四、Cgroups是如何实现的

## 4.1 数据结构

1. 每个进程都会指向一个`CSS_SET`数据结构（如上面进程结构代码所示）
2. 一个 `CSS_SET` 关联多个 `cgroup_subsys_state` 对象，`cgroup_subsys_state` 指向一个 cgroup 子系统。**所以进程和 cgroup 是不直接关联的，需要通过 `cgroup_subsys_state` 对象确定属于哪个层级，属于哪个 `Cgroup` 节点。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/css_set.png" alt="img" style="zoom:50%;" />

3. <font color='#e54d42'>**一个 `Cgroup Hierarchy(`层次）其实是一个文件系统, 可以挂载在用户空间查看和操作**</font>
4. 我们可以查看绑定任何一个cgroup节点下的所有进程Id(PID) （查看对应的文件系统）
   - 实现原理: 通过进程的fork和退出，从 `CSS_SET` attach 或者 detach 进程

## 4.2 用户空间的文件系统

Cgroup 通过 `VFS `文件系统将功能暴露给用户，**用户创建一些文件，写入一些参数即可使用**，那么用户使用Crgoup功能会创建哪些文件？

文件如下:

- `tasks` 文件: 列举绑定到某个 cgroup的 所有进程ID（PID）.
- `cgroup.procs` 文件: 列举 一个Cgroup节点下的所有线程组Id.
- `notify_on_release` flag 文件: ：填 0 或 1，表示是否在 cgroup 中最后一个 task 退出时通知运行release agent，默认情况下是 0，表示不运行。
- `release_agent` 文件： 指定 release agent 执行脚本的文件路径（该文件在最顶层 cgroup 目录中存在），在这个脚本通常用于自动化umount无用的 cgroup
- `clone_children flag`文件: 这个标志只会影响 cpuset 子系统，如果这个标志在 cgroup 中开启，一个新的 cpuset 子系统 cgroup节点 的配置会继承它的父级cgroup节点配置。
- 每个子系统也会创建一些特有的文件。

### VFS文件系统

VFS 是一个内核抽象层，能够**隐藏具体文件系统的实现细节，从而给用户态进程提供一套统一的API接口**。

VFS 使用了一种通用文件系统的设计，具体的文件系统只要实现了 VFS 的设计接口，就能够注册到 VFS 中，从而**使内核可以读写这种文件系统**。 这很像面向对象设计中的抽象类与子类之间的关系，抽象类负责对外接口的设计，子类负责具体的实现。其实，VFS本身就是用 c 语言实现的一套面向对象的接口。

# 五、cgroups作用范围

对于系统进程调度的方式中，cgrous只适用于普通调度，而不适用于实时调度

* 普通调度：关注周转时间
* 实时调度：关注响应时间，实时性要求非常高

适用的进程调度算法是：CFS（Completely Fair [Scheduler](https://so.csdn.net/so/search?q=Scheduler&spm=1001.2101.3001.7020)的缩写，即完全公平调度器，负责进程调度）

https://blog.csdn.net/XD_hebuters/article/details/79623130

现在一般的linux都是CFS的进程调度算法，其他较老的算法，cgroups也不会生效

* FIFO：先进先出
* RR: 时间片轮转
* CFS：公平调度

