---
title: 12-runtime-3-GC机制
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-11-12 14:38:00
---

<!-- more -->

# 一、GC原理

GC(Garbage Collection)现代高级编程语言管理内存的方式分为两种：自动和手动，像 C、C++、rust 等编程语言使用手动管理内存的方式，工程师编写代码过程中需要主动申请或者释放内存；而 PHP、Java 和 Go 等语言使用自动的内存管理系统，有内存分配器和垃圾收集器来代为分配和回收内存，其中垃圾收集器就是我们常说的 GC。主流的垃圾回收算法：

* **引用计数**（python、PHP）
* **追踪式垃圾回收**

Go 现在用的三色标记法就属于追踪式垃圾回收算法的一种。(go的gc机制也是不断的在更改，到了1.8才算的上及格)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/da0cKV.png" alt="da0cKV" style="zoom:50%;" />

## 1. Mark & Sweep 标记清除法

* STW

  stop the world, GC 的一些阶段需要停止所有的 mutator 以确定当前内存的引用关系。**这便是很多人对 GC 担心的来源，这也是 GC 算法优化的重点。（减少暂停的时间）**

* Root

  根对象是 mutator **不需要通过其他对象就可以直接访问到的对象**。比如<u>全局对象，栈对象中的数据</u>等。通过Root对象。可以追踪到其他存活的对象。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/eu2ZPz.png" alt="eu2ZPz" style="zoom: 33%;" />

> <font color='#39b54a'>例如上图所示：停止所有的mutator然后才能确定所有的引用关系，将有引用的mutator标记为红色，没有被引用的就是绿色。清除掉所有绿色的即可</font>

Mark Sweep 两个阶段：标记(Mark)和 清除(Sweep)两个阶段

这个算法就是严格按照**追踪式算法**的思路来实现的：

* Stop the World
* Mark：通过 Root 和 Root 直接/间接访问到的对象， 来寻找所有可达的对象，并进行标记。
* Sweep：对堆对象迭代，已标记的对象置位标记。所有**未标记的对象加入freelist， 可用于再分配。**
* Start the Wrold

这个算法最大的问题是 GC 执行期间需要把整个程序完全暂停，朴素的 Mark Sweep 是整体 STW，并且分配速度慢，内存碎片率高。

go v1.1版本中, STW可能是秒级，因为在整个过程中整个程序要全部暂停，然后进行扫描

随后进行了改进：

**go v1.3版本后，只有标记过程需要 STW，因为对象引用关系如果在标记阶段做了修改，会影响标记结果的正确性。**并引入了并发GC：

**并发 GC** 分为两层含义：

* 每个 mark 或 sweep 本身是多个线程(协程)执行的(concurrent)
* mutator 应用程序和 collector 内存分配器也同时运行(background)

concurrent 这一层是比较好实现的, GC 时整体进行STW，那么对象引用关系不会再改变，对 mark 或者sweep 任务进行分块，就能多个线程(协程) conncurrent 执行任务 mark 或 sweep。

而对于 backgroud 这一层, 也就是说 mutator 和 mark，sweep 同时运行，则相对复杂。

* 1.3**以前**的版本使用标记-清扫的方式，**整个过程**都需要 STW。
* 1.3版本分离了标记和清扫的操作，**标记过程STW，清扫过程并发执行**。

backgroup sweep 是比较容易实现的，因为 mark 后，哪些对象是存活，哪些是要被 sweep 是已知的，sweep 的是不再引用的对象。**sweep 结束前，这些对象不会再被分配到，所以 sweep 和 mutator 运行共存**。**无论全局还是栈不可能能访问的到这些对象，可以安全清理。**

> <font color='#39b54a'>其实sweep与mutator运行共存是很好理解的，因为已经STW扫描了全部栈上的对象，如果一个对象没有被引用，那么其就不可能再被访问到，后续也不可能再使用，所以可以安全的直接清理</font>

* **1.5版本在标记过程中使用三色标记法**。**标记和清扫都并发执行的，但<u>标记阶段的前后需要 STW 一定时间</u>来做 GC 的准备工作和栈的re-scan。**

