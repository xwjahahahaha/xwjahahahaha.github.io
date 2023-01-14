---
title: ebpf-7-开发技巧与常见错误总结
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-19 10:40:02
---

[TOC]

<!-- more -->

> 参考：
>
> * https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#4-bpf_histogram

# 零、开发相关手册资料

* bcc reference： https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md 
  * 能看到很多bpf相关的map和辅助函数
* linux man中的cgroup辅助函数介绍：https://man7.org/linux/man-pages/man7/bpf-helpers.7.html

# 一、开发技巧

## 1. 如何确定自己的内核版本是否有需要attach的内核函数？

### 1. cat /proc/kallsyms | grep xxxx

解决方法：查看内核符号表

`cat /proc/kallsyms | grep xxxx`

例如我在使用`nfsdist-bpfcc`工具的时候，报错提示：

![image-20220719104216734](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220719104216734.png)

那么，就可以检查一下自己的内核版本中是否有该函数

`cat /proc/kallsyms | fgrep nfs_file_read` 如果没有输出确实就没有该函数（`kernel version: 5.15.0-41-generic`），所以无法attach

而在另一个内核版本（`kernel version: 4.19.0-0.bpo.9-amd64`），则是可以找到这样的函数：

![image-20220719104532463](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220719104532463.png)

### 2. get_kprobe_functions

在BCC的python前端中，可以使用类似于如下的方式（`get_kprobe_functions`）去实现：

```python
if BPF.get_kprobe_functions(b'nfs4_file_open'):
  b.attach_kprobe(event="nfs4_file_open", fn_name="trace_entry")
  b.attach_kretprobe(event="nfs4_file_open", fn_name="trace_open_return")
else:
  b.attach_kprobe(event="nfs_file_open", fn_name="trace_entry")
  b.attach_kretprobe(event="nfs_file_open", fn_name="trace_open_return")
```

## 2. 怎样理解BPF_HISTOGRAM(name [, key_type [, size ]])的第三个参数size？

从bcc的官方使用手册中可以看到基本设定：https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#4-bpf_histogram

Syntax: `BPF_HISTOGRAM(name [, key_type [, size ]])`

Creates a histogram map named `name`, with optional parameters.

Defaults: `BPF_HISTOGRAM(name, key_type=int, size=64)`

For example:

```
BPF_HISTOGRAM(dist);
```

This creates a histogram named `dist`, **which defaults to 64 buckets indexed by keys of type int.**

This is a wrapper macro for `BPF_TABLE("histgram", ...)`.

Methods (covered later): map.increment().

也就是说size是直方图中桶的数量的设定，那么这个桶的数量就与`key_type`的种类数相关(默认是64)

例如：

```c
...
// 27 buckets for latency, max range is 33.6s .. 67.1s
  const u8 max_latency_slot = 26;
// Key for latency measurements map
struct latency_key_t {
  u8 op;
  u64 slot;
};

// Histograms to record latencies (2 is the number of ops in reclaim_op)
BPF_HISTOGRAM(xfs_reclaim_latency, struct latency_key_t, (max_latency_slot + 2) * 2);

// Possible reclaim operations
enum reclaim_op {
  S_COUNT = 1,
  S_FREE  = 2
};
...
```

这里`(max_latency_slot + 2) * 2`计算的解释如下：

* 看`latency_key_t`结构体的两个参数，首先是`slot`, 已经定义了其最大为26，从0～26共有27个槽，再加上`+Inf`（表示最大槽到无穷大），所以一共有28个槽
* `*2`的原因在注释中也解释了，因为`op`（也就是操作有两种，见下面的`reclaim_op`），所以每一种有28个，共两种就需要`*2`，这就是总槽数

类似的还可以看到：官方工具cpudist中https://github.com/iovisor/bcc/blob/c1a767b34fc800436f7fe90f2e5614b2c27bc376/tools/cpudist.py

![image-20220721103926046](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220721103926046.png)

为了得到最大槽数，还需要找到最大的pid

## 3. 如何寻找需要的内核函数attach

对于一个新的应用，期望找到对应功能的函数（也可能是获取对应的参数），然后我们进行bpf源程序的开发，下面就总结和记录一下我使用过的思路：

### 1. 查找内核tracepoint

一般来说都会优先选择静态插桩点，所以先找`tracepoint`是一个比较好的思路

可以在如下位置看到所有的`tracepoint`事件:

`/sys/kernel/debug/tracing/events/`

<font color='#e54d42'>如果不确定程序会调用哪些tracepoint，可以使用其他工具先观测一下</font>

例如：perf先观测nfs：`sudo perf stat -e 'nfs:*' -a`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220725170826673.png" alt="image-20220725170826673" style="zoom: 50%;" />

这样就比较明确了（但是有一些可能并不在perf包含的事件中，可以通过`perf list`查看支持的所有事件）

### 2. 看应用的内核源码

