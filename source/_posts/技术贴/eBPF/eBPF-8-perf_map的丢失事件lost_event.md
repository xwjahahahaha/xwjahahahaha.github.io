---
title: eBPF-8-perf_map的丢失事件lost_event
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-26 10:43:52
---

[TOC]


<!-- more -->

> 参考原文：
>
> * http://blog.itaysk.com/2020/04/20/ebpf-lost-events

本文内容基本翻译于上面的链接

# 一、什么是eBPF lost events

我们都知道bpf运行在内核态，bpc也提供了多种内核态与用户态交互的方式，例如：

* [bpf_trace_printk()](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-bpf_trace_printk)，利用`/sys/kernel/debug/tracing/trace_pipe`这个debug文件
* [BPF_PERF_OUTPUT](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_perf_output)，基于perf子系统原有的数据传递方式

不过官方还是推荐第二种方式，主要原因在于第一种的限制较多：`3 args max, 1 %s only, and trace_pipe is globally shared`

所以我们需要先了解一下perf这个先于bpf诞生的分析工具提供的`ring buffer`机制（bpf也是利用了此机制）

**Ring buffer：**

ring buffer是一个连续的内存区域，生产者和消费者能够同时在其中读写，ring的原因在于即使环形缓冲区满了，写入方还是可以继续从首部写入即使覆盖了原本的数据

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220726105444721.png" alt="image-20220726105444721" style="zoom:50%;" />

由此也不难知晓lost event的意义：当消费者来不及消费旧事件时，新的事件已经被写入者追赶上写入而导致事件未被消费就被覆盖或者说lost event

# 二、tracing the code

追踪bcc中的perf ring buffer的整个过程来更好的理解

如果使用python前端来写bcc，那么基本两个步骤就是：

1. Initialize buffer using the `BPF.open_perf_buffer` method
2. Start receiving events using `BPF.perf_buffer_poll` method

## Section 1 - BPF.open_perf_buffer

1. The user program is opening a perf buffer using the `BPF.open_perf_buffer`method **which receives a `lost_cb` callback** (function pointer): (source)

   使用`open_perf_buffer`函数会接受一个`lost_cb`的回调函数

   ```python
   def open_perf_buffer(self, callback, page_cnt=8, lost_cb=None)
   ```

2. `BPF.open_perf_buffer` ends up creating a “reader” using the `perf_reader_new` C function. “reader” is a bcc construct that facilitates reading from a buffer. Appendix 1 walks through this code path.

   `BPF.open_perf_buffer`会创建一个reader用于读取ring buffer

3. `perf_reader_new` C function is saving the callback in the newly created reader: (source)

   `perf_reader_new`函数会在reader中保存回调函数

   ```python
   reader->lost_cb = lost_cb;
   ```

所以在一开始我们就需要将回调函数传递，随后会保存在创建的reader中（用于读取ring buffer）

## Section 2 - BPF.perf_buffer_poll

1. The user program is calling `BPF.perf_buffer_poll` method to **start receiving events**. This is using bcc’s C function `perf_reader_poll` to read from the previously created “reader”: (source)

   使用`BPF.perf_buffer_poll`从reader中读取数据

   ```python
   lib.perf_reader_poll(len(readers), readers, timeout)
   ```

2.  `perf_reader_poll` is invoking the read function on every reader: (source) 在每个reader上调用read

   ```
   perf_reader_event_read(readers[i]);
   ```

3. `perf_reader_event_read` is reading an event. If it’s type is `PERF_RECORD_LOST` , **it will call our lost events callback**: (source)

   如果读出的事件类型是`PERF_RECORD_LOST`，那么就会调用我们的回调函数

   ```python
   if (e->type == PERF_RECORD_LOST) { 
     ... 
     reader->lost_cb(reader->cb_cookie, lost); 
     ... 
   }
   ```

所以，当从ring bufffer中读出的事件类型是`PERF_RECORD_LOST`就会调用我们预先设置的回调函数，但是我们没有submit这个类型的事件，它是怎么发生的呢？

## Section 3 - perf_submit

To submit events from our eBPF (C) program, we are instructed to initialize a “table” using the `BPF_PERF_OUTPUT` macro, and then call the `perf_submit` bcc C helper function.

为了模拟`perf_submit`，我们创建了table

