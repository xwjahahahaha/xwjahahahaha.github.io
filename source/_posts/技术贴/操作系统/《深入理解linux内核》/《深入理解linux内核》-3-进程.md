---
 title: 《深入理解linux内核》-3-进程
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-30 19:45:23
---

[TOC]

<!-- more -->

> 学习自书籍：
>
> * 《深入理解linux内核》第三章

# 进程

## 1. 浅述linux进程、用户线程、轻量级进程LWP的历史发展？

linux中发展可总结为：进程 => 用户线程 => 轻量级进程

**进程**：如果遵循传统OS中的概念可概括为：一个程序执行的实例，包含了此程序已经执行到何种程度的数据结构对象的汇集，同时从内核的角度看来，也是分配系统资源（cpu、内存）的基本实体

现代的Unix系统已经支持了多线程的应用程序，但是在早期并没有支持多线程应用程序，而是借助一种**用户态线程**的实现，与认知中的线程一样，都是代表一个执行流，但是最大的区别在于其是依靠**在用户态程序管理多个执行流的创建、处理、调度的**（与现在的golang的协程类似）

这就带来了问题：在内核看来仍然只是一个进程一个执行流，多用户态线程非阻塞切换逻辑非常复杂且低效（自己在用户态干了内核进程调度、切换的事）所以这样的方式就逐渐被淘汰

随后就出现了`lightwetght process(LWP)`轻量级进程，提供了对多线程应用程序更好的支持，其通过**多个进程共享一些资源（地址空间、打开的文件等）实现轻量（当然，共享访问的时候也就需要同步机制）**，同时每个进程都是一个单独的执行流，对于内核来说也可以直接当做单独的进程调度、切换，这其实也就是我们常说的线程

目前大多数多线程应用程序都是使用`pthread (POSIX thread)`库的标准函数集编写的

## 2. 进程描述符与task struct

进程描述符`process descriptor`是用来干嘛的？

=> 内核用于清楚的了解当前进程的状态信息/描述信息，例如打开了什么文件？你的调度优先级是什么？你的地址空间的分配？…

进程描述符都是`task_struct`结构的（如下图，并非全貌图），在linux中经常将进程称为任务task/线程thread

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220730203027416.png" alt="image-20220730203027416" style="zoom: 40%;" />

进程描述符中有很多字段都是指向其他数据结构的指针，右边六个数据结构都是涉及进程所拥有的特殊资源，但是在此章中只介绍进程的状态和进程父子关系

## 3. 进程状态种类？僵尸进程状态？如何设置进程状态？

如上图所示，进程的状态在`state`字段中展示，他是一组标志组成，每个标志描述一种可能的进程状态，状态是互斥的，所以一个进程严格来说只能设置一种状态

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220730204242282.png" alt="image-20220730204242282" style="zoom:50%;" />

此外还有两种状态即可放在`state`中也可放在`exit_state`中，从名字就可以看出都是进程**退出/终止**的时候才会变为这两种其中的一种：

* 僵死状态(`EXIT_ZOMBIE`)

  进程的执行被终止，但是，父进程还没有发布`wait4()`或` waitpid()`系统调用来返回有关死亡进程的信息，因为在父进程这些系统调用之前内核不能丢弃包含在已结束进程的进程描述符中的数据与资源

* 僵死撤销状态(`EXIT_DEAD`)

  最终状态，父进程刚发出`wait()`系统调用，系统将死进程回收删除。设置此状态的原因在于，可能有多个进程在`wait`此进程，所以是为了防止竞态而用作同步

设置进程状态的相关语句：

`p->state= TASK_RUNNING;`

或者使用`set_task_state`、`set_current_state`等内核代码提供的宏

## 4. 如何标识一个进程？内核识别与用户态识别

如何标识一个进程？

首先因为进程与其描述符之间是严格的一对一关系（即使是共享数据的多个轻量级进程），所以唯一标识一个进程是有意义的

在内核态：直接使用32位进程描述符的**地址**标识进程

在用户态：也就是我们熟悉的`PID`

pid是有上限的，具体描述位置在`/proc/sys/kernel/pid_max`中，每次分配都是递增+1的分配，一旦达到上限，那么就会复用之前已经销毁的进程的pid，内核是通过保存一个位图来记录哪些已分配哪些空闲的，并且这个位图存放在一个物理页框中不会释放，比较有意思的是默认的pid max值是32767就等于4KB页框32768位-1

## 5. 进程、线程、pid、tgid、pgid到底怎么区分？

我们知道LWP其实就是线程，这两个是同一个概念，那么进程与线程的区别到底在哪呢？

应该听过很多次，"进程与线程在Linux中都是一样的，没有进程线程的区分"，关于这句话要具体理解：

* 首先，进程与线程对于linux一样指的是，在 Linux 中，**进程（process）和线程（thread）都被抽象为任务（task）**，每个任务都由一个`task_struct`结构体来描述。所有的任务都有一个唯一的进程 ID（PID），而无论是线程还是进程，在 Linux 中都被视为任务。
* 其次，还是有区别的，不一样的点在于**资源共享**方面。
  * 同一个进程下的多个线程/LWP，其实指的是这些线程或者说task共享父进程的地址空间、文件描述符等
  * OS中的多个不同的进程，这指的就是毫不相关的多个进程，他们之间的资源是隔离的，但是可以为一些相同功能的进程归纳为一组，这样可以方便一起处理一些信号和I/O操作

进一步的就可以引申到tgid、pgid，其概念如下：

1. `tgid`（Thread Group ID）：当你创建一个进程（比如通过fork系统调用），它将生成一个新的 tgid，这个 tgid 就是该进程的 pid。当你在这个进程中创建**线程**（比如通过clone系统调用），这些线程的 tgid 就是它们所在的那个进程的 pid。所以，一个进程中的所有线程有同一个 tgid，可以通过这个 tgid 来区分不同的进程。
2. `pgid`（Process Group ID）：pgid 用来标识一组相关的进程，这个概念主要用于 shell 的作业控制。在一个进程组中，所有的进程都有相同的 pgid，这个 pgid 就是该进程组中第一个进程的 pid。进程可以通过改变它们的 pgid 来改变它们所属的进程组。

需要注意的是**`getpid()`系统调用返回的是当前进程的tgid而不是pid**

## 6. 内核保存进程描述符的处理：thread_info线程描述符与内核栈联合结构

一个进程的生命周期是难以确定的，有的几秒钟有的几个月，所以进程描述符不能够存储在静态内存区域（也就是永久分配的区域），所以需要放在**用户内存区而不是内核内存区**，因为用户内存区是动态内存可以分配与回收

那么问题来了：<font color='#e54d42'>既然进程的进程描述符放在用户内存区，内核如何知晓一个进程的进程描述符呢？</font>

=> Linux**将两个紧凑的数据结构存放在一个单独为进程分配的存储区域内**（当然也就是在内核内存区），通常的大小为两个**连续**页框8K（并且为了效率第一个页框的起始地址是2^13的倍数）

* `thread_info`: 线程描述符

* 内核进程堆栈，因为内核控制路径使用很少的栈，所以8K是足够的

  > <font color='green'>进程的内核态堆栈与用户态堆栈的区别是什么？</font>
  >
  > 内核态堆栈是操作系统内核在内核态下执行时用来存放中间结果的一块内存区域，用户态堆栈是用户进程在用户态下执行时用来存放中间结果的一块内存区域。两者的最大区别是：内核态堆栈由操作系统内核来管理，而用户态堆栈由用户程序自己来管理。
  
  > 进程的内核堆栈是和`thread_info`放在一起的，内核为每一个进程维护了这样的一个结构，并且内核栈是随着进程的生成周期结束而回收。

整个存储区域如下图：

![image-20220730213848590](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220730213848590.png)

堆栈由高地址向低地址拓展，底部是`thread_info`结构，`task`和`thread_info`两个指针使得进程描述符与线程描述符**互相**指向，`esp`寄存器指向栈顶（或者说保存了栈顶地址），随着栈的不断扩大其值不断减小

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220730214125166.png" alt="image-20220730214125166" style="zoom:50%;" />

可使用`alloc_threaad_info`和`free_thread_info`宏分配和释放存储这个内存区的这两个结构

> <font color='green'>延伸：如果内核堆栈发生了溢出该怎么办？</font>
>
> 内核堆栈是固定大小的（在许多架构上为8KB），并且不会动态地扩大或缩小。如果内核堆栈的使用超过了这个限制，那么就会发生所谓的 "堆栈溢出"。这可能会导致系统的不稳定甚至崩溃。
>
> 为了防止这种情况发生，Linux 内核有一种叫做 "**堆栈守护页**" 的保护机制。当尝试访问这个守护页时，内核会引发一个页故障，这样就可以检测到堆栈溢出的情况。如果发生了这种情况，内核通常会杀掉造成溢出的进程，以防止对系统的进一步破坏。
>
> 然而，值得注意的是，即使有了这个保护机制，内核代码仍然需要遵守一些规则，以尽量减少堆栈的使用。例如，**内核代码通常会尽量避免深层次的函数调用和大量的局部变量**。这些都会增加堆栈的使用，增加发生堆栈溢出的风险。
>
> 此外，Linux 还有一种叫做 "**分裂内核堆栈**" 的机制，这种机制可以将内核堆栈分为两部分，一部分用于常规的堆栈操作，另一部分用于中断处理。这种方法可以进一步降低内核堆栈溢出的风险。

## 7. 内核如何快速索引当前内核进程的用户态进程描述符？

内核提供了一个非常巧妙的方式可以实现从当前内核栈esp指针地址直接找到进程描述符，流程如下：

首先获取`thread_info`，内核中`current_thread_info()`函数的汇编实现如下：

![image-20220730215240312](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220730215240312.png)

可以看出对于当前esp指向的内核栈顶地址来说，因为此内存区域是8K大小，那么将esp地址的低13位屏蔽掉就可以获得整个union结构的基地址（如图所示），如果是4K大小，那么就将低12位屏蔽即可。

例如上面图中，esp指向的地址是`0x015fa878`屏蔽之后(经过三步的运算过后)得到的地址就是`0x015fa000`，这就是thread_info的首地址

