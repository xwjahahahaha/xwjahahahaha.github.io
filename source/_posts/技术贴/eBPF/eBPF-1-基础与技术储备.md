---
title: eBPF-1-基础与技术储备
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
> * https://www.ebpf.top/post/ebpf-overview-part-3/
> * 《BPF之巅-洞悉linux系统和应用性能》

<!-- more -->

# 一、背景与基础

## 1.1 基础之基础

在学习eBPF基础之前的基础

### 1. `gcc、llvm、clang`等是什么？

编译器分为三个：

* 前端`frontEnd` ：词法和语法分析，将源代码转换为抽象语法树
* 优化器`Optimizer`： 在前端的基础上，对中间代码进行优化
* 后端`backEnd`：将优化后的中间代码转化为各自平台的机器码

`GCC、llvm、clang`

* `GCC`: `GNU Compiler Collection`，GNU编译器套装，是一套由 GNU 开发的编程语言编译器，最先支持C语言，后来演进可处理` C++、Fortran、Pascal、Objective-C、Java`等语言

* `llvm`：`Low Level Virtual Machine`, 可用作多种编译器的后端使用（能够让程序语言的编译器优化、链接优化等），支持任何编程语言的静态和动态编译，可以使用它编写自己的编译器

  > LLVM的命名最早源自于底层虚拟机（`Low Level Virtual Machine`）的首字母缩写，由于这个项目的范围并不局限于创建一个虚拟机，这个缩写导致了广泛的疑惑。LLVM开始成长之后，成为众多编译工具及低端工具技术的统称，使得这个名字变得更不贴切，开发者因而决定放弃这个缩写的意涵，现今LLVM已单纯成为一个品牌，适用于LLVM下的所有项目

* `clang`：`clang`是`llvm`的前端，可以用来编译`c、c++、ObjectiveC`等语言，其以`llvm`作为后端支持，高效易用，并且与`IDE`有很好的结合

### 2. `.elf`对象文件处于程序编译的什么阶段？

**可执行与可链接格式** （英语：Executable and Linkable Format，缩写 ELF，此前的写法是 **Extensible Linking Format**），常被称为 **ELF格式**，在[计算](https://zh.m.wikipedia.org/wiki/计算_(计算机科学))中，是一种用于[可执行](https://zh.m.wikipedia.org/wiki/可执行)文件、[目标代码](https://zh.m.wikipedia.org/wiki/目标代码)、[共享库](https://zh.m.wikipedia.org/wiki/共享库)和[核心转储](https://zh.m.wikipedia.org/wiki/核心转储)（core dump）的标准[文件格式](https://zh.m.wikipedia.org/wiki/文件格式)。

* 编译：

  高级语言 -> 汇编语言 -> 机器语言 ： 高级语言最终变为机器语言这样的过程可以统称为编译，具体的方法基本有两种：编译型和解释型

  据此也可分为两大类：一种是编译型语言，例如`C，C++，Java`，另一种是解释型语言，例如`Python、Ruby、MATLAB 、JavaScript`

* 四个步骤：

  - 预处理（Preprocessing）

  - 编译（Compilation）

  - 汇编（Assembly）

  - 链接（Linking）

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519092739182.png" alt="image-20220519092739182" style="zoom:67%;" />

`.so`文件：动态链接库，也称为共享库, 不能直接运行

## 1.2 `BPF`基础与技术储备

### 1. 网络监控工具发展历程？发展原因？

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

详细完整的历程：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/bpf-timeline-high-watermark.png)

### 2. BPF是什么？eBPF是什么？

* **BPF**

`Berkeley Packet Filter` 伯克利包过滤器，在伯克利大学诞生，为BSD操作系统而开发，后来一直沿用

原始的`BPF`是设计用来抓取和过滤符合特定规则的网络包, 过滤器是通过程序实现的（用户定义过滤器表达式），并在基于寄存器的虚拟机上运行，使得包过滤直接在内核中执行，避免向用户态复制

![image-20220521175501426](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521175501426.png)

