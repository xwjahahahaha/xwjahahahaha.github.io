---
title: 12-runtime-1-goroutine原理与协程调度
tags:
  - golang
categories:
  - null
toc: true
date: 2021-11-06 14:37:41
---

> 学习自：
>
> * https://zhuanlan.zhihu.com/p/69554144#:~:text=用户态就是提供应,，内存，I%2FO%E3%80%82
> * https://zhuanlan.zhihu.com/p/342119843
> * 极客时间《go进阶训练营》


[TOC]

<!-- more -->

# 一、goroutine原理

## 1.1 Goroutine

### 1. 用户态与内核态

linux的系统架构如下图：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/4J34Ry.png" alt="4J34Ry" style="zoom:50%;" />

用户态或者内核态可以理解为用户应用程序空间与内核空间。

内核态就是内核，是一套在硬件之上应用程序之下的一套软件程序，管理系统的cpu调度、内存资源等重要服务。

用户态就是提供应用程序运行的空间，为了使一些特定的应用程序访问到内核管理的资源（CPU、内存等）要通过系统调用调用内核提供的一些通用的访问接口。

库函数就是为了屏蔽这些复杂的系统调用，为程序员提供的更为方便的函数上层封装接口，例如：open(), write(), read()等等。

**goroutine运行在用户态**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/acomlz.png" alt="acomlz" style="zoom: 40%;" />

### 2. goroutine定义

“Goroutine 是一个与其他 goroutines 并行运行在同一地址空间的 Go 函数或方法。一个运行的程序由一个或更多个 goroutine 组成。它与线程、协程、进程等不同。它是一个 goroutine” —— Rob Pike

Goroutines 在同一个用户地址空间里并行独立执行 functions，channels 则用于 goroutines 间的**通信**和**同步**访问控制。

### 3. goroutine与thread的区别

 <img src="http://xwjpics.gumptlu.work/qinniu_uPic/p1YMfn.png" alt="p1YMfn" style="zoom: 50%;" />

* 内存占用

  创建一个 goroutine 的栈内存消耗为 **2 KB**(Linux AMD64 Go v1.4后)，运行过程中，**如果栈空间不够用，会自动进行扩容**。
  创建一个 thread 为了尽量避免极端情况下操作系统线程栈的溢出，默认会为其分配一个较大的栈内存( 1 - 8 MB 栈内存，线程标准 POSIX Thread)，而且还需要一个被称为 “guard page” 的区域用于和其他 thread 的栈空间进行隔离。而栈内存空间一旦创建和初始化完成之后其大小就不能再有变化，这决定了在某些特殊场景下系统线程栈还是有溢出的风险。

* 创建/销毁

  线程创建和销毀都会有巨大的消耗，是**内核级的交互(trap)**。
  POSIX 线程(定义了创建和操纵线程的一套 API)通常是在已有的进程模型中增加的逻辑扩展，所以线程控制和进程控制很相似。而进入内核调度所消耗的性能代价比较高，开销较大。**goroutine 是用户态线程**，是由 goruntime 管理，创建和销毁的消耗非常小。

* 调度切换

  抛开陷入内核，线程切换会消耗 1000-1500 纳秒(上下文保存成本高，较多寄存器，公平性，复杂时间计算统计)，一个纳秒平均可以执行 12-18 条指令。所以由于线程切换，执行指令的条数会减少 12000-18000。goroutine 的切换约为 200 ns(用户态、3个寄存器)，相当于 2400-3600 条指令。因此，goroutines 切换成本比 threads 要小得多。

* 复杂性
  线程的创建和退出复杂，多个thread间通讯复杂(share memory)。不能大量创建线程(参考早期的 httpd)，成本高，使用网络多路复用，存在大量callback(参考twemproxy、nginx 的代码)。对于应用服务线程门槛高，例如需要做第三方库隔离，需要考虑引入线程池等。

### 4. M:N模型

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/aW228R.png" alt="aW228R" style="zoom:50%;" />

