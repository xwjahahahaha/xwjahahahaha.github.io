---
title: eBPF-3-现状调研
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-18 15:53:06
---

[TOC]

> 参考资料：
>
> * 

<!-- more -->

# 一、现存项目分析

## 1. 项目介绍与基本原理

* `bumblebee`:  https://bumblebee.io/EN
*  [IOVisor/Hover](https://github.com/iovisor/iomodules)
* [Pixie](https://pixielabs.ai/)

## 2. 项目对比

活跃度、生产环境可用性、兼容性、性能、缺陷和隐患



# 二、eBPF多语言Profile调研

## 前端工具

### BCC

### bpftrace

## 2.1 多语言适配性测试

火焰图输出



## 2.2 `profile`原理详解

### 1. 使用手册

代码中给出了多个样例：

```shell
./profile             # profile stack traces at 49 Hertz until Ctrl-C
./profile -F 99       # profile stack traces at 99 Hertz
./profile -c 1000000  # profile stack traces every 1 in a million events
./profile 5           # profile at 49 Hertz for 5 seconds only
./profile -f 5        # output in folded format for flame graphs
./profile -p 185      # only profile process with PID 185
./profile -L 185      # only profile thread with TID 185
./profile -U          # only show user space stacks (no kernel)
./profile -K          # only show kernel space stacks (no user)
./profile --cgroupmap mappath  # only trace cgroups in this BPF map
./profile --mntnsmap mappath   # only trace mount namespaces in the map
```

#### `./profile  `

> `Sampling at 49 Hertz of all threads by user + kernel stack`

输出：

```shell
# ./profile
Sampling at 49 Hertz of all threads by user + kernel stack... Hit Ctrl-C to end.
^C
    filemap_map_pages
    handle_mm_fault
    __do_page_fault
    do_page_fault
    page_fault
    [unknown]
    -                cp (9036)
        1

    [unknown]
    [unknown]
    -                sign-file (8877)
        1

    __clear_user
    iov_iter_zero
    read_iter_zero
    __vfs_read
    vfs_read
    sys_read
    entry_SYSCALL_64_fastpath
    read
    -                dd (25036)
        4

    func_a
    main
    __libc_start_main
    [unknown]
    -                func_ab (13549)
        5
```

解释：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220523213854569.png" alt="image-20220523213854569" style="zoom:50%;" />







### 2. 代码重点



### 3. 原理分析



先看`eBPF`程序：

```c
#include <uapi/linux/ptrace.h>
#include <uapi/linux/bpf_perf_event.h>
#include <linux/sched.h>

struct key_t {
    u32 pid;                            // 用户空间下的PID
    u64 kernel_ip;                      
    int user_stack_id;                  // 用户栈ID
    int kernel_stack_id;                // 内核栈ID
    char name[TASK_COMM_LEN];           // 命令名
};
BPF_HASH(counts, struct key_t);         // 每个key_t => 次数，用于计数
BPF_STACK_TRACE(stack_traces, STACK_STORAGE_SIZE);          // 栈存储,使用栈ID索引

// This code gets a bit complex. Probably not suitable for casual hacking.

int do_perf_event(struct bpf_perf_event_data *ctx) {
    // 分别获取pid和tgid
    u64 id = bpf_get_current_pid_tgid();
    u32 tgid = id >> 32;                // 高32位是线程组ID(在用户空间为PID)
    u32 pid = id;                       // 低32位是进程pid(在用户空间为线程id)

    // 用于替换判断某些功能是否开启
    if (IDLE_FILTER)
        return 0;

    if (!(THREAD_FILTER))
        return 0;

    if (container_should_be_filtered()) {
        return 0;
    }

    // create map key
    struct key_t key = {.pid = tgid};
    // 获取当前命令
    bpf_get_current_comm(&key.name, sizeof(key.name));

    // get stacks/获得调用栈，USER_STACK_GET、KERNEL_STACK_GET都是要被替换的方法字符串
    key.user_stack_id = USER_STACK_GET;
    key.kernel_stack_id = KERNEL_STACK_GET;

    // 获取kernel_ip
    if (key.kernel_stack_id >= 0) {
        // populate extras to fix the kernel stack
        u64 ip = PT_REGS_IP(&ctx->regs);
        u64 page_offset;

        // if ip isn't sane, leave key ips as zero for later checking / 如果IP不健全，将key IP设为0以便以后检查
        // 根据不同内核版本设置page_offset
#if defined(CONFIG_X86_64) && defined(__PAGE_OFFSET_BASE)
        // x64, 4.16, ..., 4.11, etc., but some earlier kernel didn't have it
        page_offset = __PAGE_OFFSET_BASE;
#elif defined(CONFIG_X86_64) && defined(__PAGE_OFFSET_BASE_L4)
        // x64, 4.17, and later
#if defined(CONFIG_DYNAMIC_MEMORY_LAYOUT) && defined(CONFIG_X86_5LEVEL)
        page_offset = __PAGE_OFFSET_BASE_L5;
#else
        page_offset = __PAGE_OFFSET_BASE_L4;
#endif
#else
        // earlier x86_64 kernels, e.g., 4.6, comes here
        // arm64, s390, powerpc, x86_32
        page_offset = PAGE_OFFSET;
#endif

        if (ip > page_offset) { 
            key.kernel_ip = ip;
        }
    }

    // 加入到map映射中
    counts.increment(key);  // 计数累加
    return 0;
}
```

如何理解`kernel_ip`?

* 









# 三、线上负载测试

## 3.1 当前集群环境内核支持分析


集群的大部分机器都是`debian9` 内核版本`4.9/4.19`五五开





# 四、自己的一些想法

## 1.容器使用`ebpf`的侵入问题
