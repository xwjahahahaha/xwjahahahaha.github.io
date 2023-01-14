---
title: Seccomp、BPF与容器安全
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-11 10:21:02
---

[TOC]


<!-- more -->

> 转载自：
>
> * https://xz.aliyun.com/t/11480#toc-1

本文详细介绍了关于seccomp的相关概念，包括seccomp的发展历史、`Seccomp BPF`的实现原理已经与seccomp相关的一些工具等。此外，通过实例验证了如何使用seccomp bpf 来保护Docker的安全。

# 一、简介

seccomp（全称securecomputing mode）是linux kernel支持的一种安全机制。在Linux系统里，大量的系统调用（systemcall）直接暴露给用户态程序。但是，并不是所有的系统调用都被需要，而且不安全的代码滥用系统调用会对系统造成安全威胁。通过seccomp，我们限制程序使用某些系统调用，这样可以减少系统的暴露面，同时是程序进入一种“安全”的状态。

## Seccomp 的发展历史

2005年，Linux 2.6.12中的引入了第一个版本的seccomp，通过向`/proc/PID/seccomp`接口中写入“1”来启用过滤器，最初只有一个模式：严格模式（strict mode），该模式下只允许被限制的进程使用4种系统调用：*read()*, *write()*, *_exit()*, 和 *sigreturn()* ，需要注意的是，`open()`系统调用也是被禁止的，这就意味着在进入严格模式之前必须先打开文件。一旦为程序施加了严格模式的seccomp，对于其他的所有系统调用的调用，都会触发`SIGKILL`并立即终止进程。

2007年，Linux 2.6.23 内核使用`prctl（）`操作代替了`/proc/PID/seccomp`接口来施加seccomp，通过`Prctl (PR_SET_SECCOMP,arg)`修改调用者的seccomp模式；`prctl(PR_GET_SECCOMP)`用来获取seccomp的状态，返回值为0时代表进程没有被施加seccomp，但是如果进程配置了seccomp，则会由于不能调用`prctl(）`导致进程中止，那就没有其他返回值了？？

2012年，Linux 3.5引入了”seccomp mode 2“，为seccomp带来了一种新的模式：过滤模式（ filter mode ）， 该模式使用 Berkeley 包过滤器 (BPF) 程序过滤任意系统调用及其参数,使用该模式，进程可以使用 `prctl (PR_SET_SECCOMP, SECCOMP_MODE_FILTER, ...)`来指定允许哪些系统调用。 现在已经有许多应用使用 seccomp 过滤器来对系统调用进行控制，包括 Chrome/Chromium 浏览器、OpenSSH、vsftpd 和 Firefox OS 。

2013年，Linux 3.8版本，在`/proc/PID/status`中添加了一个Seccomp字段， 可以通过读取该文件获取对应进程的 seccomp 模式的状态（0 表示禁用，1 表示严格，2 表示过滤）。

```c
/* Valid values for seccomp.mode and prctl(PR_SET_SECCOMP, <mode>) */
#define SECCOMP_MODE_DISABLED   0 /* seccomp is not in use. */
#define SECCOMP_MODE_STRICT 1 /* uses hard-coded filter. */
#define SECCOMP_MODE_FILTER 2 /* uses user-supplied filter. */
```

```shell
null@ubuntu:~/seccomp$ cat /proc/1/status | grep Seccomp
Seccomp:        0
```

2014年，Linux 3.17 引入了`seccomp()`系统调用，`seccomp()`在`prctl()`的基础上提供了现有功能的超集， **增加了将进程中的所有线程同步到同一组过滤器的能力**，这有助于确保即使在施加seccomp过滤器之前创建的线程仍然有效。

## Seccomp + BPF

seccomp 过滤模式允许开发人员编写 BPF 程序来确定是否允许给定的系统调用，基于**系统调用号和参数（寄存器）值**进行过滤。当使用`seccomp()`或`prctl()`对进程施加`seccomp` 时**，需要提前将编写好的BPF程序安装到内核，之后每次系统调用都会经过该过滤器**。而且此过程是不可逆的， 因为安装过滤器实际上是声明任何后续执行的代码都不可信

BPF在1992年的`tcpdump`程序中首次提出，`tcpdump`是一个网络数据包的监控工具， 但是由于数据包的数量很大，而且将内核空间捕获到的数据包传输到用户空间会带来很多不必要的性能损耗，所以要对数据包进行过滤，只保留感兴趣的那一部分，而在内核中过滤感兴趣的数据包比在用户空间中进行过滤更有效。BPF 就是提供了一种进行内核过滤的方法，因此用户空间只需要处理经过内核过滤的后感兴趣的数据包 。

BPF定义了一个可以在内核内实现的虚拟机(VM)。该虚拟机有以下特性：

> - 简单指令集
>   - 小型指令集
>   - 所有的指令大小相同
>   - 实现过程简单、快速
> - 只有分支向前指令
>   - 程序是有向无环图(DAGs)，没有循环
> - 易于验证程序的有效性/安全性
>   - 简单的指令集⇒可以验证操作码和参数
>   - 可以检测死代码
>   - 程序必须以 Return 结束
>   - BPF过滤器程序仅限于4096条指令

BPF 程序在Linux内核中主要在`filter.h`和`bpf_common.h`中实现，主要的数据结构包括以下几个：

