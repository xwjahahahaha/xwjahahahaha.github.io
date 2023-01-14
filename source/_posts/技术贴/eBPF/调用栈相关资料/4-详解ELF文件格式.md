---
title: 4-详解ELF文件格式
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-25 10:33:37
---

[TOC]



> 转载自：
>
> * https://zhuanlan.zhihu.com/p/286088470

<!-- more -->

# 一、简介

`ELF`文件是一种用于二进制文件、可执行文件、目标代码、共享库和`core`转存格式文件。是`UNIX`系统实验室（`USL`）作为应用程序二进制接口（`Application Binary Interface，ABI`）而开发和发布的，也是`Linux`的主要可执行文件格式。

# 二、ELF文件主要有四种类型

1. 可重定位文件（`Relocatable File`） 包含适合于**与其他目标文件链接来创建可执行文件**或者**共享目标文件的代码和数**据，即 `xxx.o `文件
2. 可执行文件（`Executable File`） 包含适合于执行的一个程序，此文件规定了` exec() `如何创建一个程序的进程映像，即` a.out`文件
3. 共享目标文件（`Shared Object File`） 包含可在**两种上下文中链接的代码和数据**。首先**链接编辑器可以将它和其它可重定位文件和共享目标文件一起处理，生成另外一个目标文件**。其次，动态链接器（`Dynamic Linker`）可能将它与某个可执行文件以及其它共享目标一起组合，创建进程映像，即` xxx.so`文件
4. 内核转储(`core dumps`)，存放当前进程的执行上下文，用于`dump`信号触发。

在内核中的定义（`/include/uapi/linux/elf.h`）：

```c
#define  ET_REL    1
#define  ET_EXEC   2
#define  ET_DYN    3
#define  ET_CORE   4
```

# 二、ELF文件格式的两种视图

ELF文件格式提供了两种视图，分别是**链接视图**和**执行视图**：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-2c33f6155faccf24839b897ee9db10a1_1440w.jpg)

链接视图是以节（`section`）为单位，执行视图是以段（`segment`）为单位

**链接视图就是在链接时用到的视图，而执行视图则是在执行时用到的视图**

目标文件`.o`里的代码段`.text` 是`section`（汇编中`.text`同理），当多个可重定向文件最终要整合成一个可执行的文件的时候（链接过程），链接器把目标文件中相同的 `section`整合成一个`segment`，在程序运行的时候，方便加载器的加载。

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-368849d36ed910b6434cc19676f6575d_1440w.jpg)

使用`readelf -l` 命令可以查看一个链接后的`elf`可执行文件，`Section to Segment` 的映射关系：

```text
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000007e4 0x00000000000007e4  R E    200000
  LOAD           0x0000000000000e30 0x0000000000600e30 0x0000000000600e30
                 0x0000000000000210 0x0000000000000220  RW     200000
  DYNAMIC        0x0000000000000e58 0x0000000000600e58 0x0000000000600e58
                 0x00000000000001a0 0x00000000000001a0  RW     8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254
                 0x000000000000005c 0x000000000000005c  R      4
  GNU_EH_FRAME   0x00000000000006d0 0x00000000004006d0 0x00000000004006d0
                 0x0000000000000034 0x0000000000000034  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x0000000000000e30 0x0000000000600e30 0x0000000000600e30
                 0x00000000000001d0 0x00000000000001d0  R      1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .rodata .eh_frame_hdr .eh_frame 
   03     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.SuSE .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .ctors .dtors .jcr .dynamic .got
```

下面的段序号和上面程序头里的段一一对应。

# 三、ELF文件结构

一个ELF文件由以下三部分组成：

## 1. ELF头

ELF头(**ELF header**) - 描述文件的主要特性

`/include/uapi/linux/elf.h` （32位）

```c
typedef struct elf32_hdr
{
	  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
	  Elf32_Half	e_type;			/* Object file type */
	  Elf32_Half	e_machine;		/* Architecture */
	  Elf32_Word	e_version;		/* Object file version */
	  Elf32_Addr	e_entry;		/* Entry point virtual address */
	  Elf32_Off	e_phoff;		/* Program header table file offset */
	  Elf32_Off	e_shoff;		/* Section header table file offset */
	  Elf32_Word	e_flags;		               /* Processor-specific flags */
	  Elf32_Half	e_ehsize;		/* ELF header size in bytes */
	  Elf32_Half	e_phentsize;		/* Program header table entry size */
	  Elf32_Half	e_phnum;		/* Program header table entry count */
	  Elf32_Half	e_shentsize;		/* Section header table entry size */
	  Elf32_Half	e_shnum;		/* Section header table entry count */
	  Elf32_Half	e_shstrndx;		/* Section header string table index */
} Elf32_Ehdr;
```