下一步就是获取thread_info的task地址，因为其指向进程描述符，巧的是，**task地址就是thread_info中偏移量为0的地址**，所以`current_thread_info->task`函数的汇编指令如下：

![image-20220730215730911](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220730215730911.png)

（其实没差）执行完这三条指令就在p中获取到了**当前内核进程的用户态进程描述符地址**，然后做进一步的操作…

在多核处理器中这样的实现优势明显，省略了需要额外全局变量标识正在运行的进程的描述符

## 8. 内核中常使用的数据结构：双向链表

内核中经常使用的双向链表结构如图所示：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814194709930.png" alt="image-20220814194709930" style="zoom: 50%;" />

next、prev指针不必多说，这两个指针都包含在`list_head`的结构中，但是其中需要注意的是，这两个指针存放的地址不是上一个/下一个链表节点的地址（整个数据结构的地址），**而是上一个/下一个的`list_head`结构体的地址**(为什么这样做的原因见下文)

具体创建的宏是`LIST_HEAD(list_name)`，相关的操作包括：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814195050175.png" alt="image-20220814195050175" style="zoom:50%;" />

注意：

* list_add_tail是插入p元素之前而不是之后(对应tail) ，这是因为首先是一个双向循环链表，p一般指向的就是首元素地址，所以在首节点之前插入一个元素其实就是在链表尾部tail插入一个新节点

## 9. 进程链表：连接起来的进程描述符(对应task_struct中的task字段)

进程链表将**所有的**进程描述符连接起来，每个进程的进程描述符`task_struct`都有一个`task`字段(`list_head`类型)，此类型连接前后双向的其他`task_struct`

> linux进程中的实现中，**不会在链表节点中添加链表数据，而是在数据(`task_struct`)中添加链表节点(`list_head`)**
>
> <font color='green'>延伸：这样做的好处是什么？</font>
>
> * 数据局部性。利用局部性原理，将链表节点与数据(task_struct)放在一起，这样就会让CPU的缓存命中效率大大提升，当遍历链表节点时因为数据已加载，此时很有可能链表节点已经在CPU缓存中了
> * 灵活性。链表节点中嵌套数据需要预先知道数据的大小并且对于所有节点来说，这个大小必须是相同的并且提前分配，但是在实际的应用中这样做对于存储空间的效率来说是非常低的；如果将链表节点嵌入到数据中，这样的方式就解决了这样的问题。
> * 泛用性：嵌入了链表节点的数据结构可以同时嵌入多个链表节点（也就是数据可以在多个链表中），这样会更加灵活。

整个链表的头是`init_task`描述符，也就是对应了系统的`0`进程（又叫swapper进程）

> init_task的pre指向什么？ => 就是最后插入的task_struct的task字段啊，别忘记了双向链表

## 10. TASK_RUNNING状态的进程链表数据结构的演变（对应run_list字段）

`TASK_RUNNING`状态的进程是可运行状态的进程，也是内核想要寻找下一个进程时需要找的进程

早期的实现中，内核将所有可运行进程都放在一个**运行队列**中，但是这样的实现问题在于依靠进程的优先级排序开销很大，并且在选择最佳进程的时候还需要全局扫描

后来的实现的目的就是：<font color='#39b54a'>在一个**固定的时间**内选出最佳运行进程，这与队列中的进程数无关</font>

具体实现思路是：

* 建立多个可运行进程链表，每个优先级权重(0～139)的对应一个链表，每个`task_strcut`中包含一个`list_head`类型的字段`run_list`（就是每个进程的结构中包含了 一个链表节点），其用于将当前进程链入对应优先级的进程链表中

  > * 进程的动态权重位于其进程描述符的`prio`字段
  > * 多核处理器情况下，每个CPU都会维护自己的一个这样的进程链表集合

* 上述的所有链表，共140个，结构如下的`prio_array_t`数据结构：

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814200909741.png" alt="image-20220814200909741" style="zoom:50%;" />

* 可以通过`enqueue_task(p, array)`插入当前进程到对应优先级的链表中，其中的参数`array`是一个指向`prio_array_t`的指针，其实现等价于：

  ![image-20220814201502560](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814201502560.png)

> <font color='#39b54a'>`run_list`和`task`都是进程的list_head类型的一个链表节点，那么两者的区别是什么？</font>
>
> * 所在的链表功能不同。`task`是用于将当前进程描述符/进程链入到**所有**进程链表中，而`run_list`则是链入到**可运行的**链表中（对应权级）

## 11. 进程之间的关系表示

进程之间有父子关系，同父进程的多个子进程之间是兄弟关系，所以需要在进程描述符中引入几个字段来描述：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814202030352.png" alt="image-20220814202030352" style="zoom:67%;" />

`real_parent`与`parent`都是指向父进程，区别点在于，前者是指向创建P时的父进程，后者是当前的父进程，但是两者一般来说是相同的

p0创建了p1、p2、p3，而随后p3又创建了p4，其关系如下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814203743911.png" alt="image-20220814203743911" style="zoom:67%;" />

> * children.next：指向孩子进程的第一个进程
> * children.prev: 指向孩子进程的最后一个进程

除了如上的关系，进程之间还可能有其他关系：是同一个进程组、登录会话、线程组等等（这些都是非亲属关系）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814204015486.png" alt="image-20220814204015486" style="zoom:67%;" />

## 12. pid与进程描述符的映射关系：较为复杂的pidHash表结构

在很多情况下，内核必须期望从进程的PID导出进程的描述符（例如kill、发送信号等）

顺序扫描的效率是不能接受的，随后就出现了hash表的结构：

为了加速查找出现了4个hash表，对应了不同类型Pid字段，每种都有自己的Hash表：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814204300596.png" alt="image-20220814204300596" style="zoom:67%;" />

这四个散列表用一个**pid_hash数组**保存，对于每一个类型的hash表来说，key还需要经过一步映射（使用`pid_hashfn`函数）为表索引

出现了hash映射就代表着会出现冲突，具体来说就是两个相同的pid经过这样的函数映射之后得到同一个表索引

内核是如何处理这样的冲突的呢？

=> 链表处理，对于每一个表项都是由冲突进程描述符组成的双向链表

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814205115364.png" alt="image-20220814205115364" style="zoom:67%;" />

目前这样的结构还有一个问题：如果内核期望快速拿到一组进程的进程描述符？（在很多场景下期望直接对一组进程操作）

这样就催生出了对应hash表内部更加复杂的结构：

对于上图进程的PID结构（在task_struct中名为pids）更为详细的结构如下表：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220814205731992.png" alt="image-20220814205731992" style="zoom: 50%;" />

* **同组的多个进程使用`pid_list`连接（横向）**
* **hash冲突的多个pid通过`pid_chain`连接成为一个链表（纵向）**（也就是上面说的解决hash冲突问题实际链表节点）

通过pid_list就可以将一系列pid（或者说同组进程/线程，例如图中就是同tgid为4351的一组线程）快速遍历到其进程描述符

## 13. 内核如何组织进程？

为了快速检索的需求，内核需要构建进程的运行队列：

* 将所有处于运行`TASK_RUNNING`状态的进程组织在一起（具体结构也就是上面所述）
* 其他状态`TASK_STOPPED`、`EXIT_ZOMBIE`、`EXIT_DEAD`因为比较简单，所以不需要使用特殊的数据结构（可以直接用pid访问）

* 根据不同的特殊的事件将`TASK_INTERRUPTIBLE`和`TASK_UNINTERRUPTIBLE`这两种不同状态的进程细分为许多类，每一类都对应某个特殊的事件，所以引入了新的数据结构（也是进程链表）：**等待队列**

## 14. 等待队列与唤醒

### 等待队列

等待队列再内核中的作用有很多：

* 中断处理
* 进程同步和定时

进程经常会等待某些事件的发生，例如访问IO、等待系统资源等等，等待队列的实现是在**事件上**的等待，每条等待队列都唯一的对应于一个特殊事件，在此事件上等待的进程就会把自己放在此等待队列中，所以等待队列就是表示<font color='#e54d42'>一组为一个特定事件等待而睡眠的一系列进程</font>，当某一个条件为真的时候，内核就会唤醒他们

等待队列也是由双向链表实现：

每个等待队列都有一个等待队列头的结构：

![image-20220821111444947](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220821111444947.png)

其中等待队列的元素类型结构如下：

![image-20220821111717265](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220821111717265.png)

每一个元素就代表一个等待进程（节点中的`task_struct`字段）

### 唤醒

一般为了避免惊群效应，所以不会直接将整个等待队列的所有进程直接全部唤醒，而是唤醒一个

> 惊群效应：对于一个互斥的资源，唤醒了一群进程去争抢，最后只有一个成功消费，其他进程又要回去睡眠

由此衍生了以下两种类型：

* **互斥进程**，一次只唤醒一个，他们的等待队列的节点`flag`都为1，内核会有选择的唤醒
* **非互斥进程**，`flag`为0，总是由内核在事件发生后唤醒 /都被唤醒（例如需要为等待磁盘传输结束的一组进程，此时就需要将他们都被唤醒）

等待队列中的元素的`func`字段就决定了睡眠中的进程应该用什么方式唤醒（内核开发者可以自定义此函数）

`TASK_INTERRUPTIBLE`和`TASK_UNINTERRUPTIBLE`这两种不同状态的进程唤醒的区别具体来说其实就是：

* 前者接收到一个信号就可以唤醒当前的进程

具体等待队列的一些函数操作见书P102

## 15. 进程的资源限制(rlimit)

每个进程都有一组相关的资源限制，限定了进程能够使用系统资源的数量，具体的如下表：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220821113024306.png" alt="image-20220821113024306" style="zoom:50%;" />

对当前进程的资源限制存放在`task_struct->signal->rlim`字段，在进程的信号描述符的一个字段中，这个字段是一个rlimit的结构数组：

![image-20220821113359119](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220821113359119.png)

超级管理员可以通过`getrlimit()`或者`setrlimit()`系统调用来提高`rlim_max`的上限

注意与`cgroup`的主要区别：

