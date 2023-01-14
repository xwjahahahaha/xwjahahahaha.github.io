---
title: 12-runtime-2-内存分配原理
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-11-09 13:09:42
---

> 学习自：
>
> * 极客时间《go进阶训练营》

<!-- more -->

# 一、堆栈 & 逃逸分析

## 1. 堆和栈的定义

Go 有两个地方可以分配内存：一个全局堆空间用来动态分配内存，另一个是每个 goroutine 都有的自身栈空间。

* 栈
      栈区的内存一般由编译器自动进行分配和释放，<u>其中存储着函数的入参以及局部变量，这些参数会随着函数的创建而创建，函数的返回而销毁。</u>(通过 CPU push & release)。一个函数可以通过帧指针直接访问帧内的内存，但是访问帧外的内存需要间接访问。

  ​	**栈的内存分配是从高位往低位拓展，堆则是反过来。**

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/nmjzK3.png" alt="nmjzK3" style="zoom: 33%;" />

  ​	**栈帧**：用来给函数运行**提供内存空间**。取内存于stack上。当函数调用时，产生栈帧，当函数调用结束时，释放栈帧。

  ​	**栈帧存储：**1.局部变量 2.形参（与局部变量地位等同） 3. 内存字段描述值

  ​	一个函数就对应一个栈

  ​	以32位的内存4G的空间为例：

  ​	<img src="http://xwjpics.gumptlu.work/qinniu_uPic/OnTiL2.png" alt="OnTiL2" style="zoom: 33%;" />

* 堆
      堆区的内存一般由编译器和工程师自己共同进行管理分配，**交给 Runtime GC 来释放**。<u>堆上分配必须找到一块足够大的内存来存放新的变量数据。后续释放时，垃圾回收器扫描堆空间寻找不再被使用的对象。</u>   

  > **任何时候一个值在函数的堆栈框架的范围之外被共享，它将被放置(或分配)在堆上。**
  > <font color='#39b54a'>在go语言中，编译器分析出局部变量产生了逃逸的行为，那么会将该变量设置在堆上（逃逸分析）。这是go的编译器自动实现的，而在C语言中，这需要程序员手动操作，否则会发生错误。</font>

  **栈分配廉价，堆分配昂贵**。stack allocation is cheap and heap allocation is expensive.

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/CUd4Vj.png" alt="CUd4Vj" style="zoom:50%;" />

## 2. 逃逸分析

### 1. 基本概念

​	写过其他语言，比如 C 的同学都知道，有明确的栈和堆的相关概念。而 **Go 声明语法并没有提到栈和堆，而是交给 Go 编译器决定在哪分配内存，保证程序的正确性**，在 <u>Go FAQ</u> 里面提到这么一段解释：

​    从正确的角度来看，你不需要知道。**Go 中的每个变量只要有引用就会一直存在。变量的存储位置(堆还是栈)和语言的语义无关。**<font color='#e54d42'>go语言中，**堆栈空间的分配**不依靠语法上的关键字**即由系统分配（编译器分析代码）而不是代码层面上的指定**</font>

​    存储位置对于写出高性能的程序确实有影响。如果可能，Go 编译器将为该函数的堆栈侦(stack frame)中的函数分配本地变量。但是<u>如果编译器在函数返回后无法证明变量未被引用，则编译器必须在会被垃圾回收的堆上分配变量以避免悬空指针错误</u>。此外，<u>如果局部变量非常大，将它存储在堆而不是栈上可能更有意义。</u>

​    <u>在当前编译器中，如果变量存在取址，则该变量是堆上分配的候选变量。</u>（取地址不一定就会分配到堆上，只是候选）但是基础的逃逸分析可以将那些生存不超过函数返回值的变量识别出来，并且因此可以分配在栈上。

堆栈的分配规则如下：

* 函数退出后不再使用则将该变量**优先**分配在栈中

  ```go
  func F() {
  	temp := make([]int, 0, 20)
  	...
  }
  ```

* 函数退出之后**返回了该变量的引用或者把指针赋值给全局变量等继续引用的方式**，那么**一定**就会把该变量分配在堆中（因为后续还要使用，所以不会放在栈上）

  ```go
  func F() []int{
  	a := make([]int, 0, 20)
  	return a		// 继续引用
  }
  
  // 或者这样的逃逸
  var global *int	
  func F2() {
    var x int
    x = 1
    global = &x
  }
  ```

**逃逸**：当一个对象的指针被多个方法、线程引用时，就可以称这个指针发生了逃逸