`ELF`头(`ELF header)`位于文件的开始位置，它的主要目的是**定位文件的其他部分**。

比较重要的成员有：`e_ident`（ELF文件幻数）、`e_machine`（比如可执行文件`ET_EXEC`）、`e_entry`（程序入口虚拟地址）等等，下面会在加载过程分析中结合源码分析各成员作用

可通过`readelf -h` 读取`ELF` 头信息，包括文件信息：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-54f35042ba1cd98fa838bfbe84143d4a_1440w.jpg)

## 2. 程序头表

程序头表(`Program header table`) - 列举了所有有效的段(`segments`)和他们的属性（执行视图）

程序头是一个结构的数组，每一个结构都表示一个段(`segments`)。在可执行文件或者共享链接库中所有的节(`sections`)都被分为不同的几个段(`segments`)

```c
typedef struct elf32_phdr{
	  Elf32_Word	p_type;    /* Magic number and other info */
	  Elf32_Off	p_offset;
	  Elf32_Addr	p_vaddr;
	  Elf32_Addr	p_paddr;
	  Elf32_Word	p_filesz;
	  Elf32_Word	p_memsz;
	  Elf32_Word	p_flags;
	  Elf32_Word	p_align;
} Elf32_Phdr;
```

程序头的索引地址(`e_phoff`)、段数量(`e_phnum`)、表项大小(`e_phentsize`)都是通过` ELF`头部信息获取的。

可通过`readelf -l`读取ELF程序头信息：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-195d184a61a66c1fecea6f37a39ea8ca_1440w.jpg)

## 3. 节头表

节头表(**Section header table**) - 包含对节(sections)的描述（链接视图）

一个`ELF`文件中到底有哪些具体的` sections`，由包含在这个`ELF`文件中的`section head table(SHT)`决定。每个`section`描述了这个段的信息，比如每个段的段名、段的长度、在文件中的偏移、读写权限及段的其它属性。

```c
typedef struct elf32_shdr{
	    Elf32_Word sh_name;   //节区名，名字是一个 NULL 结尾的字符串。
	    Elf32_Word sh_type;    //为节区类型
	    Elf32_Word sh_flags;    //节区标志
	    Elf32_Addr sh_addr;    //节区的第一个字节应处的位置。否则，此字段为 0。
	    Elf32_Off sh_offset;    //此成员的取值给出节区的第一个字节与文件头之间的偏移。
	    Elf32_Word sh_size;   //此 成 员 给 出 节 区 的 长 度 （ 字 节 数 ）。
	    Elf32_Word sh_link;   //此成员给出节区头部表索引链接。其具体的解释依赖于节区类型。
	    Elf32_Word sh_info;       //此成员给出附加信息，其解释依赖于节区类型。
	    Elf32_Word sh_addralign;    //某些节区带有地址对齐约束.
	    Elf32_Word sh_entsize;    //给出每个表项的长度字节数。
}Elf32_Shdr;
```

节区名存储在.[shstrtab](onenote:#elf加载过程&section-id={503CCFB6-C32E-4B72-8C67-CB9E878870BE}&page-id={934E7334-CB4A-4A41-814C-8742312D7636}&object-id={C81FF8EC-AF62-0DAD-105B-29D5DD1E0EA2}&7D&base-path=https://d.docs.live.net/aba64c6627629f31/文档/我的笔记本/源码整理/Linux内核学习/进程管理.one)字符串表中，`sh_name`是表中偏移。

可通过**readelf -S** 读取sections' header：

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-19cf5f3f67ade442cc800281221647eb_1440w.jpg)

## 4. 系统预订的固定section

有些节区是系统预订的，一般以点开头号，下面介绍下常见和比较重要的`section`：