Go 创建 M 个线程(CPU 执行调度的单元，内核的 task_struct)，之后创建的 N 个 goroutine 都会依附在这 M 个线程上执行，即 M:N 模型。它们能够同时运行，与线程类似，但相比之下非常轻量。因此，程序运行时，Goroutines 的个数应该是远大于线程的个数的（phread 是内核线程？）。

**同一个时刻，一个线程只能跑一个 goroutine。**当 goroutine 发生阻塞 (chan 阻塞、mutex、syscall 等等) 时，Go 会把当前的 goroutine 调度走，让其他 goroutine 来继续执行，而不是让线程阻塞休眠，尽可能多的分发任务出去，让 CPU 忙。

## 1.2 GMP调度模型

### 1. GMP概念

**G：** goroutine 的缩写，每次 go func() 都代表一个 G，无限制。使用 struct runtime.g，包含了当前 goroutine 的状态、堆栈、上下文。

**M：**工作线程(OS thread)也被称为 Machine，使用 struct runtime.m，所有 M 是有线程栈的（独自的栈）。如果不对该线程栈提供内存的话，系统会给该线程栈提供内存(不同操作系统提供的线程栈大小不同)。**当指定了线程栈，则 M.stack→G.stack（将线程的栈指向goroutine的栈），M 的 PC 寄存器指向 G 提供的函数，然后去执行，执行完毕后又会回到线程栈。**

其中找寻可用的goroutine栈的代码就是g0，找寻代码运行在线程自己的栈上

**P：**？

### 2. GM调度器

早期的调度模型，Go 1.2前的调度器实现，限制了 Go 并发程序的伸缩性，尤其是对那些有高吞吐或并行计算需求的服务程序。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211106161429689.png" style="zoom:50%;" />

> <font color='#39b54a'>把所有的G都放到一个全局队列中，这个队列的访问需要锁机制，每个线程M从其中获得一个G然后执行。锁的机制让进程访问的过程中还是有阻塞等待的情况</font>

每个 goroutine 对应于 runtime 中的一个抽象结构：G，而 thread 作为“物理 CPU”的存在而被抽象为一个结构：M(machine)。

当 goroutine 调用了一个阻塞的系统调用，运行这个 goroutine 的线程就会被阻塞，这时至少应该再创建一个线程来运行别的没有阻塞的 goroutine。线程这里可以创建不止一个，可以按需不断地创建，而活跃的线程（处于非阻塞状态的线程）的最大个数存储在变量 GOMAXPROCS中。

问题分析：

* **单一全局互斥锁(Sched.Lock)和集中状态存储**
      导致所有 goroutine 相关操作，比如：创建、结束、重新调度等都要上锁。

* **Goroutine 亲缘性问题**
   刚创建的 G 放到了全局队列，而不是本地 M 执行，被其他M抢占，亲缘性不好。

* **Per-M 持有内存缓存 (M.mcache)**
      每个 M 持有 mcache 和 stack alloc，然而只有在 M 运行 Go 代码时才需要使用的内存(每个 mcache 可以高达2mb)，当 M 在处于 syscall 时并不需要。运行 Go 代码和阻塞在 syscall 的 M 的比例高达1:100，造成了很大的浪费。同时内存亲缘性也较差。G 当前在 M运 行后对 M 的内存进行了预热，因为现在 G 调度到同一个 M 的概率不高，数据局部性不好。

* **严重的线程阻塞/解锁**
      在系统调用的情况下，工作线程经常被阻塞和取消阻塞，这增加了很多开销。比如 M 找不到G，此时 M 就会进入频繁阻塞/唤醒来进行检查的逻辑，以便及时发现新的 G 来执行。在GM模型中，当M没有任务后会休眠，一旦有新的G再唤醒，这样的不断的休眠唤醒效率很低（后面就添加了M自旋）

  > by Dmitry Vyukov “Scalable Go Scheduler Design Doc”

### 3. GMP概念