`Linux v5.18.4/include/uapi/linux/filter.h` -> [sock_fprog](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/filter.h#L24)

```c
struct sock_fprog { 										/* Required for SO_ATTACH_FILTER. */
    unsigned short      len;    				/* BPF指令的数量 */
    struct sock_filter __user *filter;  /*指向BPF数组的指针 */
};
```

这个结构体记录了**过滤规则个数与规则数组起始位置** , 而` filter `域指向了具体的规则，**每一条规则**的形式如下：

`Linux v5.18.4/include/uapi/linux/filter.h` -> [sock_filter](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/filter.h#L24)

```c
struct sock_filter {    /* Filter block */
    __u16   code;   /* Actual filter code */
    __u8    jt;     /* Jump true */
    __u8    jf; 		/* Jump false */
    __u32   k;      /* Generic multiuse field */
};
```

该规则有四个参数：

* `code`：过滤指令
* `jt`：条件真跳转
* `jf`：条件假跳转
* `k`：操作数

BPF的指令集比较简单，主要有以下几个指令：

> - 加载指令
> - 存储指令
> - 跳转指令
> - 算术逻辑指令
>   - 包括：ADD、SUB、 MUL、 DIV、 MOD、 NEG、OR、 AND、XOR、 LSH、 RSH
> - Return 指令
> - 条件跳转指令
>   - 有两个跳转目标，jt为真，jf为假
>   - jmp 目标是指令偏移量，最大 255

如何编写BPF程序呢？BPF指令可以手工编写，但是，开发人员定义了符号常量和两个方便的宏`BPF_STMT`和`BPF_JUMP`可以用来方便的编写BPF规则。

`Linux v5.18.4/include/uapi/linux/filter.h `-> [BPF_STMT&BPF_JUMP](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/filter.h#L45)

```c
/*
 * Macros for filter block array initializers.
 */
#ifndef BPF_STMT
#define BPF_STMT(code, k) { (unsigned short)(code), 0, 0, k }
#endif
#ifndef BPF_JUMP
#define BPF_JUMP(code, k, jt, jf) { (unsigned short)(code), jt, jf, k }
#endif
```

### BPF_STMT

`BPF_STMT`有两个参数，操作码(code)和值(k)，举个例子：

```c
BPF_STMT(BPF_LD | BPF_W | BPF_ABS, (offsetof(struct seccomp_data, arch)))
```

这里的操作码是由三个指令相或组成的，`BPF_LD`: 建一个 BPF 加载操作 ；`BPF_W`:操作数大小是一个字，`BPF_ABS`: 使用绝对偏移，即使用指令中的值作为数据区的偏移量,该值是体系结构字段与数据区域的偏移量 。`offsetof()`生成数据区域中期望字段的偏移量。

该指令的功能是**将体系架构数加载到累加器中**(系统调用号的值载入到累加器)

### BPF_JUMP

`BPF_JUMP` 中有四个参数：操作码、值(k)、为真跳转(jt)和为假跳转(jf)，举个例子：

```c
BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K , AUDIT_ARCH_X86_64 , 1, 0)
```

`BPF_JMP | BPF JEQ`会创建一个相等跳转指令，它将指令中的值（即第二个参数`AUDIT_ARCH_X86_64`）与累加器中的值（`BPF_K`）进行比较。判断是否相等，也就是说，如果架构是 `x86-64`，则跳过下一条指令（jt=1，代表测试为真跳过一条指令），否则将执行下一条指令（jf=0，代表如果测试为假，则**跳过0条指令，也就是继续执行下一条指令**）。

上面这两条指令**常用作系统架构的验证**

再举个实际例子，该示例用作过滤`execve`系统调用的过滤规则：

```c
struct sock_filter filter[] = {
    BPF_STMT(BPF_LD+BPF_W+BPF_ABS,0),           //将帧的偏移0处，取4个字节数据，也就是系统调用号的值载入累加器
    BPF_JUMP(BPF_JMP+BPF_JEQ,59,0,1),           //当A == 59时，顺序执行下一条规则，否则跳过下一条规则，这里的59就是x64的execve系统调用号
    BPF_STMT(BPF_RET+BPF_K,SECCOMP_RET_KILL),   //返回KILL  
    BPF_STMT(BPF_RET+BPF_K,SECCOMP_RET_ALLOW),  //返回ALLOW
};
```

> <font color='#39b54a'>可以看到如果当前的系统调用是59，那么顺序执行到下一行就会被kill掉，如果不是59号系统调用，那么就跳过kill，放通执行</font>

在[bpf_common.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf_common.h#L7)中给出了`BPF_STMT`和`BPF_JUMP`相关的操作码:

```c
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _UAPI__LINUX_BPF_COMMON_H__
#define _UAPI__LINUX_BPF_COMMON_H__

/* Instruction classes */                     
#define BPF_CLASS(code) ((code) & 0x07)    //指定操作的类别
#define     BPF_LD      0x00               //将值复制到累加器中
#define     BPF_LDX     0x01               //将值加载到索引寄存器中
#define     BPF_ST      0x02               //将累加器中的值存到暂存器
#define     BPF_STX     0x03               //将索引寄存器的值存储在暂存器中
#define     BPF_ALU     0x04               //用索引寄存器或常数作为操作数在累加器上执行算数或逻辑运算
#define     BPF_JMP     0x05               //跳转
#define     BPF_RET     0x06               //返回
#define     BPF_MISC        0x07           // 其他类别

/* ld/ldx fields */
#define BPF_SIZE(code)  ((code) & 0x18)
#define     BPF_W       0x00 /* 32-bit */       //字
#define     BPF_H       0x08 /* 16-bit */       //半字
#define     BPF_B       0x10 /*  8-bit */       //字节
/* eBPF     BPF_DW      0x18    64-bit */       //双字
#define BPF_MODE(code)  ((code) & 0xe0)
#define     BPF_IMM     0x00                  //常数  
#define     BPF_ABS     0x20                  //固定偏移量的数据包数据(绝对偏移)
#define     BPF_IND     0x40                  //可变偏移量的数据包数据(相对偏移)
#define     BPF_MEM     0x60                  //暂存器中的一个字
#define     BPF_LEN     0x80                  //数据包长度
#define     BPF_MSH     0xa0

/* alu/jmp fields */
#define BPF_OP(code)    ((code) & 0xf0)       //当操作码类型为ALU时，指定具体运算符    
#define     BPF_ADD     0x00         
#define     BPF_SUB     0x10
#define     BPF_MUL     0x20
#define     BPF_DIV     0x30
#define     BPF_OR      0x40
#define     BPF_AND     0x50
#define     BPF_LSH     0x60
#define     BPF_RSH     0x70
#define     BPF_NEG     0x80
#define     BPF_MOD     0x90
#define     BPF_XOR     0xa0
                                               //当操作码是jmp时指定跳转类型
#define     BPF_JA      0x00
#define     BPF_JEQ     0x10
#define     BPF_JGT     0x20
#define     BPF_JGE     0x30
#define     BPF_JSET        0x40
#define BPF_SRC(code)   ((code) & 0x08)
#define     BPF_K       0x00                    //常数
#define     BPF_X       0x08                    //索引寄存器

#ifndef BPF_MAXINSNS
#define BPF_MAXINSNS 4096
#endif

#endif /* _UAPI__LINUX_BPF_COMMON_H__ */
```

与`seccomp`相关的定义大多数在`seccomp.h`中定义。

<font color='#39b54a'>**一旦为程序配置了`seccomp-BPF`，每个系统调用都会经过`seccomp`过滤器，这在一定程度上会影响系统的性能**。</font>此外，Seccomp过滤器会向内核返回一个值，指示是否允许该系统调用，该返回值是一个 32 位的数值，<font color='#39b54a'>**其中最重要的 16 位（`SECCOMP_RET_ACTION`掩码）指定内核应该采取的操作，其他位（`SECCOMP_RET_DATA `掩码）用于返回与操作关联的数据 。**</font>

```c
/*
 * All BPF programs must return a 32-bit value.
 * The bottom 16-bits are for optional return data.
 * The upper 16-bits are ordered from least permissive values to most,
 * as a signed value (so 0x8000000 is negative).
 *
 * The ordering ensures that a min_t() over composed return values always
 * selects the least permissive choice.
 */
#define SECCOMP_RET_KILL_PROCESS 0x80000000U /* kill the process */
#define SECCOMP_RET_KILL_THREAD  0x00000000U /* kill the thread */
#define SECCOMP_RET_KILL     SECCOMP_RET_KILL_THREAD
#define SECCOMP_RET_TRAP     0x00030000U /* disallow and force a SIGSYS */
#define SECCOMP_RET_ERRNO    0x00050000U /* returns an errno */
#define SECCOMP_RET_USER_NOTIF   0x7fc00000U /* notifies userspace */
#define SECCOMP_RET_TRACE    0x7ff00000U /* pass to a tracer or disallow */
#define SECCOMP_RET_LOG      0x7ffc0000U /* allow after logging */
#define SECCOMP_RET_ALLOW    0x7fff0000U /* allow */

/* Masks for the return value sections. */
#define SECCOMP_RET_ACTION_FULL 0xffff0000U
#define SECCOMP_RET_ACTION  0x7fff0000U
#define SECCOMP_RET_DATA    0x0000ffffU
```

- `SECCOMP_RET_ALLOW`：允许执行
- `SECCOMP_RET_KILL`：立即终止执行
- `SECCOMP_RET_ERRNO`：从系统调用中返回一个错误（系统调用不执行）
- `SECCOMP_RET_TRACE`：尝试通知`ptrace()`， 使之有机会获得控制权
- `SECCOMP_RET_TRAP`：通知内核发送`SIGSYS`信号（系统调用不执行）

每一个`seccomp-BPF`程序都使用[seccomp_data](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/seccomp.h#L63)结构作为输入参数：

[/](https://elixir.bootlin.com/linux/latest/source)[include](https://elixir.bootlin.com/linux/latest/source/include)/[uapi](https://elixir.bootlin.com/linux/latest/source/include/uapi)/[linux](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux)/[seccomp.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/seccomp.h) :

```c
struct seccomp_data {
  int nr ;                    /* 系统调用号（依赖于体系架构） */
  __u32 arch ;                /* 架构（如AUDIT_ARCH_X86_64） */
  __u64 instruction_pointer ; /* CPU指令指针 */
  __u64 args [6];             /* 系统调用参数，最多有6个参数 */
};
```

# 二、实现

## Prctl()

[**prctl**](https://man7.org/linux/man-pages/man2/prctl.2.html) 函数是为进程制定而设计的，该函数原型如下

```c
#include <sys/prctl.h>

int prctl(int option, unsigned long arg2, unsigned long arg3, unsigned long arg4, unsigned long arg5);
```

其中明确指定哪种操作在于option选项， option有很多，与seccomp有关的option主要有两个： `PR_SET_NO_NEW_PRIVS()`和`PR_SET_SECCOMP()`。

- **PR_SET_NO_NEW_PRIVS()**：是在Linux 3.5 之后引入的特性，当一个进程或者子进程设置了`PR_SET_NO_NEW_PRIVS` 属性,则**其不能访问一些无法共享的操作**，如`setuid、chroot`等。配置seccomp-BPF的程序必须拥有Capabilities 中 的`CAP_SYS_ADMIN`，或者程序已经定义了`no_new_privs`属性。 若不这样做 非 root 用户使用该程序时 `seccomp`保护将会失效，**设置了 `PR_SET_NO_NEW_PRIVS` 位后能保证 `seccomp` 对所有用户都能起作用**

  ```c
  prctl(PR_SET_NO_NEW_PRIVS,1,0,0,0);
  ```

  如果**将其第二个参数设置为1，则这个操作能保证seccomp对所有用户都能起作用**，并且会**使子进程即`execve`后的进程依然受到seccomp的限制**。

- **PR_SET_SECCOMP()**： 为进程设置`seccomp`； 通常的形式如下

  ```c
  prctl(PR_SET_SECCOMP,SECCOMP_MODE_FILTER,&prog);
  ```

  `SECCOMP_MODE_FILTER`参数表示设置的seccomp的过滤模式，如果设置为`SECCOMP_MODE_STRICT`，则代表严格模式；

  若为过滤模式，则对应的系统调用限制通过`&prog`结构体定义（上面提到过的 `struct sock_fprog`）。

### 严格模式的简单示例

在严格模式下，进程可用的系统调用只有4个，因为`open()`也被禁用，所有在进入严格模式前，需要先打开文件，简单的示例如下：

> seccomp_strict.c：

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>

// configure_seccomp 配置seccomp严格模式
int configure_seccomp() {
    printf("Configure seccomp\n");
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
    return 0;
}

int main(int argc, char* argv[]) {
    int infd, outfd;
    ssize_t  read_bytes;
    char buffer[1024];

    if (argc < 3) {
        printf("Usage:\n\tdup_file <input path> <output path>\n");
        return -1;
    }

    configure_seccomp();

    printf("Opening '%s' for reading\n", argv[1]);
    if ((infd = open(argv[1], O_RDONLY)) > 0) {             // 因为设置了严格模式，所以会在此直接被kill
        printf("Opening '%s' for writing\n", argv[1]);
        if ((outfd = open(argv[2], O_WRONLY | O_CREAT, 0644)) > 0) {
            while((read_bytes = read(infd, &buffer, 1024)) > 0) {
                write(outfd, &buffer, (ssize_t)read_bytes);
            }
        }
    }

    close(infd);
    close(outfd);
    return 0;
}
```

代码功能实现简单的文件复制，当seccomp施加严格模式的时候运行时，seccomp 会在执行`open(argv[1], O_RDONLY)`函数调用时终止应用程序:

![image-20220711141900526](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711141900526.png)

### 过滤模式的简单示例

通过上面的介绍和程序流，如果我们想要为一个程序施加`seccomp-BPF`策略，那可以分为以下几个步骤：

* 首先定义`filter`数组
* 之后定义`prog`参数
* 最后使用`prctl`施加策略

![img](http://xwjpics.gumptlu.work/qinniu_uPic/006uFBysly1h3fzoabrv9j30ju0g9dhg.jpg)

#### 示例一：禁止execve系统调用

> seccomp_filter_execv.c:

```c
#include <stdio.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[]) {
    // 定义bpf程序的过滤规则
    struct sock_filter filter[] = {
        BPF_STMT(BPF_LD+BPF_W+BPF_ABS, 0),          // 在栈帧的偏移0处，取4个字节数据，效果也就是系统调用号的值载入累加器中
        BPF_JUMP(BPF_JMP+BPF_JEQ, 59, 0, 1),        // 判断系统的调用号是否为59，如果是则按顺序执行，否则跳过下一条
        BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL),  // 返回kill
        BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_ALLOW), // 返回allow
    };

    // 创建bpf程序
    struct sock_fprog prog = {
        .len = (unsigned short)(sizeof(filter)/sizeof(filter[0])),      // 规则条数
        .filter = filter                                                // 规则本身
    };

    // 设置prctl
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);                             // 设置NO_NEW_PRIVS
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);                  // 注册规则

    // 测试execve
    write(0, "test\n", 5);
    system("/bin/sh");
    return 0;
}
```

![image-20220711145855043](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711145855043.png)

示例二：

> seccomp_filter_open_file.c:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <stddef.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <stdlib.h>
#include <linux/unistd.h>

void configure_seccomp() {

    // 整体的作用就是如果是open系统调用，那么必须是O_RDONLY的方式，其他方式是不允许的
    struct sock_filter filter[] = {
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, (offsetof(struct seccomp_data, nr))), //将系统调用号载入累加器
        //测试系统调用号是否匹配'__NR_write',如果是则允许(执行下一条指令)，如果不是则跳过下一条指令继续过滤（__NR_前缀是获取系统调用号的前缀）
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_openat, 0, 2),   //测试是否为'__NR_open',不是就放行(往下跳过2个指令)
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, (offsetof(struct seccomp_data, args[2]))),   //第二个参数送入累加器
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, O_RDONLY, 0, 1),    //判断是否是'O_RDONLY'的方式，是则允许,否则kill
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL)
    };


    struct sock_fprog prog = {
        .len = (unsigned short)(sizeof(filter) / sizeof (filter[0])),
        .filter = filter,
    };

    printf("Configuring seccomp\n");
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
}

int main(int argc, char* argv[]) {
    int infd, outfd;
    ssize_t read_bytes;
    char buffer[1024];

    if (argc < 3) {
        printf("Usage:\n\tdup_file <input path> <output_path>\n");
        return -1;
    }
    printf("Ducplicating file '%s' to '%s'\n", argv[1], argv[2]);

    configure_seccomp(); //配置seccomp

    printf("Opening '%s' for reading\n", argv[1]);
    if ((infd = open(argv[1], O_RDONLY)) > 0) {
        printf("Opening '%s' for writing\n", argv[2]);
        if ((outfd = open(argv[2], O_WRONLY | O_CREAT, 0644)) > 0) {    // 不被允许
            while((read_bytes = read(infd, &buffer, 1024)) > 0)
                write(outfd, &buffer, (ssize_t)read_bytes);
        }
    }
    close(infd);
    close(outfd);

    return 0;
}
```

在这种情况下，`seccomp-BPF` 程序将允许使用 `O_RDONLY` 参数打开第一个调用 , 但是在使用 `O_WRONLY | O_CREAT` 参数调用 open 时终止程序

![image-20220711170521151](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711170521151.png)

## libseccomp

项目地址：libseccomp：https://github.com/seccomp/libseccomp

基于`prctl()`函数的机制不够灵活，`libseccomp`库可以提供一些函数实现prctl类似的效果，库中封装了一些函数，可以不用了解BPF规则而实现过滤。但是在c程序中使用它，需要装一些库文件：

```shell
sudo apt install libseccomp-dev libseccomp2 seccomp
```

使用示例：

> simple_syscall_seccomp.c：

```c
//gcc -g simple_syscall_seccomp.c -o simple_syscall_seccomp -lseccomp
#include <unistd.h>
#include <seccomp.h>
#include <linux/seccomp.h>

int main(void){
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
    seccomp_load(ctx);

    char * filename = "/bin/sh";
    char * argv[] = {"/bin/sh",NULL};
    char * envp[] = {NULL};
    write(1,"i will give you a shell\n",24);
    syscall(59,filename,argv,envp);//execve
    return 0;
}
```

编译运行, 在执行 execve 时程序报错退出 :

```shell
gcc -g simple_syscall_seccomp.c -o simple_syscall_seccomp -lseccomp
./simple_syscall_seccomp
i will give you a shell
Bad system call (core dumped)s
```

解释一下上诉代码：

- `scmp_filter_ctx` : 过滤器的结构体

- `seccomp_init` : 初始化的过滤状态 ,函数原型：

  ```
  seccomp_init(uint32_t def_action)
  ```

  可选的`def_action`有：

  ```c
  SCMP_ACT_ALLOW：即初始化为允许所有系统调用，过滤为黑名单模式； 
  SCMP_ACT_KILL：则为白名单模式过滤。
  SCMP_ACT_KILL_PROCESS：整个进程将被内核终止
  SCMP_ACT_TRAP:如果所有系统调用都不匹配，则给线程发送一个SIGSYS信号
  SCMP_ACT_TRACE(uint16_t msg_num)：在使用ptrace根据进程时的相关选项
  SCMP_ACT_ERRNO(uint16_t errno)：不匹配会收到errno的返回值
  SCMP_ACT_LOG：不影响系统调用，但是会被记录；
  ```

- [seccomp_rule_add](https://man7.org/linux/man-pages/man3/seccomp_rule_add.3.html) ： 添加一条规则，函数原型为：

  ```c
  int seccomp_rule_add(scmp_filter_ctx ctx, uint32_t action,int syscall, unsigned int arg_cnt, ...);
  ```

  其中`arg_cnt`参数表明是否需要对对应系统调用的参数做出限制以及指示做出限制的个数，如果**仅仅需要允许或者禁止所有某个系统调用**，`arg_cnt`直接传入0即可，如 `seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0)` 即禁用execve，不管其参数如何。 如果`arg_cnt`的参数不为0， 那 `arg_cnt` 表示后面限制的参数的个数，也就是只有调用 execve，且参数满足要求时，才会拦截 syscall 。如果想要更细粒度的过滤系统调用，把参数也考虑进去,就要设置arg_cnt不为零，然后在利用宏做一些过滤。

  举个例子， 拦截 `write `函数 参数大于 `0x11 `时的系统调用 ：

  > seccomp_write_limit.c：

  ```c
  //gcc -g ./seccomp_write_limit.cpp -o seccomp_write_limit -lseccomp
  
  #include <unistd.h>
  #include <seccomp.h>
  #include <linux/seccomp.h>
  
  int main(void){
      scmp_filter_ctx ctx;
      ctx = seccomp_init(SCMP_ACT_ALLOW);
      seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(write),1, SCMP_A2(SCMP_CMP_GT,0x11));//第2(从0)个参数大于0x11
      seccomp_load(ctx);
      write(1,"1234567812345678\n",0x11);//不被拦截
      write(1,"i will give you a shell\n",24);//会拦截
      return 0;
  }
  ```

  编译执行

  ![image-20220711171922636](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711171922636.png)

  其中`SCMP_A2`代表为第二个参数指定比较结构，`SCMP_CMP_GT`代表 大于(greater than)，详细内容如下。

  > libseccmop/include/[seccomp.h.in](https://github.com/seccomp/libseccomp/blob/3f0e47fe2717b73ccef68ca18f9f7297ee73ebb2/include/seccomp.h.in)：

  ```c
  ...
  ...
  
  /**
   * Comparison operators
   */
  enum scmp_compare {
    _SCMP_CMP_MIN = 0,
    SCMP_CMP_NE = 1,        /**< not equal */
    SCMP_CMP_LT = 2,        /**< less than */
    SCMP_CMP_LE = 3,        /**< less than or equal */
    SCMP_CMP_EQ = 4,        /**< equal */
    SCMP_CMP_GE = 5,        /**< greater than or equal */
    SCMP_CMP_GT = 6,        /**< greater than */
    SCMP_CMP_MASKED_EQ = 7,     /**< masked equality */
    _SCMP_CMP_MAX,
  };
   ...
   struct scmp_arg_cmp {
    unsigned int arg;   /**< argument number, starting at 0 */
    enum scmp_compare op;   /**< the comparison op, e.g. SCMP_CMP_* */
    scmp_datum_t datum_a;
    scmp_datum_t datum_b;
  };
   ....
  /**
   * Specify a 32-bit argument comparison struct for use in declaring rules
   * @param arg the argument number, starting at 0
   * @param op the comparison operator, e.g. SCMP_CMP_*
   * @param datum_a dependent on comparison (32-bits)
   * @param datum_b dependent on comparison, optional (32-bits)
   */
  #define SCMP_CMP32(x, y, ...) \
    _SCMP_MACRO_DISPATCHER(_SCMP_CMP32_, __VA_ARGS__)(x, y, __VA_ARGS__)
  
  /**
   * Specify a 64-bit argument comparison struct for argument 0
   */
  #define SCMP_A0_64(...)       SCMP_CMP64(0, __VA_ARGS__)
  #define SCMP_A0           SCMP_A0_64
  
  /**
   * Specify a 32-bit argument comparison struct for argument 0
   */
  #define SCMP_A0_32(x, ...)    SCMP_CMP32(0, x, __VA_ARGS__)
  
  /**
   * Specify a 64-bit argument comparison struct for argument 1
   */
  #define SCMP_A1_64(...)       SCMP_CMP64(1, __VA_ARGS__)
  #define SCMP_A1           SCMP_A1_64
  
  /**
   * Specify a 32-bit argument comparison struct for argument 1
   */
  #define SCMP_A1_32(x, ...)    SCMP_CMP32(1, x, __VA_ARGS__)
  
  /**
   * Specify a 64-bit argument comparison struct for argument 2
   */
  #define SCMP_A2_64(...)       SCMP_CMP64(2, __VA_ARGS__)
  #define SCMP_A2           SCMP_A2_64
  
  /**
   * Specify a 32-bit argument comparison struct for argument 2
   */
  #define SCMP_A2_32(x, ...)    SCMP_CMP32(2, x, __VA_ARGS__)
  
  /**
   * Specify a 64-bit argument comparison struct for argument 3
   */
  #define SCMP_A3_64(...)       SCMP_CMP64(3, __VA_ARGS__)
  #define SCMP_A3           SCMP_A3_64
  
  /**
   * Specify a 32-bit argument comparison struct for argument 3
   */
  #define SCMP_A3_32(x, ...)    SCMP_CMP32(3, x, __VA_ARGS__)
  
  /**
   * Specify a 64-bit argument comparison struct for argument 4
   */
  #define SCMP_A4_64(...)       SCMP_CMP64(4, __VA_ARGS__)
  #define SCMP_A4           SCMP_A4_64
  
  /**
   * Specify a 32-bit argument comparison struct for argument 4
   */
  #define SCMP_A4_32(x, ...)    SCMP_CMP32(4, x, __VA_ARGS__)
  
  /**
   * Specify a 64-bit argument comparison struct for argument 5
   */
  #define SCMP_A5_64(...)       SCMP_CMP64(5, __VA_ARGS__)
  #define SCMP_A5           SCMP_A5_64
  
  /**
   * Specify a 32-bit argument comparison struct for argument 5
   */
  #define SCMP_A5_32(x, ...)    SCMP_CMP32(5, x, __VA_ARGS__)
  
       ...
       ...
  ```

  除了seccomp_rule_add之外，还有其他添加规则的函数，如：`seccomp_rule_add_array ()`、 `seccomp_rule_add_exact ()`和`seccomp_rule_add_exact_array ()`，详细信息可查看参考链接。

- seccomp_load： 将当前的 seccomp 过滤器加载到内核中 ，函数原型：

  ```c
  int seccomp_load(scmp_filter_ctx ctx);
  ```

- seccomp_reset : 释放现有的过滤上下文重新初始化之前的状态，并且只能在成功调用`seccomp_init () `之后才能使用。

  ```c
  int seccomp_reset（scmp_filter_ctx ctx ，uint32_t def_action ）
  ```

# 三、其他工具

## seccmop-bpf.h

[seccomp-bpf.h](https://github.com/ahupowerdns/secfilter/blob/master/seccomp-bpf.h)是由开发人员编写的一个十分便捷的头文件用于开发seccomp-bpf 。该头文件已经定义好了很多常见的宏，如验证系统架构、允许系统调用等功能，十分便捷，如下所示。

```c
...
define VALIDATE_ARCHITECTURE \
    BPF_STMT(BPF_LD+BPF_W+BPF_ABS, arch_nr), \
    BPF_JUMP(BPF_JMP+BPF_JEQ+BPF_K, ARCH_NR, 1, 0), \
    BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL)

define EXAMINE_SYSCALL \
    BPF_STMT(BPF_LD+BPF_W+BPF_ABS, syscall_nr)

define ALLOW_SYSCALL(name) \
    BPF_JUMP(BPF_JMP+BPF_JEQ+BPF_K, __NR_##name, 0, 1), \
    BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_ALLOW)

define KILL_PROCESS \
    BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL)
...
```

应用示例：

> [seccomp_policy.c](https://gist.github.com/mstemm/1bc06c52abb7b6b4feef79d7bfff5815#file-seccomp_policy-c)

```c
//
// Created by 许文杰 on 2022/7/11.
//

#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <assert.h>
#include <linux/seccomp.h>
#include <sys/prctl.h>
#include "seccomp-bpf.h"            // 引入头文件

void install_syscall_filter()
{
    struct sock_filter filter[] = {
            /* Validate architecture. */
            VALIDATE_ARCHITECTURE,
            /* Grab the system call number. */
            EXAMINE_SYSCALL,
            /* List allowed syscalls. We add open() to the set of
               allowed syscalls by the strict policy, but not
               close(). */
            ALLOW_SYSCALL(rt_sigreturn),
#ifdef __NR_sigreturn
            ALLOW_SYSCALL(sigreturn),
#endif
            ALLOW_SYSCALL(exit_group),
            ALLOW_SYSCALL(exit),
            ALLOW_SYSCALL(read),
            ALLOW_SYSCALL(write),
            ALLOW_SYSCALL(open),
            ALLOW_SYSCALL(openat),
            KILL_PROCESS,
    };
    struct sock_fprog prog = {
            .len = (unsigned short)(sizeof(filter)/sizeof(filter[0])),
            .filter = filter,
    };

    assert(prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) == 0);

    assert(prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog) == 0);
}

int main(int argc, char **argv)
{
    int output = open("output.txt", O_WRONLY);
    const char *val = "test";

    printf("Calling prctl() to set seccomp with filter...\n");

    install_syscall_filter();

    printf("Writing to an already open file...\n");
    write(output, val, strlen(val)+1);

    printf("Trying to open file for reading...\n");
    int input = open("output.txt", O_RDONLY);

    printf("Note that open() worked. However, close() will not\n");
    close(input);

    printf("You will not see this message--the process will be killed first\n");
}
```

执行结果

![image-20220711173218655](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711173218655.png)

## seccomp-tools

一款用于分析seccomp的开源工具，项目地址：https://github.com/david942j/seccomp-tools

主要功能：

- `Dump`： 从可执行文件中自动转储 seccomp BPF
- `Disasm`： 将 seccomp BPF 转换为人类可读的格式
- `Asm`：使编写seccomp规则类似于编写代码
- `Emu`： 模拟 seccomp 规则

安装

```shell
sudo apt install gcc ruby-dev
gem install seccomp-tools
```

使用

```shell
null@ubuntu:~/seccomp$ seccomp-tools dump ./simple_syscall_seccomp
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x05 0xc000003e  if (A != ARCH_X86_64) goto 0007
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x35 0x00 0x01 0x40000000  if (A < 0x40000000) goto 0005
 0004: 0x15 0x00 0x02 0xffffffff  if (A != 0xffffffff) goto 0007
 0005: 0x15 0x01 0x00 0x0000003b  if (A == execve) goto 0007
 0006: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0007: 0x06 0x00 0x00 0x00000000  return KILL
```

从输出中可知禁用了execve系统调用。

# 四、使用Seccomp保护Docker的安全

Seccomp技术被用在很多应用程序上以保护系统的安全性，Docker支持使用seccomp来限制容器的系统调用，不过需要启用内核中的`CONFIG_SECCOMP`。

```shell
$ grep CONFIG_SECCOMP= /boot/config-$(uname -r)
CONFIG_SECCOMP=y
```

当使用docker run 启动一个容器时，Docker会使用默认的seccomp配置文件来对容器施加限制策略，该默认文件是以`json`格式编写， 在 300 多个系统调用中禁用了大约 44 个系统调用，可以在Moby项目中找到该[源码](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json)。

```shell
$ sudo docker run --rm -it ubuntu /bin/bash
root@85e01c28bd2c:/# bash
root@85e01c28bd2c:/# ps
   PID TTY          TIME CMD
     1 pts/0    00:00:00 bash
    10 pts/0    00:00:00 bash
    13 pts/0    00:00:00 ps
root@85e01c28bd2c:/# grep -i seccomp /proc/1/status
Seccomp:        2
```

Docker中**默认的配置文件提供了最大限度的包容性**，除了默认的选择之外，Docker允许我们自定义该配置文件来灵活的对容器的系统调用进行限制。

示例：以**白名单**的形式允许特定的系统调用

> example.json

```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "arch_prctl",
                "sched_yield",
                "futex",
                "write",
                "mmap",
                "exit_group",
                "madvise",
                "rt_sigprocmask",
                "getpid",
                "gettid",
                "tgkill",
                "rt_sigaction",
                "read",
                "getpgrp"
            ],
            "action": "SCMP_ACT_ALLOW",
            "args": [],
            "comment": "",
            "includes": {},
            "excludes": {}
        }
    ]
}
```

- `defaultAction` : 指定默认的seccomp 操作，具体的可选参数上面已经介绍过了，最常用的无非是`SCMP_ACT_ALLOW`、`SCMP_ACT_ERRNO`，这里选择`SCMP_ACT_ERRNO`，**表示默认禁止全部系统调用**，以白名单的形式在赋予可用的系统调用。
- `architectures` ： 系统架构，不同的系统架构系统调用可能不同。
- `syscalls`：指定系统调用以及对应的操作，name定义系统调用名，action对应的操作，这里表示允许name里边中的系统调用，args对应系统调用参数，可以为空。

这样，在使用 *docker run* 运行容器时，就可以使用 `--security-opt` 选项指定该配置文件来对容器进行系统调用定制。

```shell
$ docker run --rm -it --security-opt seccomp=/path/to/seccomp/example.json hello-world
```

举例，禁止容器创建文件夹，就可以用**黑名单**的形式禁用mkdir系统调用

> seccomp_mkdir.json:

```json
{
    "defaultAction": "SCMP_ACT_ALLOW",
    "syscalls": [
        {
            "name": "mkdir",
            "action": "SCMP_ACT_ERRNO",
            "args": []
        }
    ]
}
```

使用该策略启动容器，并在容器中创建文件夹时，就会收到禁止信息，不允许创建文件夹。

```shell
$ sudo docker run --rm -it --security-opt seccomp=seccomp_mkdir.json busybox /bin/sh
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
/ # mkdir test
mkdir: can't create directory 'test': Operation not permitted
```

当然也可以不适用任何`seccomp`策略启动容器，只需要在启动选项中加上`--security-opt seccomp=unconfined`即可。

## zaz

*zaz seccomp* 是一个可以为容器自动生成json格式的seccomp文件的开源工具，项目地址：https://github.com/pjbgf/zaz。

主要用法为

```shell
zaz seccomp docker IMAGE COMMAND
```

它能够**为特定的可执行文件定制系统调用**，以只允许特定的操作，禁止其他操作

举个例子：为alpine中的ping命令生成seccomp配置文件

```shell
$ sudo ./zaz seccomp docker alpine "ping -c5 8.8.8.8" > seccomp_ping.json
$ cat seccomp_ping.json | jq '.'
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "arch_prctl",
        "bind",
        "clock_gettime",
        "clone",
        "close",
        "connect",
        "dup2",
        "epoll_pwait",
        "execve",
        "exit",
        "exit_group",
        "fcntl",
        "futex",
        "getpid",
        "getsockname",
        "getuid",
        "ioctl",
        "mprotect",
        "nanosleep",
        "open",
        "poll",
        "read",
        "recvfrom",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sendto",
        "set_tid_address",
        "setitimer",
        "setsockopt",
        "socket",
        "write",
        "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

如上所示，`zaz`检测到了33个系统调用，使用白名单的形式过滤系统调用。那它以白名单的形式生成的系统调用能否很好的过滤系统系统呢？是否能够满足运行ping命令，而不能运行除了它允许的系统调用之外的命令呢？做个测试，首先用下面Dockerfile构建一个简单的镜像。

> Dockerfile

```dockerfile
FROM alpine:latest
CMD ["ping","-c5","8.8.8.8"]
```

构建成功后，使用默认的seccomp策略启动容器，没有任何问题，可以运行。

```shell
$ sudo docker build -t pingtest .
$ sudo docker run --rm -it pingtest
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=127 time=42.139 ms
64 bytes from 8.8.8.8: seq=1 ttl=127 time=42.646 ms
64 bytes from 8.8.8.8: seq=2 ttl=127 time=42.098 ms
64 bytes from 8.8.8.8: seq=3 ttl=127 time=42.484 ms
64 bytes from 8.8.8.8: seq=4 ttl=127 time=42.007 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 42.007/42.274/42.646 ms
```

接着我们使用上述zaz生成的策略试试，

```shell
$ sudo docker run --rm -it --security-opt seccomp=seccomp_ping.json pingtest
docker: Error response from daemon: failed to create shim: OCI runtime create failed: container_linux.go:380: starting container process caused: close exec fds: open /proc/self/fd: operation not permitted: unknown.
```

容器并没有成功启动，在创建OCI的时候就报错了，报错原因是`operation not permitted`,这个报错上面似乎提到过，是想要使用的系统调用被禁用的缘故，可能zaz这种白名单的模式鲁棒性还是不够强，而且Docker更新那么多次，`zaz`缺乏维护导致捕获的系统调用不足，在容器启动过程中出现了问题。奇怪的是，当我再次运行同样的命令，却引发了panic报错：`No error following JSON procError payload`。

```shell
$ sudo docker run --rm -it --security-opt seccomp=seccomp_ping.json pingtest
docker: Error response from daemon: failed to create shim: OCI runtime create failed: runc did not terminate successfully: exit status 2: panic: No error following JSON procError payload.

goroutine 1 [running]:
github.com/opencontainers/runc/libcontainer.parseSync(0x56551adf30b8, 0xc000010b20, 0xc0002268a0, 0xc00027f9e0, 0x0)
        github.com/opencontainers/runc/libcontainer/sync.go:93 +0x307
github.com/opencontainers/runc/libcontainer.(*initProcess).start(0xc000297cb0, 0x0, 0x0)
        github.com/opencontainers/runc/libcontainer/process_linux.go:440 +0x5ef
github.com/opencontainers/runc/libcontainer.(*linuxContainer).start(0xc000078700, 0xc000209680, 0x0, 0x0)
        github.com/opencontainers/runc/libcontainer/container_linux.go:379 +0xf5
github.com/opencontainers/runc/libcontainer.(*linuxContainer).Start(0xc000078700, 0xc000209680, 0x0, 0x0)
        github.com/opencontainers/runc/libcontainer/container_linux.go:264 +0xb4
main.(*runner).run(0xc0002274c8, 0xc0000200f0, 0x0, 0x0, 0x0)
        github.com/opencontainers/runc/utils_linux.go:312 +0xd2a
main.startContainer(0xc00025c160, 0xc000076400, 0x1, 0x0, 0x0, 0xc0002275b8, 0x6)
        github.com/opencontainers/runc/utils_linux.go:455 +0x455
main.glob..func2(0xc00025c160, 0xc000246000, 0xc000246120)
        github.com/opencontainers/runc/create.go:65 +0xbb
github.com/urfave/cli.HandleAction(0x56551ad3b040, 0x56551ade81e8, 0xc00025c160, 0xc00025c160, 0x0)
        github.com/urfave/cli@v1.22.1/app.go:523 +0x107
github.com/urfave/cli.Command.Run(0x56551aa566f5, 0x6, 0x0, 0x0, 0x0, 0x0, 0x0, 0x56551aa5f509, 0x12, 0x0, ...)
        github.com/urfave/cli@v1.22.1/command.go:174 +0x579
github.com/urfave/cli.(*App).Run(0xc000254000, 0xc000132000, 0xf, 0xf, 0x0, 0x0)
        github.com/urfave/cli@v1.22.1/app.go:276 +0x7e8
main.main()
        github.com/opencontainers/runc/main.go:163 +0xd3f
: unknown.
```

[![img](http://xwjpics.gumptlu.work/qinniu_uPic/006uFBysly1h3fzoaenjnj30xr0jy1kx.jpg)](http://tva1.sinaimg.cn/large/006uFBysly1h3fzoaenjnj30xr0jy1kx.jpg)

这种报错或许是不应该的，我尝试在网上寻找报错的相关信息，类似的情况很少，而且并不是每次运行都是出现这种panic，正常情况下应该是`operation not permitted`，这是由于我们的白名单没有完全包括必须的系统调用导致的。目前将此情况汇报给了Moby [issue](https://github.com/moby/moby/issues/43730)，或许能够得到一些解答。

类似panic信息：

https://bugzilla.redhat.com/show_bug.cgi?format=multiple&id=1714183

无论是哪种报错，看起来都是runc出了问题，尝试解决这个问题，我们就要知道Docker到底是如何在运行时加载seccomp？

当我们要创建一个容器的时候 ，容器守护进程 Dockerd会请求 `containerd` 来创建一个容器 ， `containerd` 收到请求后，也并不会直接去操作容器，而是创建一个叫做 `containerd-shim` 的进程，让这个进程去操作容器，之后`containerd-shim`会通过OCI去调用容器运行时`runc`来启动容器， `runc` 启动完容器后本身会直接退出，`containerd-shim` 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 `containerd`, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程 。也就是说调用顺序为

```
Dockerd -> containerd -> containerd-shim -> runc
```

启动一个容器ubuntu，并在容器中再运行一个bash

```shell
$ sudo docker run --rm -it ubuntu /bin/bash
root@ef57fff95b80:/# bash
root@ef57fff95b80:/# ps
   PID TTY          TIME CMD
     1 pts/0    00:00:00 bash
     9 pts/0    00:00:00 bash
    12 pts/0    00:00:00 ps
```

查看调用栈，`containerd-shim`（28051-28129）并没有被施加seccomp,而容器内的两个bash（1 -> 28075;9->28126）被施加了seccomp策略。

```shell
root@ubuntu:/home/null# pstree -p | grep containerd-shim
           |-containerd-shim(28051)-+-bash(28075)---bash(28126)
           |                        |-{containerd-shim}(28052)
           |                        |-{containerd-shim}(28053)
           |                        |-{containerd-shim}(28054)
           |                        |-{containerd-shim}(28055)
           |                        |-{containerd-shim}(28056)
           |                        |-{containerd-shim}(28057)
           |                        |-{containerd-shim}(28058)
           |                        |-{containerd-shim}(28059)
           |                        |-{containerd-shim}(28060)
           |                        `-{containerd-shim}(28129)
root@ubuntu:/home/null# grep -i seccomp /proc/28051/status
Seccomp:        0
root@ubuntu:/home/null# grep -i seccomp /proc/28075/status
Seccomp:        2
root@ubuntu:/home/null# grep -i seccomp /proc/28126/status
Seccomp:        2
root@ubuntu:/home/null# grep -i seccomp /proc/28052/status
Seccomp:        0
...
...
root@ubuntu:/home/null# grep -i seccomp /proc/28129/status
Seccomp:        0
```

也就是说对容器施加seccomp 是在`container-shim`启动之后，在调用runc的时候出现了问题，是否我们的seccomp策略也要将runc所必须的系统调用考虑进去呢？Zaz是否考虑了容器启动时候的runc所必须的系统调用?

这就需要捕获容器在启动时，runc所必要的系统调用了。

## Sysdig

为了获取容器运行时runc用了哪些系统调用，可以有很多方法，比如ftrace、strace、fanotify等，这里使用`sysdig`来监控容器的运行，`sisdig`时一款原生支持容器的系统可见性工具，项目地址：https://github.com/draios/sysdig。具体的安装和使用方法可以参考GitHub上给出的详细教程，这里只做简单介绍。

安装完成后，直接在命令行运行sysdig，不加任何参数， sysdig 会捕获所有的事件并将其写入标准输出 ：

```shell
$ sysdig
285304 01:21:51.270700399 7 sshd (50485) > select
285306 01:21:51.270701716 7 sshd (50485) < select res=2
285307 01:21:51.270701982 7 sshd (50485) > rt_sigprocmask
285308 01:21:51.270702258 7 sshd (50485) < rt_sigprocmask
285309 01:21:51.270702473 7 sshd (50485) > rt_sigprocmask
285310 01:21:51.270702660 7 sshd (50485) < rt_sigprocmask
285312 01:21:51.270702983 7 sshd (50485) > read fd=13(<f>/dev/ptmx) size=16384
285313 01:21:51.270703971 1 sysdig (59131) > switch next=59095 pgft_maj=0 pgft_min=1759 vm_size=280112 vm_rss=18048 vm_swap=0
...
```

默认情况下，sysdig 在一行中打印每个事件的信息，格式如下

```c
%evt.num %evt.time %evt.cpu %proc.name (%thread.tid) %evt.dir %evt.type %evt.args
```

其中

- evt.num 是递增的事件编号
- evt.time 是事件时间戳
- evt.cpu 是捕获事件的 CPU 编号
- proc.name 是生成事件的进程的名称
- thread.tid 是产生事件的TID，对应单线程进程的PID
- evt.dir 是事件方向，> 表示进入事件，< 表示退出事件
- evt.type 是事件的名称，例如“open”或“read”
- evt.args 是事件参数的列表。在系统调用的情况下，这些往往对应于系统调用参数，但情况并非总是如此：出于简单或性能原因，某些系统调用参数被排除在外。

启动一个终端A，输入以下命令进行监控，`container.name`指定捕获容器名为ping，`proc.name`指定进程名为runc的包，保存为runc.scap.

```
$sysdig -w runc.scap container.name=ping&&proc.name=runc
```

接着在另一个终端B启动该容器

```shell
$sudo docker run --rm -it --name=ping pingtest
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=127 time=44.032 ms
64 bytes from 8.8.8.8: seq=1 ttl=127 time=42.069 ms
64 bytes from 8.8.8.8: seq=2 ttl=127 time=42.066 ms
64 bytes from 8.8.8.8: seq=3 ttl=127 time=42.073 ms
64 bytes from 8.8.8.8: seq=4 ttl=127 time=42.112 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 42.066/42.470/44.032 ms
```

执行完毕后，在终端A使用ctrl+c停止捕获，并筛选捕获的内容，只留系统调用，将结果保存到`runc_syscall.txt`中，这样我们就得到了启动容器时runc使用了哪些系统调用。

```
$ sysdig  -p "%syscall.type" -r runc.scap | runc_syscall.txt
$ cat -n runc_syscall.txt
  ...
  3437  rt_sigaction
  3438  exit_group
  3439  procexit
```

可以发现筛选出的系统调用数还是有很多的，其中包含很多重复的系统调用，这里可以简单的写一个脚本，进行过滤，通过过滤后，一共有72个系统调用。

```shell
$ python analyse.py runc_syscall.txt
Filter syscall num: 72
filter syscall:['clone', 'close', 'prctl', 'getpid', 'write', 'unshare', 'read', 'exit_group', 'procexit', 'setsid', 'setuid', 'setgid', 'sched_getaffinity', 'openat', 'mmap', 'rt_sigprocmask', 'sigaltstack', 'gettid', 'rt_sigaction', 'mprotect', 'futex', 'set_robust_list', 'munmap', 'nanosleep', 'readlinkat', 'fcntl', 'epoll_create1', 'pipe', 'epoll_ctl', 'fstat', 'pread', 'getdents64', 'capget', 'epoll_pwait', 'newfstatat', 'statfs', 'getppid', 'keyctl', 'socket', 'bind', 'sendto', 'getsockname', 'recvfrom', 'mount', 'fchmodat', 'mkdirat', 'symlinkat', 'umask', 'mknodat', 'fchownat', 'unlinkat', 'chdir', 'fchdir', 'pivot_root', 'umount', 'dup', 'sethostname', 'fstatfs', 'seccomp', 'brk', 'fchown', 'setgroups', 'capset', 'execve', 'signaldeliver', 'access', 'arch_prctl', 'getuid', 'getgid', 'geteuid', 'getcwd', 'getegid']
```

将zaz生成的系统调用与我们捕获的系统调用合二为一，系统调用数到了85个。如下：

```shell
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "clone",
                "close",
                "prctl",
                "getpid",
                "write",
                "unshare",
                "read",
                "exit_group",
                "procexit",
                "setsid",
                "setuid",
                "setgid",
                "sched_getaffinity",
                "openat",
                "mmap",
                "rt_sigprocmask",
                "sigaltstack",
                "gettid",
                "rt_sigaction",
                "mprotect",
                "futex",
                "set_robust_list",
                "munmap",
                "nanosleep",
                "readlinkat",
                "fcntl",
                "epoll_create1",
                "pipe",
                "epoll_ctl",
                "fstat",
                "pread",
                "getdents64",
                "capget",
                "epoll_pwait",
                "newfstatat",
                "statfs",
                "getppid",
                "keyctl",
                "socket",
                "bind",
                "sendto",
                "getsockname",
                "recvfrom",
                "mount",
                "fchmodat",
                "mkdirat",
                "symlinkat",
                "umask",
                "mknodat",
                "fchownat",
                "unlinkat",
                "chdir",
                "fchdir",
                "pivot_root",
                "umount",
                "dup",
                "sethostname",
                "fstatfs",
                "seccomp",
                "brk",
                "fchown",
                "setgroups",
                "capset",
                "signaldeliver",
                "access",
                "getuid",
                "getgid",
                "geteuid",
                "getcwd",
                "getegid",
                "arch_prctl",
                "clock_gettime",
                "connect",
                "dup2",
                "execve",
                "exit",
                "ioctl",
                "open",
                "poll",
                "rt_sigreturn",
                "set_tid_address",
                "setitimer",
                "setsockopt",
                "socket",
                "writev"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

通过该文件再次运行容器，发现可以成功运行！

```shell
null@ubuntu:~/seccomp/docker/zaz/cmd$ sudo docker run -it --rm --security-opt seccomp=seccomp_ping.json pingtest
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=127 time=43.424 ms
64 bytes from 8.8.8.8: seq=1 ttl=127 time=42.873 ms
64 bytes from 8.8.8.8: seq=2 ttl=127 time=42.336 ms
64 bytes from 8.8.8.8: seq=3 ttl=127 time=48.164 ms
64 bytes from 8.8.8.8: seq=4 ttl=127 time=42.260 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 42.260/43.811/48.164 ms
```

尝试运行其他命令，有些命令由于缺乏必须的系统调用，会出现`Operation not permitted`的报错。

```shell
$ sudo docker run -it --rm --security-opt seccomp=seccomp_ping.json pingtest ls
ls: .: Operation not permitted
$ sudo docker run -it --rm --security-opt seccomp=seccomp_ping.json pingtest mkdir test
mkdir: can't create directory 'test': Operation not permitted
```



> **BPF操作码**
>
> https://www3.physnet.uni-hamburg.de/physnet/Tru64-Unix/HTML/MAN/MAN7/0012____.HTM
>
> **seccomp_rule_add**
>
> https://man7.org/linux/man-pages/man3/seccomp_rule_add.3.html
>
> **seccomp和seccomp bfp**
>
> https://ajxchapman.github.io/linux/2016/08/31/seccomp-and-seccomp-bpf.html
>
> **seccomp 概述**
>
> https://lwn.net/Articles/656307/
>
> **seccomp沙箱机制 & 2019ByteCTF VIP**
>
> https://bbs.pediy.com/thread-258146.htm
>
> **prctl(2) — Linux manual page**
>
> https://man7.org/linux/man-pages/man2/prctl.2.html
>
> **seccomp-tools**
>
> https://github.com/david942j/seccomp-tools
>
> **libseccomp**
>
> https://github.com/seccomp/libseccomp/blob/3f0e47fe2717b73ccef68ca18f9f7297ee73ebb2/include/seccomp.h.in
>
> docker seccomp
>
> https://docs.docker.com/engine/security/seccomp/
>
> Docker seccomp 与OCI
>
> https://forums.mobyproject.org/t/docker-seccomp-prevents-system-calls-issued-by-oci-runtime/297/9
