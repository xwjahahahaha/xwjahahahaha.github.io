---
title: 动手实现一个os-1-HellOS(操作系统层面的helloworld)
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-17 13:34:14
---

[TOC]

<!-- more -->

> 学习自：
>
> * 极客时间-《操作系统45讲》
>   * 购买地址： https://time.geekbang.org/column/intro/100078401 
>   * 作者：LMOS

# 一、HelloOS

使用几行汇编代码和C实现一个极简操作系统（或者说操作系统层面的`HelloWorld`），先看效果吧：

![image-20220718181633081](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718181633081.png)

![image-20220718183912052](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718183912052.png)

不打算从PC的引导程序开始写起，原因是目前我们的知识储备还不够，所以先借用一下 GRUB 引导程序，只要我们的 PC 机上安装了 Ubuntu Linux 操作系统， GRUB 就已经存在了

## 1.1 基本引导流程

HelloOS的引导流程图如下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220717141530939.png" alt="image-20220717141530939" style="zoom:33%;" />

1. 首先BIOS固件是固化在PC主板的ROM芯片中的，所以即使没电也可以保存
2. 当PC上电之后，执行的第一个指令就是BIOS固件的指令，其负责检测和初始化CPU、内存和主板平台
3. 然后再加载引导设备（大概率是硬盘）中的第一个扇区数据，到`0x7c00`地址的内存空间中
4. 然后在此内存地址执行引导指令（在这里就是GRUB引导程序）

## 1.2 编写汇编

接下来需要写一段汇编代码

> 为什么不直接使用C编写？
>
> * C 作为通用的高级语言，不能直接操作特定的硬件，而且 C 语言的函数调用、函数传参， 都需要用栈，而栈是需要内存分配的，所以我们先需要用汇编去实现这些C语言执行依赖的环境

汇编代码如下：

```asm
MBT_HDR_FLAGS	EQU 0x00010003
MBT_HDR_MAGIC	EQU 0x1BADB002 ;多引导协议头魔数
MBT_HDR2_MAGIC	EQU 0xe85250d6 ;第二版多引导协议头魔数
global _start ;导出_start符号root
extern main ;导入外部的main函数符号

[section .start.text] ;定义.start.text代码节
[bits 32] ;汇编成32位代码
_start:
	jmp _entry
ALIGN 8
mbt_hdr:
	dd MBT_HDR_MAGIC
	dd MBT_HDR_FLAGS
	dd -(MBT_HDR_MAGIC+MBT_HDR_FLAGS)
	dd mbt_hdr
	dd _start
	dd 0
	dd 0
	dd _entry
;以上是GRUB所需要的头

ALIGN 8
mbt2_hdr:
	DD	MBT_HDR2_MAGIC
	DD	0
	DD	mbt2_hdr_end - mbt2_hdr
	DD	-(MBT_HDR2_MAGIC + 0 + (mbt2_hdr_end - mbt2_hdr))
	DW	2, 0
	DD	24
	DD	mbt2_hdr
	DD	_start
	DD	0
	DD	0
	DW	3, 0
	DD	12
	DD	_entry
	DD      0
	DW	0, 0
	DD	8
mbt2_hdr_end:
;以上是GRUB2所需要的头
;包含两个头是为了同时兼容GRUB、GRUB2

ALIGN 8

_entry:
	;关中断
	cli
	;关不可屏蔽中断
	in al, 0x70
	or al, 0x80
	out 0x70,al
	;重新加载GDT（全局描述符表）
	lgdt [GDT_PTR]
	jmp dword 0x8 :_32bits_mode ;跳转到下面执行

_32bits_mode:
	;下面初始化C语言可能会用到的寄存器
	mov ax, 0x10
	mov ds, ax
	mov ss, ax
	mov es, ax
	mov fs, ax
	mov gs, ax
	xor eax,eax
	xor ebx,ebx
	xor ecx,ecx
	xor edx,edx
	xor edi,edi
	xor esi,esi
	xor ebp,ebp
	xor esp,esp
	;初始化栈，C语言需要栈才能工作
	mov esp,0x9000 ;esp指向栈顶
	;调用C语言函数main
	call main
	;让CPU停止执行指令
halt_step:
	halt
	jmp halt_step ;循环调用


GDT_START:
knull_dsc: dq 0
kcode_dsc: dq 0x00cf9e000000ffff
kdata_dsc: dq 0x00cf92000000ffff
k16cd_dsc: dq 0x00009e000000ffff
k16da_dsc: dq 0x000092000000ffff
GDT_END:

GDT_PTR:
GDTLEN	dw GDT_END-GDT_START-1
GDTBASE	dd GDT_START
```

以上代码分为四个部分：