基本原理是<font color='#e54d42'>`BPF`提供了一种在内核事件和用户程序事件发生时，安全注入代码的机制(运行一小段程序的机制)</font>, 这样就让非内核开发人员也可以对内核进行监控和控制

> 著名的`tcpdump`就是基于此实现：
>
> <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521175742989.png" alt="image-20220521175742989" style="zoom:50%;" />

* **eBPF**

`extend Berkeley Packet Filter` 拓展`BPF`, 它演进成为了一套通用执行引擎

BPF的功能升级版，[Alexei Starovoitov](https://twitter.com/alexei_ast)为了更好地利用的现代硬件，提出了扩展型BPF（eBPF）设计

`eBPF`的基本架构在于：<font color='#e54d42'>借助即时编译器JIT，在内核中运行了一个执行引擎（因为编译后直接在cpu上运行，所以相比虚拟机可能执行引擎更为合适），被验证安全的`eBPF`指令最终都会被内核执行</font>

> * 老版本的`Berkeley Packet Filter`目前称为`cBPF`: `classic BPF`，目前基本废弃，新的`Linux`内核只运行`eBPF`, 内核会将`cBPF`透明的转换为`eBPF`
> * 特别的，当前`cBPF`和`eBPF`都统称为`BPF`了，或者说提到`BPF`不做特殊说明就是`eBPF`，`BPF`应该看作是一个技术名词而不是单纯的缩写（或者说包过滤器了）

* 二者比较

  ![image-20220521180300520](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521180300520.png)

### 3. eBPF做了哪些提升？

* 速度更快

  * 更贴近硬件的指令集架构ISA，特别是适应64位寄存器以及提升使用的寄存器数量（从2个提升到10个），这样有助于即时编译提高性能；此外`eBPF`的指令仍然运行在内核中, 不需要向用户态复制数据（也是BPF拥有的），提高了事件处理效率。

    > 对于某些网络过滤器微基准测试上显示，`eBPF`在 `x86-64`架构上的速度比旧的经典`BPF` (`cBPF`)实现最高快四倍，大多数都在1.5倍

* 支持使用一些受限的系统调用

  * 新的`BPF_CALL`指令，可以更廉价地调用内核函数

* 拓展性
  * 功能从包过滤拓展到更多（跟踪过滤、链路追踪、可观测）

    > 2014年的[daedfb22451d](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=daedfb22451dd02b35c0549566cbb7cc06bdd53b)这次代码提交中，eBPF虚拟机[直接暴露给了用户空间来调用](https://lwn.net/Articles/603983/)（或者说运行用户空间代码）

### 4. eBPF感知代码流程？能够使用eBPF做什么？

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

### 5. eBPF如何保证安全性？（内核验证器）

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

### 6. eBPF虚拟机的内部架构是什么？

BPF虚拟机的内部架构

![image-20220521192823981](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521192823981.png)



### 7. eBPF的执行流程是什么？

下面就是整个`eBPF`程序的工作流程图：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/ebpfworkflow-20220517161733498.png)

或者是这张图：

![image-20220521142259971](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521142259971.png)

整体的流程可以总结为：

1. 通过`llvm`将程序编译为`eBPF`字节码
2. 通过`bpf`系统调用交给内核(也叫`load`加载)
3. 内核在接受字节码之前会进行检查检验
4. 检验通过的字节码提交到即时编译器中运行

### 8. eBPF的插桩类型有哪些？

插桩是什么？

* 也就是对某个函数进行埋点为了跟踪系统某些事件的发生

具体的事件源：

![image-20220522155105084](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220522155105084.png)

具体类型？

#### 动态插桩：kprobes && uprobes

动态插桩：对正在运行的软件插入观测点的能力；如果软件未启动，那么动态插桩的开销为0；具体插桩的位置可以是软件栈中所有函数中任一个

与`debugger`调试器的区别？

* 调试器是在任意指令地址插入断点的技术，动态插桩则是在软件记录完信息后自动继续执行，不会把控制权交给调试器（无侵入？）

动态插桩有两种探针：

* 内核态插桩`kprobes`

  * 可以对任意内核函数进行插桩，还可以对内部指令进行插桩，可以在实时生产环境中使用无需重启系统或内核
  * `kretprobes`: 对内核函数返回时进行插桩以获取返回值；两者配合可以用来计算函数执行时间

  * 上层追踪器/前端的使用
    * `BCC` : `attach_kprobe()`和`attach_kretprobe()`
    * `bpftrace`： `kprobe`和`kretprobe`探针类型
    * 注：`BCC`的`kprobe`支持函数开始以及某一偏移量放置探针，而`bpftrace`只支持函数开始位置插桩

* 用户态插桩`uprobes`

  * 几乎类似于`kprobes`，但是是对用户态程序
  * 上层追踪器/前端的使用
    * `BCC` : `attach_uprobe()`和`attach_uretprobe()`
    * `bpftrace`： `uprobe`和`uretprobe`探针类型
    * 注：`BCC`的`uprobe`支持函数开始以及某一偏移量放置探针，而`bpftrace`只支持函数开始位置插桩


<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521170720532.png" alt="image-20220521170720532" style="zoom:80%;" />

缺点：

* 注意：虽然插桩性能消耗很小(已做了优化)， 但是超高频事件(百万级别)插桩后还是会影响性能，所以建议插桩低频函数或者在测试环境下插桩高频函数
* 可能随着软件的更新，被插桩的函数也可能会被重新命名或移除（并且在旧版插桩使用的时候可能没有任何输出提示），要重新进行适配 
* 编译器的内联优化，将函数内联处理导致被插桩函数无法被插桩

#### 静态插桩：tracepoint && USDT

为了解决动态插桩的问题，出现了静态插桩，静态插桩会将稳定的事件名字编码到软件代码中，由开发者维护

具体有两种：

* `tracepoint`(内核跟踪点)

  * 用来对内核进行静态插桩，内核开发者在某些位置特定放了插桩点，最终会被编译到内核的二进制文件中，用于跟踪
  * 如果有内核跟踪点可以满足需求，优先使用它而不是`kprobes`, 因为前者更加稳定
  * 跟踪点一般格式：`<子系统>:<事件名>`
  * 上层追踪器/前端的使用
    * `BCC` : `TRACEPOINT_PROBE()`
    * `bpftrace`：跟踪点探针类型
  * 原始跟踪点`BPF_RAW_TRACEPOINT`: 避免一些没必要的参数传递，提高性能 (比`kprobes`稳定，因为其探针函数名是稳定的，参数可能是不稳定的)

* `USDT`(用户态静态定义跟踪插桩技术 `user-level statically defined tracing`)

  * 用户空间版的跟踪点机制
  * 依赖于外部系统跟踪器唤起，如果没有，应用中的`USDT`也不会做任何事 (不用担心写在源代码中的探针会带来性能开销，如果外部没启用，那么该代码会被直接跳过)
  * 上层追踪器/前端的使用
    * `BCC` : `USDT().enable_probe()`
    * `bpftrace`：`USDT`探针类型

  * 动态`USDT`: 
    * 对于编译型语言在静态`USDT`直接编译到二进制文件中，而对于动态编译型语言(运行时编译，例如`java/jvm`)，动态`USDT`就起作用了
    * `JVM` 已经内置在`C++`代码库中许多动态`USDT`探针, 包括GC、类加载等；但是`java`这种依靠`JIT`即时编译的方式难以使用动态`USDT`（因为动态`USDT`是一个动态链接库，需要提前编译好、带一个包含了探针描述的`notes`的`ELF`文件）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521170732924.png" alt="image-20220521170732924" style="zoom:80%;" />

缺点：增加维护成本，数量一般较少

> 动态插桩、静态插桩的基本原理将在后续介绍

### 9. 如何理解eBPF中的Map?

`map`又称为映射，`BPF`程序可以利用其进行**存储**

核心职责：**存储eBPF运行时状态即用户程序与运行在内核的`eBPF`程序交互载体**

运行在内核的`eBPF`程序收集目标状态存储在`map`中，随后用户程序再从映射中读取这些状态

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518213529969.png" alt="image-20220518213529969" style="zoom:67%;" />

### 10. eBPF程序的限制有哪些？

保证安全性就意味着限制，所以`eBPF`并不是万能的

具体的限制包括：

* 校验限制：必须通过校验才可以执行，并且不能包含不可到达的指令
* 内核函数限制：内核函数只可以调用API定义的辅助函数
* 存储限制：`eBPF`程序栈最大只有512字节，需要更大的存储要借助映射存储
* 指令数量限制：内核5.2之前支持4096条指令，之后支持100万条
* 移植性/适配性：不同内核版本下的移植可能需要修改源码并重新编译

具体内核版本要求和函数功能支持见：https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md

### 11. eBPF程序编写的组件层次是什么样的？相关工具的实现程度？

eBPF程序编写的整体的组织架构：

* **后端**：这是在内核中加载和运行的 eBPF 字节码。它将数据写入内核 map 和环形缓冲区的**数据结构**中。
* **加载器：它将字节码后端**加载到内核中。通常情况下，当加载器进程终止时，字节码会被内核自动卸载。
* **前端：从数据结构**中读取数据（由之前的**后端**写入）并将其显示给用户。
* **数据结构**：这些是**后端**和**前端**之间的通信手段。它们是由内核管理的 map 和环形缓冲区，可以通过文件描述符访问，并需要在**后端**被加载之前创建。它们会持续存在，直到没有更多的**后端**或**前端**进行读写操作

eBPF 程序可以更加复杂：多个**后端**可以由一个（或单独的多个）**加载器**进程加载，写入多个**数据结构**，然后由多个**前端**进程读取，所有这些都可以发生在一个跨越多个进程的用户 eBPF 应用程序中。

![img](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521142814379-20220524001311740.png)

### 12. 工具集`llvm`、`BCC`、`bpftrace`、`IOVisor`层次架构与比较

与之相关的知名工具包括：

* 层级一：`llvm`

  一个编译器，帮助高级语言(`c、GO、Rust`)的子集被编译成为`eBPF`字节码程序；将“”受限的C语言“”(符合eBPF验证规范的)编译为`ELF`对象文件，随后即可通过`bpf`等系统调用实现加载到内核中；受限的c语言的引入带来的好处是更加容易用高级语言编写，带来的坏处在于**加载器**程序的复杂性变高(需要解析`ELF`对象)

* 层级二：`BCC`

  一个`BPF`工具链集合（`libbcc、libbpf`的前身）, 解决了上述整体四个组织架构之间的整合关系，尽量实现自动化和标准化，其本身组成分为两个部分：

  * 编译器集合（BCC 本身）：这是用于编写 BCC 工具的框架
  * `BCC-tools`：这是一个不断增长的基于 `eBPF `且经过测试的程序集，提供了使用的例子和手册(基于`BCC`开发的成熟工具)

  重新定义了组织结构，`eBPF` 程序组件在` BCC `组织方式如下：

  - **后端**和**数据结构**：用 “受限制的C语言” 编写(本身也依赖于`llvm/clang`进行编译成`eBPF`程序)。可以在单独的文件中，或直接作为多行字符串存储在**加载器/前端**的脚本中，以方便使用(很多方便的宏定义)。参见：[语言参考](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#bpf-c)

  - **加载器**和**前端**：可用非常简单的高级` python/lua `脚本编写

    例如`python`的`BPF(text='BPF_program'))`即可加载`BPF`字节码到内核, 详细参见：[语言参考](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#bcc-python)

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521144808474.png" alt="image-20220521144808474" style="zoom:57%;" />

* 层级三：`bpftrace`

  在某些用例中，`BCC` 仍然过于底层，例如在事件响应中检查系统时，时间至关重要，需要快速做出决定，而编写 `python/“限制性 C” `会花费太多时间，因此 `BPFtrace` 建立在 `BCC `之上，通过特定领域语言（受 `AWK` 和` C `启发实现的一种自定义的高级语言）提供更高级别的抽象，根据[声明帖](http://www.brendangregg.com/blog/2018-10-08/dtrace-for-linux-2018.html)，该语言类似于 DTrace 语言实现，也被称为 DTrace 2.0，并提供了良好的介绍和例子。

  例如：这个单行 shell 程序统计了每个用户进程系统调用的次数（访问[内置变量](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#1-builtins)、[map 函数](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#map-functions) 和[count()](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#2-count-count)文档获取更多信息）

  `bpftrace -e 'tracepoint:raw_syscalls:sys_enter {@[pid, comm] = count();}'`

  局限性：上层的封装抽象会受限于特殊的功能需求，在某些场景下很难直接用一个`bpftrace`命令实现，所以还是需要`BCC`工具

  > `BCC`与`bpftrace`适用场景对比：
  >
  > * `BCC`: 开发复杂的脚本和作为后台进程使用
  > * `bpftrace`：编写强大的单行程序、短小的脚本使用

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521161933299.png" alt="image-20220521161933299" style="zoom:67%;" />

* 层级四：云环境中的`eBPF`-`IOVisor`

  [IOVisor](https://www.iovisor.org/) 是 Linux 基金会的一个[合作项目](https://www.linuxfoundation.org/projects/)，基于本系列文章中介绍的 eBPF 虚拟机和工具。它使用了一些非常高层次的热门概念，如 “通用输入/输出”，专注于向云/数据中心开发人员和用户提供 eBPF 技术。其重新定义了概念，更加模块化、组件化：

  - 内核` eBPF `虚拟机成为 “IO Visor 运行时引擎”
  - 编译器后端成为 “`IO Visor` 编译器后端”
  - 一般的` eBPF `程序被重新命名为 “`IO `模块”  ，例如：实现包过滤器的特定 `eBPF` 程序称为 “`IO `数据平面模块/组件”等等 
  - `IOVisor` 项目创建了 [Hover 框架](https://github.com/iovisor/iomodules)，也被称为 “`IO `模块管理器”，专门用于管理`eBPF`程序/`IO`模块的用户空间的后台服务程序, 目标是类似于`Docker daemon`发布/获取镜像的方式, 能够将` IO `模块推送和拉取到云端，分发和部署到多台主机

### 13. `BPF`运行时模块组成？

* 解释器
* `JIT`即时编译器：将BPF指令动态的转换为本地化指令
* `verifer`验证器：用于eBPF程序指令的安全检查, 保护内核安全

## 1.3 `eBPF`可观测性方向基础

### 1.`eBPF`可观测性的术语

目前`eBPF`应用领域分别是网络、可观测性、安全。对于可观测性有以下术语解释：

* 跟踪`trace、snoop`: 基于事件的跟踪，记录追踪系统发生的一系列事件
* 采样`sampling、profiling`: 通过获取全部观测信息的子集（定时采样）来描绘大值的图像； 采样工具的优点在于性能开销比跟踪工具小，缺点就在于可能会遗漏部分关键事件
* 可观测性`observability`：通过全面观测来理解一个系统，只要可以实现此目标就可归为此类；上面两种和固定计数器工具都包括在其内，但是不包括基准(`benchmark`)测量工具

### 2. 基础库`libbcc`、`libbpf`的理解？

基于这两个基础库分别实现了：`BCC`、`bpftrace`, `libbpf`目前已经是内核代码的一部分

### 3. `eBPF`与传统的性能分析工具的不同点？

eBPF是在内核层面使用非常低的损耗实现观测，这与传统的用户态性能分析工具有着本质区别, ：

* 高效：`eBPF`同时具备高效率
* 安全性：`eBPF`同时具备生产环境安全性特点
* 侵入性：`eBPF`已经在内核中，可以直接在生产环境中使用，无需添加新的内核组件

**为什么效率高？**

* 举个例子，需要追踪当前内核I/O尺寸分布的数据，区别就在于：（将工作都移植到到了内核中，减少内核、用户态之间数据复制量）

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521193907607.png" alt="image-20220521193907607" style="zoom:67%;" />

  * 不使用`BPF`：对于每个事件都需要向缓冲区中写一条记录到`perf buffer`, 然后用户程序<font color='#e54d42'>周期性</font>拷贝所有缓冲区数据到用户态生成直方图
  * 使用`BPF`: 对于每次事件，运行BPF程序，只获取字节字段，保存在自定义的`Map`映射数据结构中, 用户空间<font color='#e54d42'>一次性</font>读取BPF直方图映射表并输出结果

  效率提升显著，以至于工具的额外开销减小到可以在生产环境下直接使用

**为什么安全？**

* 相比于传统性能检测工具可能会为了高效而需要修改内核等，`BPF`采用的验证器和受限的调用更加正规和安全

相比于内核模块（`tracepoint、kprobes`这些内核模块已经出现很多年了）：

* 安全性：有自带的安全性检查
* 丰富的数据结构支持
* 稳定性：`eBPF`程序`CO-RE`一次编译到处运行，BPF指令集、映射表结构、辅助函数等相关基础设施都是稳定的`ABI`;(当然也包括不稳定的因素，例如`kprobes`，但是也有对应的解决方案)
* 原子性替换`BPF`程序的能力

所以适用场景也有所区别:

* 传统性能分析工具可以作为分析的起点
* 然后再使用`BPF`跟踪工具进一步分析

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220521164355616.png" alt="image-20220521164355616" style="zoom:67%;" />

### 4. 调用栈回溯(`stack-trace`)

#### 1. 函数调用栈作用？

* 用于理解某些事件产生的代码路径
* 剖析内核和用户代码，观测执行开销的具体位置

`BPF`的支持：

* 专用的存储调用栈信息的映射表结构
* 保存基于帧指针或基于`ORC`的调用栈回溯信息

#### 2. 基于帧指针的栈回溯原理

一个惯例：

* 函数调用栈的头部始终保存在寄存器`RBP`中(`x86_64`)
* 函数调用的返回地址永远位于`RBP`的值指向的位置+固定偏移量(`+8`)

追溯过程：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220522150153663.png" alt="image-20220522150153663" style="zoom:50%;" />

读取`RBP`的值遍历以此值为头部的链表，同时在固定偏移位置获取返回地址(`+8`)，从而轻松的实现栈回溯

> 注：
>
> * `RBP`不一定在所有体系结构中都为保存栈头部地址，也可以用作通用寄存器
> * `gcc`编译时默认不开启帧指针，通过`-fno-omit-frame-pointer`来开启，开启的性能消耗很低并且给栈回溯带来的收益很大
> * 帧指针并不是唯一的栈回溯方法，还可以通过调试信息(`debuginfo`)、`LBR`以及`ORC`实现

#### 3. 其他栈回溯方法

* 调试信息(`debuginfo`)

  将软件的额外调试信息以软件的调试信息包来提供(包含了`DWARF`格式的`ELF`调试信息)，`BPF`目前还不支持, 此技术非常耗费处理器资源

* `LBR`

  最后分支记录(`Last branch record`)技术是intel处理器提供的特性，在硬件层面上讲程序分支、函数调用分支信息保存；`BPF`未来可能会支持

* `ORC`

  `Oops`回滚能力(`Oops Rewind Capability`), 相比于`DWARF`格式，此格式对处理器的要求比较低, 目前`linux`内核已经有相关支持, 如果是寄存器数量受限的体系结构上，可以用其代替帧指针; `BPF`中可以通过`perf_callchain_kernel()`函数支持, 但是目前没有对用户态支持

### 5. 性能监控计数器`PMC`是什么？

别名：`PIC、CPC、PMU event`

作用：

* 处理器上的硬件可编程计数器

两种模式：

* **计数**：`PMC`跟踪事件发生的频率，内核随时可以读取，开销基本可以忽略不计
* **溢出采样**：`PMC`所监控的事件发生到一定次数时通知内核（中断），让内核获取额外的状态