* cgroup主要用于将限制作用于一组进程
* rlimit限制的是单个进程
* 两者都能做限制，但是互相独立，其实除了这些限制以外系统本身还有很多限制，所以一个进程的限制可以有多个但是都必须全部满足在其内

## 16. 进程切换过程中的硬件上下文存储在哪？（TSS&&thread字段）

什么是进程切换：操作系统挂起某个正在运行的进程，并恢复之前某个挂起进程的运行

### 硬件上下文

所有进程有自己的虚拟地址空间，但是需要**共享CPU的寄存器**，所以需要保证CPU寄存器的上下文环境的切换

必须装入寄存器的数据称之为**硬件上下文**，其是进程上下文/可执行上下文的一个子集

linux中具体存放在哪？

* 一部分存放在TSS段（任务状态段，注意是老版本intel的原始设计保存在这里，之后被取消）
* 一部分存放在内核堆栈中（也就是上面说的，和`thread_info`存放在一起）

切换方式的发展：

* 最早是使用硬件支持来切换的，通过`far jmp`指令跳到下一个进程的TSS段描述符的选择符（修改寄存器的值），cpu会自动保存原进程的硬件上下文（在其自己的task_struct中）并重放新的硬件上下文

  > 缺点：硬件不会检查ds和es段寄存器的值

* 现在转为使用一组`mov`指令（软件层面）切换，这样可控制性更高一些，性能也差不多

进程切换的过程在**内核态发生**，在切换之前上一个进程的所有寄存器内容都已经保存在内核堆栈上了（包括ss和esp这对描述用户态堆栈地址的寄存器）

### 任务状态段TSS

8086体系结构的一个特殊段类型，叫做**任务状态段**`TSS`，专门用于存放硬件上下文

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220821120041881.png" alt="image-20220821120041881" style="zoom:67%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220821120113487.png" alt="image-20220821120113487" style="zoom:50%;" />

一个TSS段对应于一个CPU（注意Linux中一对一的不是进程，就像一个GDT对应于一个CPU一样）

虽然Linux中不使用硬件上下文，但是还是会为每个CPU创建这样的TSS段

其主要功能就是（或者说存储的值是）：

* 当一个CPU从用户切换到内核态的时候，其要从TSS段中获取**内核态堆栈的地址**
* 一个用户态进程通过in或者out指令访问一个IO端口的时候，会**从TSS中获取IO许可权位图**，判断自己是否有权利访问

每次进程切换的时候内核都更新TSS的某些字段以便相应的cpu控制单元可以安全的检索到它需要的信息

因此TSS反应了当前在CPU上运行的进程的特权级别，但是没有运行的进程并不会保留其TSS

Linux中每个CPU的`tr`寄存器包含了相应的TSS的TSSD选择符（TSSD：任务状态段描述符），也包含了两个非编程字段：TSSD的Base字段和Limit字段，这样CPU就可以直接对TSS寻址而不用从GDT中检索TSS的地址

所以：<font color='#e54d42'>对于Linux来说，目前TSS只用做给CPU一对一使用的保存权级切换时的内核态堆栈地址以及IO许可的权级位图，并不会保存进程切换的某个进程的硬件上下文</font>

> 对于x86架构来说，任务状态段（Task State Segment, TSS）在早期的设计中，被用于存储一个任务（即进程或线程）的上下文信息，以支持硬件上的任务切换。然而，在现代操作系统中，如 Linux，硬件任务切换已经不再被使用，因为它的开销过大，且不够灵活。
>
> 在 Linux 中，TSS 的主要作用已经转变为存储内核栈的地址和 IO 位图。当 CPU 从用户态切换到内核态（例如，因为系统调用或者硬件中断），CPU 需要知道在哪里找到内核栈，这个地址就存储在 TSS 中。同样，当 CPU 返回到用户态时，也需要用到这个地址。这就是“保存权级切换时的内核态堆栈地址”的意思。
>
> IO 位图则用于定义哪些 IO 端口可以被哪些特权级别的代码访问。这是一种对硬件资源访问的保护机制。
>
> 至于进程的上下文（包括寄存器的值，虚拟内存环境等等），它们是在进程切换时由内核保存到进程的 `task_struct` 结构体中，而不是存储在 TSS 中。当内核需要切换到另一个进程时，会从那个进程的 `task_struct` 结构体中恢复上下文。

### thread字段

如上所述，硬件上下文必须保存在别处而不在TSS中，因为TSS针对的是CPU

具体的目前Linux将硬件上下文信息保存在`thread_struct`中的**`thread`字段**

## 17. 进程切换的两大步骤

1. 切换页全局目录安装一个新的地址空间（第九章详述）
2. 切换内核态堆栈与硬件上下文（这一步主要由switch_to宏函数完成）

## 18. switch_to函数的last参数到底作用何在？内核函数__switch_to()解析？

### last参数

switch_to宏函数的参数有三个：

* pre：待替换进程的描述符地址
* next：下一个要运行进程的描述符地址
* last：这是一个输出参数，是一个地址，在本函数中会地址指向的位置赋值。其地址值指向的是next进程的prev局部变量位置/地址

> 参考链接：https://developer.aliyun.com/article/296904

```c
#define switch_to(prev,next,last) \
	asm volatile(SAVE_CONTEXT						    \
		     "movq %%rsp,%P[threadrsp](%[prev])\n\t" /* save RSP */	  \
		     "movq %P[threadrsp](%[next]),%%rsp\n\t" /* restore RSP */	  \
		     "call __switch_to\n\t"					  \
		     ".globl thread_return\n"					\
		     "thread_return:\n\t"					    \
		     "movq %%gs:%P[pda_pcurrent],%%rsi\n\t"			  \
		     "movq %P[thread_info](%%rsi),%%r8\n\t"			  \
		     LOCK "btr  %[tif_fork],%P[ti_flags](%%r8)\n\t"		  \
		     "movq %%rax,%%rdi\n\t" 					  \
		     "jc   ret_from_fork\n\t"					  \
		     RESTORE_CONTEXT						    \
		     : "=a" (last)					  	  \
		     : [next] "S" (next), [prev] "D" (prev),			  \
		       [threadrsp] "i" (offsetof(struct task_struct, thread.rsp)), \
		       [ti_flags] "i" (offsetof(struct thread_info, flags)),\
		       [tif_fork] "i" (TIF_FORK),			  \
		       [thread_info] "i" (offsetof(struct task_struct, thread_info)), \
		       [pda_pcurrent] "i" (offsetof(struct x8664_pda, pcurrent))   \
		     : "memory", "cc" __EXTRA_CLOBBER)
```



书上用了A->B->C三个进程的切换说的很难理解，只关注进程A-进程>B这样的过程：

