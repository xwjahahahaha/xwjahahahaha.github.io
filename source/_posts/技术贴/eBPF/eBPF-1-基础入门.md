---
title: eBPF-1-基础入门
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-17 10:49:44
---

[TOC]

> 参考资料：
>
> * https://davidlovezoe.club/wordpress/archives/1122#BPF%E7%A4%BE%E5%8C%BA%E5%92%8C%E7%94%9F%E6%80%81
> * [《A thorough introduction to eBPF》](https://davidlovezoe.club/wordpress/archives/867#%E5%90%8E%E8%AE%B0-%E7%BF%BB%E8%AF%91%E5%B0%8F%E7%BB%93)
> * https://davidlovezoe.club/wordpress/archives/874
> * https://davidlovezoe.club/wordpress/archives/988
> * 极客时间《eBPF核心技术与实战》
> * https://zhuanlan.zhihu.com/p/44922656

# 一、背景与基础

## 1.1 网络监控工具发展历程？发展原因？

* 监控内容：监控内核空间运行的数据(数据包), 抓取和过滤符合特定规则的网络包

* 发展历程：

  ![image-20220517110503597](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517110503597.png)

* 发展原因：
  * 1->2: 
    * 效率问题：用户态进程处理监控数据包需要将内核态大量数据包拷贝到用户进程地址空间从而进行分析，设计用户态内核态模式切换（上下文切换），无法处理现在日益增长的数据量(音频、流媒体数据)需求，**急迫的需要一个在内核内部运行用户提供的程序的能力**
  * 2->3: 
    * 原始BPF的设计：设计较差，难以适配新的硬件环境更新。例如：BPF专注于提供少量的RISC指令的特点，因为当前64位寄存器以及多核处理器新指令的的出现，已不再与现代处理器的实际情况相匹配
    * 不仅仅满足于内核数据包的监控，而将功能拓展到例如：性能分析、系统追踪、网络优化等多种类型的工具和平台

> [Brendan Gregg](http://www.brendangregg.com/)，他在2017年的[linux.conf.au大会上的演讲](https://www.youtube.com/watch?v=JRFNIKUROPE)提到「内核虚拟机eBPF」，表示，“超能力终于来到了Linux操作系统“
>
> 因此，[Alexei Starovoitov](https://twitter.com/alexei_ast)为了更好地利用的现代硬件，提出了扩展型BPF（eBPF）设计。eBPF虚拟机更类似于现代的处理器，允许eBPF指令映射到更贴近硬件的ISA以获得更好的性能

<!-- more -->

详细完整的历程：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/bpf-timeline-high-watermark.png)

## 1.2 BPF是什么？eBPF是什么？

### 1. BPF

`Berkeley Packet Filter` 伯克利包过滤器（因为在伯克利大学诞生）

原始的`BPF`是设计用来抓取和过滤符合特定规则的网络包, 过滤器是通过程序实现的，并在基于寄存器的虚拟机上运行

基本原理是<font color='#e54d42'>`BPF`提供了一种在内核事件和用户程序事件发生时，安全注入代码的机制</font>, 这样就让非内核开发人员也可以对内核进行监控和控制

### 2. eBPF

`extend Berkeley Packet Filter` 拓展`BPF`, 它演进成为了一套通用执行引擎

BPF的功能升级版，[Alexei Starovoitov](https://twitter.com/alexei_ast)为了更好地利用的现代硬件，提出了扩展型BPF（eBPF）设计

`eBPF`的基本架构在于：<font color='#e54d42'>借助即时编译器JIT，在内核中运行了一个执行引擎（因为编译后直接在cpu上运行，所以相比虚拟机可能执行引擎更为合适），被验证安全的`eBPF`指令最终都会被内核执行</font>

> * 老版本的`Berkeley Packet Filter`目前称为`cBPF`: `classic BPF`，目前基本废弃，新的`Linux`内核只运行`eBPF`, 内核会将`cBPF`透明的转换为`eBPF`
> * 特别的，当前`cBPF`和`eBPF`都统称为`BPF`了，或者说提到`BPF`不做特殊说明就是`eBPF`

## 1.3 eBPF做了哪些提升？

* 速度更快

  * 更贴近硬件的指令集架构ISA，特别是适应64位寄存器以及提升使用的寄存器数量（从2个提升到10个），这样有助于即时编译提高性能；此外`eBPF`的指令仍然运行在内核中, 不需要向用户态复制数据（也是BPF拥有的），提高了事件处理效率。

    > 对于某些网络过滤器微基准测试上显示，`eBPF`在 `x86-64`架构上的速度比旧的经典`BPF` (`cBPF`)实现最高快四倍，大多数都在1.5倍

  * 新的`BPF_CALL`指令，可以更廉价地调用内核函数

* 拓展性
  * 2014年的[daedfb22451d这次代码提交中](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=daedfb22451dd02b35c0549566cbb7cc06bdd53b)，eBPF虚拟机[直接暴露给了用户空间来调用](https://lwn.net/Articles/603983/)（或者说运行用户空间代码）

## 1.4 eBPF感知代码流程？能够使用eBPF做什么？

`eBPF`怎样感知代码？

* 将代码放在`eBPF`的指定的代码路径中，当代码路径被遍历到时，任何附加的`eBPF`代码都会被执行

能够做什么？

* 网络数据包/流量的过滤转发

  通过编写程序实现对网络数据包/流量的过滤分发，甚至是修改`socket`的设置

  > 实例：
  >
  > * [XDP](https://www.iovisor.org/technology/xdp)这个项目就是专门使用eBPF来执行高性能数据包处理，方法是在收到数据包之后，立即在网络栈的最低层执行eBPF程式
  > * [seccomp BPF](https://lwn.net/Articles/656307/)实现了限制一个进程的系统调用方法的使用

* 调试内核/性能分析

  程序可以添加跟踪点、`kprobes`和`perf`事件; 对于实时运行的系统，可以不重新编译内核而实现编写和测试新的调试代码

  > 甚至可以使用`eBPF`通过「[用户空间静态定义的跟踪点](http://blog.memsql.com/bpf-linux-performance/)」来调试用户空间程序

## 1.5 eBPF如何保证安全性？（内核验证器）

如果允许用户空间代码在内核中运行，`eBPF`如何保证安全性？

* `eBPF`分为两个阶段的检查：
  * 第一阶段：加载每个`eBPF`程序之前
    * 禁止内核锁定：确保`eBPF`终止时不包含任何可能导致内核锁定的循环逻辑（就是不能有循环），通过程序控制流图`CFG`来实现
    * 禁止不可达指令：任何有不可达指令的程序都无法加载
  * 第二阶段：模拟执行
    * 禁止越界跳转和越界数据访问：验证器模拟执行`eBPF`程序，每执行完一次指令就在指令执行之前和之后检查虚拟机的状态，确保寄存器和堆栈状态的有效性，禁止越界跳转和越界数据访问
* 检查时的优化：裁剪
  * `eBPF`验证器会智能的检测出已经检查过程序的子集，从而裁剪分支跳过模拟验证的过程
* 禁止指针运算的安全模式
  * 时机：当没有使用`CAP_SYS_ADMIN`特权加载`eBPF`程序的时候就会进入安全模式，安全模式下会确保内核地址不会泄露给没有特权的用户，并且指针不能写入到内存
  * 如果未启用安全模式，则必须在通过检查之后才允许指针运算（检查计算后的指针是否出现类型、位置、边界违反情况等）
* 无法读取未被初始化（从未被写入内容）的寄存器
  * 寄存器R0-R5的内容在函数调用时会被标记为不可读
  * 对读取栈上的变量也进行了类似的检查，以确保没有指令写入只读类型的帧指针寄存器
* 最后，验证器使用**`eBPF`程序类型**(后面将介绍)来限制可以从`eBPF`程序调用哪些内核函数以及可以访问哪些数据结构

## 1.6 如何理解GCC、llvm、clang等编译器？

编译器分为三个：

* 前端`frontEnd` ：词法和语法分析，将源代码转换为抽象语法树
* 优化器`Optimizer`： 在前端的基础上，对中间代码进行优化
* 后端`backEnd`：将优化后的中间代码转化为各自平台的机器码

GCC、llvm、clang

* `GCC`: `GNU Compiler Collection`，GNU编译器套装，是一套由 GNU 开发的编程语言编译器，最先支持C语言，后来演进可处理` C++、Fortran、Pascal、Objective-C、Java`等语言

* `llvm`：`Low Level Virtual Machine`, 可用作多种编译器的后端使用（能够让程序语言的编译器优化、链接优化等），支持任何编程语言的静态和动态编译，可以使用它编写自己的编译器

  > LLVM的命名最早源自于底层虚拟机（`Low Level Virtual Machine`）的首字母缩写，由于这个项目的范围并不局限于创建一个虚拟机，这个缩写导致了广泛的疑惑。LLVM开始成长之后，成为众多编译工具及低端工具技术的统称，使得这个名字变得更不贴切，开发者因而决定放弃这个缩写的意涵，现今LLVM已单纯成为一个品牌，适用于LLVM下的所有项目

* `clang`：`clang`是`llvm`的前端，可以用来编译`c、c++、ObjectiveC`等语言，其以`llvm`作为后端支持，高效易用，并且与`IDE`有很好的结合

## 1.7 eBPF的执行流程是什么？

下面就是整个`eBPF`的工作流程图：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/ebpfworkflow-20220517161733498.png)

或者是这张图：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518212515703.png" alt="image-20220518212515703" style="zoom:60%;" />

整体的流程可以总结为：

1. 通过`llvm`将程序编译为`eBPF`字节码
2. 通过`bpf`系统调用交给内核
3. 内核在接受字节码之前会进行检查检验
4. 检验通过的字节码提交到即时编译器中运行

## 1.8 eBPF的插桩类型有哪些？

eBPF可以在内核和应用的任意位置进行插桩，主要借助于两种探针：

* 内核态插桩`kprobe`
* 用户态插桩`uprobe`

## 1.9 如何理解Map

`map`又称为映射，`BPF`程序可以利用其进行**存储**

核心职责：**存储eBPF运行时状态即用户程序与运行在内核的`eBPF`程序交互载体**

运行在内核的`eBPF`程序收集目标状态存储在map中，随后用户程序再从映射中读取这些状态

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518213529969.png" alt="image-20220518213529969" style="zoom:67%;" />

## 2.0 eBPF程序的限制有哪些？

保证安全性就意味着限制，所以`eBPF`并不是万能的

具体的限制包括：

* 校验限制：必须通过校验才可以执行，并且不能包含不可到达的指令
* 内核函数限制：内核函数只可以调用API定义的辅助函数
* 存储限制：`eBPF`程序栈最大只有512字节，需要更大的存储要借助映射存储
* 指令数量限制：内核5.2之前支持4096条指令，之后支持100万条
* 移植性/适配性：不同内核版本下的移植可能需要修改源码并重新编译

具体内核版本要求和函数功能支持见：https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md

## 2.1 什么是CO-RE？

一次编译到处运行即`CO-RE`, 也是`eBPF`最大的改进之一

## 2.2 `.elf`文件处于程序编译的什么阶段？

**可执行与可链接格式** （英语：Executable and Linkable Format，缩写 ELF，此前的写法是 **Extensible Linking Format**），常被称为 **ELF格式**，在[计算](https://zh.m.wikipedia.org/wiki/计算_(计算机科学))中，是一种用于[可执行](https://zh.m.wikipedia.org/wiki/可执行)文件、[目标代码](https://zh.m.wikipedia.org/wiki/目标代码)、[共享库](https://zh.m.wikipedia.org/wiki/共享库)和[核心转储](https://zh.m.wikipedia.org/wiki/核心转储)（core dump）的标准[文件格式](https://zh.m.wikipedia.org/wiki/文件格式)。

* 编译：

  高级语言 -> 汇编语言 -> 机器语言 ： 高级语言最终变为机器语言这样的过程可以统称为编译，具体的方法基本有两种：编译型和解释型

  据此也可分为两大类：一种是编译型语言，例如`C，C++，Java`，另一种是解释型语言，例如`Python、Ruby、MATLAB 、JavaScript`

* 四个步骤：

  - 预处理（Preprocessing）

  - 编译（Compilation）

  - 汇编（Assembly）

  - 链接（Linking）

    ![image-20220519092739182](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519092739182.png)

## eBPF技术概览

![image-20220518173621584](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518173621584.png)

# 二、编程接口

## 2.1 系统调用`bpf()`

基本使用：

```c
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

![img](http://xwjpics.gumptlu.work/qinniu_uPic/DraggedImage.png)

* `cmd`表示不同的类型(例如使用`bps()`系统调用和`BPF_PROG_LOAD`命令用于加载程序)
* `bpf_attr union` 允许在内核和用户空间之间传递数据; 确切的格式取决于 `cmd` 参数。
* `size` 这个参数表示`bpf_attr union` 这个对象以字节为单位的大小

`CMD`/命令：

* 可以使用命令创建和修改`eBPF maps`数据结构，这个数据结构是一个通用键值对数据结构，用于在`eBPF`程序和内核或用户空间之间通信

* 分类：

  * 使用`eBPF`程序的命令

  * 使用`eBPF maps`的命令

  * 同时使用`eBPF`程序和`eBPF maps`的命令

## 2.2 eBPF程序类型

函数`BPF_PROG_LOAD`加载的程序类型规定了四件事：

1. 程序可以附加在哪里
2. 验证器允许调用内核哪些帮助函数
3. 网络包数据是否可以直接访问
4. 作为第一个参数传递给程序的对象类型

实际上，程序类型本质上定义了一个`API`，通过不同的程序类型区分允许调用的不同函数列表

目前内核支持的eBPF程序类型列表如下所示：

- `BPF_PROG_TYPE_SOCKET_FILTER`: 一种网络数据包过滤器
- `BPF_PROG_TYPE_KPROBE`: 确定`kprobe`是否应该触发
- `BPF_PROG_TYPE_SCHED_CLS`: 一种网络流量控制分类器
- `BPF_PROG_TYPE_SCHED_ACT`: 一种网络流量控制动作
- `BPF_PROG_TYPE_TRACEPOINT`: 确定` tracepoint`是否应该触发
- `BPF_PROG_TYPE_XDP`: 从设备驱动程序接收路径运行的网络数据包过滤器
- `BPF_PROG_TYPE_PERF_EVENT`: 确定是否应该触发`perf`事件处理程序
- `BPF_PROG_TYPE_CGROUP_SKB`: 一种用于控制组的网络数据包过滤器
- `BPF_PROG_TYPE_CGROUP_SOCK`: 一种由于控制组的网络包筛选器，它被允许修改套接字选项
- `BPF_PROG_TYPE_LWT_*`: 用于轻量级隧道的网络数据包过滤器
- `BPF_PROG_TYPE_SOCK_OPS`: 一个用于设置套接字参数的程序
- `BPF_PROG_TYPE_SK_SKB`: 一个用于套接字之间转发数据包的网络包过滤器
- `BPF_PROG_CGROUP_DEVICE`: 确定是否允许设备操作

## 2.3 eBPF使用的数据结构

eBPF程序使用的主要数据结构是`eBPF map`（键值对）数据结构，这是一种**通用的**数据结构，允许在内核内部或内核与用户空间之间来回传递数据.

使用`bpf()`系统调用创建和操作`map`数据结构。成功创建`map`后，将返回与该`map`关联的文件描述符。每个`map`由四个值定义:

1. 类型
2. 元素的最大个数
3. 值大小(以字节为单位)
4. 键大小(以字节为单位)

有不同的`map`类型，每种类型都提供不同的行为和一些权衡：

- `BPF_MAP_TYPE_HASH`: 一种哈希表
- `BPF_MAP_TYPE_ARRAY`: 一种为快速查找速度而优化的数组类型map键值对，通常用于计数器
- `BPF_MAP_TYPE_PROG_ARRAY`: 与eBPF程序相对应的一种文件描述符数组;用于实现跳转表和处理特定（网络）包协议的子程序
- `BPF_MAP_TYPE_PERCPU_ARRAY`: 一种基于每个cpu的数组，用于实现展现延迟的直方图
- `BPF_MAP_TYPE_PERF_EVENT_ARRAY`: 存储指向`perf_event`数据结构的指针，用于读取和存储`perf`事件计数器
- `BPF_MAP_TYPE_CGROUP_ARRAY`: 存储指向控制组的指针
- `BPF_MAP_TYPE_PERCPU_HASH`: 一种基于每个CPU的哈希表
- `BPF_MAP_TYPE_LRU_HASH`: 一种只保留最近使用项的哈希表
- `BPF_MAP_TYPE_LRU_PERCPU_HASH`: 一种基于每个CPU的哈希表，只保留最近使用项
- `BPF_MAP_TYPE_LPM_TRIE`: 一个匹配最长前缀的字典树数据结构，适用于将IP地址匹配到一个范围
- `BPF_MAP_TYPE_STACK_TRACE`: 存储堆栈跟踪信息
- `BPF_MAP_TYPE_ARRAY_OF_MAPS`: 一种`map-in-map`数据结构
- `BPF_MAP_TYPE_HASH_OF_MAPS`: 一种`map-in-map`数据结构
- `BPF_MAP_TYPE_DEVICE_MAP`: 用于存储和查找网络设备的引用
- `BPF_MAP_TYPE_SOCKET_MAP`: 存储和查找套接字，并允许使用BPF帮助函数进行套接字重定向

可以使用`bpf_map_lookup_elem()`函数和`bpf_map_update_elem()`函数从eBPF程序或用户空间程序访问所有map对象。某些map类型，如套接字类型map，它是与那些执行特殊任务的eBPF帮助函数，一起工作

# 三、开发方式

## 3.1 手动执行每一步

### 1. 手动的具体步骤包括哪些？

简单的总结为：

1. 使用C语言开发一个`eBPF`程序
2. 借助`LLVM`将`eBPF`程序编译为`BPF`字节码
3. 通过`bpf`系统调用将`BPF`字节码提交给内核
4. 内核验证并运行`BPF`字节码，并把相关的状态保存到`BPF`映射`map`中
5. 用户程序通过`BPF`查询`BPF`字节码的运行状态

一些细节：

> 以下来自*《A thorough introduction to eBPF》*

在最开始的开发中，必须通过手工编写`eBPF`汇编代码，并使用内核的`bpf_asm`汇编程序来生成BPF字节码，现在只需要使用`LLVM Clang`编译器增加了对`eBPF`后端的支持，现在可以将C语言写的程序通过`LLVM Clang`编译器，编译成字节码。然后可以使用`bpf()`系统调用函数和`BPF_PROG_LOAD`命令，直接加载包含这个字节码的对象文件

通过使用`Clang`编译器，配合`-march=bpf`参数，您就可以用C语言编写自己的`eBPF`程序了。在内核代码的 [samples/bpf/](http://elixir.free-electrons.com/linux/v4.14.2/source/samples/bpf) 目录下有很多eBPF程序的示例，它们的文件名称大部分都具有「`_kern.c`」的后缀。`Clang`编译出来的目标文件(eBPF字节码)，需要由在本机运行的一个程序进行加载(这些示例的文件名称中通常具有「`_user.c`」)

> `kern.c`和`user.c`分别对应内核态和用户态的`eBPF`使用

为了更容易地编写`eBPF`程序，内核提供了`libbpf`库，其中包括用于加载程序、创建和操作`eBPF`对象的帮助函数。举个例子，一个`eBPF`程序和使用`libbpf`库的用户程序的抽象的工作流程一般像如下这样的：

- 读取eBPF字节码到用户应用程序中的缓冲区，并将其传递给`bpf_load_program()`函数
- eBPF程序，当在内核运行时，它将调用`bpf_map_lookup_elem()`函数来查找`map`中的元素，并存储新值给这个元素。
- 用户应用程序调用`bpf_map_lookup_elem()`函数来读取eBPF程序存储在内核中的值。

## 3.2 借助BCC工具

### 1. bcc出现原因？什么是BCC？

* 出现原因：

  对于在生产环境机器或者客户机上工作的工程师来说，从内核源代码中编译程序并链接到`eBPF`库是比较困难的（避免手动的繁琐流程）

* BCC是什么：

  `BPF Compiler Conllection`, 一个BPF编译器集合，包括**用于编写、编译和加载`eBPF`程序的工具链，以及用于调试和诊断性能问题的示例程序和久经考验的工具**，并向上提供了高级语言支持`Python、C++`等

> 代码仓库：https://github.com/iovisor/bcc

bcc工具大全：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518215612105.png" alt="image-20220518215612105" style="zoom:67%;" />

### 2. 借助BCC开发的优势

* 支持高级语言接口

  BCC支持高级语言进行编程（`Python`和`Lua`）例如，开发人员可以将`eBPF map`类比为`Python`字典，并可以直接访问映射内容，这是通过使用BPF帮助函数，它在内部实现这个功能。

* 错误提示

  BCC调用`LLVM Clang`编译器，这个编译器具有BPF后端，可以将C代码转换成eBPF字节码。然后，`BCC`负责使用`bpf()`系统调用函数，将`eBPF`字节码加载到内核中

  如果加载失败，例如内核验证器检查失败，则BCC提供有关加载失败原因的提示，如，“提示：如果在没有首先检查指针是否为空的情况下，从map查找中取消引用指针值，可能就会出现**`The 'map_value_or_null'`**”。这是创建BCC的另一个动机——因为很难写出明显正确的BPF程序;当你犯了错误时，BCC会通知你

# 四、环境搭建

## 1. `linux`环境搭建

最好使用最新版本的内核才能够使用大部分的功能，可以通过`vagrant`创建一个linux虚拟机

```shell
sudo apt-get install virtualbox vagrant
vagrant init ubuntu/impish64
vagrant up
vagrant ssh  # 连接到虚拟机
uanme -a
lsb_release -a
```

> 如果嫌麻烦可以用大佬已经做好的docker镜像，一步到位不用担心环境（bcc也在）：
>
> `https://github.com/Jun10ng/ebpf-docker-for-desktop`

## 2. `bcc`环境搭建

前提：`a Linux kernel version 4.1 or newer is required` 此外，内核应该已经编译并设置了以下标志

```shell
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
# [optional, for tc filters]
CONFIG_NET_CLS_BPF=m
# [optional, for tc actions]
CONFIG_NET_ACT_BPF=m
CONFIG_BPF_JIT=y
# [for Linux kernel versions 4.1 through 4.6]
CONFIG_HAVE_BPF_JIT=y
# [for Linux kernel versions 4.7 and later]
CONFIG_HAVE_EBPF_JIT=y
# [optional, for kprobes]
CONFIG_BPF_EVENTS=y
# Need kernel headers through /sys/kernel/kheaders.tar.xz
CONFIG_IKHEADERS=y
```

可以从`/proc/config.gz` or `/boot/config-<kernel-version>`中查看自己主机内核的配置是否适配 

(我的文件在`vim /boot/config-4.15.0-48-generic `, 注意除非重新编译，否则此文件不能更改)

[建议按照安装`BCC`官方教程](https://github.com/iovisor/bcc/blob/master/INSTALL.md#kernel-configuration)

最简单的方式：包管理器安装

```shell
# 第一种方式：包安装
sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
```

比较复杂的方式（不建议）：

```shell

# 第二种方式：源码安装（包括下面的安装和编译）
# Trusty (14.04 LTS) and older
VER=trusty
echo "deb http://llvm.org/apt/$VER/ llvm-toolchain-$VER-3.7 main
deb-src http://llvm.org/apt/$VER/ llvm-toolchain-$VER-3.7 main" | \
  sudo tee /etc/apt/sources.list.d/llvm.list
wget -O - http://llvm.org/apt/llvm-snapshot.gpg.key | sudo apt-key add -
sudo apt-get update

# For Bionic (18.04 LTS)
sudo apt-get -y install bison build-essential cmake flex git libedit-dev \
  libllvm6.0 llvm-6.0-dev libclang-6.0-dev python zlib1g-dev libelf-dev libfl-dev python3-distutils

# For Eoan (19.10) or Focal (20.04.1 LTS)
sudo apt install -y bison build-essential cmake flex git libedit-dev \
  libllvm7 llvm-7-dev libclang-7-dev python zlib1g-dev libelf-dev libfl-dev python3-distutils
  
# For Hirsute (21.04) or Impish (21.10)
sudo apt install -y bison build-essential cmake flex git libedit-dev libllvm11 llvm-11-dev libclang-11-dev python zlib1g-dev libelf-dev libfl-dev python3-distutils

# For other versions
sudo apt-get -y install bison build-essential cmake flex git libedit-dev \
  libllvm3.7 llvm-3.7-dev libclang-3.7-dev python zlib1g-dev libelf-dev python3-distutils

# For Lua support
sudo apt-get -y install luajit luajit-5.1-dev
```

安装和编译`bcc`

```shell
git clone https://github.com/iovisor/bcc.git
mkdir bcc/build; cd bcc/build
cmake ..   # 注意使用的llvm环境是哪一个
make
sudo make install
cmake -DPYTHON_CMD=python3 .. # build python3 binding 绑定到python3而不是python2
pushd src/python/
make
# 安装到/usr/lib/python3/dist-packages/下
sudo make install
popd				# 返回栈顶目录
```

# 五、eBPF实践demos

## 5.1 使用BCC工具

### 1. python ebpf 监控内核clone

```python
#!/usr/bin/python
# Copyright (c) PLUMgrid, Inc.
# Licensed under the Apache License, Version 2.0 (the "License")

# run in project examples directory with:
# sudo ./hello_world.py"
# see trace_fields.py for a longer example

from bcc import BPF

# This may not work for 4.17 on x64, you need replace kprobe__sys_clone with kprobe____x64_sys_clone
BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
```

解释：

* `bpf_trace_printk`是一个`BPF`的辅助函数, 其作用是打印输出，由于运行在内核中，所以打印输出并不是标准输出，而是内核调试文件`/sys/kernel/debug/tracing/trace_pipe`
* `BPF(text=“xxx”)`：表示传入要执行的`eBPF`源代码, 并依托BPF模块编译为字节码
* `trace_print()`：读取内核文件`/sys/kernel/debug/tracing/trace_pipe`到标准输出

运行`sudo ././helloword.py` 或者`sudo /usr/bin/python3 ./helloword.py `

![image-20220519095327510](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519095327510.png)

**效果：一旦内核中发生了`clone`, 就打印`Hello World!`**

输出解释：

* `systemd-xxx`：是进程的名字-PID
* `[001]`: 表示CPU编号
* `……`：表示一系列选项
* `182.317265`：表示时间戳
* `do_sys_open`: 表示函数名

### 2.  python + C ebpf 监控打开文件

> 注意，此Demo需要内核`5.6`以上

编写代码`hello.c`

```c
int hello_world(void *ctx) 
{
    bpf_trace_printk("Hello, World!\n");
    return 0;
}
```

编写代码`hello.py`

```python
#!/usr/bin/python3
# 1) import bcc library
from bcc import BPF

# 2) load BPF program
b = BPF(src_file="hello.c")
# 3) attach kprobe
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")
# 4) read and print /sys/kernel/debug/tracing/trace_pipe
b.trace_print()
```

测试：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519143340214.png" alt="image-20220519143340214" style="zoom:50%;" />

输出的问题：

* 可能不需要某些输出，例如CPU编号等
* 输出格式不够灵活
* 所有的`eBPF`程序都会输出到`trace_pipe`文件，比较混乱

### 3. 改进程序：使用BPF映射Map

> 注意，此Demo需要内核`5.6`以上

为了解决输出凌乱的问题，我们借助`map`映射来交互

`BCC`为了简化和`BPF`的交互，定义了一系列的[库函数和辅助宏定义](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)

比如可以使用`BPF_PERF_OUTPUT`定义一个`Perf`事件类型的`BPF`映射map

https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_perf_output

```c
// 包含头文件
#include <uapi/linux/openat2.h>
#include <linux/sched.h>

// 定义数据结构
struct data_t {
  u32 pid;
  u64 ts;
  char comm[TASK_COMM_LEN];
  char fname[NAME_MAX];
};

// 定义性能事件映射map
BPF_PERF_OUTPUT(events);
```

然后在eBPF程序中，填充这个数据结构，并调用`perf_submit()`提交到刚刚定义的性能映射中

```c
// 定义kprobe处理函数
int hello_world(struct pt_regs *ctx, int dfd, const char __user * filename, struct open_how *how)
{
  struct data_t data = { };

  // 获取PID和时间
  data.pid = bpf_get_current_pid_tgid();
  data.ts = bpf_ktime_get_ns();

  // 获取进程名
  if (bpf_get_current_comm(&data.comm, sizeof(data.comm)) == 0)
  {
    bpf_probe_read(&data.fname, sizeof(data.fname), (void *)filename);
  }

  // 提交性能事件
  events.perf_submit(ctx, &data, sizeof(data));
  return 0;
}
```

> 以bpf开头的都是BPF提供的辅助函数，如：
>
> * `bpf_get_current_pid_tgid`: 用于获取进程的`pid`和`tgid`（线程组`ID`，同一个进程下的所有线程都是同一个`tgid`）,该函数返回一个`64`位的`uint`值，高32位为线程组`tgid`，低32为进程的`pid`
> * `bpf_ktime_get_ns`:  获取系统从启动以来执行的时间, 单位纳秒
> * `bpf_get_current_comm`: 获取进程名，并将进程名复制到预定义的缓冲区中
> * `bpf_probe_read`:  从指定指针处读取固定大小的数据，这里用于读取进程打开的文件名
>
> 如果理解`hello_world`函数的入参？
>
> * `struct pt_regs *ctx` : bcc默认的参数
> * ` int dfd, const char __user * filename, struct open_how *how`：这些参数都是`openat2`函数的参数
>
> 所以，编写`eBPF`程序的时候都可以在`ctx`后加入对应系统调用接口的入参即可，在`eBP`F执行的时候会自动进行参数绑定
>
> 为什么`data_t`的c结构体最后会在`python`中读取为`python`对象, 此过程透明?
>
> * `perf_submit`传入的c对象/结构会通过event方法自动转换为python对象(`bcc`脚本的功劳)

然后在用户态程序借助该map对应的辅助函数`open_perf_buffer()`，向其中传入一个回调函数处理从`perf`事件类型的`BPF`映射中读取到的数据:

```Python
from bcc import BPF

# 1) load BPF program
b = BPF(src_file="trace-open.c")
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")

# 2) print header
print("%-18s %-16s %-6s %-16s" % ("TIME(s)", "COMM", "PID", "FILE"))

# 3) define the callback for perf event
start = 0
def print_event(cpu, data, size):   # cpu, data, size 三个参数都是bbc框架默认使用的
    global start
    event = b["events"].event(data)
    if start == 0:
            start = event.ts
    time_s = (float(event.ts - start)) / 1000000000
    print("%-18.9f %-16s %-6d %-16s" % (time_s, event.comm, event.pid, event.fname))

# 4) loop with callback to print_event
# 定义一个events的Perf事件映射，然后循环读取
b["events"].open_perf_buffer(print_event)  # 传入回调函数jie
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```

> 注意：`events`要和`eBPF`程序匹配，只是一个自定义的map名字
>
> `perf_buffer_poll`是不是一个非阻塞函数？
>
> * 其作用就是对所有的`perf`缓冲区`buff`调用回调函数，然后结束，所以外层要用一个for循环

执行结果：

![image-20220519203413623](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519203413623.png)

结果较为清晰

流程总结：

* 在`eBPF`程序中定义要捕获的对象结构
* 定义`kprobe`处理函数，通过`bpf`提供的辅助函数获取必要信息填充对象结构
* 将填充好的对象结构放入/提交到`map`中存储（给用户态程序获取做准备）
* 用户态程序通过在`open_perf_buffer`传入回调函数并编写逻辑，处理从缓冲区中读取的`map`中的对象

## 5.2 linux内核源码自带样例

### 1. 环境准备

需要先下载对应自己`linux`内核对应版本的源代码，我这里采用`apt`仓库维护的方式自己下载（其他方式见：https://davidlovezoe.club/wordpress/archives/988）

```shell
# 先搜索
sudo apt-cache search linux-source
linux-source - Linux kernel source with Ubuntu patches
linux-source-4.15.0 - Linux kernel source for version 4.15.0 with Ubuntu patches
linux-source-4.18.0 - Linux kernel source for version 4.18.0 with Ubuntu patches
linux-source-5.0.0 - Linux kernel source for version 5.0.0 with Ubuntu patches
linux-source-5.3.0 - Linux kernel source for version 5.3.0 with Ubuntu patches
# 再安装
sudo apt install linux-source-4.15.0
cd /usr/src/linux-source-4.15.0
sudo tar xjf linux-source-4.15.0.tar.bz2 
cd linux-source-4.15.0/
```

内核中所有的样例就在`<根目录>/samples/bpf`

![image-20220518102408314](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518102408314.png)

### 2. 开始编译

> 在真正开始编译工作之前，请确保你的实验环境已经安装`clang`和`llvm`
>
> - `clang >= version 3.4.0`
> - `llvm >= version 3.7.1`

所谓编译其实就是利用`Makefile`中写好的命令安装

```shell
# 切换到内核源代码根目录
cd linux_sourcecode/
# 生成内核编译时需要的头文件
sudo make headers_install
# 可视化选择你想为内核添加的内核模块，最终生成保存了相关模块信息的.config文件，为执行后面的命令做准备
sudo make menuconfig
# 使用make命令编译samples/bpf/目录下所有bpf示例代码，注意需要加上最后的/符号
sudo make samples/bpf/ # or  make M=samples/bpf
```

### 3. 执行测试

编译完成的所有样例都可以直接执行尝试：

![image-20220518174209994](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518174209994.png)

## 5.3 golang编写相关demo

> 参考：
>
> * https://blog.csdn.net/susu_xi/article/details/124147863
> * https://blog.csdn.net/susu_xi/article/details/124202867

### 1. 监控文件被打开

BPF工作的一个简易的流程是：`BPF`程序注入内核hook点之后，当hook点处的系统调用被调用时，BPF程序就会执行

`bpf`经常会使用`bpf_trace_printk`来完成打印/输出到标准输出，该函数会将需要打印的内容输出到`trace_pipe`中

所以查看`trace_pipe`文件的输出可以有两种方式：

* 监听`trace_pipe`文件
* `Golang`读取`trace_pipe`通道数据

这里使用第一种

核心要使用的包就是[github.com/iovisor/gobpf/bcc](https://github.com/iovisor/gobpf)

安装依赖

```shell
go get -u "github.com/iovisor/gobpf/bcc"
go get -u "github.com/iovisor/gobpf/pkg/tracepipe"
```

`hello.go`

```go
package main

import (
	"fmt"
	bpf "github.com/iovisor/gobpf/bcc"
	"github.com/iovisor/gobpf/pkg/tracepipe"
	"os"
)

import "C"

const source string = `
#include <uapi/linux/ptrace.h>

int kprobe__do_sys_open(struct pt_regs *ctx, void *dummy, char* fname) 
{
	char buf[256];
	bpf_probe_read(&buf, sizeof(buf), (void *) fname );
  u32 pid = bpf_get_current_pid_tgid() >> 32;
  bpf_trace_printk("pid=%d, file= %s\n", pid, &buf);
	return 0;
}
`

func main() {
  // 加载模块
	m := bpf.NewModule(source, []string{})
	defer m.Close()
	// 加载探针
	kp, err := m.LoadKprobe("kprobe__do_sys_open")
	if err != nil {
		fmt.Printf("Failed to load kprobe count: %s\n", err)
		os.Exit(1)
	}
	
  // 对应系统调用函数埋入探针
	err = m.AttachKprobe("do_sys_open", kp, -1)
	if err != nil {
		fmt.Printf("Failed to attach kprobe to strlen: %s\n", err)
		os.Exit(1)
	}

  // 逐行读取tracepipe中的数据，如果输出没有换行，就不会读取到数据
	tp, err := tracepipe.New()
	if err != nil {
		fmt.Printf("tracepipe.New err: %s\n", err)
		os.Exit(1)
	}

	defer tp.Close()
	channel, errChannel := tp.Channel()
	for {
		select {
		case event := <-channel:
			fmt.Printf("%+v\n", event)
		case err := <-errChannel:
			fmt.Printf("%+v\n", err)
		}
	}
}
```















## 5.4 bcc-tools中的`profile`详解





















# 四、遇到的问题总结

## 4.1 环境构建遇到的问题

### 1. BCC工具编译

错误描述：编译`bcc`时`cmake Warning`：

![image-20220519110150982](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519110150982.png)

原因：缺少依赖

解决：这些错误其实都不影响后面的编译（可能部分功能会影响，例如`debug`），同样会构建成功

> 相关参考`issue`:
>
> * https://github.com/iovisor/bcc/issues/3601
> * https://github.com/isage/lua-imagick/issues/16

```shell
sudo apt install libdebuginfod-dev    # 我尝试了并不能直接下载
sudo apt install libluajit-5.1-dev
sudo apt install arping netperf
```

---

错误描述：`make`编译阶段提示`getName()`函数不带参数，而`llvm-6.0`版本中的调用却带参数

![image-20220517202125777](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517202125777.png)

原因：使用的`llvm`的版本太低了，我的版本是`6.0.0`

![image-20220517202400162](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517202400162.png)

根据issue，bcc将逐渐不支持旧版本的`llvm`，所以现在最好升级到`11.0`以上([相关issue](https://github.com/iovisor/bcc/issues/3808))

同样的问题`issue`：https://github.com/iovisor/bcc/issues/3881

解决方法（二选一）：

1. 升级`llvm`的版本（一劳永逸）

   几种安装方式都行：

   > 参考自：https://zhuanlan.zhihu.com/p/102028114

   - 源码安装方式

       ```shell
     # 采用源码安装方式
     cd /usr/local
     wget https://github.com/llvm/llvm-project/releases/download/llvmorg-13.0.0/llvm-project-13.0.0.src.tar.xz
     tar xvf llvm-project-13.0.0.src.tar.xz
     mv llvm-project-13.0.0.src/ llvm
     cd llvm
     mkdir build && cd build
     # cmake生成编译信息
     cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS="clang;lldb" -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_BUILD_TYPE="Release" -DLLVM_INCLUDE_TESTS=OFF -DCMAKE_INSTALL_PREFIX="/usr/local/llvm" ../llvm
     # 编译
     make     # 注意：这里会非常慢，耐心等待
     # 安装
     make install
       ```

       其中因为我的`cmake`版本过低，所以要先更新一下`cmake`

       ![image-20220517205409738](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517205409738.png)

       详细见：https://blog.csdn.net/u013925378/article/details/106945800

     ---

     `make`编译过程中到这一步就会失败：

     ![image-20220518111936362](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518111936362.png)

     并且会导致我的服务器网络出现问题无法连接，最后被`kill`掉

     现象：这里的编译可能会重启服务器的网卡/网络，因为会导致与服务器的ssh链接断开

     原因：目前还未解决，尝试使用第二种方式

   - 使用官方脚本安装

     ```shell
     wget https://apt.llvm.org/llvm.sh
     chmod +x llvm.sh
     # 后面的参数是版本号
     sudo ./llvm.sh 11
     ```

2. download the release [0.24.0](https://github.com/iovisor/bcc/releases/tag/v0.24.0), it will compile successfully. （临时解决）

   ```shell
   wget https://github.com/iovisor/bcc/releases/download/v0.24.0/bcc-src-with-submodule.tar.gz
   tar xvf bcc-src-with-submodule.tar.gz 
   mkdir build && cd build
   cmake ..
   make
   sudo make install
   cmake -DPYTHON_CMD=python3 .. # build python3 binding
   # 将目录添加到目录堆栈顶，简单理解就是进入这个目录，和popd一起使用
   pushd src/python/   
   make
   # 安装到/usr/lib/python3/dist-packages/下
   sudo make install
   popd	# 返回栈顶目录
   ```

---

问题描述：编译`bcc`的时候`cmake ..`失败

![image-20220519191926700](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519191926700.png)

原因：检查`ls /usr/lib/llvm-13/lib/`下是否有如下所需要的依赖（`.a`后缀）

![image-20220519192513782](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519192513782.png)

如果没有的话安装`sudo apt-get install libclang-13-dev` (中间的版本号对应你的`clang`版本)

## 4.2 bcc demos 遇到的问题

**注意默认使用的`python`源,** 检查不要用`anaconda3`的`python3`或者是系统默认的`python2`，否则会报如下错误：

`Helloword.py`直接执行`sudo ./hello_world.py"` 执行的`python`引擎是文件第一行`/usr/bin/python`, 请确保`bcc`绑定的`python`是这个

![image-20220518113919028](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518113919028.png)

```shell
# 检查
ls -l `which python`
lrwxrwxrwx 1 lywh lywh 9 3月  28 14:28 /home/lywh/anaconda3/bin/python -> python3.9  # 不能用这个
```

原因：之前bcc安装是安装在系统`python3`下`/usr/lib/python3/dist-packages/bcc`，所以会找不到

解决：`sudo /usr/bin/python3 ./helloword.py `

---

错误：`apt-get update `错误

![image-20220518123206678](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518123206678.png)

解决：在`apt`密码管理器中添加`NO_PUBKEY`

`sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 15CF4D18AF4F7421`

## 4.2 linux内核源码自带样例遇到的问题

错误描述：编译`samples/bpf/`出现问题：

![image-20220518164510584](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518164510584.png)

解决方法：因为是测试的代码，所以可以屏蔽掉，https://davidlovezoe.club/wordpress/archives/988

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518164951510.png" alt="image-20220518164951510" style="zoom:57%;" />

将71行的`#ifdef`改为`#if` (注意，谨慎修改)

----

错误描述：

写`Python + C ebpf 监控打开文件`demo的时候执行`sudo ./hello.py`出错：

![image-20220519133845251](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519133845251.png)

原因：可能是执行的`Python`版本用到2.7了。。

解决：在程序末尾加上`;`后可以执行，后来删除掉分号后也可以执行了，很诡异

错误描述：

![image-20220519133731209](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519133731209.png)

原因：版本内核太低`4.15`

https://man7.org/linux/man-pages/man2/openat2.2.html#VERSIONS

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519140256941.png" alt="image-20220519140256941" style="zoom:67%;" />

解决：