这就要去找Linux对应版本的内核源码，找到对应的模块，然后去寻找合适的函数和参数了

linux内核源码在线查看（注意：找对应的版本，因为内核函数名可能会有所改动）：https://github.com/torvalds/linux/tags

> 对于用户态的应用，也可以使用`uprobe`，但是要注意函数的稳定性，不会随着版本轻易修改（当然这难以避免）

## 4. BPF_HASH的使用注意点

首先引用bcc的指南部分介绍：

Syntax: `BPF_HASH(name [, key_type [, leaf_type [, size]]])`

Creates a hash map (associative array) named `name`, with optional parameters.

Defaults: `BPF_HASH(name, key_type=u64, leaf_type=u64, size=10240)`

For example:

```c
BPF_HASH(start, struct request *);
```

This creates a hash named `start` where the key is a `struct request *`, and the value defaults to u64. This hash is used by the disksnoop.py example for saving timestamps for each I/O request, where the key is the pointer to struct request, and the value is the timestamp.

This is a wrapper macro for `BPF_TABLE("hash", ...)`.

Methods (covered later): map.lookup(), map.lookup_or_try_init(), map.delete(), map.update(), map.insert(), map.increment().

Examples in situ: [search /examples](https://github.com/iovisor/bcc/search?q=BPF_HASH+path%3Aexamples&type=Code), [search /tools](https://github.com/iovisor/bcc/search?q=BPF_HASH+path%3Atools&type=Code)

核心的问题在于使用结构体指针作为key或者value时，其本身的key就是一个指针的参数，所以如果传参使用的是结构体的指针，那么就是一个二级指针，在拿出的时候需要处理二级指针

## 5. 如何查看目前的bpf程序状态

有时候我们需要查看自己的bpf程序是否已经加载，并想获得更多的信息

`bpftool`解决了这个问题

### bpftool的安装：

```shell
# 我在debian 9的环境上可以直接apt安装
sudo apt update
sudo apt install bpftool
sudo bpftool
```

> 其他安装方式可见：https://blog.csdn.net/weixin_44260459/article/details/123036982

### bpftool的使用

* 查看当前已经加载的bpf程序：

  `sudo bpftool prog show` 或者期望json格式，还可以通过id过滤：`sudo bpftool prog show --json id 3 | jq`

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220728094726754.png" alt="image-20220728094726754" style="zoom:50%;" />

* 查看程序的字节码信息：

  `bpftool prog dump xlated id 1154`

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220728094800088.png" alt="image-20220728094800088" style="zoom:50%;" />

其他更多使用见：https://blog.csdn.net/Longyu_wlz/article/details/109931993

# 二、常见错误

## 1.bcc load bpf程序提示invalid indirect read from stack off xxx+xxx size xxx

### 问题描述

![image-20220721100455632](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220721100455632.png)

load bpf程序的时候，被拒绝了

### 原因分析

找到报错的函数：

https://patchwork.ozlabs.org/project/netdev/patch/20171215015517.409513-5-ast@kernel.org/

![image-20220721100410320](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220721100410320.png)

函数的注释翻译为：当寄存器'regno'被传递给将从指针读取'access_size'字节的函数时，确保它在堆栈边界内，并初始化堆栈的所有元素

其实初步原因就可以判断为访问的地址超出了栈的边界

### 解决

发现错误的差异点所在，关于bpf的源代码：

如果像下面这样赋值就会报如上的错：

```c
struct dist_key{
  u8 op;
  u64 slot;
};

....
  
static int trace_return(struct pt_regs *ctx, enum opration op) {
  ...
  // store as histogram 
  struct dist_key key = {
    .op = op,
    .slot = latency_slot
  };
  ...
}
```

而改成如下就没问题：

```c
struct dist_key{
  u8 op;
  u64 slot;
};

....
  
static int trace_return(struct pt_regs *ctx, enum opration op) {
  ...
  // store as histogram 
  struct dist_key key = {};
  key.op = op;
  key.slot = latency_slot;
  ...
}
```

但是还未深入知晓原因所在。。。

## 2. 引用其他包错误

例如引用`string.h`, 下面的方式是错误的

```c
#include <string.h>
```

会报错：`not found`

![image-20220721203502671](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220721203502671.png)

解决方法：

```c
#include <linux/string.h>
```

## 3. 引用只读数据或静态区数据错误

![image-20220721203253028](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220721203253028.png)

> 如果你引用一个全局变量或静态变量，或只读区域中的数据，可能会发生'unknown opcode'。例如，'char *p = "hello"'将导致p引用一个只读section， 'char p[] = "hello"'将把"hello"存储在堆栈中

```c
// 屏蔽一些命令
char comm[TASK_COMM_LEN];
bpf_get_current_comm(&comm, sizeof(comm));
if (strcmp(comm, "bash") == 0 || strcmp(comm, "sh") == 0) {
  return 0;
}
```