P： “Processor”是一个抽象的概念，并不是真正的物理 CPU。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/llOn2F-20211106165131708.png" style="zoom:50%;" />

Dmitry Vyukov的方案是引入一个结构 P，它代表了 M 所需的**上下文环境**，也是处理用户级代码逻辑的**处理器**。它负责衔接 M 和 G 的调度上下文，将等待执行的 G 与 M 对接。当 P 有任务时需要创建或者唤醒一个 M 来执行它队列里的任务。所以 **P/M 需要进行绑定，构成一个执行单元**。**P 决定了并行任务的数量，可通过 runtime.GOMAXPROCS 来设定**。在 <u>Go1.5 之后GOMAXPROCS 被默认设置可用的核数</u>，而之前则默认为1。

> <font color='#39b54a'>P可以理解为一个本地队列，作为本地缓存，防止频繁的访问全局队列</font>

> docker中的一个坑：创建的docker会自动设置为宿主机的核数，导致多个docker创建后过多的使用cpu
>
> 解决方法：
>
> Tips: https://github.com/uber-go/automaxprocs
> Automatically set GOMAXPROCS to match Linux container CPU quota.

mcache 等从 M 移到了 P，而 G 队列也被分成两类，保留全局 G 队列，同时每个 P 中都会有一个本地的 G 队列。

引入了 local queue，因为 P 的存在，runtime 并不需要做一个集中式的 goroutine 调度，每一个 M 都会在 P's local queue、global queue 或者**其他 P 队列中找 G 执行**，减少全局锁对性能的影响。**这也是 GMP Work-stealing 调度算法的核心**。

> <font color='#e54d42'>注意，虽然P与M一一绑定，但是M仍然可能会去其他P的队列中寻找G，所以P的本地队列也面临着并发访问的问题</font>

注意 P 的本地 G 队列还是可能面临一个并发访问的场景，为了避免加锁，这里 P 的本地队列是一个 LockFree的队列，窃取 G 时使用 CAS 原子操作来完成。关于LockFree 和 CAS 的知识参见 Lock-Free。



## 1.3 Goroutine Lifecycle生命周期

### 1. go程序启动

**runtime.GOMAXPROCS 指定了P的数量，M的数量可能大于等于P，可以把P的数量理解为并行数**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/B3phMQ.png" alt="B3phMQ" style="zoom: 23%;" />

启动的步骤如下：

0. 整个程序始于一段汇编，而在随后的`runtime·rt0_go`（也是汇编程序）中，会执行很多初始化工作。

   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/MLBbwa.png" alt="MLBbwa" style="zoom:50%;" />

1. 绑定 M0 和 g0，M0就是程序的主线程，程序启动必然会拥有一个主线程(运行主函数`main()`)，这个就是 M0。g0 负责调度，即 shedule() 函数。

2. 创建P0：首先会创建**逻辑 CPU （不是物理CPU核数）核数个 P ，存储在 sched 的 空闲链表(pidle)。**然后绑定P0与M0

3. 新建任务G（就是main）到 P0 本地队列，M0 的 g0 会创建一个指向`runtime.main() `的 G ，并放到 P0 的本地队列。

4. `runtime.main()`: 启动 sysmon 线程M1；启动 GC 协程；执行 init，即代码中的各种 init 函数；执行 `main.main` 函数。

每一个goroutine的创建即`go func(){...}()`的代码都是由一个特殊的goroutine创建的，那就是**g0**

关于g0、sysmon都会在后面介绍

### 2. g0 - 特殊的goroutine

*（本节基于Go 1.13）*

Go必须在每个正在运行的线程上调度和管理goroutine。该角色被委派给名为**`g0`的特殊goroutine，这是为每个系统线程创建的第一个goroutine：**