| sh_name   | sh_type      | description                                                  |
| --------- | ------------ | ------------------------------------------------------------ |
| .text     | SHT_PROGBITS | 代码段，包含程序的可执行指令                                 |
| .data     | SHT_PROGBITS | 包含初始化了的数据，将出现在程序的内存映像中                 |
| .bss      | SHT_NOBITS   | 未初始化数据，因为只有符号所以                               |
| .rodata   | SHT_PROGBITS | 包含只读数据                                                 |
| .comment  | SHT_PROGBITS | 包含版本控制信息                                             |
| .eh_frame | SHT_PROGBITS | 它生成描述如何unwind 堆栈的表                                |
| .debug    | SHT_PROGBITS | 此节区包含用于符号调试的信息                                 |
| .dynsym   | SHT_DYNSYM   | 此节区包含了动态链接符号表                                   |
| .shstrtab | SHT_STRTAB   | 存放section名，字符串表。Section Header String Table         |
| .strtab   | SHT_STRTAB   | 字符串表                                                     |
| .symtab   | SHT_SYMTAB   | 符号表                                                       |
| .got      | SHT_PROGBITS | 全局偏移表                                                   |
| .plt      | SHT_PROGBITS | 过程链接表                                                   |
| .relname  | SHT_REL      | 包含了重定位信息，例如 .text 节区的重定位节区名字将是：.rel.text |
| …         |              |                                                              |

### 1、`.text` 代码段

可以通过objdump -d 反汇编，查看ELF文件代码段内容。

### 2、`.strtab / .shstrtab` 字符串表

在ELF文件中，会用到很多字符串，比如节名，变量名等。所以ELF将所有的字符串集中放到一个表里，每一个字符串以`\0`分隔，然后使用字符串在表中的偏移来引用字符串。这样在ELF中引用字符串只需要给出一个数组下标即可。字符串表在ELF也以段的形式保存，` .shstrtab`是专供`section name`的字符串表。

可以用一下命令查看：`readelf -S xxx.o`

```text
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000045  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000658
       0000000000000048  0000000000000018          12     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000085
       0000000000000000  0000000000000000  WA       0     0     1
                         … …
  [11] .shstrtab         STRTAB           0000000000000000  00000108
       0000000000000074  0000000000000000           0     0     1
  [12] .symtab           SYMTAB           0000000000000000  00000500
       0000000000000138  0000000000000018          13    10     8
  [13] .strtab           STRTAB           0000000000000000  00000638
       000000000000001e  0000000000000000           0     0     1
```

### 3、`.symtab `符号表

在链接的过程中需要把多个不同的目标文件合并在一起，不同的目标文件相互之间会引用变量和函数。**在链接过程中，我们将函数和变量统称为符号，函数名和变量名就是符号名**。

每个定义的符号都有一个相应的值，叫做符号值(`Symbol Value`)，对于变量和函数，**符号值就是它们的地址**

可以使用下面命令查看：readelf -s xxx.o

![img](http://xwjpics.gumptlu.work/qinniu_uPic/v2-c9e476401c19307c82b3fe89e3555224_1440w.jpg)

### 4、`.eh_frame / .eh_frame_hdr`

在调试程序的时候经常需要进行**堆栈回溯**，早期使用通用寄存器(`ebp`)来保存每层函数调用的栈帧地址，但局限性很大。后来现代Linux操作系统在`LSB(Linux Standard Base)`标准中定义了一个`.eh_frame section`，用来描述如何去`unwind the stack`。gcc编译器默认打开，如果不想把`.eh_frame section`编入`elf`文件，可以通过`gcc`选项` -fno-asynchronous-unwind-tables `去除。

`GAS(GCC Assembler)`汇编编译器定义了一组伪指令来协助`eh_frame`生成调用栈信息`CFI(Call Frame Information)`。具体原理在后续《栈回溯》章节分析，这里不再阐述。

### 5、重定位表（`.relname`）

链接器在处理目标文件时，需要对目标文件中的某些部位进行重定位，即代码段和数据中中那些绝对地址引用的位置。对于每个需要重定位的代码段或数据段，都会有一个相应的重定位表。比如”.rel.text”就是针对”.text”的重定位表，”.rel.data”就是针对”.data”的重定位表。

`GOT`是全局偏移表（ `Global Offset Table`），用于存储外部符号地址；PLT是程序链接表（Procedure Link Table），用于存储记录定位信息的额外代码。关于linux动态链接重定位原理，可以参考[海枫](https://link.zhihu.com/?target=https%3A//linyt.blog.csdn.net/)写的《[Linux动态链接中的PLT和GOT](https://link.zhihu.com/?target=https%3A//blog.csdn.net/linyt/article/details/51635768)》。

关于elf32_hdr 等数据结构中成员具体细节可参考mergerly转载的《[ELF文件格式解析](https://link.zhihu.com/?target=https%3A//blog.csdn.net/mergerly/article/details/94585901)》