> <font color='#e54d42'>“通过检查**变量的作用域是否超出了它所在的栈**来决定是否将它分配在堆上”的技术，其中“变量的作用域超出了它所在的栈”这种行为即被称为**逃逸**</font>

**逃逸分析**：分析一个变量是分配在栈上还是堆上。逃逸分析在大多数语言里属于静态分析：在编译期由静态代码分析来决定一个值是否能被分配在栈帧上，还是需要“逃逸”到堆上。

**为什么要逃逸分析**：让合适的变量存在合适的地方，节省GC内存回收的压力

**编译器逃逸分析原理：** 就是上面的分配原则，如果函数中的变量被多个引用就分配到堆上，Go中的变量只有在编译器可以证明在函数返回后不会再被引用的，才分配到栈上，其他情况下都是分配到堆上。

### 2. 查看是否逃逸的方法

```go
package main

import "fmt"

func foo() *int {
    t := 3
    return &t  //返回对局部变量的引用
}

func main() {
    x := foo()
    fmt.Println(*x)
}
```

运行查看：

```shell
//-m: 查看逃逸过程；-l: 禁止函数内联
$ go build -gcflags '-m -l' main.go
```

输出结果：

```shell
# command-line-arguments
./main.go:11:2: moved to heap: t				// t 逃逸到了堆上
./main.go:17:13: ... argument does not escape		 
./main.go:17:14: *x escapes to heap			// x 逃逸到了堆上
```

逃逸我们可以理解，x逃逸是因为fmt输出语句的参数为空接口类型，编译期间很难确定其参数的具体类型，也会发生逃逸。

### 3. 逃逸案例

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/UsG3l0.png" alt="UsG3l0" style="zoom:50%;" />