![QZSRZr](http://xwjpics.gumptlu.work/qinniu_uPic/QZSRZr.png)



然后，它将安排就绪的<u>goroutine在系统线程上</u>运行。

为了更好地了解在`g0`上的调度方式，让我们回顾一下通道的使用情况。这是当goroutine阻塞在通道上发送时：

```go
ch := make(chan int)
[...]
ch <- v
```

在通道上阻塞时，当前goroutine将被停放，即处于等待模式，并且不会在任何goroutine队列中被推送：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/YvpWjn.png" alt="YvpWjn" style="zoom:50%;" />

(G7阻塞进入等待模式)

然后，`g0`**替换**goroutine并进行第一轮调度：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/ROY4zI.png" alt="ROY4zI" style="zoom:50%;" />

在调度期间，**本地队列拥有优先级(但不是一定就在本地队列中调度，优先级详细见Scheduler)**，并且goroutine 2 (G2) 现在将运行：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Xp71cd.png" alt="Xp71cd" style="zoom:50%;" />

一旦接收器将读取通道，则goroutine＃7 (G7) 将被解除阻塞：

`v := <-ch`

接收到消息的goroutine将切换到`g0`并通过将其放置在本地队列中来解锁停放的goroutine（后面会知道这是`runnext`字段的作用）：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/2LQPiy.png" alt="2LQPiy" style="zoom:50%;" />

特殊的goroutine(g0)除了管理调度，它还有更多的职责。

与一般goroutine相反，**`g0`拥有固定的较大的栈**。这样，Go可以在需要更大栈并且在不希望栈增长的情况下执行操作。在`g0`的职责中，我们可以列出：

- **Goroutine创建**。当调用`go func(){ ... }()`或`go myFunction()`时，Go会将函数创建委托给`g0`，然后再将其<u>放置在本地队列中</u>。<u>**新创建的goroutine优先运行，并放置在本地队列的顶部。**</u>

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/G7WkRM.png" alt="G7WkRM" style="zoom:50%;" />

- **Defer方法分配。**
- **GC操作**，例如stw，扫描goroutine的栈以及一些标清操作。
- **栈增长**。在需要时，Go会增加goroutine的大小。该操作由`g0`在prolog方法中完成。

这个特殊的goroutine `g0`涉及许多其他操作（大量分配，cgo等），使我们的程序可以更高效地管理操作，并且需要更大的栈，以保持我们的程序在低内存下更加高效。

### 3. OS thread线程的创建

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211107142659520.png" alt="image-20211107142659520" style="zoom: 33%;" />

准备运行的新 goroutine 时，如果没有新的P存在，那么将唤醒新的 P 以更好地分发工作（在P的空闲链表中选一个，图中是P3）。<u>这个 P 将创建一个与之关联的 M 绑定到一个OS thread。</u>

go func() 中 触发 **Wakeup 唤醒机制：**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Ss9Uzw.png" alt="Ss9Uzw" style="zoom:43%;" />

**有空闲的 Processor 而没有在 `spinning `状态Machine （该状态中M不断循环找寻可以执行的G, 会在后面详述） 时候, 需要去唤醒一个空闲(睡眠)的 M** （M也有一个空闲链表，如果spinning一段时间后还没有找到G，那么这个M就会放入到M的空闲链表中），**如果M空闲链表中也没有那么就会新建一个**，新建的M其实是P0的M创建的（已存在的线程M创建的），**新建的M会创建g0并执行g0**，因为没有G在本地队列，所以g0会从全局队列中寻找G执行

程序启动后，Go 已经将主线程和 M 绑定(`rt0_go`)。
当 goroutine 创建完后，它是放在当前 P 的 local queue 还是 global queue ？

`runtime.runqput `这个函数会尝试把 `new g` 放到本地队列上，<u>**如果本地队列满了**，它会将本地队列的**前半部分**和 `new g`  迁移到全局队列中</u>。剩下的事情就等待 M 自己去拿任务了。

## 1.4 Work-stealing调度算法

### 1. 调度G到线程M上

 Go 基于两种断点将 G 调度到线程上：

* 当 G 阻塞时：系统调用、互斥锁或 chan。阻塞的 G 进入睡眠模式/进入队列，并允许Go 安排和运行等待其他的 G。
* 在函数调用期间，如果 G 必须扩展其堆栈。这个断点允许 Go 调度另一个 G 并避免运行 G 占用CPU。

在这两种情况下，**运行调度程序的 g0 将当前G 替换为另一个 G，即 ready to run。然后，<u>选择的 G 替换 g0 并在线程上运行</u>**

### 2. G的切换过程

在 Go 中，G 的切换相当轻便，其中需要保存的状态仅仅涉及以下两个：

* **Goroutine 在停止运行前执行的指令**，程序当前要运行的指令是记录在程序计数器（PC）中的， G 稍后将在同一指令处恢复运行；
* **G 的堆栈**，以便在再次运行时还原局部变量；在切换之前，堆栈将被保存，以便在 G 再次运行时进行恢复：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/9iLmh8.png" alt="9iLmh8" style="zoom:50%;" />

**从 G 到 g0 或从 g0 到 G 的切换是相当迅速的，它们只包含少量固定的指令。**相反，对于调度阶段，调度程序需要检查许多资源以便确定下一个要运行的 G。

> <font color='#39b54a'>切换很快，寻找新的G则需要花费一些时间</font>

* 当前 G 阻塞在 chan 上并切换到 g0：
  1. PC 和堆栈指针一起保存在内部结构中；
  2. 将 g0 设置为正在运行的 goroutine；
  3. g0 的堆栈替换当前堆栈；

* g0 寻找新的 Goroutine 来运行
* g0 使用所选的新 Goroutine 进行切换： 
  1. PC 和堆栈指针是从其内部结构中获取的；
  2. 程序跳转到对应的 PC 地址；

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211107191350700.png" alt="image-20211107191350700" style="zoom:50%;" />

### 3. Goroutine Recycle 回收

G 很容易创建，栈很小以及快速的上下文切换。基于这些原因，开发人员非常喜欢并使用它们。

然而，**一个产生许多 `shortlive `的 G 的程序将花费相当长的时间来<u>创建和销毁</u>它们。**

每个 P 维护一个` freelist G`(**就是本地队列`local queue`**)，保持这个列表是本地的，这样做的好处是不使用任何锁来 push/get 一个空闲的 G。当 G 退出当前工作时，它将被 push 到这个空闲列表中。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/igXjJM.png" alt="igXjJM" style="zoom: 33%;" />

为了**更好地分发空闲的 G ，调度器也有自己的列表。**它实际上有两个列表：**一个包含已分配栈的 G，另一个包含释放过堆栈的 G（无栈）。**

锁保护` central list`(中心列表)，因为任何 M 都可以访问它。

**本地队列满的解决方法：**当本地列表长度超过64时，调度程序持有的列表从 P 获取 G。然后<u>*一半的* G 将移动到中心列表</u>。需求回收 G 是一种节省分配成本的好方法。

但是，由于堆栈是动态增长的，现有的G 最终可能会有一个大栈。<u>因此，当堆栈增长（即超过2K）时，Go 不会保留这些栈。</u>（<font color='#39b54a'>拓展过栈空间的go都不会保留</font>）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/flSm54.png" alt="flSm54" style="zoom:50%;" />

### 4. Schedule优先级与Work-stealing

M 绑定的 P 没有可执行的 goroutine 时，它会去**按照优先级**去抢占任务：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/hKLIS0.png" alt="hKLIS0" style="zoom: 33%;" />

优先级策略（`runtime.schedule`）顺序如下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/BzHxuL.png" alt="BzHxuL" style="zoom:33%;" />

* 1/61的概率直接去全局队列中寻找新的G
* 如果没有找到，检查本地队列有没有G
* 如果还没有找到：
  * 尝试向其他P的本地队列中偷取G
  * 还没找到，检查全局队列获取G
  * 还没找到，找寻一些网络处理的G

> <font color='#e54d42'>核心思路：避免饥饿，首先防止全局队列饥饿，然后是本地队列等等</font>

窃取的选择思路如下：

为了保证公平性，**从随机位置上的 P 开始**，而且遍历的顺序也**随机化**了(选择一个小于 GOMAXPROCS，且和它互为质数的步长)，**保证遍历的顺序也随机化了。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/htZgAK.png" alt="htZgAK" style="zoom:40%;" />

### 5. Spining thread

**线程自旋**是相对于线程阻塞而言的，表象就是**循环执行一个指定逻辑**(就是上面提到的**调度逻辑**，目的是不停地寻找 G)。这样做的问题显而易见，如果 G 迟迟不来，CPU 会白白浪费在这无意义的计算上。但好处也很明显，降低了 M 的上下文切换成本，提高了性能。

* M 带 P 的找 G 运行 （M已将与一个P绑定了）
* M 不带 P 的找 P 挂载 （此时的M处于“游离”状态，从idle空闲链表中寻找）
* G 创建又没 spining M 唤醒一个 M （如果有一个新的G创建，会检查让至少有一个M在spining）

Go 的设计者倾向于高性能的并发表现，选择了后者。当然前面也提到过，<u>为了避免过多浪费 CPU 资源，自旋的线程数不会超过 GOMAXPROCS (Busy P)</u>，<u>这是因为一个 P 在同一个时刻只能绑定一个 M，P 的数量不会超过 GOMAXPROCS，自然被绑定的 M 的数量也不会超过</u>。对于未被绑定的“游离态”的 M，会进入休眠阻塞态。

### 6. Syscall系统调用

**Go 有自己封装的 syscall**，也就是进入和退出 syscall 的时候执行 `entersyscall/exitsyscall`， 也<u>只有被封装了系统调用才有可能触发重新调度</u>，<u>它将改变 P 的状态为 syscall。</u>

**系统监视器 (system monitor)，称为 sysmon**，**会定时扫描。在执行系统调用时, <u>如果某个 P 的 G 执行超过一个 sysmon tick，脱离 M</u>。**(小优化)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Dwneyp.png" alt="Dwneyp" style="zoom: 33%;" />

P3 和 M 脱离后 目前在 idle list 中等待被绑定。而 syscall 结束后 M 按照如下规则执行直到满足其中一个条件：

* 尝试获取同一个 P(P3)，恢复执行 G (为了同源性更好)

* 尝试获取 `idle list `中的空闲 P

* 把 G 放回 global queue，M 放回到M的idle list

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/H56QU9.png" alt="H56QU9" style="zoom: 33%;" />



> **当使用了 Syscall，Go 无法限制 Blocked OS threads （系统调用Syscall会将线程阻塞Blocked，变为Blocked mode）的数量： 因为只可以通过runtime.GOMAXPROCS限制P的数量，但是Blocked OS threads的数量是无法限制的**
> The GOMAXPROCS variable limits the number of operating system threads that can execute user-level Go code simultaneously. There is no limit to the number of threads that can be blocked in system calls on behalf of Go code; those do not count against the GOMAXPROCS limit. This package’s GOMAXPROCS function queries and changes the limit.
>
> Tips: **使用 syscall 写程序要认真考虑 pthread exhaust 问题。**

### 7. Sysmon

Sysmon（system monitor）是一个系统级别的goroutine，挂在一个M上执行，**不需要P，始终执行**，会不断的检测P在M上执行的G的时间，如果过长就会主动将其断开

sysmon 也叫监控线程，它无需 P 也可以运行，他是一个死循环，每20us~10ms循环一次，循环完一次就 sleep 一会，为什么会是一个变动的周期呢，主要是避免空转，如果每次循环都没什么需要做的事，那么 sleep 的时间就会加大。

Sysmon的主要作用：

* 释放闲置超过5分钟的 span 物理内存；
* 如果超过2分钟没有垃圾回收，强制执行；
* <u>*将长时间未处理的 netpoll 添加到全局队列；*</u>
* *向长时间运行的 G 任务发出抢占调度；* （信号抢占）
* *收回因 syscall 长时间阻塞的 P*；

go是***协作式抢占***，也就是<u>需要依靠携程阻塞主动释放自己与P的连接，无法直接强制的阻断一个携程</u>。当 P 在 M 上执行时间超过10ms，sysmon 调用 `preemptone `将 G 标记为` stackPreempt `。因此需要在某个地方触发检测逻辑，Go 当前是在检查栈是否溢出的地方判定(morestack())，M 会保存当前 G 的上下文，重新进入调度逻辑。

> <font color='#39b54a'>如果goroutine执行的是一个死循环，那么可能会一直占用这个线程P与M，如果死循环中有哦一些其他函数调用或者需要扩展内存空间，那么会被检测出来（`stackPreempt`），断开链接。如果是一个单纯的死循环那么可能就会一直占有(缺点)</font>

> Tips：
> 死循环：issues/11462
> 信号抢占：**go1.14基于信号的抢占式调度实现原理**

go1.14基于信号的抢占式调度实现原理:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/072nly.png" alt="072nly" style="zoom: 50%;" />

异步抢占，注册` sigurg `信号，通过sysmon 检测，对 M 对应的线程发送信号，触发注册的 handler，它往当前 G 的 PC 中插入一条指令(调用某个方法)，在处理完 handler，G 恢复后，自己把自己推到了 global queue 中。

> Tips: 发生程序 hang 死情况时，通常使用什么工具诊断？
>
> * `go tool pprof`
> * `perf top`

### 8. Network poller

**Go 所有的 I/O 都是阻塞的。然后通过 goroutine + channel 来处理并发。**因此所有的 IO 逻辑都是直来直去的，你不再需要回调，不再需要 future，要的仅仅是 step by step。这对于代码的可读性是很有帮助的。

**G 发起网络 I/O 操作也不会导致 M 被阻塞(仅阻塞G)，从而不会导致大量 M 被创建出来。<u>将异步 I/O 转换为阻塞 I/O 的部分称为net poller。</u>**

打开或接受连接都被设置为非阻塞模式。如果你试图对其进行 I/O 操作，并且文件描述符数据还没有准备好，G 会进入 gopark 函数，将当前正在执行的 G 状态保存起来，然后切换到新的堆栈上执行新的 G。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/BaBWHX.png" alt="BaBWHX" style="zoom: 33%;" />

network poller中G被调度回来执行的触发点：

* sysmon  （调度到全局队列）
* runtime.schedule()  (调用到P的本地队列)
* GC: start the world：从 ready 的网络事件中恢复 G。

**gopark**：G 置为 waiting 状态，等待显示**goready**唤醒，在poller中用得较多，还有锁、chan等。

### 9.  Scheduler Affinity 亲缘性调度

亲缘性调度的一些限制：

* Work-stealing 
      当 P 的 local queue 任务不够，同时 global queue、network poller 也会空，这时从其他 P 窃取任务运行，然后任务就运行到了其他线程。*<u>那么，一些内存的一些亲缘性问题可能会因此放大</u>*
* 系统调用
      当 syscall 产生，Go 把当前线程置为 blocking mode，让一个新的线程接管了这个 P (过一个 sysmon tick 才会交给其他 M，大多数syscall都是很快的)。

**communicate-and-wait问题：**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211107162042595.png" alt="image-20211107162042595" style="zoom:33%;" />

目前M只执行到了G5，但是在使用channel通信的另一方释放了G9，G9可以执行，另一方通信的携程希望M能够快速的反应过来执行G9，但是M需要执行G5，G9可能需要被别的M窃取才可以执行，这就造成了通信上的时间等待问题。

针对 communicate-and-wait 模式，进行了亲缘性调度的优化。
当前 local queue，使用了 FIFO 实现，unlock 的 G 无法尽快执行，如果队列中前面存在占用线程的其他 G。
**Go 1.5 在 P 中引入了` runnext` 特殊的一个字段，可以高优先级执行 unblock G （插队）。加速channel通信机制的时间效率。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/t9uU29.png" alt="t9uU29" style="zoom:33%;" />