![img](http://xwjpics.gumptlu.work/img_afe0e695081bc753fbcb87bb3bb87180.png)

1. （见图1）在进程切换之前，A是当前进程，esp寄存器指向A的内核栈，prev，next这两个局部变量保存在栈中，也就是在A的内核栈中。那么B的内核栈有没有这两个参数呢？当然也有，因为B既然是在等待队列中，很可能B也经历过被其他进程切换出去这一个过程，在那个过程中，B的内核栈同样保存了这两个变量(如果B是新创建的进程，可以在创建时，或在schedule函数中将两个值压入内核栈），但是这两个值肯定跟A中的prev=A，next=B不同，因为那个过程中，B是被切换的，因此，这时，B的内核栈中应该是prev=B.为了在切换到B进程执行时，prev参数是正确的，就需要借助于第三个参数last 。在schedule函数中（它挑选的B，当然也知道B进程描述符的地址），它从B的进程描述符中得到B的内核栈的地址（书中是thread_info参数，3.2.54版本代码中改成了stack参数，原理是一样的），从而得到B的prev参数的地址，作为第三个参数传给switch_to宏。switch_to宏还将A的进程描述符地址加载到eax寄存器中，而在进程切换过程中，eax寄存器内容是不会改变的。
2. （见图2）执行进程切换，主要是内核栈的切换，因为内核实现中，将thread_info结构与内核栈放在一起，esp改变了，current参数得到的当前进程描述符地址也跟着改变。这时，当前进程变成了B进程，并在B的内核栈上工作。注意，这时B内核栈的prev参数还是不正确的，它指向的依然是B。
3. （见图3）将EAX寄存器内容复制到last指向的内存，即B内核栈的prev参数所在的地址。这样，B内核栈上的prev参数就指向了正确的A进程描述符的地址。

总的来说：

（还是对于A->B为例）

* 总体上的目标是期望下一个被切换的进程的内核栈的局部变量prev、next要符合切换时的逻辑（也就是说要正确）
* 下一个进程原本的prev局部变量一定就是自己，因为自己之前进程是被切换掉挂起在等待队列中的（对应例子中切换前B的prev=B）
* 目标是改为此次切换时的上一个进程（例子中是A，即prev=A）
* 所以修改需要明确两件事：
  * 改哪里？切换的下一个进程的prev地址如何记录 => last变量保存，保存下一个进程的prev地址
  * 内容是什么？也就是切换时上一个进程的描述符地址，问题在于如何在进程切换的时候存储？（因为进程切换寄存器上下文也会覆盖） => eax寄存器，在进程切换过程中值不会改变

关于switch_to函数的详细汇编代码解释见书P111，其中还有一个关键内部函数`__switch_to()`

### __switch_to()内部函数

在执行__switch_to()纯c函数之前，会有一个汇编指令将prev和next存入到eax和edx两个寄存器中

这两个值也是该函数的两个参数prev_p和next_p的来源（注意，与一般的函数不同，该函数的参数直接来自于寄存器而不是栈，通过`__attribute__`和`regparm`这样的关键字实现）

关于`__switch_to()`函数的详细执行内容/作用可以简单总结为：（具体一些细节见P113）

* 执行`__unlazy_fpu()`宏代码，保存prev_p进程的FPU、MMX、XMM寄存器的值

  * 这几个寄存器用来适配不同的算术函数指令/汇编指令集，当切换的两个进程有用到这些指令集或使用的并不兼容的时候就需要保存这些寄存器上下文的值（此部分详细可见P115～P118）
  
* 执行`smp_processor_id()`宏获得本地cpu的下标（也就是当前执行的CPU）

* 装入下一个进程的某些上下文到当前CPU

  * 装入next_p（也就是下一个进程）的esp到当前CPU的TSS.esp0字段（TSS任务状态段，保存进程内核栈地址，见上面15or见书108）
  * 装入next_p的线程局部存储TLS段（概念见书P49）到当前CPU的全局描述符表的tls_array字段

* 保存上一个进程的段寄存器值，保存fs、gs段寄存器的内容到prev_p->thread.fs和prev_p->thread.gs中

  > 在x86体系结构中，FS和GS是两个额外的段寄存器，除了CS、DS、ES和SS之外。它们的一般作用如下：
  >
  > 1. 线程本地存储（Thread Local Storage，TLS）：FS和GS段寄存器经常被用于线程本地存储。每个线程都可以为自己分配一部分内存空间，用于存储线程相关的数据。通过使用FS或GS段寄存器，线程可以快速访问自己的TLS数据。
  >
  > 2. 操作系统扩展：某些操作系统在内核级别使用FS和GS段寄存器来存储特定的数据结构或上下文信息。例如，Linux内核中使用GS段寄存器来访问当前进程的线程描述符表。
  >
  > 需要注意的是，FS和GS的具体用途是由特定的操作系统或编程环境定义的，并且可能因为不同的上下文而有所变化。因此，上述的一般作用只是在常见情况下的示例，实际使用会根据具体情况有所不同。

* 装入下一个进程的调试字段到CPU的调试寄存器

  * next_p -> thread.debugreg数组内容装载到dr0...dr7（6个调试寄存器）
  * 对于prev_p即上一个进程的调试寄存器状态不需要保存

* （非必要）更新TSS中的I/O位图

  * 一般采用的是懒加载的方式，当前进程在此时间片执行的过程中实际访问了I/O端口的时候，其真实的位图才会被拷贝到本地CPU的TSS I/O位图段

* 执行结束后，`__switch_to`函数会返回prev_p的值，默认会被拷贝到eax寄存器中（等于返还了prev的值，整个`__switch_to()`函数的执行过程中将prev的值保护起来了，这是非常重要的）

## 19. 创建进程

最早Unix操作系统创建一个进程时效率很低，子进程需要拷贝整个父进程的地址空间，在很多情况下子进程会立即调用execve()函数并清除掉父进程的地址空间，这是很浪费的

随后linux内核通过三种机制优化了这个过程：

* COW写时复制，父子进程可以共享读相同的物理页（或者说映射到同一个物理页），如果两者其中有一个需要改变实际的数据，那么此时才复制一个新的物理页并修改映射进行修改，这也是fork()系统调用使用的原理
* 轻量级进程LWP：父子进程能共享许多在内核中的数据结构，例如页表、打开的文件表以及信号处理
* vfork()创建的子进程能共享父进程的内存地址空间，与COW不同的是，共享读的同时也**共享写**，但是为了保证一致性，父进程创建子进程后会阻塞，等待子进程完成退出

linux中总体创建进程的各类关键函数/系统调用之间的关系是：

![image-20230722190449026](http://xwjpics.gumptlu.work/image-20230722190449026.png)

### clone()库函数

在linux中创建LWP的库函数是`clone()`，其基本的参数如下图：

<img src="http://xwjpics.gumptlu.work/image-20230722182806373.png" alt="image-20230722182806373" style="zoom:67%;" />

关于flags剩余三个字节详细的各类标志：

<img src="http://xwjpics.gumptlu.work/image-20230804192632174.png" alt="image-20230804192632174" style="zoom:67%;" />

<img src="http://xwjpics.gumptlu.work/image-20230804192657877.png" alt="image-20230804192657877" style="zoom:67%;" />

```c
/*
 * cloning flags:
 */
#define CSIGNAL		0x000000ff	/* signal mask to be sent at exit */
#define CLONE_VM	0x00000100	/* set if VM shared between processes */
#define CLONE_FS	0x00000200	/* set if fs info shared between processes */
#define CLONE_FILES	0x00000400	/* set if open files shared between processes */
#define CLONE_SIGHAND	0x00000800	/* set if signal handlers and blocked signals shared */
#define CLONE_PTRACE	0x00002000	/* set if we want to let tracing continue on the child too */
#define CLONE_VFORK	0x00004000	/* set if the parent wants the child to wake it up on mm_release */
#define CLONE_PARENT	0x00008000	/* set if we want to have the same parent as the cloner */
#define CLONE_THREAD	0x00010000	/* Same thread group? */
#define CLONE_NEWNS	0x00020000	/* New namespace group? */
#define CLONE_SYSVSEM	0x00040000	/* share system V SEM_UNDO semantics */
#define CLONE_SETTLS	0x00080000	/* create a new TLS for the child */
#define CLONE_PARENT_SETTID	0x00100000	/* set the TID in the parent */
#define CLONE_CHILD_CLEARTID	0x00200000	/* clear the TID in the child */
#define CLONE_DETACHED		0x00400000	/* Unused, ignored */
#define CLONE_UNTRACED		0x00800000	/* set if the tracing process can't force CLONE_PTRACE on this clone */
#define CLONE_CHILD_SETTID	0x01000000	/* set the TID in the child */
#define CLONE_STOPPED		0x02000000	/* Start in stopped state */
```

关于`ptid` 和 `ctid` 进一步的理解：

它们都是用来设置新创建的线程的线程ID。

1. **ptid（Parent Thread ID）**：这个参数是一个指针，它指向一个在父进程用户空间的变量。当 `clone()` 调用成功返回后，这个变量将被设置为子线程/子LWP的线程ID。这样，父线程就可以通过这个变量来获取子线程的线程ID。
2. **ctid（Child Thread ID）**：这个参数也是一个指针，它同样指向一个在父进程用户空间的变量。不同的是，这个变量将被设置为子线程的线程ID，并且这个值**可以被子线程和父线程同时访问**。**当子线程退出时，这个变量将被设置为0，这样，父线程就可以知道子线程已经退出**。

这两个参数都是可选的，如果不需要这些功能，它们可以被设置为NULL

关于此库函数使用的一个示例是：

```c
#define _GNU_SOURCE     // 开启GUN特性，使用linux glibc的clone函数，要写在所有include的前面
#include <sched.h>      // clone函数必须的头文件
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>

#define STACK_SIZE (1024 * 1024)    /* Stack size for cloned child */

static char child_stack[STACK_SIZE]; /* Space for child's stack */

/* The child_process function will be executed in the child process created by clone. */
static int child_process(void *arg)
{
    printf("child_process() : Hello from the child!\n");
    return 0;
}

int main()
{
    pid_t child_pid;

    /* The flags argument is a bit mask that specifies flags. For simplicity, we'll use the 
    most common CLONE_NEWPID, which isolates the new process in its own PID namespace. */

    /* child_stack + STACK_SIZE 表示从子进程栈的最高位地址*/
    /* CLONE_NEWPID | SIGCHLD 两个标志按位或都开启 */
    child_pid = clone(child_process, child_stack + STACK_SIZE, CLONE_NEWPID | SIGCHLD, NULL);

    if (child_pid < 0) {
        perror("clone");
        exit(1);
    }

    printf("main() : PID of child created by clone() is %ld\n", (long) child_pid);

    /* Parent process waits child process to finish. */
    if (waitpid(child_pid, NULL, 0) == -1) {
        perror("waitpid");
        exit(1);
    }

    printf("main() : Child process has ended\n");

    return 0;
}
```

### clone()系统调用

`clone()`库函数对编程者隐藏了`clone()`系统调用，此系统调用是`sys_clone()`服务例程支持的

`clone()`库函数与`clone()`系统调用的区别在于：clone系统调用是**没有fn和arg参数**的

所以clone库函数做了如下的处理：

clone库函数将fn函数的指针放在调用这个封装函数子进程堆栈的某个位置，这个位置就是clone库函数返回地址的位置，arg就刚好放在fn的下面，这样就等同于函数入栈的过程，那么当封装函数clone执行结束后，子进程就会调用fn函数了

```c
asmlinkage long sys_clone(unsigned long clone_flags, unsigned long newsp, void __user *parent_tid, void __user *child_tid, struct pt_regs *regs)
{
	if (!newsp)
		newsp = regs->rsp;
	return do_fork(clone_flags, newsp, regs, 0, parent_tid, child_tid);
}
```

### fork()系统调用

也是通过clone()系统调用实现的，对于传入clone()的参数有一定的指定以完成其场景：

* flags：指定SIGCHILD信号，其他所有标志都为0
* child_stack参数：传递父进程（也就是调用frok()函数的进程）的堆栈指针（regs->rsp），因为有COW写时复制，所以父子进程共享地址空间

```c
asmlinkage long sys_fork(struct pt_regs *regs)
{
	return do_fork(SIGCHLD, regs->rsp, regs, 0, NULL, NULL);
}
```

### vfork()系统调用

同样基于clone实现，传入的参数：

* flags：SIGCHILD信号以及CLONE_VM和CLONE_VFORK这两个标志被打开
  * CLONE_VM表示共享内存描述符和所有的页表
  * CLONE_VFORK作用就是会让父进程进入等待队列等待子进程释放自己的地址空间，详可见下面的do_fork第8步
* child_stack参数：也是一样传递父进程当前的栈指针

```c
/*
 * This is trivial, and on the face of it looks like it
 * could equally well be done in user mode.
 *
 * Not so, for quite unobvious reasons - register pressure.
 * In user mode vfork() cannot have a stack frame, and if
 * done by calling the "clone()" system call directly, you
 * do not have enough call-clobbered registers to hold all
 * the information you need.
 */
asmlinkage long sys_vfork(struct pt_regs *regs)
{
	return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, regs->rsp, regs, 0,
		    NULL, NULL);
}
```

> 关于注释的解释：
>
> 在这个注释中，作者在解释**为什么`vfork()`系统调用不能直接在用户模式下通过`clone()`来实现**。这个原因并不直观，关键在于所谓的"register pressure"。
>
> "**register pressure**"指的是程序在运行过程中对寄存器的需求。在任何给定的时刻，程序可能需要存储一些临时的值，而这些值的数量可能会超过可用的寄存器的数量。如果一个程序有很高的寄存器压力，那么它将<u>不得不频繁地将数据从寄存器移出到内存，然后再从内存移入到寄存器，这将会降低程序的性能</u>。
>
> `vfork()`的实现中，有很多需要保存的状态信息，例如程序计数器、栈指针等。如果`vfork()`在用户模式下通过`clone()`来实现，由于寄存器的数量有限，可能无法保存所有这些信息。而且，在用户模式下，`vfork()`不能有栈帧（这是因为vfork父子进程共享用户模式的地址空间/栈帧，但是在内核态下父子进程都是有自己的内核栈的），这就更加限制了我们可以使用的空间。
>
> 所以，尽管看上去`vfork()`可以在用户模式下实现，但由于寄存器压力和其它因素，它实际上需要在内核模式下通过特定的系统调用来实现。

### do_fork()函数

而 `fork()`, `vfork()`, `clone()` 等函数则是用户层的API，它们为用户提供了创建新进程或线程的接口，但最终都是通过 `do_fork()` 函数在内核中实现的

各类不同架构的clone都会用到底层的do_fork

![image-20230804104256007](http://xwjpics.gumptlu.work/image-20230804104256007.png)

其基本的参数：

<img src="http://xwjpics.gumptlu.work/image-20230722185254563.png" alt="image-20230722185254563" style="zoom:67%;" />

<img src="http://xwjpics.gumptlu.work/image-20230722185308810.png" alt="image-20230722185308810" style="zoom:67%;" />

函数原型如下（注意：不同版本的内核参数可能会有一些差距）：

`fork.c` :

```c
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 */
long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      struct pt_regs *regs,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
```

do_fork的所有执行步骤如下：

<img src="http://xwjpics.gumptlu.work/image-20230722191551256.png" alt="image-20230722191551256" style="zoom: 67%;" />

<img src="http://xwjpics.gumptlu.work/image-20230722191607713.png" alt="image-20230722191607713" style="zoom:67%;" />

关于5.b详细的解释：

*为什么dofork的时候父子进程在一个cpu上时让子进程先运行对于写时复制来说更加高效？*

在fork后让子进程先运行的原因是因为通常子进程会立即执行exec来运行新的程序，这样会使得大部分的页面都被替换掉，因此即使子进程在运行过程中修改了页面，也不会对父进程造成影响，因为父进程所需的数据都还在原来的页面中。如果父进程先运行并修改了页面，那么当子进程开始运行时，可能需要复制这些被修改的页面，这就会产生额外的内存复制操作，降低了系统的效率。

内核相关源码：

```c
long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      struct pt_regs *regs,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
{
	struct task_struct *p;
	int trace = 0;
	long pid = alloc_pidmap();		  // 查找pidmap分配一个新的PID

	if (pid < 0)
		return -EAGAIN;
	if (unlikely(current->ptrace)) {  // current是父进程
		trace = fork_traceflag (clone_flags);
		if (trace)
			clone_flags |= CLONE_PTRACE;
	}

	// 调用copy_process复制进程描述符
	p = copy_process(clone_flags, stack_start, regs, stack_size, parent_tidptr, child_tidptr, pid);
	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	if (!IS_ERR(p)) {
		struct completion vfork;

		if (clone_flags & CLONE_VFORK) {
			p->vfork_done = &vfork;
			init_completion(&vfork);
		}
		
		// 对应第4步
		if ((p->ptrace & PT_PTRACED) || (clone_flags & CLONE_STOPPED)) {
			/*
			 * We'll start up with an immediate SIGSTOP.
			 */
			sigaddset(&p->pending.signal, SIGSTOP); // 给进程设置SIGSTOP信号
			set_tsk_thread_flag(p, TIF_SIGPENDING);	
		}

		if (!(clone_flags & CLONE_STOPPED))
			wake_up_new_task(p, clone_flags);		// 没有设置CLONE_STOPPED，则唤醒新的子进程
		else
			p->state = TASK_STOPPED;				// 设置了CLONE_STOPPED则将子进程其设置为STOPPED状态

		if (unlikely (trace)) {						// 如果父进程被追踪
			current->ptrace_message = pid;			// 父进程存储子进程的pid
			ptrace_notify ((trace << 8) | SIGTRAP);	// 让父进程停止运行并向父进程的父进程(debugger进程)发送SIGCHILD信号,通知debugger进程可以通过父进程的ptrace_message字段获取其子进程pid
		}

		if (clone_flags & CLONE_VFORK) {			// 设置了VFORK
			wait_for_completion(&vfork);			// 让父进程进入等待队列 等待子进程释放自己的地址空间
			if (unlikely (current->ptrace & PT_TRACE_VFORK_DONE))
				ptrace_notify ((PTRACE_EVENT_VFORK_DONE << 8) | SIGTRAP);
		}
	} else {
		free_pidmap(pid);
		pid = PTR_ERR(p);
	}
	return pid;
}
```

```c
void fastcall wake_up_new_task(task_t * p, unsigned long clone_flags)
{
	unsigned long flags;
	int this_cpu, cpu;
	runqueue_t *rq, *this_rq;

	rq = task_rq_lock(p, &flags);
	cpu = task_cpu(p);
	this_cpu = smp_processor_id();

	BUG_ON(p->state != TASK_RUNNING);

	schedstat_inc(rq, wunt_cnt);
	/*
	 * We decrease the sleep average of forking parents
	 * and children as well, to keep max-interactive tasks
	 * from forking tasks that are max-interactive. The parent
	 * (current) is done further down, under its lock.
	 */
	p->sleep_avg = JIFFIES_TO_NS(CURRENT_BONUS(p) *
		CHILD_PENALTY / 100 * MAX_SLEEP_AVG / MAX_BONUS);

	p->prio = effective_prio(p);

	if (likely(cpu == this_cpu)) {  			// 如果父进程与子进程在同一个cpu上
		if (!(clone_flags & CLONE_VM)) {		// 并且不共享一个页表
			/*
			 * The VM isn't cloned, so we're in a good position to
			 * do child-runs-first in anticipation of an exec. This
			 * usually avoids a lot of COW overhead.
			 */
			if (unlikely(!current->array))
				__activate_task(p, rq);
			else {
				// 将子进程插入到父进程运行队列
				// 将子进程刚好插入在父进程前面，让子进程先运行
				p->prio = current->prio;
				list_add_tail(&p->run_list, &current->run_list); // 把子进程p节点插入到父进程节点之前（run_list是一个链表节点）
				p->array = current->array;						// array字段是一个指向prio_array_t(优先队列)的指针，这里是父子进程指向一个
				p->array->nr_active++;							// 进程描述符计数器+1
				rq->nr_running++;								// 增加调度队列（runqueue，rq）中运行状态进程的数量
			}
			set_need_resched();
		} else									// 子进程创建新的地址空间，不共享一组页表
			/* Run child last */
			__activate_task(p, rq);				// 那么就让子进程最后运行（插入父进程运行队列队尾）
		/*
		 * We skip the following code due to cpu == this_cpu
	 	 *
		 *   task_rq_unlock(rq, &flags);
		 *   this_rq = task_rq_lock(current, &flags);
		 */
		this_rq = rq;
	} else {									// 父子进程不在同一个cpu上
		this_rq = cpu_rq(this_cpu);

		/*
		 * Not the local CPU - must adjust timestamp. This should
		 * get optimised away in the !CONFIG_SMP case.
		 */
		p->timestamp = (p->timestamp - this_rq->timestamp_last_tick)
					+ rq->timestamp_last_tick;
		__activate_task(p, rq);					// 父子进程不在同一个cpu上
		if (TASK_PREEMPTS_CURR(p, rq))
			resched_task(rq->curr);

		schedstat_inc(rq, wunt_moved);
		/*
		 * Parent and child are on different CPUs, now get the
		 * parent runqueue to update the parent's ->sleep_avg:
		 */
		task_rq_unlock(rq, &flags);
		this_rq = task_rq_lock(current, &flags);
	}
	current->sleep_avg = JIFFIES_TO_NS(CURRENT_BONUS(current) *
		PARENT_PENALTY / 100 * MAX_SLEEP_AVG / MAX_BONUS);
	task_rq_unlock(this_rq, &flags);
}
```



总的来说，do_fork函数的核心作用为：

* 为子进程分配一个PID
* 执行copy_process()为子进程创建进程描述符
* 根据一些标志(追踪、STOPPEDN)等做一些相应的措施

其中核心是调用辅助函数copy_process()来创建**子进程描述符**以及子进程需要的其他内核数据结构

### copy_process()

此函数的参数与do_fork()参数相同，唯一的多个一个参数就是上面为子进程分配的PID

此函数细节的步骤有很多，详细见书P123～P126页（省略了一些flag细节相关的详细配置），源码位于fork.c

总的来说，可以总结为以下几个关键步骤：

* 检查一些Clone参数的一致性以及一些安全性检查

* 调用dup_task_struct()创建进程描述符
  * alloc_task_struct()宏为子进程创建进程描述符，并保存地址到tsk局部变量中
  * alloc_thread_info()宏获取一块空闲的内存区域存储新进程的thread_info和内核栈
  * 将tsk(进程描述符)与thread_info之间的指针相互指向对方，建立联系
  
* 检查当前用户创建的进程数量是否超出限制(user_struct结构)，没有的话就递增累积数量+1；检查系统的进程数量是否超出max_threads变量的最大值(所有进程thread_info+内核栈的空间应该不超过物理内存的1/8)

* 设置**进程状态**相关的几个字段：
  * tsk->lock_depth初始化为-1（大内核锁计数器）
  * tsk->did_exec字段初始化为0，该字段记录了进程exec()系统调用的次数
  * 更新从父进程复制到tsk->flags字段中的一些标志(例如清除PF_SUPERPRIV标志表示进程是否使用某些超级用户权限等)
  
* 初始化子进程**描述符数据结构**
  * 复制PID到tsk->pid字段，初始化子进程的list_head数据结构和自旋锁等
  * 调用各类copy函数，复制父进程数据结构到子进程中

* 初始化子进程的**内核栈**
  * 调用copy_thread()，是用CPU寄存器的值（保存在父进程的内核栈中）初始化子进程的内核栈，但是这里不是完全拷贝，会有一些不同：
    * eax寄存器的值(fork/clone系统调用在子进程中的返回值)强制设置为0
    * 子进程描述符的thread.esp字段初始化为子进程自己的内核栈基地址
    * 把汇编函数ret_from_fork地址存放在thread.eip字段中
  
  ```c
  int copy_thread(int nr, unsigned long clone_flags, unsigned long rsp, 
  		unsigned long unused,
  	struct task_struct * p, struct pt_regs * regs)
  {
  	int err;
  	struct pt_regs * childregs;
  	struct task_struct *me = current;
  
  	childregs = ((struct pt_regs *) (THREAD_SIZE + (unsigned long) p->thread_info)) - 1;
  
  	*childregs = *regs;
  
  	childregs->rax = 0;						// rax寄存器设置为0，让子进程返回值为0（因为(新)子进程开始运行的时候，eax/rax就是其返回值的地方）
  	childregs->rsp = rsp;					// 设置rsp
  	if (rsp == ~0UL) {
  		childregs->rsp = (unsigned long)childregs;
  	}
  
  	p->thread.rsp = (unsigned long) childregs;	// 设置rsp为子进程内核栈的基地址
  	p->thread.rsp0 = (unsigned long) (childregs+1);
  	p->thread.userrsp = me->thread.userrsp; 
  
  	....
  }
  ```
  
* 初始化新进程**调度程序**相关的数据结构

* 初始化亲子关系的字段（tsk->parent、tsk->real_parent）

* 执行SET_LINKS宏将子进程描述符插入到进程链表中

* 调用attach_pid()把子进程描述符的PID插入pidhash的各类型散列表中

* 递增nr_threads变量的值和total_forks变量的值（记录被创建进程的数量）

* 终止并返回子进程描述符指针tsk

### 当do_fork()结束之后发生了什么

do_fork之后的进程已经是一个可以运行的完整的子进程了

但是在进程调度的时候会继续完善子进程：

* 把进程硬件上下文相关的thread字段的值装入几个CPU寄存器中
  * thread.esp -> esp寄存器 （子进程内核堆栈地址）
  
  * ret_from_fork()函数的地址 -> eip寄存器（用于接下来执行）
  
    > 这里源码可以见switch_to函数
  
* ret_from_fork会依次调用finish_task_switch()函数（这部分会在第七章详述）

* 用存放在子进程内核栈中的值再装载所有寄存器，然后强迫CPU返回用户态
  * 在copy_thread部分已经将寄存器的值初始化到子进程的内核栈中（复制父进程的值and一些特定修改）
  
* 然后在fork/vfork/clone系统调用结束后，新的进程开始执行

* 系统调用返回值放在eax寄存器中，父进程返回子进程的PID，子进程返回0 ，除非fork系统调用返回0否则父子进程只想相同的代码（这里也与copy_thread对应）

## 20. 内核线程

什么是内核线程：

* 一些始终在内核态运行的内核线程
* 操作系统将一些重要的周期性执行的系统任务函数托付给这些内核线程运行

那也意味着：

* 这些线程始终只在内核态运行，不会切换到用户态
* 这些线程只使用大于PAGE_OFFSET的线性地址空间

创建内核线程的函数kernel_thread()：

(这是i386架构下的源码)

```c
extern void kernel_thread_helper(void);
__asm__(".section .text\n"
	".align 4\n"
	"kernel_thread_helper:\n\t"
	"movl %edx,%eax\n\t"	// 复制一份edx的值到eax
	"pushl %edx\n\t"		// 压入args入栈，在接下来的函数执行中会pop使用
	"call *%ebx\n\t"		// 执行fn(地址存放在ebx中)，函数执行之后的返回值会放入eax
	"pushl %eax\n\t"		// 将上一步函数返回值压入eax，让其返回值作为do_exit的参数
	"call do_exit\n"		// 执行退出
	".previous");

/*
 * Create a kernel thread
 * args:
 * 	fn: 内核线程执行的函数
 * 	arg: 函数的参数
 * 	flags: 一组clone标志
 */
int kernel_thread(int (*fn)(void *), void * arg, unsigned long flags)
{
	struct pt_regs regs;				// 寄存器结构体，用此初始化新内核线程的CPU寄存器

	memset(&regs, 0, sizeof(regs));

	// 借助于do_fork的copy_process函数，将这些寄存器的值初始化新内核线程的CPU寄存器
	regs.ebx = (unsigned long) fn;		// 分别将内核函数地址fn、以及函数的参数放入寄存器
	regs.edx = (unsigned long) arg;

	regs.xds = __USER_DS;
	regs.xes = __USER_DS;
	regs.orig_eax = -1;
	regs.eip = (unsigned long) kernel_thread_helper;	// 让下一条指令执行这个helper汇编函数，总体作用就是运行ebc的函数fn(arg)并返回
	regs.xcs = __KERNEL_CS;
	regs.eflags = X86_EFLAGS_IF | X86_EFLAGS_SF | X86_EFLAGS_PF | 0x2;

	/* Ok, create the new process.. */
	// CLONE_VM避免复制调用进程的页表，CLONE_UNTRACED用于禁止追踪内核进程
	return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
}

```

高层次看kernel_thread_helper函数的作用简单来说就是：

* 调用ebx中指向的函数fn，使用edx中的值即函数的参数执行函数
* 然后调用do_exit函数退出

## 21. 进程0与进程1

进程0是整个OS所有进程的祖先进程，又叫idle进程、swapper进程】

这个进程不同于其他进程的地方在于：其使用**静态**分配的数据结构进行初始化（一般进程的数据结构都是动态分配的）

其创建过程来自start_kernel函数：

```c
/*
 *	Activate the first processor. 
 * 	激活0号进程（所有进程的祖先进程，idle进程orswapper进程）
 */
asmlinkage void __init start_kernel(void)  // asmlinkage宏的作用是表明函数的所有参数来自堆栈而不是寄存器
{
	char * command_line;
	extern struct kernel_param __start___param[], __stop___param[];
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 * 在设置完一些必要的设置后再开启中断
 */
	lock_kernel();
	page_address_init();
	printk(linux_banner);
	setup_arch(&command_line);
	setup_per_cpu_areas();

	/*
	 * Mark the boot cpu "online" so that it can call console drivers in
	 * printk() and can access its per-cpu storage.
	 */
	smp_prepare_boot_cpu();

	/*
	 * Set up the scheduler prior starting any interrupts (such as the
	 * timer interrupt). Full topology setup happens at smp_init()
	 * time - but meanwhile we still have a functioning scheduler.
	 * 在中断开启前先设置调度器
	 */
	sched_init();			// 调度初始化
	/*
	 * Disable preemption - early bootup scheduling is extremely
	 * fragile until we cpu_idle() for the first time.
	 * 禁止抢占，第一次调用cpu_idle之前，早期的cpu调度非常脆弱
	 */
	preempt_disable();
	build_all_zonelists();
	page_alloc_init();
	printk("Kernel command line: %s\n", saved_command_line);
	parse_early_param();
	parse_args("Booting kernel", command_line, __start___param,
		   __stop___param - __start___param,
		   &unknown_bootoption);
	sort_main_extable();
	trap_init();			// 开始各种初始化
	rcu_init();
	init_IRQ();
	pidhash_init();
	init_timers();
	softirq_init();
	time_init();

	/*
	 * HACK ALERT! This is early. We're enabling the console before
	 * we've done PCI setups etc, and console_init() must be aware of
	 * this. But we do want output early, in case something goes wrong.
	 */
	console_init();
	if (panic_later)
		panic(panic_later, panic_param);
	profile_init();
	local_irq_enable();
#ifdef CONFIG_BLK_DEV_INITRD
	if (initrd_start && !initrd_below_start_ok &&
			initrd_start < min_low_pfn << PAGE_SHIFT) {
		printk(KERN_CRIT "initrd overwritten (0x%08lx < 0x%08lx) - "
		    "disabling it.\n",initrd_start,min_low_pfn << PAGE_SHIFT);
		initrd_start = 0;
	}
#endif
	vfs_caches_init_early();
	mem_init();
	kmem_cache_init();
	numa_policy_init();
	if (late_time_init)
		late_time_init();
	calibrate_delay();
	pidmap_init();
	pgtable_cache_init();
	prio_tree_init();
	anon_vma_init();
#ifdef CONFIG_X86
	if (efi_enabled)
		efi_enter_virtual_mode();
#endif
	fork_init(num_physpages);
	proc_caches_init();
	buffer_init();
	unnamed_dev_init();
	security_init();
	vfs_caches_init(num_physpages);
	radix_tree_init();
	signals_init();
	/* rootfs populating might need page-writeback */
	page_writeback_init();
#ifdef CONFIG_PROC_FS
	proc_root_init();
#endif
	check_bugs();

	acpi_early_init(); /* before LAPIC and SMP init */

	/* Do the rest non-__init'ed, we're now alive */
  // 处理那些剩下的、不需要被标记为xxx__init的函数
	rest_init();
}
```

可以看到有一些的xxx_init宏的初始化函数是为了帮助进程0的创建，总体上都是用于内核初始化

reset_init函数如下：

```c
static void noinline rest_init(void)
	__releases(kernel_lock)
{
	kernel_thread(init, NULL, CLONE_FS | CLONE_SIGHAND);	// 创建另一个进程为1的线程(init进程)
	numa_default_policy();
	unlock_kernel();
	preempt_enable_no_resched();
	cpu_idle();			// 让进程0循环空闲等待，让系统其他部分继续初始化
} 
```

进程0调用kernel_thread创建init子进程(也是1号进程)，传入了init函数让其之后运行

随后进程0进入cpu_idle状态，不断自旋空闲等待，系统其他部分继续初始化，一般情况进程0都不会被调度了，除非当没有其他进程TASK_RUNNING的时候才运行，所以可以理解进程 0 是当 CPU 没有其他事情可以做时的“占位符”，确保 CPU 总是有某种任务可以运行

进程1开始执行init函数，具体如下：

```c
static int init(void * unused)
{
	lock_kernel();
	...							// 一些其他内核初始化
	
	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are 
	 * trying to recover a really broken machine.
	 */

	if (execute_command)
		run_init_process(execute_command);

	// 尝试按顺序执行以下的init程序，如果上一个成功则execve直接开始执行该代码，而不会继续向下执行
	// 所以会按序逐个尝试，直到有一个成功
	run_init_process("/sbin/init");
	run_init_process("/etc/init");
	run_init_process("/bin/init");
	run_init_process("/bin/sh");

	panic("No init found.  Try passing init= option to kernel.");
}
```

其中run_init_process的代码如下：

```c
static void run_init_process(char *init_filename)
{
	argv_init[0] = init_filename;
	execve(init_filename, argv_init, envp_init);
}
```

所以总的来说init进程期望通过execve函数，按顺序尝试替换自己为/sbin/init等init程序（但是pid不变还是1）（或者说装入init可执行程序），从一个内核线程变为一个普通用户态进程，此进程就一直存活，创建和监视在整个OS外层其他进程的所有活动

除了0，1这两个标志的内核线程外，还有一些其他内核线程，他们有时候可能是按需创建也可能伴随系统一直存在：

<img src="http://xwjpics.gumptlu.work/image-20230813163526244.png" alt="image-20230813163526244" style="zoom:67%;" />

## 22. 进程的终止

在C语言的库函数中，默认在main函数的最后会隐式调用一个exit()函数，表示退出，exit库函数也是进程终止的一般函数

那么用户态终止进程的两个系统调用与上层库函数的关系可以总结为如下图：

* 库函数层：exit()、pthread_exit()
* 系统调用：exit_group()、exit()
* 内核函数：do_group_exit()、do_exit()

其中第一列是退出/终止整个线程组，第二列是终止某一个进程而不会终止进程组的其他进程

下面就详细的看一下两个核心的内核函数的实现

### do_group_exit函数

do_group_exit的代码如下：

（详细的解释已经在代码注释中了）

```c
/*
 * Take down every thread in the group.  This is called by fatal signals
 * as well as by sys_exit_group (below).
 */
NORET_TYPE void
do_group_exit(int exit_code)  				// exit_code进程的终止代号
{
	BUG_ON(exit_code & 0x80); /* core dumps don't get here */

	if (current->signal->flags & SIGNAL_GROUP_EXIT)		
		exit_code = current->signal->group_exit_code;	
	else if (!thread_group_empty(current)) {
		struct signal_struct *const sig = current->signal;
		struct sighand_struct *const sighand = current->sighand;
		read_lock(&tasklist_lock);
		spin_lock_irq(&sighand->siglock);
		if (sig->flags & SIGNAL_GROUP_EXIT)  	// 当前进程的退出标志不为0，表示当前内核已经在为当前进程组执行退出了
			/* Another thread got here before we took the lock.  SIGNAL_GROUP_EXIT标志的作用是同步 */ 
			exit_code = sig->group_exit_code;	// 赋值/更新终止代号（把已经执行的退出码复制过来）
		else {									
			sig->flags = SIGNAL_GROUP_EXIT;		// 否则设置进程的SIGNAL_GROUP_EXIT标志（当前是第一次执行）
			sig->group_exit_code = exit_code;	// 并设置终止代号到当前进程字段
			zap_other_threads(current);			// 杀死其他进程（让其他进程退出）
		}
		spin_unlock_irq(&sighand->siglock);
		read_unlock(&tasklist_lock);
	}

	do_exit(exit_code);							// 接下来传递exit_code执行do_exit，在退出其他进程后，自己退出
	/* NOTREACHED */
}
```

输入的进程的终止代号可能是系统调用exit_group正常结束指定的一个值也可能是内核提供的一个异常错误代号

设置SIGNAL_GROUP_EXIT代表着当前current进程的所有线程（或者说整个线程组）都在退出

其中zap_other_threads的详细代码如下：

```c
/*
 * Nuke all other threads in the group.
 */
void zap_other_threads(struct task_struct *p)
{
	struct task_struct *t;

	p->signal->flags = SIGNAL_GROUP_EXIT;
	p->signal->group_stop_count = 0;

	if (thread_group_empty(p))	// 其他进程集合为空就不执行了
		return;

	// 遍历与当前tgid对应散列表的每个PID列表
	for (t = next_thread(p); t != p; t = next_thread(t)) {
		/*
		 * Don't bother with already dead threads
		 */
		if (t->exit_state)
			continue;

		/*
		 * We don't want to notify the parent, since we are
		 * killed as part of a thread group due to another
		 * thread doing an execve() or similar. So set the
		 * exit signal to -1 to allow immediate reaping of
		 * the process.  But don't detach the thread group
		 * leader.
		 */
		if (t != p->group_leader)
			t->exit_signal = -1;

		sigaddset(&t->pending.signal, SIGKILL);		// 发送SIGKILL信号，这样所有进程都会执行do_exit函数被杀死
		rm_from_queue(SIG_KERNEL_STOP_MASK, &t->pending);
		signal_wake_up(t, 1);
	}
}
```

### do_exit

do_exit()函数用于退出自己这个进程

关于do_exit()函数的源码逻辑如下：

````c
// fastcall使用寄存器获取函数参数, NORET_TYPE表示函数不返回给调用者（可能会长调用）
fastcall NORET_TYPE void do_exit(long code) 
{
	struct task_struct *tsk = current;
	int group_dead;

	profile_task_exit(tsk);

	if (unlikely(in_interrupt()))
		panic("Aiee, killing interrupt handler!");
	if (unlikely(!tsk->pid))
		panic("Attempted to kill the idle task!");
	if (unlikely(tsk->pid == 1))
		panic("Attempted to kill init!");
	if (tsk->io_context)
		exit_io_context();

	if (unlikely(current->ptrace & PT_TRACE_EXIT)) {
		current->ptrace_message = code;
		ptrace_notify((PTRACE_EVENT_EXIT << 8) | SIGTRAP);
	}

	tsk->flags |= PF_EXITING;		// 将进程的flag字段设置为PF_EXITING，表示正在被删除
	del_timer_sync(&tsk->real_timer);	// 从动态定时器队列中删除进程描述符

	if (unlikely(in_atomic()))
		printk(KERN_INFO "note: %s[%d] exited with preempt_count %d\n",
				current->comm, current->pid,
				preempt_count());

	acct_update_integrals();
	update_mem_hiwater();
	group_dead = atomic_dec_and_test(&tsk->signal->live);
	if (group_dead)
		acct_process(code);
	exit_mm(tsk);		
	// 这些exit函数都是分离进程描述符的部分数据结构，如果这些单独的结构没有共享，那么就直接删除
	exit_sem(tsk);
	__exit_files(tsk);
	__exit_fs(tsk);
	exit_namespace(tsk);
	exit_thread();
	exit_keys(tsk);

	if (group_dead && tsk->signal->leader)
		disassociate_ctty(1);

	module_put(tsk->thread_info->exec_domain->module);
	if (tsk->binfmt)
		module_put(tsk->binfmt->module);

	tsk->exit_code = code;		// 设置进程的终止代号为进程描述符的exit_code字段（这个值要么是exit/exit_group系统调用的正常代号要么是内核的异常终止错误代号）
	exit_notify(tsk);			// 向所有亲属进程发送通告，自己（当前进程）要结束了
#ifdef CONFIG_NUMA
	mpol_free(tsk->mempolicy);
	tsk->mempolicy = NULL;
#endif

	BUG_ON(!(current->flags & PF_DEAD));
	schedule();			// 选择一个新的进程运行，当前进程被设置成EXIT_ZOMBIE状态，在调度程序中不会再被调度执行(switch_to宏调用后停止)（为什么被设置为EXIT_ZOMBIE，可以先看下面的exit_notify细节）
	BUG();
	/* Avoid "noreturn function does return".  */
	for (;;) ;
}
````

其中较为关键的函数exit_notify因为比较复杂，需要进一步的展开：

```c
/*
 * Send signals to all our closest relatives so that they know
 * to properly mourn us..
 */
static void exit_notify(struct task_struct *tsk)
{
	int state;
	struct task_struct *t;
	struct list_head ptrace_dead, *_p, *_n;

	if (signal_pending(tsk) && !(tsk->signal->flags & SIGNAL_GROUP_EXIT) // signal_pending的作用是检查线程是否还有其他未处理的信号
	    && !thread_group_empty(tsk)) {
		// 条件：确保当前的进程或线程有待处理的信号，但是整个线程组没有在退出，并且线程组中还有其他线程
		/*
		 * This occurs when there was a race between our exit
		 * syscall and a group signal choosing us as the one to
		 * wake up.  It could be that we are the only thread
		 * alerted to check for pending signals, but another thread
		 * should be woken now to take the signal since we will not.
		 * Now we'll wake all the threads in the group just to make
		 * sure someone gets all the pending signals.
		 * 处理一种情况：当前线程current被唤醒处理另一组线程组的信号，但是目前它在退出，所以需要唤醒同组的另一个线程来执行
		 */
		read_lock(&tasklist_lock);
		spin_lock_irq(&tsk->sighand->siglock);
		for (t = next_thread(tsk); t != tsk; t = next_thread(t))	// 遍历同线程组(tgid)的进程
			if (!signal_pending(t) && !(t->flags & PF_EXITING)) {	// 当某个同组线程没有未处理的信号并且不在退出时
				recalc_sigpending_tsk(t);							// 重新计算一次该线程的待处理信号
				if (signal_pending(t))
					signal_wake_up(t, 0);							// 满足条件，唤醒其处理
			}
		spin_unlock_irq(&tsk->sighand->siglock);
		read_unlock(&tasklist_lock);
	}

	write_lock_irq(&tasklist_lock);

	/*
	 * This does two things:
	 *
  	 * A.  Make init inherit all the child processes init 
	 * B.  Check to see if any process groups have become orphaned
	 *	as a result of our exiting, and if they have any stopped
	 *	jobs, send them a SIGHUP and then a SIGCONT.  (POSIX 3.2.2.2)
	 *	A.进程init继承所有子进程
	 *  B.检查是否有任何进程组由于我们的退出而变得孤立，如果它们有任何停止的作业，
	 *   请向它们发送 SIGHUP，然后发送 SIGCONT
	 */

	INIT_LIST_HEAD(&ptrace_dead);
	forget_original_parent(tsk, &ptrace_dead);	// 转交自己所有孩子进程给同组的其他进程，如果没有的话那么转交给init进程
	BUG_ON(!list_empty(&tsk->children));
	BUG_ON(!list_empty(&tsk->ptrace_children));

	/*
	 * Check to see if any process groups have become orphaned
	 * as a result of our exiting, and if they have any stopped
	 * jobs, send them a SIGHUP and then a SIGCONT.  (POSIX 3.2.2.2)
	 * 当一个进程组中的所有进程的父进程都不在该进程组时，这个进程组被认为是孤立的。
	 * 这是一个特殊的情况，因为当一个进程组变成孤立并且其中有被停止的进程时，
	 * 该孤立进程组应该接收到SIGHUP信号，随后接收到SIGCONT信号。
	 * 这是按照POSIX规范来确保没有进程被遗留在停止状态。
	 *
	 * Case i: Our father is in a different pgrp than we are
	 * and we were the only connection outside, so our pgrp
	 * is about to become orphaned.
	 * 特殊情况：父进程与我们不属于同一个进程组，我们是外部的唯一链接，因此我们的进程组将成为孤儿
	 */
	 
	t = tsk->real_parent;
	
	if ((process_group(t) != process_group(tsk)) &&
	    (t->signal->session == tsk->signal->session) &&
	    will_become_orphaned_pgrp(process_group(tsk), tsk) &&
	    has_stopped_jobs(process_group(tsk))) {
		__kill_pg_info(SIGHUP, (void *)1, process_group(tsk));		// 先发送SIGHUP，作用是通知进程组中的进程，其状态即将变化（父进程已退出，将变成孤儿）
		__kill_pg_info(SIGCONT, (void *)1, process_group(tsk));		// 再发送SIGCONT，作用是让暂停的进程继续执行，知道自己的状态变化
	}

	/* Let father know we died 
	 *
	 * Thread signals are configurable, but you aren't going to use
	 * that to send signals to arbitary processes. 
	 * That stops right now.
	 *
	 * If the parent exec id doesn't match the exec id we saved
	 * when we started then we know the parent has changed security
	 * domain.
	 *
	 * If our self_exec id doesn't match our parent_exec_id then
	 * we have changed execution domain as these two values started
	 * the same after a fork.
	 *
	 * 确保在发送死亡信号给父进程时，要考虑子进程或父进程可能发生的安全域和执行域的变化。
	 * 如果这些域在子进程创建之后发生了变化，那么直接发送信号可能会导致安全问题。
	 * 为了防止这种情况，需要进行相应的检查。
	 */
	
	if (tsk->exit_signal != SIGCHLD && tsk->exit_signal != -1 &&
	    ( tsk->parent_exec_id != t->self_exec_id  ||
	      tsk->self_exec_id != tsk->parent_exec_id)
	    && !capable(CAP_KILL))				// 如果子进程的执行域发生了改变，我们希望只发送安全的信号，也就是SIGCHLD
		tsk->exit_signal = SIGCHLD;			// 子进程(当前进程)的exit_signal就代表了传递给父进程的退出信号


	/* If something other than our normal parent is ptracing us, then
	 * send it a SIGCHLD instead of honoring exit_signal.  exit_signal
	 * only has special meaning to our real parent.
	 * 如果还有正常父进程之外的追踪进程在跟踪我们，那么向其发送一个SIGCHLD信号（exit_signal是父子进程特有的发送信号的方式，对其不适用）
	 */
	if (tsk->exit_signal != -1 && thread_group_empty(tsk)) {
		int signal = tsk->parent == tsk->real_parent ? tsk->exit_signal : SIGCHLD;
		do_notify_parent(tsk, signal);	// 这里的parent是调试程序
	} else if (tsk->ptrace) {
		do_notify_parent(tsk, SIGCHLD);
	}

	state = EXIT_ZOMBIE;			// 设置退出默认状态为EXIT_ZOMBIE
	if (tsk->exit_signal == -1 &&
	    (likely(tsk->ptrace == 0) ||		// 如果当前进程(子进程)的exit_signal仍然为-1并且没有被追踪
	     unlikely(tsk->parent->signal->flags & SIGNAL_GROUP_EXIT)))
		state = EXIT_DEAD;					// 那么就设置其退出状态为正常的EXIT_DEAD
	tsk->exit_state = state;		// 否则可能就是默认的EXIT_ZOMBIE，成为僵尸进程

	/*
	 * Clear these here so that update_process_times() won't try to deliver
	 * itimer, profile or rlimit signals to this task while it is in late exit.
	 */
	tsk->it_virt_value = cputime_zero;
	tsk->it_prof_value = cputime_zero;

	write_unlock_irq(&tasklist_lock);

	list_for_each_safe(_p, _n, &ptrace_dead) {
		list_del_init(_p);
		t = list_entry(_p,struct task_struct,ptrace_list);
		release_task(t);
	}

	/* If the process is dead, release it - nobody will wait for it */
	if (state == EXIT_DEAD)
		release_task(tsk);		// 释放已经死亡进程的其他数据结构占用的内存

	/* PF_DEAD causes final put_task_struct after we schedule. */
	preempt_disable();
	tsk->flags |= PF_DEAD;		// 设置进程的flag为PF_DEAD
}
```

在下一节会再进一步的详细介绍release_task这个函数

## 23. 进程删除

这一块的描述更好解释了**僵尸进程、孤儿进程之间的关系**

### 为什么会有僵尸进程/为什么要设置进程的僵死状态？

* 因为基于Unix的设计原则来说，父进程期望能获得子进程的执行状态，那么类似的就提供了wait()等之类的库函数检查子进程是否已经成功完成
* 那么，在父进程调用wait之前就不允许子进程在终止之后自己就把进程描述符在内存中释放掉了，这样父进程调用wait也就无法得知任何状态
* 所以就会设置此时的子进程的状态为EXIT_ZOMBIE，并且一直占用着内存
* 总结：**从技术上来说子进程已经死亡，但是还是必须保存其描述符直到父进程得到通知**

### 僵尸进程如何变为孤儿进程的？

* 当父进程一直没处理已经结束的子进程的时候会出现僵尸进程

* 那么如果当父进程在处理子进程结束(例如wait)之前就已经结束时，子进程首先是僵尸进程
* 然后系统为了避免这样过多的僵尸进程占据RAM，会将此时的子进程转交给init进程处理（成为init进程的子进程）
* 这样这些进程就成为了孤儿进程
* init进程在调用wait类系统调用的时候就会将子进程的僵尸状态撤销掉

### release_task

源码详细如下，其中__unhash_process函数的源码也注释了：

```c
static void __unhash_process(struct task_struct *p)
{
	nr_threads--;					// 线程组计数器-1						
	detach_pid(p, PIDTYPE_PID);		
	detach_pid(p, PIDTYPE_TGID);	// 分别从PIDTYPE_PID和PIDTYPE_TGID类型的PID散列表中删除进程描述符
	if (thread_group_leader(p)) {	// 如果该进程是线程组的领头进程，那么也很有可能是进程组的领头进程（即使不是的话也没关系，detach_pid会什么也不做）
		detach_pid(p, PIDTYPE_PGID);	// 那么再从PIDTYPE_PGID和PIDTYPE_SID中删除其进程描述符
		detach_pid(p, PIDTYPE_SID);
		if (p->pid)
			__get_cpu_var(process_counts)--;
	}

	REMOVE_LINKS(p);				// 从进程链表中删除该进程描述符的链接
}

void release_task(struct task_struct * p)
{
	int zap_leader;
	task_t *leader;
	struct dentry *proc_dentry;

repeat: 
	atomic_dec(&p->user->processes);		// 递减当前用户拥有的进程个数
	spin_lock(&p->proc_lock);
	proc_dentry = proc_pid_unhash(p);
	write_lock_irq(&tasklist_lock);
	if (unlikely(p->ptrace))
		__ptrace_unlink(p);					// 如果进程正在被追踪，那么解除追踪，让其重新属于初始父进程
	BUG_ON(!list_empty(&p->ptrace_list) || !list_empty(&p->ptrace_children));
	__exit_signal(p);						// 删除所有挂起信号并释放进程的signal_struct描述符
	__exit_sighand(p);						// 删除进程的信号处理函数
	__unhash_process(p);					// 对进程的pid hash做一系列处理

	/*
	 * If we are the last non-leader member of the thread
	 * group, and the leader is zombie, then notify the
	 * group leader's parent process. (if it wants notification.)
	 * 如果线程组的最后一个成员不是领头进程并且此时领头进程是一个僵尸进程，那么
	 * 向领头进程的父进程发送一个信号，通知其自己已死亡
	 */
	zap_leader = 0;
	leader = p->group_leader;
	if (leader != p && thread_group_empty(leader) && leader->exit_state == EXIT_ZOMBIE) {
		BUG_ON(leader->exit_signal == -1);
		do_notify_parent(leader, leader->exit_signal);
		/*
		 * If we were the last child thread and the leader has
		 * exited already, and the leader's parent ignores SIGCHLD,
		 * then we are the one who should release the leader.
		 *	
		 * do_notify_parent() will have marked it self-reaping in
		 * that case.
		 * 如果当前线程是线程组中的最后一个子线程，并且领导线程已经退出，
		 * 同时领导线程的父进程忽略了 SIGCHLD 信号（领导线程的exit_signal设置为-1），
		 * 那么当前线程应该负责释放领导线程
		 */
		zap_leader = (leader->exit_signal == -1);	// zap_leader是清除领导线程的bool判断值
	}

	sched_exit(p);			// 调整父进程的时间片
	write_unlock_irq(&tasklist_lock);
	spin_unlock(&p->proc_lock);
	proc_pid_flush(proc_dentry);
	release_thread(p);
	put_task_struct(p);		// 递减进程描述符的使用计数器

	p = leader;				// 对应上面的特殊情况，如果zap_leader为正，则清除父进程
	if (unlikely(zap_leader))
		goto repeat;
}
```