1. 代码 1~43 行，用汇编定义的 **GRUB 的多引导协议头**，其实就是一定格式的数据，我们的Hello OS是用GRUB引导的，当然要遵循GRUB的多引导协议标准，让GRUB能识别我们的Hello OS。之所以有两个引导头，是为了兼容 GRUB1 和 GRUB2
2. 代码 44~55行，关掉中断，设定CPU的工作模式（作者说现在不懂没关系，后面CPU相关的课程我们会专门再研究它）
3. 代码 58~75 行，初始化CPU的寄存器和C语言的运行环境
4. 代码 76~最后，GDT_START开始的，是CPU工作模式所需要的数据（同样，后面讲 CPU 时会专门介绍）

## 1.3 Hello OS主函数

上面的汇编代码调用了 `main `函数，而在其代码中并没有看到其函数体，而是从外部引入了一个符号

那是因为**这个函数是用C语言写的在`main.c`中**，最终它们分别由`nasm`和`GCC`编译成可链接模块，由`LD`链接器链接在一起，形成可执行的程序文件:

```c
#include "vgastr.h"
void main() {
    printf("Hello, World!");
    return 0;
}
```

只不过这里面的`printf`函数并不能使用`stdio.h`标准库里面的实现了，我们需要自己实现(`vgastr.h`)

## 1.4 控制计算机屏幕

显卡控制计算机屏幕的显示，显卡有很多形式:集成在主板的叫集显，做在CPU芯片内的叫核显，独立存在通过PCIE接口连接的叫独显，性能依次上升，价格上也是，要在屏幕上显示字符，那么就需要**编程操作显卡**

其实无论我们 PC 上是什么显卡，它们都支持一种叫 `VESA `的标准，这种标准下有两种工作模式:**字符模式和图形模式**。显卡们为了兼容这种标准，不得不自己提供一种叫 `VGABIOS` 的固件程序

**显卡的字符模式的工作细节：**

它把屏幕分成 24 行，每行 80 个字符，把这(24*80)个位置映射到以` 0xb8000` 地址开 始的内存中，每两个字节对应一个字符，其中一个字节是字符的 ASCII 码，另一个字节为 字符的颜色值：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220717150921114.png" alt="image-20220717150921114" style="zoom:67%;" />

下面就实现这样的逻辑`vgastr.c`：

```c
int _strwrite(char* string) {
    char* p_strdst = (char*)(0xb8000); // 指向现存的开始地址
    while (*string) {
        *p_strdst = *string++;
        p_strdst += 2;      // +2是不管控制颜色的第二个字节
    }

    return 0;
}

int printf(char* fmt, ...) {
    _strwrite(fmt);
    return 0;
}
```

`vgastr.h`:

```c
int _strwrite(char* string);
int printf(char* fmt, ...);
```

printf 函数直接调用了` _strwrite `函数，而` _strwrite `函数正是将字符串里每 个字符依次定入到 `0xb8000` 地址开始的显存中，而` p_strdst `每次加 2，这也是为了跳过字符的颜色信息的空间

可以尝试着先在自己的操作系统中运行一下：

`gcc -o main main.c && ./main`

![image-20220717152302658](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220717152302658.png)

完美！那么代码就写完了，但是剩下来的就是**编译和安装**

## 1.5 编译和安装Hello OS

### 编译

编译的过程如下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220717161143897.png" alt="image-20220717161143897" style="zoom: 50%;" />

主要通过如下makefile实现：

```makefile
MAKEFLAG = -sR
MKDIR = mkdir
RMDIR = rmdir
CP = cp
CD = cd
DD = dd
RM = rm

ASM = nasm
CC = gcc
LD = ld
OBJCOPY = objcopy

ASMBFLAGS = -f elf -w-orphan-labels
CFLAGS = -c -Os -std=c99 -m32 -Wall -Wshadow -W -Wconversion -Wno-sign-conversion -fno-stack-protector -fomit-frame-pointer -fno-builtin -fno-common  -ffreestanding  -Wno-unused-parameter -Wunused-variable
LDFLAGS = -s -static -T hello.lds -n -Map HelloOS.map
OJCYFLAGS = -S -O binary

HELLOOS_OBJS :=
HELLOOS_OBJS += entry.o main.o vgastr.o
HELLOOS_ELF = HelloOS.elf
HELLOOS_BIN = HelloOS.bin

.PHONY: build clean link bin

all: clean build link bin

clean:
	$(RM) -f *.o *.bin *.elf

build: $(HELLOOS_OBJS)   	# 指定依赖的所有.o文件

link: $(HELLOOS_ELF) 		# 指定依赖的elf文件
$(HELLOOS_ELF): $(HELLOOS_OBJS)	# elf依赖.o文件
	$(LD) $(LDFLAGS) -o $@ $(HELLOOS_OBJS)  # 链接所有.o文件

bin: $(HELLOOS_BIN) 	 	# 最后的可执行文件见依赖elf文件
$(HELLOOS_BIN): $(HELLOOS_ELF)
	$(OBJCOPY) $(OJCYFLAGS) $< $@

%.o : %.asm
	$(ASM) $(ASMBFLAGS) -o $@ $<
%.o : %.c
	$(CC) $(CFLAGS) -o $@ $<
```

