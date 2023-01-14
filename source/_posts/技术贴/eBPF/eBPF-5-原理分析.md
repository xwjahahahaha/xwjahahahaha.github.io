---
title: eBPF-4-原理分析
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-24 00:07:47
---



<!-- more -->

# 原理

## kprobes && uprobes基本原理

基本流程如下：

1. 复制插桩目的地址的字节内容并存储，给断点指令腾出位置
2. 用单步中断指令(`int3(x86)、jmp(优化)`)覆盖目标地址：
3. 当指令流执行到断点后会检查是否是`kprobes`注册的，是则执行`kprobes`处理函数
4. 原始指令流继续执行
5. 当不再需要`kprobes`时，原始的字节内容会被复制回目标地址

> 注意：第三步执行到断点处时并不会中断原本指令流的执行，`kprobes`处理函数和原始指令流并行

例子（反汇编`readline()`函数，插桩前和插桩后）：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220522163637027.png" alt="image-20220522163637027" style="zoom:47%;" />

如果使用了`Ftrace`优化，则执行流程变为：

1. 将一个`Ftrace kprobes`处理函数注册为对应函数的`Ftrace`处理器
2. 在函数起始处执行内建入口函数时会调用`Ftrace`，然后调用`kprobe`处理函数
3. 当`kprobe`不再使用时可以从`Ftrace`中移除

`kretprobes`的流程：

1. 对函数的入口进行`kprobes`插桩
2. `kprobes`命中后，将原函数的返回地址保存并替换为一个`蹦床`函数地址
3. 当函数最终返回时会交给蹦床返回函数进行处理
4. 在`kretprobes`处理完之后再返回到之前保存的地址
5. 不需要时就会被移除

> 注意点：
>
> * `kprobes`处理的过程中一般会禁止抢占或者中断
> * `kprobes`从设计上就保证了修改字节指令的安全性，例如不可给自己插桩、`int3、jmp`等指令也保证了在修改代码时其他`cpu`核心不会执行指令
> * 某些`ARM`不允许修改内核代码区，所以要注意可能无法正常工作

## tracepoint基本原理

## USDT基本原理

