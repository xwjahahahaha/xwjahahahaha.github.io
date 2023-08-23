---
title: linux2-6-11版本源码相关
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2023-08-23 15:29:01
---

[TOC]

<!-- more -->

> 参考：
>
> *  https://github.com/Antonio-Zhou/Linux-2.6.11

在看这本书的时候不要纯看理论，可以边看边找对应的源代码来看，这样记忆深刻并且理解的更加透彻

所以这篇文章就主要是如何下载和辅助阅读源码以及源码的基本结构、常见宏函数等的描述

**注意：本篇针对的都是2.6.11的内核版本**

# 准备工作

下载2.6.11版本：

* 可以直接fork别人的github库
  * nobody: https://github.com/Antonio-Zhou/Linux-2.6.11
  * 我的库：https://github.com/xwjahahahaha/linux_kernel_2.6.11_study
* 或者用下面的链接下载：
  * https://mirrors.edge.kernel.org/pub/linux/kernel/v2.6/

vscode+Global让源代码在vscode中可跳转：

https://blog.mxslly.com/archives/170.html

内核源码GDB调试：（目前我也还没做）

https://www.jianshu.com/p/a12c89a4f409

# 源码目录结构

```shell
./
├── COPYING 
├── CREDITS
├── Documentation
├── MAINTAINERS
├── Makefile
├── README
├── README.md
├── REPORTING-BUGS
├── arch    # 特定体系结构的代码，比如x86、ARM、MIPS等。每个子目录都对应一个支持的硬件体系结构。
├── crypto	# 包含了内核的加密API的实现
├── drivers # 驱动设备相关，包括图形卡、声卡、网络设备等
├── fs			# 文件系统的代码，ext4、NTFS、FAT32等
├── include # 所有的头文件，一些给内核本身使用，一些给用户空间程序使用
├── init		# 启动内核和初始化内核的代码
├── ipc			# 进程间通信的代码，信号量、消息队列、共享内存
├── kernel	# 内核核心，包括调度器、模块加载和卸载等代码
├── lib			# 内核操作的通用函数库
├── mm			# 内存管理相关的代码，页内存、虚拟内存、物理内存
├── net			# 网络协议的实现，TCP/IP、IPv6等
├── scripts	# 编译内核的各种脚本
├── security	# 安全模块的代码，例如SELinux
├── sound		# 声音子系统的代码
└── usr			# initramfs相关
```

# 常见宏函数解释

## BUG_ON

其源码如下：

```assembly
#define BUG_ON(x) do {						\
	__asm__ __volatile__(					\
		"1:	tdnei %0,0\n"				\
		".section __bug_table,\"a\"\n\t"		\
		"	.llong 1b,%1,%2,%3\n"			\
		".previous"					\
		: : "r" (x), "i" (__LINE__), "i" (__FILE__),	\
		    "i" (__FUNCTION__));			\
} while (0)
```

是使用汇编写的一段代码宏，不用关心其具体细节，只需要了解其功能：

=> 类似与assert，一般用于断言某个条件应该为**false**，如果断言失败，那么就会产生内核错误

例如

```c
BUG_ON(pointer == NULL);
```

表示，“我们绝不期望 `pointer` 是 `NULL`”。如果 `pointer` 真的是 `NULL`，那么这是一个错误，我们希望立即知道。