* 需要手动切换堆栈的问题: **超过栈帧(stack frame)**

  ![vqctDy](http://xwjpics.gumptlu.work/qinniu_uPic/vqctDy.png)

  当一个函数被调用时，会在两个相关的帧边界间进行上下文切换。从调用函数切换到被调用函数，如果函数调用时需要传递参数，那么这些参数值也要传递到被调用函数的帧边界中。**Go 语言中帧边界间的数据传递是按*<u>值</u>*传递的。**任何在函数 getRandom 中的变量在函数返回时，都将不能访问。Go 查找所有变量超过当前函数栈侦的，把它们分配到堆上，避免 outlive 变量。

* go编译器的逃逸分析的作用

  上述情况中，num 变量不能指向之前的栈。**Go 查找所有变量超过当前函数栈侦的，把它们分配到堆上，避免 outlive 变量。变量 tmp 在栈上分配，<font color='#e54d42'>但是它包含了指向堆内存的地址</font>，**所以可以安全的从一个函数的栈侦复制到另外一个函数的栈帧。

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/OqdYP8.png" alt="OqdYP8" style="zoom: 33%;" />

还存在大量其他的 case 会出现逃逸，比较典型的就是 “**多级间接赋值容易导致逃逸**”，<u>**这里的多级间接指的是，对某个引用类对象中的引用类成员进行赋值**</u>。

> **<font color='#e54d42'>记住公式：`Data.Field=Value`如果`Data.Field`都是引用类的数据类型，则会导致`Value`的逃逸</font>**

Go 语言中的引用类数据类型有 func, interface, slice, map, chan, *Type ：

* 一个值被分享到函数栈帧范围之外
* 在 for 循环外申明，在 for 循环内分配，同理闭包
* 发送指针或者带有指针的值到 channel 中
* 在一个切片上存储指针或带指针的值
* slice 的背后数组被重新分配了
* *在 interface 类型上调用方法*
* ….

### 4. 总结

1. Go编译器会在编译期对考察变量的作用域，并作一系列检查，如果**它的作用域**在运行期间对编译器一直是可知的，那么就会分配到栈上；
2. 堆上动态分配内存比栈上静态分配内存，开销大很多；尽可能的将不使用的变量减少引用，放在栈上减少GC的压力，栈会随函数执行完毕后直接回收。
3. 变量分配在栈上需要能在编译期确定它的作用域，否则会分配到堆上；
4. <font color='#e54d42'>**函数传递变量的指针可以避免复制值的开销，但是这种情况不是一定的，因为复制值的过程在栈上进行这个的开销远比变量逃逸后在堆上动态分配内存少的多**</font>

# 二、连续栈

## 1. 分段栈(Segmented stacks)

Go 应用程序运行时，**每个 goroutine 都维护着一个自己的栈区，这个栈区只能自己使用不能被其他 goroutine 使用。**栈区的初始大小是2KB(比 x86_64 架构下线程的默认栈2M要小很多)，**在 goroutine 运行的时候栈区会按照需要增长和收缩**，占用的内存最大限制的默认值在64位系统上是1GB。

* v1.0 ~ v1.1 — 最小栈内存空间为 4KB
* v1.2 — 将最小栈内存提升到了 8KB
* v1.3 — 使用连续栈替换之前版本的分段栈
* v1.4 — 将最小栈内存降低到了 2KB

分段栈的实例图如下：

如果栈的空间不够，那么就会分配一个更大的空间，然后指向它。这样的好处是不需要内存的拷贝就可以实现扩容。

![](http://xwjpics.gumptlu.work/qinniu_uPic/anPrHn-20211111142321213.png)

但是也面临着`Hot split`热分裂的问题：

当前的分段栈的实现方式存在 “hot split” 问题，**如果栈快满了，那么下一次的函数调用会强制触发栈扩容。当函数返回时，新分配的 “stack chunk” 会被清理掉。如果这个函数调用产生的范围是在一个循环中，会导致严重的性能问题，频繁的` alloc/free`。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/MOYgIq.png" alt="MOYgIq" style="zoom:50%;" />

Go 不得不在1.2版本把栈默认大小改为8KB，降低触发热分裂的问题，但是每个 goroutine 内存开销就比较大了。直到实现了连续栈(`contiguous stack`)，栈大小才改为2KB。

## 2. 连续栈(contiguous stack)

采用**复制栈**的实现方式，在热分裂场景中不会频发释放内存，**即不像分配一个新的内存块并链接到老的栈内存块，而是<u>会分配一个两倍大的内存块并把老的内存块内容*复制*到新的内存块里</u>**，当栈缩减回之前大小时，我们不需要做任何事情。

* `runtime.newstack` 分配更大的栈内存空间
* `runtime.copystack `将旧栈中的内容复制到新栈中
* *<u>将指向旧栈对应变量的指针重新指向新栈</u>*
* `runtime.stackfree` 销毁并回收旧栈的内存空间

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211111095415722.png" alt="image-20211111095415722" style="zoom:50%;" />

如果之前扩容到了很大的空间，但是目前程序的使用率却不高，那么GC也会**栈缩容**：

如果栈区的空间使用率不超过1/4，那么在垃圾回收的时候使用 `runtime.shrinkstack` 进行**栈缩容**，同样使用 copystack拷贝到一个小栈空间继续使用

## 3. 栈扩容

Go 运行时判断栈空间是否足够，所以在` call function `调用新的函数的时候会插入` runtime.morestack`，但每个函数调用都判定的话，成本比较高。

所以go的做法是：在编译期间通过计算 `sp`、`func stack framesize` 确定需要哪个函数调用中插入 `runtime.morestack`。

具体的规则如下：

* 当函数是叶子节点（就是此函数之后不再调用其他函数了），且栈帧小于等于 112 ，不插入指令
* 当叶子函数栈帧大小为 120 -128 或者 非叶子函数栈帧大小为 0 -128，SP < `stackguard0`
* 当函数栈帧大小为 128 - 4096
  *  SP - framesize < `stackguard0` - `StackSmall`
* 大于 StackBig
  * SP-`stackguard`+`StackGuard` <= framesize + (`StackGuard`-`StackSmall`)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Twd538.png" alt="Twd538" style="zoom: 33%;" />

可以认为`stackguard0`就是一个保底线，如果大于这个保底线就会优先通过判断大小、减法等判断是否需要扩容，能够不插入`morestack`就不插入，为了性能考量。

# 三、内存结构

## 1. 编码时的内存优化技巧

* 小对象结构体的合并

```go
type A struct {...}
type B struct {
  a	A								// 组合模式
  ...
}
newB := new(B)			// 这样只会创建一个B的对象，但是A对象也得到了初始化
```

​	这样可以节省对象的数量，节省GC扫描的时间

* bytes.Buffer

  一开始使用就创建较大的合适空间，防止扩展

* slice、map预创建

  一开始make就确定大小而不是创建一个空的，不断append

* 长调用栈

  长调用指的是goroutine在短时间内不会被释放掉（例如网络传输等），这时内存栈的调用就会比较多。大量的defer也会影响代码的效率

* 避免频繁创建临时对象

  在一个高频率的环境下频繁的创建对象会给GC带来压力。例如Http编程中不断的request和require的过程每次都new一个新对象的话，性能会不是很好。解决方式是：1.保存对象变量复用 2.sigpool

* 字符串拼接`strings.Builder`

  比`bytes.buffer`的方式更高效

* 不必要的memory copy

  例如文件复制的时候创建一个中间buffer，在writer与reader之间传递。其实`io.copy`实现的逻辑更加高效，其没新创建一个buffer而是直接使用reader的buffer，避免拷贝

* 分析内存逃逸

## 2. 内存管理

`TCMalloc `是` Thread Cache Malloc` 的简称，是Go 内存管理的起源，Go的内存管理是借鉴了`TCMalloc`。内存的管理一直面临着两个问题：

* 内存碎片
      随着内存不断的申请和释放，内存上会存在大量的碎片，降低内存的使用率。为了解决内存碎片，可以将2个连续的未使用的内存块合并，减少碎片。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/ZaiH3E.png" alt="ZaiH3E" style="zoom:50%;" />

* 大锁
      **同一进程下的所有线程共享相同的内存空间，它们申请内存时需要加锁**，如果不加锁就存在同一块内存被2个线程同时访问的问题。

### 1. 内存概念

我们需要先知道几个重要的概念：

* **page: 内存页**，一块 8K 大小的内存空间。Go 与操作系统之间的内存申请和释放，都是以 page 为单位的。
* **span: 内存块**，一个或多个连续的 page 组成一个 span。
* **sizeclass: 空间规格**，每个 span 都带有一个 sizeclass，标记着该 span 中的 page 应该如何使用。（例如图中分割的8、16、32等规格）
* **object: 对象**，用来**存储一个变量数据内存空间**，一个 span 在初始化时，会被切割成一堆等大的 object。假设 object 的大小是 16B，span 大小是 8K，那么就会把 span 中的 page 就会被初始化 8K / 16B = 512 个 object。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/X25Kj7.png" alt="X25Kj7" style="zoom: 33%;" />

### 2. 再谈mcache

当程序里发生了 32kb 以下的小块内存申请时（new一个对象其小于32k），Go 会从一个叫**` mcache `**的本地缓存给程序分配内存。**这样的一个内存块里叫做 `mspan`，它是要给程序分配内存时的分配单元。**

> <font color='#39b54a'>在GMP模型调度时说道过这个mcache，原本的模型调度方式是mcache挂载在M上，当M调用系统调用的时候不会使用goutine的内存，所以这是一个额外的负担。后来的GMP将mcache转移到了P上，解决了这个问题</font>

在 Go 的调度器模型里，每个线程  M 会绑定给一个处理器 P，在单一粒度的时间里只能做多处理运行一个 goroutine，**每个 P 都会绑定一个上面说的本地缓存 mcache**。**当需要进行内存分配时，当前运行的 goroutine 会从 mcache 中查找可用的 mspan。从本地 mcache 里分配内存时不需要加锁，这种分配策略效率更高。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/sL3a1R-20211112154927027.png" style="zoom:50%;" />

<font color='#e54d42'>mcache的作用：为线程内存分配提供一个本地缓存，快速分配避免访问全局内存大锁</font>

申请内存时都分给他们一个 mspan 这样的单元会不会产生浪费。其实**` mcache `持有的这一系列的`mspan `并不都是统一大小的，而是按照大小，从8kb 到 32kb 分了大概 67*2 类的 mspan。**

> <font color='#39b54a'>*2的原因在于每个对象还有scan、noscan两个种类，在后面的GC机制中会详细阐述</font>

**每个内存页分为多级固定大小的“空闲列表”，这有助于减少碎片。**类似的思路在` Linux Kernel`、`Memcache `都可以见到` Slab-Allactor`。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/RMb7CJ.png" alt="RMb7CJ" style="zoom: 33%;" />

> <font color='#39b54a'>也就是说，go解决内存碎片/内存分配的方法是通过在mcache中挂载的一系列mspan上用不同/多级sizeclass规格的设置不同的大小规格，这样细粒度越小，分配内存时的适配性就更强，产生的碎片就越小</font>

### 3. mcentral

如果分配内存时` mcachce `里没有空闲的对口 `sizeclass` 的 mspan 了，Go 里还为每种类别的 mspan 维护着一个 `mcentral`。

**`mcentral `的作用是为所有 mcache 提供切分好的 mspan 资源。**每个` central `会持有一种特定大小的全局 mspan 列表，包括已分配出去的和未分配出去的。 每个 `mcentral `对应一种 mspan，**当工作线程的 mcache 中没有合适(也就是特定大小的)的mspan 时就会从 `mcentral `去获取。**

**`mcentral` 被所有的工作线程共同享有，**存在多个 goroutine 竞争的情况，因此**从 `mcentral` 获取资源时需要加锁。**`mcentral `里维护着两个双向链表，non empty 表示链表里还有空闲的 mspan 待分配。empty 表示这条链表里的 mspan 都被分配了object 或缓存 mcache 中。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/HeIivs.png" alt="HeIivs" style="zoom: 33%;" />

> <font color='#39b54a'>如果P的本地mcache不够用了/没有合适的了，那么就会向全局的mcentral中索取合适的mspan使用，但是这个过程就需要加锁了。</font>

> <font color='#e54d42'>**mcache的核心设计思路与P的本地队列有异曲同工之处。都是为了避免全局大锁的限制，在本地制造缓存，并且为了防止内存碎片将细粒度下拉到很小**。</font>

程序申请内存的时候，mcache 里已经没有合适的空闲 mspan了，那么工作线程就会像下图这样去 `mcentral` 里去申请。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/evwtXN.png" alt="evwtXN" style="zoom:50%;" />

mcache 从 `mcentral `获取和归还 mspan 的流程：

* **获取 加锁**；从 non empty 链表找到一个可用的mspan；并将其从 non empty 链表删除；将取出的 mspan 加入到 empty 链表；将 mspan 返回给工作线程；解锁。

* **归还 加锁**；将 mspan 从 empty 链表删除；将mspan 加入到 non empty 链表；解锁。

**`mcentral `是 sizeclass 相同的 span 会以链表的形式组织在一起**, 就是指该 span 用来存储哪种大小的对象。

### 4. mheap

**当 `mcentral` 没有空闲的 mspan 时，会向 `mheap`申请。而 `mheap `没有资源时，会向操作系统申请新内存。**

**`mheap `主要用于大对象的内存分配，以及管理未切割的 mspan，用于给` mcentral `切割成小对象。**

`mheap `中含有<u>所有规格</u>的` mcentral`，所以当一个 mcache 从 mcentral 申请 mspan 时，<u>只需要在独立的 mcentral 中使用锁，并不会影响申请其他规格的 mspan。</u>

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/GDwBB2.png" alt="GDwBB2" style="zoom:50%;" />

**<font color='#e54d42'>整体分配的过程：mcache本地mspan => mcentral => mheap => 操作系统</font>**

<u>所有 mcentral 的集合则是存放于 `mheap` 中的</u>。` mheap `里的 <u>arena 区域是真正的堆区</u>，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象。运行时使用二维的` runtime.heapArena `数组管理所有的内存，每个 `runtime.heapArena` 都会管理 64MB 的内存。

如果 arena 区域没有足够的空间，会调用 `runtime.mheap.sysAlloc `从操作系统中申请更多的内存。（如下图：**Go 1.11 前**的内存布局）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/WoTLoF.png" alt="WoTLoF" style="zoom:67%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/OBFgcY.png" alt="OBFgcY" style="zoom:50%;" />

### 5. tiny

对于小于16字节的对象(且无指针)，Go 语言将其划分为了tiny 对象。划分 **tiny 对象的主要目的是为了处理极小的字符串和独立的转义变量。**对 json 的基准测试表明，使用 tiny 对象减少了12%的分配次数和20%的堆大小。tiny 对象会被放入class 为2的 span 中。

* 首先查看**之前分配**的元素中是否有空余的空间
* 如果当前要分配的大小不够，例如要分配16字节的大小，这时就需要找到下一个空闲的元素

tiny 分配的第一步是<u>尝试利用分配过的前一个元素的空间</u>，达到**节约内存**的目的。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/a4PtWE.png" alt="a4PtWE" style="zoom: 33%;" />



### 6. 大于32kb内存分配

Go 没法使用工作线程的本地缓存 mcache 和全局中心缓存 mcentral 上管理超过32KB的内存分配，所以对于那些**超过32KB的内存申请，会直接从堆上(mheap)上分配对应的数量的内存页(每页大小是8KB)给程序。**

历史版本的过程中，搜索满足需求大小的内存空间使用的方法：

* freelist
* treap （二叉搜索树）
* radix tree + pagecache （1.14之后）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220824090956314.png" alt="image-20220824090956314" style="zoom: 40%;" />

### 7. 总结

![K7w6uc](http://xwjpics.gumptlu.work/qinniu_uPic/K7w6uc.png)



**一般小对象通过 mspan 分配内存；大对象则直接由 mheap 分配内存。**

* Go 在程序启动时，会向操作系统申请一大块内存，由 mheap 结构全局管理(现在 Go 版本不需要连续地址了，所以不会申请一大堆地址)
* Go 内存管理的基本单元是 mspan，每种 mspan 可以分配特定大小的 object。
* **mcache, mcentral, mheap 是 Go 内存管理的三大组件，mcache 管理线程在本地缓存的 mspan；mcentral 管理全局的 mspan 供给所有线程**





