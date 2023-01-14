---
title: 内核漏洞-COW写时复制
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-13 09:53:52
---

[TOC]


<!-- more -->

> 参考：
>
> * https://tech.meituan.com/2020/03/12/cloud-native-security.html

# 经典的Dirty CoW

在Linux内核的内存子系统处理私有只读内存映射的写时复制（Copy-on-Write，CoW）机制的方式中发现了一个竞争冲突。一个没有特权的本地用户，可能会利用此漏洞获得对其他情况下只读内存映射的写访问权限，从而增加他们在系统上的特权，这就是知名的Dirty CoW漏洞。

Dirty CoW漏洞的逃逸的实现思路和上述的思路不太一样，采取Overwrite vDSO技术。

vDSO（Virtual Dynamic Shared Object）是内核为了减少内核与用户空间频繁切换，提高系统调用效率而设计的机制。它同时映射在内核空间以及每一个进程的虚拟内存中，包括那些以root权限运行的进程。通过调用那些不需要上下文切换（context switching）的系统调用可以加快这一步骤（定位vDSO）。vDSO在用户空间（userspace）映射为R/X，而在内核空间（kernelspace）则为R/W。这允许我们在内核空间修改它，接着在用户空间执行。又因为容器与宿主机内核共享，所以可以直接使用这项技术逃逸容器。

**利用步骤如下：**

1. 获取vDSO地址，在新版的glibc中可以直接调用getauxval()函数获取；
2. 通过vDSO地址找到clock_gettime()函数地址，检查是否可以hijack；
3. 创建监听socket；
4. 触发漏洞，Dirty CoW是由于内核内存管理系统实现CoW时产生的漏洞。通过条件竞争，把握好在恰当的时机，利用CoW的特性可以将文件的read-only映射该为write。子进程不停地检查是否成功写入。父进程创建二个线程，ptrace_thread线程向vDSO写入shellcode。madvise_thread线程释放vDSO映射空间，影响ptrace_thread线程CoW的过程，产生条件竞争，当条件触发就能写入成功。
5. 执行shellcode，等待从宿主机返回root shell，成功后恢复vDSO原始数据。
