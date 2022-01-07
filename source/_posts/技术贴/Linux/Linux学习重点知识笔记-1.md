---
title: Linux学习重点知识笔记(1)
tags:
  - linux
categories:
  - technical
  - linux
toc: true
date: 2020-08-03 17:14:54
---

学习《鸟哥的Linux私房菜（基础学习篇）》记录一些重点。

# 第一章 linux是什么以及如何学习

linux其实就是一个操作系统底层的**内核**和**提供的内核工具**。

linux的版本与linux发行版的版本是不同的。

linux发行版(Linux distribution)的构成：Linux Kernel 内核 + SoftWare软件 + (Tools+documentation) 工具 + 可完全安装程序。

**X-Window System(可看作是图像化操作)只是Linux上的一个软件，而不是内核，我们要学习的是他的内核，所以要学习命令行。**

linux强大稳定性能(ps:创造者Linux Torvalds是个性能癖)、强大的运算能力使得目前应用广泛：

* 企业环境：关键任务、高性能计算
* 个人环境：桌面计算机、手持系统(ps:Android系统就是Linux的一个分支)、嵌入式系统
* 云端应用： 云计算
等等

linux系统中各个组件或设备都是一个**文件**，这与windows系统是大相径庭的。

<!-- more -->

# 第二章 主机规划与磁盘分区

- 分区表有两种形式：MBR(Ms-DOS)和GPT磁盘分区表(partition table)

- 磁盘的第一个扇区主要会放置：
  - 主引导记录：安装启动引导程序
  - 分区表：记录硬盘分区状态

- 所谓的分区其实就是针对64字节的的分区表进行设置

- MBR规定主要分区和拓展分区最多一共四个，其中拓展分区最多一个

- MBR中拓展分区被破坏，那么对应的逻辑分区就会被删除

- GPT没有主、拓展、逻辑分区的概念，每个分区都是独立存在

- 硬件识别操作系统的流程：
  - BIOS：固件程序，会认识第一个可启动设备
  - MBR: 第一个可启动设备，内含启动引导代码
  - 启动引导程序(boot loader)：读取内核文件来执行的软件
  - 内核文件：开始启动操作系统

- 挂载：目录作为进入点，将磁盘分区数据放置在该目录下也就是该目录可以读取该分区数据。

- **磁盘容量小于2TB，系统会默认使用MBR分区表来安装**

# 第三章 安装CentOS 7.x

[Linux操作系统-安装Centos7环境](https://myblog.gumptlu.work/2020/08/07/%E6%8A%80%E6%9C%AF%E8%B4%B4/Linux/Linux%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F-%E5%AE%89%E8%A3%85Centos7%E7%8E%AF%E5%A2%83/)

# 第四章 首次登录与在线求助

- **Linux系统中的英文大小写代表不同的内容**

