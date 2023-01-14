---
title: 3-linux栈回溯x86_64
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-30 11:33:16
---

[TOC]

<!-- more -->

> 转载自：
>
> * https://zhuanlan.zhihu.com/p/302726082

# 前序

前面几个章节我们了解了《[ELF文件格式](https://zhuanlan.zhihu.com/p/286088470)》、《[ELF文件加载过程](https://zhuanlan.zhihu.com/p/287863861)》、《[x86通用寄存器](https://zhuanlan.zhihu.com/p/288636064)》、《[x86栈帧原理](https://zhuanlan.zhihu.com/p/290689333)》和《[linux 进程内核栈](https://zhuanlan.zhihu.com/p/296750228)》，对x86平台上程序运行和调试机制有了一定认识。接下来我们从程序调试的角度，来一同学习下x86栈回溯的原理和使用。

# 栈回溯发展

我们在在调试的时候，经常需要获取`CFI（Call Frame Information）`，进行堆栈回溯。在《[x86栈帧原理](https://zhuanlan.zhihu.com/p/290689333)》一文中，我们知道x86上栈帧有多种结构，栈回溯根据栈帧结构不同，而采取不同的回溯方式。下面一起来看下栈帧的发展和对应的栈回溯方式。

## 1. frame pointer

`frame pointer`是经典栈帧结构。在《[X86栈帧原理](https://zhuanlan.zhihu.com/p/290689333)》一文中，介绍过x86经典的栈帧结构：即使用`ebp`寄存器保存栈帧地址，`esp`保存栈顶指针，在过程调用时将上一个栈帧地址入栈保存：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-42cbcdf6a7e09ff0e1598b588ef1c640_1440w.jpg)

这种方式，在栈回溯时，**调试器可以轻松获取旧的堆栈指针并继续展开其他栈帧**。这种方式的优缺点如下：

优点：使用起来方便快捷，回溯栈比较简单；

缺点：

* 需要固定占用一个通用寄存器
* 保存回溯信息有指令开销（保存ebp寄存器）
* 最主要的是没有足够的信息被编码，只能恢复堆栈寄存器

前面章节提到，这种栈帧结构，在`x86_64`中被抛弃

gcc 在64位编译器中默认不使用`rbp`保存栈帧地址，即不再保存上一个栈帧地址，因此在`x86_64`中也就无法使用这种栈回溯方式（不过可以使用`-fno-omit-frame-pointer`选项保存栈帧rbp）。

## 2. DWARF 

调试信息标准**DWARF** (`Debugging With Attributed Record Formats`)定义了一个叫`.debug_frame `的`section`。

该调试信息格式支持处理无基址指针的方法，可以将`ebp`用作常规寄存器，但是**当保存`esp`时，它必须在`.debug_frame`节中产生一个注释**，告诉调试器什么指令将其保存在何处。

> 也就是通过`.debug_frame`保存，而不是占用`ebp`寄存器

它可以告诉调试器如何还原更多信息，而不仅仅是`ebp`。而且没有指令开销。不过方案仍然不足以进行异常处理，因为它不支持指定原始语言，在手写汇编代码时用起来很挣扎。

## 3. EH_FRAME段

现代Linux操作系统在LSB (Linux Standard Base)标准中定义了一个`.eh_frame`的` section`来解决上述的难题

这个`section`和`.debug_frame`非常类似，但是它编码紧凑，可以**随程序一起加载**

我们在《[ELF文件格式](https://zhuanlan.zhihu.com/p/286088470)》一文中简要介绍了`.eh_frame `的`section`。这个`.eh_frame`段中存储着跟函数入栈相关的关键数据。当函数执行入栈指令后，**在该段会保存跟入栈指令一一对应的编码数据**，根据这些编码数据，就能计算出当前函数栈大小和cpu的哪些寄存器入栈了，在栈中什么位置。

无论是否有-g选项，gcc默认都会生成`.eh_frame`和`.eh_frame_hdr section`

`-fno-asynchronous-unwind-tables`选项可以禁止生成`.eh_frame`和`.eh_frame_hdr section`

## 4. CFI directives

无论是`.debug_frame`还是`.eh_frame`，都有一个问题：生成它们的最直接方法是在`asm`汇编文件中写一个固定格式的长表。由于编译器不知道代码的确切大小，因此这会导致编码效率低下，表格也很难阅读。

`CFI`伪指令是改进`.debug_frame`和`.eh_frame`生成的一种方法

`CFI directives`伪指令是一组生成`CFI`调试信息的高级语言，它的形式截取如下：

```c
.cfi_startproc
pushl %ebp
.cfi_def_cfa_offset 8
.cfi_offset ebp, -8
```

关于汇编器利用这些伪指令来生成`.debug_frame`还是`.eh_frame`，在`.cfi_sections`指令中定义

如果只是调试需求可以生成`.debug_frame`，如果需要在运行时调用需要生成`.eh_frame`

在认识了CFI存储发展历史后，下面会进行详解分析

`linux userspace` 上运行的`ELF`可执行文件，使用的就是`eh_frame`这种调试信息格式，我们下面就详细分析`eh_frame` 原理和`unwind`式。

# 详解CFI 伪指令

gcc 使用CFI 伪指令来生成`eh_frame`信息，在使用c语言编写时，gcc会自动帮我们产生CFI伪指令。我们在深入了解eh_frame格式之前，先看下CFI 伪指令。

在`GAS(GCC Assembler)`汇编编译器[CFI(Call Frame Information)](https://link.zhihu.com/?target=https%3A//sourceware.org/binutils/docs-2.31/as/CFI-directives.html)/[ARM CFI](https://link.zhihu.com/?target=https%3A//sourceware.org/binutils/docs/as/ARM-Directives.html%23arm_005ffnstart)文档中，对所有CFI伪指令的含义有详细描述。这里挑选几个重要的伪指令分析。

（1）.**cfi_startproc**

> 用在每个函数的入口处。

（2）.**cfi_endproc**

> `.cfi_endproc`用在函数的结束处，和`.cfi_startproc`对应。

（3）.**cfi_def_cfa_offset** [offset]

> 用来修改修改CFA计算规则，基址寄存器不变，offset变化：
> `CFA = register + offset(new)`

（4）.**cfi_def_cfa_register** register

> 用来修改修改CFA计算规则，基址寄存器从rsp转移到新的register。
> `register = new register`

（5）.**cfi_offset** register, offset

> 寄存器register上一次值保存在CFA偏移offset的堆栈中：
> `*(CFA + offset) = register(pre_value)`

（6）.**cfi_def_cfa** register, offset

> 用来定义CFA的计算规则：
> `CFA = register + offset`
> 默认基址寄存器`register = rsp`
> x86_64的register编号从0-15对应下表。rbp的register编号为6，rsp的register编号为7。
> `%rax，%rbx，%rcx，%rdx，%esi，%edi，%rbp，%rsp，%r8，%r9，%r10，%r11，%r12，%r13，%r14，%r15`，参考[x86_64 ABI](https://link.zhihu.com/?target=https%3A//refspecs.linuxfoundation.org/elf/x86_64-abi-0.95.pdf)

在使用c语言编写时，gcc会自动帮我们产生CFI伪指令。我们通过一个例子来看下x86_64上汇编里的CFI伪指令：

```c
#include <stdio.h>
 
int test(int x)
{
        int c =10;
        return x*c;
}
 
void main()
{
        int a,b;

        a = 10;
        b = 11;
        printf("hello test~, %d\n", a+b); 
        a = test(a+b);
}
```

test.c 程序汇编代码（test函数部分）：

```c
//gcc -S test.c -o test.s
      5 test:
      6 .LFB0:
      7         .cfi_startproc
      8         pushq   %rbp
      9         .cfi_def_cfa_offset 16
     10         .cfi_offset 6, -16
     11         movq    %rsp, %rbp
     12         .cfi_def_cfa_register 6
     13         movl    %edi, -20(%rbp)
     14         movl    $10, -4(%rbp)
     15         movl    -20(%rbp), %eax
     16         imull   -4(%rbp), %eax
     17         popq    %rbp
     18         .cfi_def_cfa 7, 8
     19         ret
     20         .cfi_endproc
     21 .LFE0:
     22         .size   test, .-test
     23         .globl  main
     24         .type   main, @function
```

`FDT`中包含`PC`范围，可以根据函数符号地址，找到函数对应的`FDT`

`readelf -s `找到`test`和`main`符号地址：

```c
    95: 000000000040054d    43 FUNC    GLOBAL DEFAULT   11 main
    96: 0000000000601020     0 OBJECT  GLOBAL HIDDEN    21 __TMC_END__
    97: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
    98: 0000000000400428     0 FUNC    GLOBAL HIDDEN    10 _init
    99: 0000000000400536    23 FUNC    GLOBAL DEFAULT   11 test
```

`readelf -wF` 根据test函数入口地址，找到对应的FDE如下：

```c
00000040 000000000000001c 00000044 FDE cie=00000000 pc=0000000000400536..000000000040054d
   LOC           CFA      rbp   ra    
0000000000400536 rsp+8    u     c-8   
0000000000400537 rsp+16   c-16  c-8   
000000000040053a rbp+16   c-16  c-8   
000000000040054c rsp+8    c-16  c-8 
```

`FDE`和汇编语句对照关系：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-c8cc1e5c0ae20f6b4826e2d2add9c09e_1440w.jpg)

1. 将 `return addr`压栈，此时对应`FDE`中的`CFA = rsp+8`;
2. 将`rbp`压栈，基址寄存器不变情况下修改CFA计算规则，`CFA = rsp+16`;
3. 寄存器`6（rbp）`上一次的值，相对CFA的位置。即`rbp(old) = CFA-16`；
4. 把`rsp`的值赋给`rbp`，后续的堆栈计算，使用`rbp`作为基址寄存器。3和4组合起来，`CFA=rbp+16`；
5. 定义CFA的计算规则，将`rsp`恢复成`rbp`的值，此时`CFA = 寄存器7(rsp)+8`；
6. 返回上一层函数（main），弹出`return addr`

# 详解eh_frame

上面简要介绍了几种调用栈信息保存方式，以及CFI伪指令的使用。本文接下来重点围绕linux 中x86_64中，`.eh_frame` 格式**unwind栈回溯**原理

每个`.eh_frame section `包含一个或多个**CFI**(`Call Frame Information`)记录，记录的条目数量由`.eh_frame `段大小决定

每条CFI记录包含一个**CIE**(`Common Information Entry Record`)记录，每个CIE包含一个或者多个**FDE**(`Frame Description Entry`)记录。

通常情况下，**CIE对应一个文件，FDE对应一个函数**。

CFI、FDE格式和字段含义，请参考[LSB官方手册](https://link.zhihu.com/?target=https%3A//refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html)，这里不再罗列。接下来通过一个实际的例子来分析下`CFI`、`FDE`数据。

```c
#include <stdio.h>
 
int test(int x)
{
        int c =10;
        return x*c;
}
 
void main()
{
        int a,b;

        a = 10;
        b = 11;
        printf("hello test~, %d\n", a+b); 
        a = test(a+b);
}
```

`gcc test.c `编译，使用`readelf -wF a.out `查看elf文件中的`.eh_frame`解析信息（节选）：

```text
Contents of the .eh_frame section:
 
00000000 0000000000000014 00000000 CIE "zR" cf=1 df=-8 ra=16
   LOC           CFA      ra   
0000000000000000 rsp+8    u    
...
000000c8 0000000000000044 0000009c FDE cie=00000030 pc=00000000000006b0..0000000000000715
   LOC           CFA      rbx   rbp   r12   r13   r14   r15   ra   
00000000000006b0 rsp+8    u     u     u     u     u     u     c-8  
00000000000006b2 rsp+16   u     u     u     u     u     c-16  c-8  
00000000000006b4 rsp+24   u     u     u     u     c-24  c-16  c-8  
00000000000006b9 rsp+32   u     u     u     c-32  c-24  c-16  c-8  
00000000000006bb rsp+40   u     u     c-40  c-32  c-24  c-16  c-8  
00000000000006c3 rsp+48   u     c-48  c-40  c-32  c-24  c-16  c-8  
00000000000006cb rsp+56   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
00000000000006d8 rsp+64   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
000000000000070a rsp+56   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
000000000000070b rsp+48   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
000000000000070c rsp+40   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
000000000000070e rsp+32   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
0000000000000710 rsp+24   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
0000000000000712 rsp+16   c-56  c-48  c-40  c-32  c-24  c-16  c-8  
0000000000000714 rsp+8    c-56  c-48  c-40  c-32  c-24  c-16  c-8
```

可以看到.eh_frame总体架构就是由`CIE`和`FDE`组成的。其中最核心的就是`FDE`的组织，读懂它条目的所有字段基本就理解了`unwind`的含义：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-6754891e8dff9e416cd3e07dd5ded3cd_1440w.jpg)

CFA (`Canonical Frame Address, which is the address of %rsp in the caller frame`)，**CFA就是上一级调用者的堆栈指针**。

上图详细说明了怎么样利用`.eh_frame`来进行栈回溯：

1. 根据当前的PC在`.eh_frame`中找到对应的条目，根据条目提供的各种偏移计算其他信息
2. 首先根据`CFA = rsp+8`，把当前`rsp+8`得到`CFA`的值。再根据CFA的值计算出通用寄存器和返回地址在堆栈中的位置
3. 通用寄存器栈位置计算：例如：`rbx = CFA-56`
4. 返回地址`ra`的栈位置计算：`ra = CFA-8`
5. 根据`ra`的值，重复步骤1到4，就形成了完整的栈回溯。

还可以使用`readelf -wf xxx`命令来查看elf文件中的`.eh_frame`原始信息，这里不再列出。

# 用户态栈回溯

在实际编程过程中，我们起始并不需要自己去处理`unwind`信息来进行栈回溯，在linux上有很多已经封装好的栈回溯API，我们在使用调试工具（如gdb）时，工具本身就集成了栈回溯代码。下面一起来看下有哪些已经实现好的API供调用。

## 1. gcc提供的取栈 API

gcc提供了`__builtin_return_address() `宏来获取函数的返回地址（栈中的`return addr`）level为参数，如果level为0，那么就是请求当前函数的返回地址；如果level为1，那么就是请求进行调用的函数的返回地址。然后我们通过`objdump`出来的文件去查找打印出来的函数地址，就可以知道函数调用栈了

下面还是通过一个例子来看下效果：

```c
#include <stdio.h>

void f()
{
        printf("%p,%p\n" , __builtin_return_address(0), __builtin_return_address(1));
}

void g()
{
        f();
}
int main()
{
        g();
}
```

执行：

```c
[root@localhost]# ./a.out 
0x4005cc, 0x4005dd
```

反汇编查看两个地址位置：

```c
0000000000400596 <f>:
  400596:	55                   	push   %rbp
	… …
  4005b6:	e8 e5 fe ff ff       	callq  4004a0 <printf@plt>
  4005bb:	90                   	nop
  4005bc:	5d                   	pop    %rbp
  4005bd:	c3                   	retq   

00000000004005be <g>:
  4005be:	55                   	push   %rbp
  4005bf:	48 89 e5             	mov    %rsp,%rbp
  4005c2:	b8 00 00 00 00       	mov    $0x0,%eax
  4005c7:	e8 ca ff ff ff       	callq  400596 <f>
  4005cc:	90                   	nop
  4005cd:	5d                   	pop    %rbp
  4005ce:	c3                   	retq   

00000000004005cf <main>:
  4005cf:	55                   	push   %rbp
  4005d0:	48 89 e5             	mov    %rsp,%rbp
  4005d3:	b8 00 00 00 00       	mov    $0x0,%eax
  4005d8:	e8 e1 ff ff ff       	callq  4005be <g>
  4005dd:	b8 00 00 00 00       	mov    $0x0,%eax
  4005e2:	5d                   	pop    %rbp
… …
```

1. 打印的地址0：`0x4005cc`

   由于地址是函数返回地址，因此查看其上一句汇编语句是函数g 里的`callq 400596 <f>`。我们通过《[x86栈帧原理](https://zhuanlan.zhihu.com/p/290689333)》知道，call指令其实对应于`push`和`jump`两条指令，`0x4005cc`就是`push`压栈的返回地址。因此“当前函数（printer）”上一个函数就是函数g；

2. 打印的地址0：`0x4005dd`

   计算同上，找到`callq 4005be <g>`，g的上一个函数就是main函数。

## 2. glibc取栈

`glibc`提供了一对函数**backtrace**()和**backtrace_symbols**()来回溯栈信息。man手册中函数原型：

```c
#include <execinfo.h>

int backtrace(void **buffer, int size);

char **backtrace_symbols(void *const *buffer, int size);

void backtrace_symbols_fd(void *const *buffer, int size, int fd);

//The array backtrace_symbols return is malloc(3)ed  by  backtrace_symbols(),  and  must  be  freed by the caller.
```

`backtrace_symbols `可以将`backtrace `收到的地址数据翻译成易阅读的符号，因此可以两个组合使用（注意需要主动free掉 `backtrace_symbols` 返回的地址！）：

```c
#include <stdio.h>
#include <execinfo.h>

#define BACKTRACE_SIZ   64
void do_backtrace()
{
    void    *array[BACKTRACE_SIZ];
    size_t   size, i;
    char   **strings;

    size = backtrace(array, BACKTRACE_SIZ);
    strings = backtrace_symbols(array, size);

    for (i = 0; i < size; i++) {
        printf("%p : %s\n", array[i], strings[i]);
    }

    free(strings);  // malloced by backtrace_symbols
}

void func1()
{
    do_backtrace();
}

int  main()
{
    func1();
    return 0;
}
```

执行：

```c
[root@localhost]# ./a.out 
0x400695 : ./a.out() [0x400695]
0x400720 : ./a.out() [0x400720]
0x400731 : ./a.out() [0x400731]
0x7f94b08666a3 : /lib64/libc.so.6(__libc_start_main+0xf3) [0x7f94b08666a3]
0x4005be : ./a.out() [0x4005be]
```

同样采用1 中`__builtin_return_address() `宏 的方法，使用`objdump -d`反汇编查看汇编地址：

```c
0000000000400676 <do_backtrace>:
	… …
  400690:	e8 db fe ff ff       	callq  400570 <backtrace@plt>
  400695:	48 98                	cltq   
	… …
0000000000400712 <func1>:
	… …
  40071b:	e8 56 ff ff ff       	callq  400676 <do_backtrace>
  400720:	90                   	nop
	… …
0000000000400723 <main>:
	… …
  40072c:	e8 e1 ff ff ff       	callq  400712 <func1>
  400731:	b8 00 00 00 00       	mov    $0x0,%eax
  … …
```

三个函数返回地址对比，得到函数调用栈

不过可以在编译时使用`-rdynamic`把调试信息链接进文件，运行会打印出更详细的符号信息：

```c
0x400865 : ./a.out(do_backtrace+0x1f) [0x400865]
0x4008f0 : ./a.out(func1+0xe) [0x4008f0]
0x400901 : ./a.out(main+0xe) [0x400901]
0x7f73ee2af6a3 : /lib64/libc.so.6(__libc_start_main+0xf3) [0x7f73ee2af6a3]
0x40078e : ./a.out(_start+0x2e) [0x40078e]
```

## 3. unwind库

这里的`libunwind`库指的是非gcc内置的外部库。目前有很多unwind库，有LLVM内置的libunwind 、[http://nongnu.org](https://link.zhihu.com/?target=http%3A//nongnu.org)的“[The libunwind project](https://link.zhihu.com/?target=https%3A//savannah.nongnu.org/news/%3Fgroup%3Dlibunwind)”、Android 9.0开始使用[新的unwind库](https://link.zhihu.com/?target=https%3A//android.googlesource.com/platform/system/core/%2B/master/libunwindstack)等等。

本文主要关心x86_64平台主要在用的unwind库。笔者在用的SUSE 和 centos发行版都是用的 [http://nongnu.org](https://link.zhihu.com/?target=http%3A//nongnu.org)的“[The libunwind project](https://link.zhihu.com/?target=https%3A//savannah.nongnu.org/news/%3Fgroup%3Dlibunwind)”版本。因此后面讨论的均以此unwind库为基础。该开源计划网址是：[http://savannah.nongnu.org/projects/libunwind/](https://link.zhihu.com/?target=http%3A//savannah.nongnu.org/projects/libunwind/) ，可选择源码安装或使用如下命令安装：

```c
// suse11sp4
zypper install libunwind libunwind-devel
```

本文实验环境是：

```c
suse11sp4
libunwind-0.98.6-26.6
libunwind-devel-0.98.6-26.6
```

接下来利用`libunwind`来做栈回溯：

```c
void dump_backtrace() {
    unw_cursor_t cursor;
    unw_context_t uc;
    unw_word_t ip, sp;
    char buf[4096];
    unw_word_t offset;
    unw_getcontext(&uc);            // store registers
    unw_init_local(&cursor, &uc);   // initialze with context

    printf("==========\n");

    while (unw_step(&cursor) > 0) {                         // unwind to older stack frame
        unw_get_reg(&cursor, UNW_REG_IP, &ip);              // read register, rip
        unw_get_reg(&cursor, UNW_REG_SP, &sp);              // read register, rbp
        unw_get_proc_name(&cursor, buf, 4095, &offset);     // get name and offset
        printf("0x%016lx <%s+0x%lx>\n", ip, buf, offset);   // x86_64, unw_word_t == uint64_t
    }

    printf("==========\n");
}

void show_backtrace()
{
        dump_backtrace();
}

int main()
{
        show_backtrace();
        return 0;
}
```

编译：`gcc test.c -o test -g -lunwind`

运行：

```c
0x0000000000400936 <show_backtrace+0xe>
0x0000000000400946 <main+0xe>
0x00007f692e189c36 <__libc_start_main+0xe6>
0x0000000000400709 <_start+0x29>
```

其中 `unw_get_reg `函数读取指定的寄存器到变量中。`UNW_REG_IP`和`UNW_REG_SP`在x86_64中分别表示rip和rsp寄存器。除此之外，还可以通过参数#2 指定读取其他寄存器：

| UNW_X86_64_RAX rax |
| ------------------ |
| UNW_X86_64_RDX rdx |
| UNW_X86_64_RCX rcx |
| UNW_X86_64_RBX rbx |
| … …                |

# kernel 栈回溯实现

我们在内核开发时，当内核`panic`后，经常可以看到类似如下的call trace信息：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-76f6b53c9006b8e0ae09948d67185937_1440w.jpg)

因为在内核中，已经有现成的代码实现了`kernel`空间的栈回溯。因此我们在自己driver代码里，可以主动调用**dump_stack**()函数打印栈回溯信息。下面简要分析下内核空间栈回溯是如何实现的。

不过在x86中，linux内核 并非像用户空间那样使用unwind方式进行栈回溯。linux x86体系下栈回溯有一段发展历史：

1. 在早期时候，内核只支持`frame point`方式，即`CONFIG_FRAME_POINTER=y`；

2. 在Linux kernels: 2.6.18 期间，内核加入了对unwind方式支持，提供`CONFIG_STACK_UNWIND`，在早期SUSE发行版中有使用（比如作者使用过的**SUSE11**）

3. 不过随后在Linux kernels: 2.6.19 中就被移除，理由是不稳定引入了过多的问题，感兴趣的可以看[linus提交邮件](https://link.zhihu.com/?target=https%3A//git.congatec.com/android/qmx6_kernel/commit/d1526e2cda64d5a1de56aef50bad9e5df14245c2)；

4. 到了Linux kernels: 4.15 ，内核使用`ORC (Oops Rewind Capability) unwinder` 替代了以前出现过的`DWARF unwinder`

5. 至此以后，内核提供下面两个`kconfig`选项供选择：

   `CONFIG_UNWINDER_FRAME_POINTER=y (Ubuntu, etc.) ` or
   `CONFIG_UNWINDER_ORC=y (RHEL 8, SuSE, Debian, ...)`

由于本文重点在`unwind`栈回溯（user space 常用），内核中常见的`frame point` 和 `ORC`栈回溯专门章节介绍。

> 参考
>
> [http://web.archive.org/web/20130111101034/http://blog.mozilla.org/respindola/2011/05/12/cfi-directives](https://link.zhihu.com/?target=http%3A//web.archive.org/web/20130111101034/http%3A/blog.mozilla.org/respindola/2011/05/12/cfi-directives)
>
> [https://refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html](https://link.zhihu.com/?target=https%3A//refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html)
>
> [https://github.com/euspectre/kedr/commit/cc51514f6dc44a42c37a4244ceb4064acdb7d5ec](https://link.zhihu.com/?target=https%3A//github.com/euspectre/kedr/commit/cc51514f6dc44a42c37a4244ceb4064acdb7d5ec)
>
> [https://git.congatec.com/android/qmx6_kernel/commit/d1526e2cda64d5a1de56aef50bad9e5df14245c2](https://link.zhihu.com/?target=https%3A//git.congatec.com/android/qmx6_kernel/commit/d1526e2cda64d5a1de56aef50bad9e5df14245c2)
>
> [https://cateee.net/lkddb/web-lkddb/STACK_UNWIND.html](https://link.zhihu.com/?target=https%3A//cateee.net/lkddb/web-lkddb/STACK_UNWIND.html)
>
> [https://lwn.net/Articles/727553/](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/727553/)
>
> [https://blog.csdn.net/pwl999/article/details/107569603](https://link.zhihu.com/?target=https%3A//blog.csdn.net/pwl999/article/details/107569603)
>
> [https://wdv4758h-notes.readthedocs.io](https://link.zhihu.com/?target=https%3A//wdv4758h-notes.readthedocs.io/zh_TW/latest/libunwind.html)