1. The user eBPF program is using `BPF_PERF_OUTPUT` to define a struct. The created struct holds a pointer to the `perf_submit` function: [(source:)](https://github.com/iovisor/bcc/blob/2fa54c0bd388898fdda58f30dcfe5a68d6715efc/src/cc/export/helpers.h#L137)

2. The user eBPF program calls `table.perf_submit()` to submit an event

3. **`bpf_perf_event_output` ends up calling `perf_event_output` function from the perf subsystem.** Appendix 2 walks through this code path.

   > <font color='#e54d42'>table.perf_submit最终会调用perf子系统的perf_event_output函数，向ring buffer写入</font>

4. `perf_event_output` calls `perf_output_begin` function before it actually submits an event. [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/events/core.c#L5608)

5. `perf_output_begin` kernel function is the one that creates the “lost events”: [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/events/ring_buffer.c#L184) 

   ```c
   struct { struct perf_event_header header; u64 id; u64 lost; } lost_event;
   ```

`perf_output_begin`最终在实际提交event之前会检查和创建lost event :

```c
if (unlikely(have_lost)) { … lost_event.header.type = PERF_RECORD_LOST; … }
```

What is this `have_lost` indicator? Let's dig (final stretch, bear with me)

关键的判断指标`hava_lost`是如何工作的呢？

## Section 4 - Tracking the ring buffer's `have_lost` indicator

If we look at the `perf_output_begin` function from the kernel's perf ring buffer implementation:
1. `have_lost` variable is holding the ring buffer's `lost` field: [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/events/ring_buffer.c#L134)  `have_lost` 变量保存了环形缓冲区的 `lost` 字段：

   ```c
   have_lost = local_read(&rb->lost);
   ```
2. There’s a check if there’s enough space in the ring buffer (and also the buffer is configured to not overwrite), then we go to `fail`: [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/events/ring_buffer.c#L147)

   ```c
   if (!rb->overwrite && unlikely(CIRC_SPACE(head, tail, perf_data_size(rb)) < size))
     goto fail;
   ```

   会检查如果ring buffer已经满了或者被配置超越某个大小，那么就直接失败

3. Under `fail`, `rb->lost` is being incremented: [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/events/ring_buffer.c#L198) 在fail的实现中就是标记lost发生了，+1

   ```c
   fail: 
     local_inc(&rb->lost);
   ```

不难想象，发生lost event的原因就是写入时已满/达到最大容量修改flag，然后读取方进行了判断，如果是lost event类型，那么就会执行对应的callback函数

总体流程如下：

![lost events depiction](http://xwjpics.gumptlu.work/qinniu_uPic/2020-04-20-ebpf-lost-events_2.png)

# 三、Lost events in gobpf

如何在gobpf中使用lost event？作者已经在提交了issue：https://github.com/iovisor/gobpf/pull/235. （已合并）

重点的改变在于：

1. change `callbackData` struct to contain a lost channel in addition to the main channel:

   在`callbackData`中除了主通道之外创建`lostChan`

   ```go
   callbackDataIndex := registerCallback(&callbackData{
        receiverChan,
        lostChan,
    })
   ```

2. change the signature of the `InitPerfMap` user facing function making it also accept a channel for lost events:

   ```go
   func InitPerfMap(table *Table, receiverChan chan []byte, lostChan chan uint64) (*PerfMap, error) {...}
   ```

3. in the call to the lower level bcc C function `bpf_open_perf_buffer`, pass the registered lost callback:

   ```go
   reader, err := C.bpf_open_perf_buffer(
        (C.perf_reader_raw_cb)(unsafe.Pointer(C.rawCallback)),
        (C.perf_reader_lost_cb)(unsafe.Pointer(C.lostCallback)),
        unsafe.Pointer(uintptr(callbackDataIndex)),
        -1, cpuC, pageCntC)
   ```

# 四、Appendix

拓展阅读：https://www.kernel.org/doc/Documentation/circular-buffers.txt

## Appendix 1 - from BPF.open_perf_buffer to perf_reader_new

1. `BPF.open_perf_buffer` method is calling into bcc’s C function `lib.bpf_open_perf_buffer` [(source)](https://github.com/iovisor/bcc/blob/98c18b640117b10f923c8697be8bfff8ad39834b/src/python/bcc/table.py#L705)
2. `bpf_open_perf_buffer` function is creating a reader using `perf_reader_new` function [(source)](https://github.com/iovisor/bcc/blob/2fa54c0bd388898fdda58f30dcfe5a68d6715efc/src/cc/libbpf.c#L1192)

## Appendix 2 - from bpf_perf_event_output to perf_event_output

1. `table.perf_submit()` function is converted to `bpf_perf_event_output()` [(source)](http://blog.itaysk.com/2020/04/20/`https://sourcegraph.com/github.com/iovisor/bcc@454b138e6b75a47d4070a4f99c8f2372b383f71e/-/blob/src/cc/frontends/clang/b_frontend_action.cc#L844:15`)
2. `bpf_perf_event_output` is implemented by `BPF_FUNC_perf_event_output` [(source)](https://github.com/iovisor/bcc/blob/2fa54c0bd388898fdda58f30dcfe5a68d6715efc/src/cc/export/helpers.h#L405)
3. `BPF_FUNC_perf_event_output` is an eBPF helper: [(source)](https://elixir.bootlin.com/linux/v4.5/source/include/uapi/linux/bpf.h#L271)
4. `BPF_FUNC_perf_event_output` is creating the `bpf_perf_event_output` prototype: `bpf_perf_event_output_proto`: [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/trace/bpf_trace.c#L300)
5. `bpf_perf_event_output_proto` is pointing to the `bpf_perf_event_output` function [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/trace/bpf_trace.c#L262)
6. `bpf_perf_event_output` function is calling the `perf_event_output` function [(source)](https://elixir.bootlin.com/linux/v4.5/source/kernel/trace/bpf_trace.c#L258)