编译：`make all`

> 注意：这里可能会提示没有nasm编译器，所以我们要先安装：
>
> * `sudo apt install nasm`

### 安装启动

当编译好`helloOS.bin`执行文件之后，接下来就要让`GRUB`能够找到它，然后才会在计算机启动的时候找到它，这个过程我们称为安装，不过这里没有写安装程序，得我们手动来做。

GRUB 在启动时会加载一个 `grub.cfg `的文本文件，根据其中的内容执行相应的操作，其中一部分内容就是启动项。

GRUB 首先会显示启动项到屏幕，然后让我们选择启动项，最后 GRUB 根据启动项对应的信息，加载 OS 文件到内存

#### 设置到GRUB的引导菜单

我们将HelloOS作为一个操作系统启动项供grub启动，因此需要能够在PC启动时进入grub引导菜单，并选择启动HelloOS

为了能够每次启动时进入grub引导菜单，需要进行如下设置

**修改/etc/default/grub**：

![image-20220718180517805](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718180517805.png)

* 注释掉`HIDDEN`所在的2行（我这里没有）
* 将`GRUB_TIMEOUT`设置为30（使用默认值10其实也可以）
* 将`GRUB_CMDLINE_LINUX_DEFAUL`设置为`text`

**执行如下命令，更新grub配置**:

`sudo update-grub `

#### 物理机安装启动

首先查询一下自己`boot`目录挂载的分区

`df /boot`

![image-20220718180207996](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718180207996.png)

那么Hello OS 的启动项可以配置为如下:

```shell
menuentry 'HelloOS' {
    insmod part_msdos 		#GRUB加载分区模块识别分区
    insmod ext2       		#GRUB加载ext文件系统模块识别ext文件系统
    set root='hd0,msdos3' #注意boot目录挂载的分区，这是我机器上的情况
    multiboot2 /boot/HelloOS.bin
    boot
}
```

其中的“sda3”就是硬盘的第三个分区，但是 GRUB 的 menuentry 中不能写 sda3，而是要写“hd0,msdos3”，这是 GRUB 的命名方式，hd0 表示第一块硬盘，结合起来就是 第一块硬盘的第四个分区

> <font color='#e54d42'>注意：这里的root路径按照上面的规则并不一定正确，具体的要看`/boot/grub/grub.cfg`文件中底部其他menuentry的root路径，具体可见下面的错误记录</font>

把上面启动项的代码插入到你的 Linux 机器上的` /boot/grub/grub.cfg `文件中，然后把 `HelloOS.bin `文件复制到` /boot/ `目录下，最后重启计算机，你就可以看到 Hello OS 的启动选项了

#### qemu虚拟机安装启动

> 参考：https://gitee.com/gaopedro/pos

在上面的链接中下载`floppy.img`软盘

修改`Makefile`文件，在最下方添加如下伪指令：

```makefile
....

update_image:
	sudo mount floppy.img /mnt/kernel
	sudo cp HelloOS.bin /mnt/kernel/hx_kernel
	sleep 1
	sudo umount /mnt/kernel

umount_image:
	sudo umount /mnt/kernel

qemu:
	qemu-system-i386 -fda floppy.img -boot a
	#add '-nographic' option if using server of linux distro, such as fedora-server,or "gtk initialization failed" error will occur.
```

> 注意，我这里添加了`-nographic`标志。如果你的机器上也报了`gtk initialization failed`的错误，可以在qemu的命令上添加此选项

如果没有`qemu-system-i386`指令，则可以安装：`apt-get update && apt-get install qemu-system-i386 `

安装启动：

```shell
make update_image
make qemu
```

#### docker启动

可参考：

https://github.com/vizv/learn_os/tree/master/hello-os

`make docker`

![image-20220718163710023](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718163710023.png)

# 错误记录

## 1. 找不到msdos3

选择HelloOS之后，提示：

![image-20220718182107624](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718182107624.png)

查看`/boot/grub/grub.cfg`文件中的其他`menuentry`发现应该使用`gpt3`，也就是`hd0,gpt3`

![image-20220718183705880](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220718183705880.png)
