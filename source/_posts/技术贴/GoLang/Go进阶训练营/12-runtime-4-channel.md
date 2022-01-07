---
title: 12-runtime-4-channel
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-11-19 20:07:31
---

[TOC]

<!-- more -->

# 一、channel

## 1. 简介

go语言的两项特点：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/XIMnJ3.png" alt="XIMnJ3" style="zoom: 33%;" />

channel最基本的使用：`task queue`：

![image-20211119201738475](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119201738475.png)

channel的特点：

* 多访问goroutine安全
* 存储并且在多个goroutine间传递值
* FIFO先进先出
* 能够让goroutine锁住或解锁

## 2. making channels - the chan struct

将原本自己设计的结构体变为语言层面自己包含的结构channel进行使用

### 带缓冲的channel

一个带缓冲的`channel`内部核心就是一个**环形数组**，分别用了两个指针（发送、接收指针）以及一个锁机制

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211120142649000.png" alt="image-20211120142649000" style="zoom:50%;" />

（开始时两个指针都指向0）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119202710535.png" alt="image-20211119202710535" style="zoom: 33%;" />

（放入一个元素）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/ndpPZx.png" alt="ndpPZx" style="zoom:33%;" />

（数组满时）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/v6mBI5.png" alt="v6mBI5" style="zoom:33%;" />

（取一个元素时）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119203144223.png" alt="image-20211119203144223" style="zoom:33%;" />

hchan创建后一定分配在堆上，make初始化之后会返回一个hchan的指针，所以我们不需要再取地址就可以在函数之间使用。

![GO259M](http://xwjpics.gumptlu.work/qinniu_uPic/GO259M.png)

## 3. sends and receives - grouting scheduling

### 1. 基本过程

channel的发送和接受消息

场景如下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119203906547.png" alt="image-20211119203906547" style="zoom:33%;" />

放入一个任务的过程：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/fAVrrX.png" alt="fAVrrX" style="zoom:33%;" />

拿出一个任务的过程：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119204658432.png" alt="image-20211119204658432" style="zoom:33%;" />

整个过程并没有在多个goroutine共享内存，仅仅是在hchan这个结构体上进行了**拷贝**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119205035068.png" alt="image-20211119205035068" style="zoom:33%;" />

**不依靠共享内存通信，而是通过通信实现共享内存 -> 仅仅只是一些copy操作**

### 2. 等待与唤醒 - sudog

#### 1. 发送阻塞等待

当发送task4的时候G1会阻塞，但是需要注意的是，挂载G1的线程M并不会阻塞，而是会切换到g0根据优先级在P的本地队列中寻找新的goroutine （G1会变成gopark状态）

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/dDnYyC.png" alt="dDnYyC" style="zoom:33%;" />

当队列中有元素被拿走后G1就会恢复

 那么等待与恢复的过程是怎样实现的呢？

有两个特殊的字段：`sendq`和`recvq`（注意与上面的数组指针区分开）分指向一个等待队列叫做**`sudog`**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/6vZd8D.png" alt="6vZd8D" style="zoom:33%;" />

**在goroutine执行调度之前，如果G阻塞了会创建一个sudog的对象，并把task4和G1放进入，并把整体放入sendq中（指向）**

> <font color='#e54d42'>把等待的进程都存起来，等到要满的时候再将其唤醒</font>

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119211754768.png" alt="image-20211119211754768" style="zoom:33%;" />

当G2拿走一个任务后，就会唤醒`sendq`中的`sudog`中的G1

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/JP7Unt.png" alt="JP7Unt" style="zoom:33%;" />

 填充tesk4：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/pHBt6d.png" alt="pHBt6d" style="zoom:33%;" />

然后将G1变为`runnable`，状态变为`goready`,然后为了**亲缘性调度**将其放到**`runNext`**上（进程的调度部分说过），让其立即下一个执行

#### 2. 接收阻塞等待

同样的对于一个空的channel，G2取读取的话会阻塞。

1. 首先会将G2暂停将其的状态变为`gopark`

2. 在其被调度之前，创建`sudog`并保存G2以及数据
3. 将`sudog`放入到`recvq`中保存
4. 等待channel加入数据，唤醒

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119212707847.png" alt="image-20211119212707847" style="zoom:33%;" />

有意思的是，`sudog`存储的是值是**t，即栈上的地址**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119212843985.png" alt="image-20211119212843985" style="zoom:33%;" />

当G1将一个新的任务放入channel中时，按照正常的思路，我们应该使用`sudog`唤醒G2然后将task复制给t，<font color='#e54d42'>但是，这里的思路更加的聪明！</font>

**我们直接根据t在栈上的地址，将task赋值给t，因为此时G2一定处于`gopark`状态，所以这样做是一定安全的**

> <font color='#e54d42'>G1直接操作了G2的栈，没有经过中间的hchan来回复制，也避免了来回拿锁等操作</font>

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119213528624.png" alt="image-20211119213528624" style="zoom:33%;" />

像这样直接修改栈地址一般只会在`runtime`中使用

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119213750799.png" alt="image-20211119213750799" style="zoom:33%;" />

#### 3. direct send机制

对于有缓冲channel：

* 当接受者等待一个空channel的时候，一旦发送者发送就会使用“直接发送”机制（直接修改对方栈数据）

对于无缓冲的channel（循环队列长度为0）：

* 如果是发送者先行，那么接受者直接发送（修改栈）
* 如果是接受者先行，那么发送者直接发送（修改栈）