## 2. Tri-color Mark & Sweep 三色标记法

三色标记是对标记清除法的改进，标记清除法在整个执行时要求长时间 STW，Go 从1.5版本开始改为三色标记法，**初始将所有内存标记为白色，然后将 roots 加入待扫描队列(进入队列即被视为变成灰色)，然后使用并发 goroutine 扫描队列中的指针，如果指针还引用了其他指针，那么被引用的也进入队列，被扫描的对象视为黑色。**

* 白色对象：**潜在的垃圾**，其内存可能会被垃圾收集器回收。
* 黑色对象：**活跃的对象**，包括不存在任何引用外部指针的对象以及从根对象可达的对象，**垃圾回收器不会扫描这些对象的子对象。**
* 灰色对象 ：**活跃的对象**，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象。

> <font color='#39b54a'>未扫描时所有对象为白色，然后将roots（全局对象、栈数据等）标记为灰色，进入待扫描队列，然后经过一系列的扫描，如果一个对象被引用就加入扫描队列标记为灰色，过程中被扫描过的对象就是黑色</font>

垃圾收集器从 root 开始然后跟随指针递归整个内存空间。**分配于 `noscan `的 `span` 的对象(mcache会区分这两种mspan),不会进行下一步扫描。 (因为这些对象不含有指针，所以不用下一步扫描了) **

然而，此过程不是由同一个 goroutine 完成的，每个指针都排队在工作池中 然后，先看到的被标记为工作协程的后台协程从该池中出队，扫描对象，然后将在其中找到的指针排入队列。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/r7m7CB.png" alt="r7m7CB" style="zoom: 33%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/nT4XUm.png" alt="nT4XUm" style="zoom:50%;" />

染色流程：

* 一开始所有对象被认为是白色

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/CXHfHm.png" alt="CXHfHm" style="zoom:33%;" />

* 根节点(stacks，heap，global variables)被染色为灰色

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/z8Ux7p.png" alt="z8Ux7p" style="zoom: 33%;" />

  ​	此时S2会被直接被标记为黑色，因为他是noscan对象

一旦主流程走完，gc会：

* 选一个灰色对象，标记为黑色（S1）

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/sOodQk.png" alt="sOodQk" style="zoom:33%;" />

* 遍历这个对象的所有指针，标记所有其引用的对象为灰色（上图S2被标灰）

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/HaELTI.png" alt="HaELTI" style="zoom:33%;" />

最终直到所有对象需要被染色。

标记结束后，**黑色对象是内存中正在使用的对象，而白色对象是要收集的对象**。

由于struct2的实例是在匿名函数中创建的，并且无法从堆栈访问，因此它保持为白色，可以清除。（对应于上图的白色S2）

**颜色在内部实现原理：**

**每个 span 中有一个名为 `gcmarkBits `的位图属性，该属性跟踪扫描，并将相应的位设置为1。**(标记是否被扫描过)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/16gSw8.png" alt="16gSw8" style="zoom:50%;" />

## 3. Write Barrier

1.5版本在标记过程中使用三色标记法。回收过程主要有四个阶段，其中，标记和清扫都并发执行的，但**标记阶段的前后需要 STW 一定时间来做GC 的准备工作和栈的 re-scan。**

各个版本gc效率对比：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/pnpbKC.png" alt="pnpbKC" style="zoom:67%;" />

使用并发的垃圾回收，也就是**多个 Mutator 与 Mark 并发执行（不需要整个标记阶段都停止）**，<u>想要在<font color='#e54d42'>**并发或者增量的标记算法（程序边执行边标记）**</font>中保证正确性，我们需要达成以下两种三色不变性(Tri-color invariant)中的任意一种：</u>

* **强三色不变性**：<u>黑色对象不会指向白色对象</u>，只会指向灰色对象或者黑色对象。
* **弱三色不变性** ：如果需要黑色对象可以指向白色对象，那么就一定需要黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径。

