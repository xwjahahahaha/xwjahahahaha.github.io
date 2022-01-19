---
title: “/lib64/ld-linux-x86-64.2”文件的作用
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-19 19:07:12
---

> 参考：
>
> * https://unix.stackexchange.com/questions/400621/what-is-lib64-ld-linux-x86-64-so-2-and-why-can-it-be-used-to-execute-file

[TOC]

<!-- more -->

# 一、小trick

当我们在Linux下对于一个文件没有执行权限时会被禁止，例如`chmod`命令文件

首先在`root`下将`/bin/chmod`文件的权限改为

```shell
chmod -x /bin/chmod
-rw-r--r--. 1 root root 58592 8月  20 2019 /bin/chmod
```

这样没有用户可以执行这个文件（包括`root`）:

```shell
[root@localhost ~]$ chmod
-bash: /usr/bin/chmod: 权限不够
```

此时，可以使用`/lib64/ld-linux-x86-64.2`文件恢复这个文件的权限

```shell
/usr/lib64/ld-2.17.so /bin/chmod +x /bin/chmod
[root@localhost]# /usr/lib64/ld-2.17.so /bin/chmod +x /bin/chmod
[root@localhost]# ll /bin/chmod
-rwxr-xr-x. 1 root root 58592 8月  20 2019 /bin/chmod
```

# 二、功能描述

运行该程序本身可以得到：

```shell
$ /lib64/ld-linux-x86-64.so.2
Usage: ld.so [OPTION]... EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]
You have invoked `ld.so', the helper program for shared library executables.
This program usually lives in the file `/lib/ld.so', and special directives
in executable files using ELF shared libraries tell the system's program
loader to load the helper program from this file.  This helper program loads
the shared libraries needed by the program executable, prepares the program
to run, and runs it.  You may invoke this helper program directly from the
command line to load and run an ELF executable file; this is like executing
that file itself, but always uses this helper program from the file you
specified, instead of the helper program file specified in the executable
file you run.  This is mostly of use for maintainers to test new versions
of this helper program; chances are you did not intend to run this program.
```

这个文件的功能可以归结为：

* 一个**动态链接加载器**，用于运行动态链接的程序，所以当我们运行`chmod`, 实际上内核运行的是`/lib64/ld-linux-x86-64.so.2 chmod`就像手动执行一样，即使这个文件不可以执行。所以一旦丢失这个程序就会导致很多程序不可用；

* 本身就是一个链接程序：

  ```shell
  $ readlink -f /lib64/ld-linux-x86-64.so.2
  /usr/lib64/ld-2.17.so    # 真正的位置
  ```

* 作用：加载/链接程序执行所需要的共享库，准备程序的运行，可以用其运行一个`ELF`可执行文件

# 三、不同版本位置

## x86

一般：`/lib64/ld-linux-x86-64.so.2`

`i386`:

## ARM

` /lib64/ld-linux-aarch64.so.1`

**所以也可以用于识别主机架构**