可以看出，一个白色对象被黑色对象引用，是注定无法通过这个黑色对象来保证自身存活的，与此同时，如果所有能到达它的灰色对象与它之间的可达关系全部遭到破坏，那么这个白色对象必然会被视为垃圾清除掉。 故当上述两个条件同时满足时，就会出现**对象丢失的问题**。**如果这个白色对象下游还引用了其他对象，并且这条路径是指向下游对象的唯一路径，那么他们也是必死无疑的。**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211112162556193.png" alt="image-20211112162556193" style="zoom:50%;" />



为了防止这种现象的发生，最简单的方式就是 STW，直接禁止掉其他用户程序对对象引用关系的干扰，但是 STW 的过程有明显的资源浪费，对所有的用户程序都有很大影响，**如何能在保证对象不丢失的情况下合理的尽可能的提高 GC 效率，减少 STW 时间呢？**

### Dijkstra 写屏障

**插入屏障拦截将白色指针插入黑色对象的操作，标记其对应对象为灰色状态，这样就不存在黑色对象引用白色对象的情况了，满足强三色不变式，在插入指针 f 时将 C 对象标记为灰色。**

> <font color='#39b54a'>一旦一个黑色对象指向一个白色对象，那么就把这个白色对象变为灰色。也就是上面的第二步的时候直接将C标记为灰色</font>

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/puBFlQ.png" alt="puBFlQ" style="zoom:50%;" />

其实具体的做法就是在编译期间在你的代码上强制插入了一段代码让这个C变成了灰色

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/JIPpCG.png" alt="JIPpCG" style="zoom:50%;" />

如果对栈上的写做拦截，那么流程代码会非常复杂，并且性能下降会非常大，得不偿失。根据局部性的原理来说，其实我们程序跑起来，大部分的其实都是操作在栈上，函数参数啊、函数调用导致的压栈出栈、局部变量啊，协程栈，这些如果也弄起写屏障，那么可想而知了，根本就不现实，复杂度和性能就是越不过去的坎。

**所以go 1.5只对堆上的对象做这个拦截操作/写屏障**

1. 内存屏障只是对应一段特殊的代码；
2. 内存屏障这段代码在编译期间生成；
3. 内存屏障本质上**在运行期间拦截内存写操作，相当于一个 hook 调用**

Go 团队在实现上选择了在标记阶段完成时暂停程序、将所有栈对象标记为灰色并重新扫描，在活跃 goroutine 非常多的程序中，重新扫描的过程需要占用 10 ~ 100ms 的时间。

**1.5版本的整体过程：**

1. 初始化 GC 任务，包括**开启写屏障(write barrier)和开启辅助 GC(mutator assist)**，统计 root 对象的任务数量等，这个过程需要 STW（非常快）。
2. 扫描所有 root 对象，包括全局指针和 goroutine(G) 栈上的指针(**扫描对应 G 栈时需停止该 G**)，将其加入标记队列(灰色队列)，并循环处理灰色队列的对象，直到灰色队列为空，**该过程后台并行执行**。
3. 完成标记工作，**重新扫描(re-scan)全局指针和栈**。因为 Mark 和 mutator 是并行的，所以在 Mark 过程中可能会有新的对象分配和指针赋值，这个时候就需要通过写屏障(write barrier)记录下来，re-scan 再检查一下，这个过程也是会 STW 的。
4. 按照标记结果回收所有的白色对象，该过程后台并行执行。

![oBUQRD](http://xwjpics.gumptlu.work/qinniu_uPic/oBUQRD.png)

> <font color='#39b54a'>在编译阶段就通过写屏障将堆上的上述情况进行拦截，栈上的不管。然后扫描标记与内存创建、应用程序并行运行，在完成标记之前再re-scan标记刚刚并行期间以及栈上所有的未被引用的对象，最后统一删除清理</font>

### Yuasa 删屏障 

删除屏障也是**拦截写操作**的，但是是**通过保护灰色对象到白色对象的路径不会断**来实现的。如下图例中，在删除指针 e 时将对象 C 标记为灰色，这样 C 下游的所有白色对象，即使会被黑色对象引用，最终也还是会被扫描标记的，满足了弱三色不变式。这种方式的**回收精度低**，**一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮**，在下一轮 GC 中被清理掉。(一个本来就需要删除的白色对象需要在下一轮才会被回收)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Wapt6e.png" alt="Wapt6e" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/StITg9.png" alt="StITg9" style="zoom: 50%;" />

### 混合屏障

插入屏障和删除屏障各有优缺点：

**Dijkstra 的插入写屏障在标记开始时*无需 STW（只需要一小段的STW开启它）*，可直接开始，并发进行**，**但结束时需要 STW 来重新扫描栈**，标记栈上引用的白色对象的存活；

Yuasa 的**删除写屏障则<u>需要在 GC 开始时 STW 扫描堆栈</u>来记录初始快照，**这个过程会保护开始时刻的所有存活对象，但**结束时无需 STW。**

**Go1.8 混合写屏障结合了Yuasa的删除写屏障和Dijkstra的插入写屏障**

Golang 中的混合写屏障满足的是变形的弱三色不变式，<font color='#e54d42'>**同样允许黑色对象引用白色对象，白色对象处于灰色保护状态，但是只由堆上的灰色对象保护。**</font>

例如，一个堆上的灰色对象B，引用白色对象C，在GC并发运行的过程中，如果栈已扫描置黑，而赋值器将指向C的唯一指针从B中删除，并让栈上其他对象引用它，这时，写屏障会在删除指向白色对象C的指针的时候就将C对象置灰，就可以保护下来了，且它下游的所有对象都处于被保护状态。 

如果对象B在栈上，引用堆上的白色对象C，将其引用关系删除，且新增一个黑色对象到对象C的引用，那么就需要通过shade(ptr)来保护了，在指针插入黑色对象时会触发对对象C的置灰操作。如果栈已经被扫描过了，那么栈上引用的对象都是灰色或受灰色保护的白色对象了，所以就没有必要再进行这步操作。

> * <font color='#e54d42'>写屏障：对于堆上的黑色对象，如果已经变为黑色再引用一个白色对象，那么就将这个白色对象变为灰色保护起来。（写屏障对堆上的强三色不变性处理）</font>
>
> * <font color='#e54d42'>删屏障：对于栈上的灰色对象，如果其和一个黑色对象同时引用一个白色对象，当运行时删除了灰色对象对白色对象的引用的时候就将这个白色对象变为灰色。（这保证了栈上所有对象不会出错，弥补了写屏障对栈不处理的缺点）</font>

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/f6mhsY.png" alt="f6mhsY" style="zoom:50%;" />

**所以，只需要一开始的扫描STW，后续不再需要重新扫描了，因为后面的白色由灰色来保护（删屏障）**

由于结合了 Yuasa 的删除写屏障和 Dijkstra 的插入写屏障的优点，<font color='#e54d42'>只需要在开始时并发扫描各个goroutine 的栈，使其变黑并一直保持，这个过程不需要 STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行 re-scan 操作了，减少了 STW 的时间。</font>

**为了移除栈的重扫描过程**，除了引入混合写屏障之外，在垃圾收集的标记阶段，**我们还需要将创建的所有新对象都标记成黑色，防止新分配的栈内存和堆内存中的对象被错误地回收**，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/bDHanw.png" alt="bDHanw" style="zoom: 50%;" />

### Sweep

Sweep 让 Go 知道哪些内存可以重新分配使用，然而，Sweep 过程并不会处理释放的对象内存置为0(`zeroing the memory`)。而是在分配重新使用的时候，重新` reset bit`。

> <font color='#39b54a'>在释放的时候不会将其内存空间重制为0，而是在**再次使用的时候将其设置为0**。也就是说标记了对象0、1之后，再次使用该内存空间的时候如果是0（可清除）那么才将其所有位清空为0</font>

![YLH8Gt](http://xwjpics.gumptlu.work/qinniu_uPic/YLH8Gt.png)

每个 span 内有一个 `bitmap allocBits`，他表示上一次 GC 之后每一个 object 的分配情况

1：表示已分配，0：表示未使用或释放。

内部还使用了 `uint64 allocCache(deBruijn)`，加速寻找 freeobject。

这个`allocBits`怎样赋值呢？

GC 将会启动去释放不再被使用的内存。在标记期间，GC 会用一个位图` gcmarkBits` 来跟踪在使用中的内存。
正在被使用的内存被标记为黑色，然而当前执行并不能够到达的那些内存会保持为白色。

现在，我们可以使用` gcmarkBits `精确查看可用于分配的内存。**Go 使用 `gcmarkBits` 赋值了` allocBits`**，这个操作就是内存清理。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/NicGa3.png" alt="NicGa3" style="zoom:50%;" />

然而必须每个 扫描一遍 来一次类似的处理，需要耗费大量时间。Go 的目标是在清理内存时不阻碍执行，并为此提供了两种策略：

Go 提供两种方式来清理内存：

* **在后台启动一个 worker 等待清理内存，一个一个 mspan 处理**
  当开始运行程序时，Go 将设置一个后台运行的 Worker(唯一的任务就是去清理内存)，它将进入睡眠状态并等待内存段扫描。

* **当申请分配内存时候 lazy 触发**

  当应用程序 goroutine 尝试在堆内存中分配新内存时，会触发该操作。**清理导致的延迟和吞吐量降低被分散到每次内存分配时。**

清理内存段的**第二种方式是即时执行**。但是，由于这些内存段已经被分发到每一个处理器 P 的本地缓存 mcache 中，因此很难追踪首先清理哪些内存。这就是为什么 **Go 首先将所有内存段移动到 mcentral 的原因。然后，它将会让本地缓存 mcache 再次请求它们，去即时清理。**

即时扫描确保所有内存段都会得到清理（节省资源），同时不会阻塞程序执行。

由于后台只有一个 worker 在清理内存块，清理过程可能会花费一些时间。但是，我们可能想知道如果另一个 GC 周期在一次清理过程中启动会发生什么。在这种情况下，这个运行 GC 的 Goroutine 就会在开始标记阶段前去协助完成剩余的清理工作。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211112183631161.png" alt="image-20211112183631161" style="zoom:50%;" />

## 4. Stop The World

在垃圾回收机制 (GC) 中，"Stop the World" (STW) 是一个重要阶段。顾名思义， 在 "Stop the World" 阶段， **当前运行的所有程序将被暂停， 扫描内存的 root 节点和添加<u>写屏障</u> (write barrier) 。**

这个阶段的第一步， 是抢占所有正在运行的 goroutine（sysmon发送信号量抢占），被抢占之后， 这些 goroutine 会被悬停在一个相对安全的状态。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/9FfIXL.png" alt="9FfIXL" style="zoom:50%;" />



处理器 P (无论是正在运行代码的处理器还是已在 idle 列表中的处理器)， 都会被被标记成停止状态 (stopped)， 不再运行任何代码。 调度器把每个处理器的 M  从各自对应的处理器 P 分离出来， 放到 idle 列表中去。

对于 Goroutine 本身， 他们会被放到一个全局队列中等待。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211112183915560.png" alt="image-20211112183915560" style="zoom:50%;" />



<img src="http://xwjpics.gumptlu.work/qinniu_uPic/KbT2m6.png" alt="KbT2m6" style="zoom: 33%;" />



运行时中有 GC` Percentage `的配置选项，默认情况下为100。

此值表示在下一次垃圾收集必须启动之前可以分配多少新内存的比率。将 GC 百分比设置为100意味着，基于在垃圾收集完成后标记为活动的堆内存量，下次垃圾收集前，堆内存使用可以增加100%。

如果超过2分钟没有触发，会强制触发 GC。

> 使用环境变量 GODEBUG 和 gctrace = 1选项生成GC trace GODEBUG=gctrace=1 ./app

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/CUQ2nC.png" alt="CUQ2nC" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/lFwUfq.png" alt="lFwUfq" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/xqFqyM.png" alt="xqFqyM" style="zoom: 33%;" />

