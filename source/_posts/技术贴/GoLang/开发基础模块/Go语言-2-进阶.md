---
title: Go语言-2-进阶
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2020-11-16 16:26:40
---

# 三、Go进阶

## 1. goroutine

### 1.1 进程与线程：

* 程序: 编译成功的二进制文件.(只占用磁盘存储)   => 剧本
* 进程: 运行起来的程序. 占用系统资源. (内存)     => 戏剧

* 线程: LWP(Light weight process 轻量级(资源分配)的进程, 本质仍然是进程(linux下))

程序是死的, 进程是活的, 一个程序可以开启多个进程

#### 进程的基本工作状态

五种:初始态、就绪态、运行态、挂起/阻塞态与终止态 . 其中初始态为进程准备阶段,常与就绪态结合来看

![G9Eurj](http://xwjpics.gumptlu.work/qinniu_uPic/G9Eurj.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122193733.png)

<!-- more -->

#### 进程与线程

进程:  <font color='red'>最小的**资源分配**单位    </font>

线程: LWP轻量级的进程,<font color='red'> 最小的**执行单元**   </font> —cpu分配时间轮片的对象

### 1.2 并发与并行与同步 

#### 并发与并行

1) 多线程程序在**单核**上运行，就是**并发**, 软件上可以进行调度设计
2) 多线程程序在**多核**上运行，就是**并行**, 并行是硬件上实现的(不研究)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122193959.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122194040.png)

并发:

* 宏观并发: 在用户体验上来说,程序在**并行**执行 <font color='green'> (因为人能感受到的时间变化是毫秒, 假并行)</font>
* 微观并发: 多个计划任务,顺序执行,在飞快的切换.轮换使用cpu**时间轮片**

#### 同步

当多个线程在**同一时刻**访问**共享资源**时,产生与时间相关的错误, 需要同步机制, 避免由时间导致的数据混乱.

**同步就是协同步骤,规划顺序**

同步机制在线程中完成 => 线程同步

同步机制在进程中完成 => 进程同步

同步机制在协程中完成 => 协程同步

### 1.3 协程

协程: `goroutine `也叫轻量级的线程,其最大的优势在于轻量级,轻量级可以达到只对应一个函数

Python、Lua、Ruset都有协程的概念

<font color='red'> 进程、线程的并发时为了争夺cpu的使用权,提高cpu执行的速度, 而协程的并发是为了提高程序执行的效率(在线程阻塞等待时进一步的利用其进行工作)</font>

### 1.4 进程、线程、协程总结

三者都可以完成并发.

* **进程**独自占用资源,<font color='green'>**稳定性较强,但是开销较大**</font>

* **线程**轻量级的进程能够<font color='green'>**减少资源开销**</font>(线程之间的切换时体现)
* **协程**则是在线程的进一步基础上<font color='green'>**提高了使用效率**</font>

**所以需要按需求选择并发**

#### 进一步理解

进程、线程、协程对比

通俗描述

有一个老板想要开个工厂进行生产某件商品（例如剪子） 
他需要花一些财力物力制作一条生产线，这个生产线上有很多的器件以及材料这些所有的 为了能够<u>生产剪子而准备的**资源**称之为：进程</u>

只有生产线是不能够进行生产的，所以老板的找个工人来进行生产，这个工人能够利用这些材料最终一步步的将剪子做出来，这个<u>来做事情的工人称之为：线程</u>

这个老板为了提高生产率，想到3种办法：

> 方式1
>
> 在这条生产线上多招些工人，一起来做剪子，这样效率是成倍増长，即<font color='red'>单进程 多线程</font>
>
> 方式 2 
> 老板发现这条生产线上的工人不是越多越好，因为一条生产线的资源以及材料毕竟有限，所以老板又花了些财力物力购置了另外一条生产线，然后再招些工人这样效率又再一步提高了，即<font color='red'>多进程 多线程</font>
>
> 方式 3 
> 老板发现，现在已经有了很多条生产线，并且每条生产线上已经有很多工人了（即程序是多进程的，每个进程中又有多个线程），为了再次提高效率，老板想了个损招， 
> 规定：如果某个员工在上班时临时没事或者再等待某些条件（比如等待另一个工人生产完谋道工序 之后他才能再次工作） ，那么这个员工就利用这个时间去做其它的事情， 那么也就是说：**如果一个线程等待某些条件，可以充分利用这个时间去做其它事情**，其实这就是：<font color='red'>协程</font>方式

简单总结 
1 **进程是资源分配的基本单位** 
2 **线程是操作系统调度的基本单位** 
3 进程切换需要的资源很最大，效率很低 
4 线程切换需要的资源一般，效率一般 
5 **协程切换任务资源很小，效率高** 
6 **多进程、多线程根据cpu核数不一样可能是并行的 也可能是并发的**。<font color='green'>**协程的本质就是使用当前进程在不同的函数代码中切换执行，可以理解为并行**</font>。 协程是一个用户层面的概念，**不同协程的模型实现可能是单线程，也可能是多线程**。

**进程拥有自己独立的堆和栈，既不共享堆，亦不共享栈**，**进程由操作系统调度**。（全局变量保存在堆中，局部变量及函数保存在栈中）

**线程拥有自己独立的栈和共享的堆，共享堆，<u>不共享栈</u>**，**线程亦由操作系统调度**(标准线程是这样的)。

**协程和线程一样共享堆，不共享栈**，<font color='red'>**协程由程序员在协程的代码里显式调度**。</font>

**一个应用程序一般对应一个进程，一个进程一般有一个主线程，还有若干个辅助线程，<u>线程之间是平行运行的，在线程里面可以开启协程</u>，让程序在特定的时间内运行。**

协程和线程的区别是：

**协程避免了无意义的调度，由此可以提高性能，但也因此，<u>程序员必须自己承担调度的责任</u>，同时，<u>协程也失去了标准线程使用多CPU的能力</u>。**

### 1.5 Go协程（goroutine）和Go主线程

Go 主线程(有程序员直接称为线程/也可以理解成**进程**): 一个Go 线程上，可以起多个协程，可以这样理解，**协程是轻量级的线程[编译器做优化]。**

理解： **Go主线程  类比于  进程   Go协程  类比于  优化后的线程**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122194855.png)

![4jwZav](http://xwjpics.gumptlu.work/qinniu_uPic/4jwZav.png)

**<font color='red'>Go 协程的特点：</font>**

<font color='red'> **相比于其他程序在程序设计层面上实现并发,Go是从语言层面就支持并发**</font>

1) **有==独立的==栈空间**
2) **==共享==程序堆空间**
3) **调度由用户控制**
4) 协程是轻量级的线程
5) 协程是是**用户态**的

### 1.6 协程的使用：

传统的写法，会先执行test，在执行主线程下面的内容

```go
func test()  {
	for i:=0; i<=10; i++ {
		fmt.Println("test : Hello")
		time.Sleep(time.Second)	//每一秒休眠一次
	}
}

func main() {
	test()

	for i:=0; i<=10; i++ {
		fmt.Println("main : Hello")
		time.Sleep(time.Second)	//每一秒休眠一次

	}
}
```

使用协程的写法：

```go
func test()  {
	for i:=0; i<=10; i++ {
		fmt.Println("test : Hello")
		time.Sleep(time.Second)	//每一秒休眠一次
	}
}

func main() {
	go test()  //只需要加个go  开启一个协程

	for i:=0; i<=10; i++ {
		fmt.Println("main : Hello")
		time.Sleep(time.Second)	//每一秒休眠一次

	}
}


//交替调用  test与主线程同时执行
test : Hello 0
main : Hello 0
main : Hello 1
test : Hello 1
test : Hello 2
main : Hello 2
main : Hello 3
test : Hello 3
test : Hello 4
main : Hello 4
test : Hello 5
main : Hello 5
test : Hello 6
main : Hello 6
test : Hello 7
main : Hello 7
test : Hello 8
main : Hello 8
test : Hello 9
```

程序执行的流程图：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122201217.png)

<font color='red'>**协程的结束与否以主线程的运行为准**</font>

1. 主线程是一个物理线程，直接作用在cpu 上（操作系统控制）的。是重量级的，**非常耗费cpu 资源**。

2.  协程从主线程开启的，是轻量级的线程，是逻辑态。对资源消耗相对小。
3. Golang 的协程机制是重要的特点，**可以轻松的开启上万个协程**。<u>其它编程语言的并发机制是一般基于线程的</u>，开启过多的线程，资源耗费大，这里就突显Golang 在并发上的优势了

理解：<font color='green'>其实协程就是轻量级的线程，golang通过程序员控制这样的协程去避免大量线程的创建并且在线程的执行效率上做了优化，从而实现强大的并发功能</font>

### 1.7 runtime包

#### Gosched

作用: 出让当前此次go程序所占用的时间片(后续分配到了还会从出让位置继续执行)

func [Gosched](https://github.com/golang/go/blob/master/src/runtime/extern.go?name=release#78)

```
func Gosched()
```

Gosched使当前go程放弃处理器，以让其它go程运行。它不会挂起当前go程，因此当前go程未来会恢复执行。

```go
func main() {

	go func() {
		for {
			fmt.Println("--------goroutine--------")
		}
	}()


	for {
		runtime.Gosched()			// 输出中goroutine占大多数了
		fmt.Println("----------main-----------")
	}

}
```

#### Goexit

goexit将会立即终止当前goroutine的执行

goexit与return之间的差异:

* return: 返回当前**函数**调用,后续的函数不会执行,但是其之**前**如果有defer语句的话在当前函数结束之前还是会执行     

* Goexit: 退出**当前go程**,在其之**前**的defer会注册,其后的也不会执行

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/IfHFDz.png" alt="IfHFDz" style="zoom:50%;" />

#### GOMAXPROCS

设置并行计算的CPU核数最大值, 返回上一次设置cpu的值

```go
func main() {

	n := runtime.GOMAXPROCS(4)
	fmt.Println("n = ", n)		//8， 原本默认为8
	n = runtime.GOMAXPROCS(1)
	fmt.Println("n = ", n)		//4， 输出上次设置的值
	for {
		go fmt.Print("1")		// 设置成单核之后，对于一个语句会一直执行到时间片用完为止，所以1和0出现会很有规律
		time.Sleep(1*time.Second)	// 1010101010101010....
		fmt.Print("0")
	}

}
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122204102.png)

### 1.6 goroutine的调度模型

MPG 模式基本介绍：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122201831.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122202016.png)



##### MPG 模式运行的状态1

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122202033.png)

##### MPG 模式运行的状态2

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122202508.png)

理解：**<font color='red'>动态的为单个线程中的协程队列切换其他线程（在线程池中取）执行</font>**

## 2. go程通信与channel、锁

### 2.1 协程资源共享问题

例子：需求：现在要计算1-200 的各个数的阶乘，并且把各个数的阶乘放入到map 中。最后显示出来。要求使用goroutine 完成

```go
var newMap = make(map[int]int, 100)

func Cal(a int) {
	var res int
	for i := 1; i <= a; i++ {
		res *= i
	}
	//写入
	newMap[a] = res
}


func main() {
	for i:=1; i<=200; i++ {
		//多协程计算
		go Cal(i)
	}
	for i, v := range newMap {
		fmt.Println(i," : ", v)
	}
}
```

代码逻辑上没有问题，但是程序执行起来就会出现问题：**<font color='red'>资源竞争</font>**

**go判断是否会出现资源竞争问题：在go run 运行的时候添加参数 -race**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122205458.png)

1) 使用goroutine 来完成，效率高，但是会**出现并发/并行安全问题**.
2) 这里就提出了**不同goroutine 如何通信的问题**

**多个协程同时对一个map空间写入：**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122210208.png)

### 2.2 解决方法：

#### 2.2.1 全局变量锁

加入**全局变量锁**机制

包：<font color='red'>**sync**</font>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122211012.png)

代码：

```go
var (
	newMap = make(map[int]int, 100)
	lock sync.Mutex
)



func Cal(a int) {
	res := 1
	for i := 1; i <= a; i++ {
		res *= i
	}
	//写入
	lock.Lock()  //锁住资源
	newMap[a] = res
	lock.Unlock() //写入完毕后，解锁
}


func main() {
	for i:=1; i<=10; i++ {
		//多协程计算
		go Cal(i)
	}

	time.Sleep(time.Second * 5) //添加睡眠，让主线程等待协程结束

	for i, v := range newMap {
		fmt.Println(i," : ", v)
	}
}

```

**这种方法不是推荐的方法，因为不知道主进程要等多久，channel是更好的办法**

1.**主线程在等待所有goroutine 全部完成的时间很难确定**，我们这里设置10 秒，仅仅是估算。

2.如果主线程休眠时间长了，会加长等待时间，如果等待时间短了，可能还有goroutine 处于工作
状态，这时也会随主线程的退出而销毁

3.**通过全局变量加锁同步来实现通讯，也并不利用多个协程对全局变量的读写操作。**

4.上面种种分析都在呼唤一个新的通讯机制-channel

**<font color='red'>主进程需要等待所有的协程都跑完了才能继续执行，但是这样的等待是不可知晓的，要用特殊的方法去实现</font>**

### 2.3 channel

#### 2.3.1 概念

channel是一个go语言的数据结构,主要用来解决go程/协程的**资源同步问题**以及写成之间的**数据共享/传递问题**

goroutine运行在**相同的**地址空间,因此访问共享内存必须做好同步.goruntine奉行<font color='red'>**通过通信来共享内存,而不是共享内存来通信** </font>

* 传统的方式是通过共享内存来实现通信, 因为进程每个开辟的内存空间中, 内存中内核的空间都是相同的(同一台机器同一个内核), 所以通过对此空间中的数据进行读/写实现进程之间的通信.

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/LAuAz3.png" alt="LAuAz3" style="zoom: 33%;" />

* 而goroutine则是通过与channel通信来实现资源的共享

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/Y2xR61.png" alt="Y2xR61" style="zoom:50%;" />

  ![ce0YlV](http://xwjpics.gumptlu.work/qinniu_uPic/ce0YlV.png)



1) channel 本质可以看作是一个**数据结构-队列**【示意图】
2) 数据是先进先出**【FIFO : first in first out】**
3) <font color='red'>线程安全，多goroutine 访问时，不需要加锁</font>，就是说**channel 本身就是线程安全**的。<font color='red'>**可认为不管多少协程通过管道去操作共享资源都可以认为是安全的，由底层编译器实现**</font>
4) **channel 有类型**的，一个string 的channel 只能存放string 类型数据。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201122213233.png)



#### 2.3.2 基本语法

* var 变量名 chan 数据类型
*  举例：
  var intChan chan int (intChan 用于存放int 数据)
  var mapChan chan map[int]string (mapChan 用于存放map[int]string 类型)
  var perChan chan Person
  var perChan2 chan *Person
  ...

说明：

1. channel 是**引用类型**
2. **channel 必须初始化（make）才能写入数据, 即make 后才能使用**
3. 管道是有类型的，intChan 只能写入整数int

4. <font color='red'>**channel是在创建时固定大小的，不能够自动扩容（能自动扩容点的目前就是map）**</font>

```go
func main() {
	//创建一个channel
	var intChannel chan int
	//初始化
	intChannel = make(chan int, 3)
	//输出一下
	fmt.Println(intChannel)
	fmt.Println("容量:",cap(intChannel), len(intChannel)) //3 0
	//加入数据进管道
	intChannel<- 1
	num := 55
	intChannel<- num
	intChannel<- 108
	//输出容量
	fmt.Println("容量:",cap(intChannel), len(intChannel))// 3 2
	//取管道，读取数据
	num2 := <-intChannel
	fmt.Println(num2)		//108
	num3 := <-intChannel
	fmt.Println(num3)		//55
	//也可以取出来不处理
	<-intChannel
	fmt.Println("容量:",cap(intChannel), len(intChannel))// 3 0
}

```

#### 2.3.3 注意事项

1) channel 中只能存放指定的数据类型
2) channle 的数据放满后，就不能再放入了
3) 如果从channel 取出数据后，可以继续放入
4) **在没有使用协程的情况下，如果channel 数据取完了，再取，就会报dead lock**

#### 2.3.4 案例演示

* 放Map

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123165836.png)

* 结构体

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123165917.png)

* 指针

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123170003.png)

* <font color='red'>存放任何数据类型，使用空接口</font>

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123170125.png)

注意：<font color='red'>**使用空接口取出数据时一定要注意类型断定**</font>

```go
func main() {
   var allChannel chan interface{}
   allChannel = make(chan interface{}, 20)
   dog := Dog{
      Name:  "xx",
      Age:   10,
      Phone: 10,
      sex:   "mu",
   }
   allChannel<- dog
   dog2 := &Dog{
      Name:  "哈哈哈",
      Age:   20,
      Phone: 20,
      sex:   "lal",
   }
   allChannel<- dog2
   allChannel<- 10
   allChannel<- "shjdhaj"
   pop := <- allChannel
   //取出
   //弹出的类型main.Dog, 值：{%!V(string=xx) %!V(int=10) %!V(int8=10) %!V(string=mu)}
   fmt.Printf("弹出的类型%T, 值：%V\n", pop, pop)
   //重点！！！！虽然运行期把底层编译器会识别interface为Dog类型，但是在编译器编译器不知道，所以会直接报错
   //fmt.Println(pop.Name)
   //解决方法：类型断言
   a := pop.(Dog) //断言是这个类型
   fmt.Println(a.Name)
}
```

#### 2.3.5 channel遍历

不要使用普通的for循环遍历

channel 支持**for--range** 的方式进行遍历，请注意两个细节
1) 在遍历时，**如果channel 没有关闭，则回出现<u>deadlock</u> 的错误**
2) 在遍历时，**如果channel 已经关闭，则会正常遍历数据**，遍历完后，就会退出遍历。

```go
func main() {
	intChannel := make(chan int, 100)
	//写入数据
	for i:=1; i<=100; i++ {
		intChannel<-i
	}
	//遍历管道  不能用len，因为每次都会弹出一个len也是递减的
	//for i:=1; i<=len(intChannel); i++ {
	//	fmt.Println(<-intChannel)
	//}

	//可以用cap，但是不建议
	//for i:=1; i<=cap(intChannel); i++ {
	//	fmt.Println(<-intChannel)
	//}
	close(intChannel)			//在遍历之前关闭管道则会正确输出，理解为在末尾添加EOF标识
	for v := range intChannel{
		fmt.Println(v)			//fatal error: all goroutines are asleep - deadlock! 如果没有关闭则会出现死锁
	}
}
```

==>知道那个协程能够做完时就使用管道去关闭它

### 2.4<font color='red'> **goroutine与channel协同工作**</font>

* **阻塞**:由于某种原因数据没有到达,当前协程(线程)处于持续等待状态,直到条件满足才解除阻塞
* **同步**:在两个或者多个协程(线程)间,保持数据内容的一致性的机制

#### 1.读写阻塞通信

基本的使用:打印机示例

这样的方式打印出来才是有顺序的“Hello World”

```go
func main()  {

	// 由于channel在此案例中本身不作为共享资源访问，所以对于chan是什么类型没有影响
	// 其仅仅作为协程通信的载体
  // 设置无缓冲通道
	var channel = make(chan bool)

	go func() {
		PrintWord("Hello")
		// 打印完之后写入channel, 实现channel写方向上的"畅通"
		channel <- true
	}()

	go func() {
		// 始终读取channel，实现channel读方向上的始终"畅通"
		// 注意：只有当channel读和写双方向的畅通才能够通信，当没有数据写入channel时，这时此协程一直在此阻塞
		<- channel
		PrintWord("World")
	}()

	// 为了不让主线程立即结束
	for {
		;
	}
}

// 打印
func PrintWord(s string)  {
	for _, c := range s {
		fmt.Printf("%c", c)
		time.Sleep(1500 * time.Millisecond)
	}
}
```

原理:**channel有两个端: 1.写端(传入端) 2. 读端(输出端), 要求:<font color='red'>读与写必须同时满足,才可以在channel上实现数据的流动,否则会导致go程的阻塞    </font>**

![J9PbsW](http://xwjpics.gumptlu.work/qinniu_uPic/J9PbsW.png)

#### 2.同步传递数据

在第一种读写阻塞的方式上通过channel在线程之间传递数据

```go
func main()  {
	ch := make(chan int)
	go func(num int) {
		var sum int
		for i:=1; i<=num; i++ {
			fmt.Println(i)
			sum += i
		}
		// 计算结束后写入channel，共享给其他go程
		ch <- sum
	}(3)
	//不断读取结果，没有写入时一直阻塞
	sum := <- ch
	fmt.Println("go程的计算结果为：", sum)		//6
}
```

#### 3.无缓存channel

上面两种方式都是无缓冲的channel,不设置len和cap,即通**道容量为0**

一般应用于两个go程中,一方读另一方写,==<font color='red'> **具备<u>同步</u>的能力,需要读/写go程同时在线(类似于打电话)**   </font>==

```go
func main() {
	// 创建无缓冲channel
	ch := make(chan int)
	fmt.Println("len(ch) = ", len(ch), "cap(ch) = ", cap(ch))	// len和cap始终为0
	go func() {
		for i:=0; i<10; i++ {
			// 写入后，如果主进程一直未读取那么就写入就会阻塞
			ch <- i
			// stdout的io操作（很耗时）可能会导致更换时间分片，所以打印的顺序可能不符合过程
			fmt.Println("子go程写入：", i)
			fmt.Println("len(ch) = ", len(ch), "cap(ch) = ", cap(ch))
		}
	}()

	time.Sleep(1 * time.Second)

	for i:=0; i<10; i++ {
		fmt.Println("主进程读出： ", <-ch)
	}
}
```

##### 无缓冲channel的存储优化

只针对于用于**通信的channel**

创建channel时可以使用空数组或者空数据结构来减少通道元素赋值的元素开销:

**空数组方式：**

```go
// 空数组方式
var c1 = make(chan [0]int)
go func() {
  time.Sleep(2 * time.Second)
  fmt.Println(c1)
  c1 <- [0]int{}  	// 一旦写入，主进程就结束
}()
<- c1
```

**空结构体方式（推荐）：**

```go
// 空结构体方式
var c1 = make(chan struct{})
go func() {
  time.Sleep(2 * time.Second)
  fmt.Println(c1)
  c1 <- struct{}{}  	// 一旦写入，主进程就结束
}()
<- c1
```

#### 4.有缓冲的channel

通道容量>0, ==**<font color='red'> 缓冲区可以进行数据存储.存储达到容量上限才会阻塞,具备<u>异步</u>的能力,不需要同步操作channel缓冲区(类似于发短信)</font>**==

```go
func main() {
	// 创建有缓冲区的channel， cap为3
	ch := make(chan int, 6)
	fmt.Println("len(ch) = ", len(ch), "cap(ch) = ", cap(ch))	
	go func() {
		for i:=0; i<10; i++ {
			// 写入后，如果达到缓冲区的极限即cap的大小，就会写入阻塞
			ch <- i
			// stdout的io操作（很耗时）可能会导致更换时间分片，所以打印的顺序可能不符合过程
			fmt.Println("子go程写入：", i)
			fmt.Println("len(ch) = ", len(ch), "cap(ch) = ", cap(ch))		// len不断的改变
		}
	}()

	time.Sleep(1 * time.Second)

	for i:=0; i<10; i++ {
		fmt.Println("主进程读出： ", <-ch)
	}

}
```

#### 5.关闭channel

当**==协程通信的双方不知道对方channel中数据大小/容量时==**,就需要使用关闭channel

* 基本使用

  使用内置函数**close** 可以关闭channel, <font color='green'>当channel 关闭后，就不能再向channel 写数据了</font>，但是<font color='green'>仍然可以从该channel 读取数据</font>

  简单示例:

  ```go
  func main() {
    intChan := make(chan int, 20)
    intChan<- 1
    intChan<- 44
    intChan<- 56
    intChan<- 56
    intChan<- 46
    intChan<- 7
    intChan<- 6
    //读出数据
    fmt.Println(<-intChan)
    close(intChan)			//panic: send on closed channel
    intChan<- 55  			//写入
    fmt.Println(<-intChan)	//读取
  }
  ```

  ==**关闭可以看做在管道最后加入一个“EOF”标志**，**能够防止死锁**==：当管道中的最后一个元素被取出去，下一次管道中已经没有值了，协程只能死死的等待取管道的值从而陷入死锁。

* 判断是否关闭

  ```go
  func main() {
  	ch := make(chan int, 10)
  	go func() {
  		for i:=0; i<10; i++ {
  			ch <- i
  		}
  		// 写入完毕
  		close(ch)
  	}()
  
  	for true {
  		if num, ok := <- ch; ok {
  			fmt.Println("读取到数据", num)
  		}else {
        fmt.Println(<-ch)		// 0
  			fmt.Println("channel关闭！")
  			break
  		}
  	}
  }
  ```

* 总结
  1. 数据确定发送完,再关闭
  1. **注意第二个参数true表示没关闭，false表示关闭**
  2. 已关闭的channel不能再写入数据,可以读取数据
  3. <font color='red'> 当channel写端关闭后,即使channel数据已经被读取完,后面读端就会一直读到0(不会阻塞),这也代表着写端已经关闭了 </font>

#### 6.一个案例

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123215651.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123224127.png)

<font color='red'>**重点！！！**</font>（这是一种方法，必须将管道关闭，后面还有select的方法）

```go
func WriteData(intChannel chan int)  {
	for i:=0; i<50; i++ {
		intChannel <- i //因为管道是线程安全的，不许要加锁就可以多协程共享读取
		fmt.Println("写入数据 ", i)
		//time.Sleep(time.Second * 1) 	//【非必须】等待一下不然写入太快看不到交叉
	}
	//写完了之后就关闭数据管道（打上结束标记）,因为读取协程一直在阻塞读取，等待结束
	close(intChannel)
}

func ReadData(intChannel chan int, overChannel chan bool) {
	for {
		v, ok := <-intChannel	//读取管道数据
		//当管道数据被读完，就阻塞，直到读取到结束标记(返回的ok表示当前管道是否被关闭了,如果是关闭了那么就返回false)
		if !ok {
			break
		}
		fmt.Println("读取数据 ", v)
		//time.Sleep(time.Second * 1)		//【非必须】等待一下
	}
	//读取完毕了就写入结束标志，并关闭
	overChannel<-true
	close(overChannel)
}

func main() {
	//创建两个channel
	var intChannel = make(chan int, 50)			//数据管道
	var overChannel = make(chan bool, 1)		//创建一个新的channel储存读完标记
	//写和读是两个线程，是交互、并行的
	go WriteData(intChannel) 				//创建一个协程写文件进管道
	go ReadData(intChannel, overChannel)	//创建一个协程读取文件
	//主进程不等待上面的两个协程运行，继续跑！
	//主进程一直阻塞读取该channel,直到读取到值并且为true则代表go协程跑完了，那么主进程就可以继续了
	for {
		res, ok := <-overChannel
		if !ok {
			break
		}
		fmt.Println("等待goroutine中。。。。。", res)	//读到true时会输出
	}
	//要分析一下：
	//if !<-overChannel{
	//	fmt.Println("等待goroutine中。。。。。")
	//}
	fmt.Println("主进程结束！")
}

```

### 2.5 单向channel

#### 1.基本性质

* channel 可以声明为**只读**，或者**只写**性质

  只写：var intChan chan<- int

  只读：var intChan <-chan int

  应用: 在只需要写/读的函数参数中加入此声明, 这样可以防止**误操作**，并且效率更加高

  注意：不管是读还是写，管道的类型仍然还是chan int， **只读只写只是一种属性**
  
  ![MQhiYy](http://xwjpics.gumptlu.work/qinniu_uPic/MQhiYy.png)
  
* 单向channel(只读/只写channel)与双向channel的转换

  1. 双向channel可以隐性的转换为任何一种单向channel (直接赋值即可转换)

  2. 单向channel不可以向双向channel转换

  3. 在传递参数时,单纯的写/读函数可以在函数参数中指定好为单向channel<font color='green'>**(良好的语法习惯)**    </font>,调用时可以使用双向channel

     <img src="http://xwjpics.gumptlu.work/qinniu_uPic/zrVCh0.png" alt="zrVCh0" style="zoom:50%;" />

#### 2.生产者消费者模型

![gNPYKk](http://xwjpics.gumptlu.work/qinniu_uPic/gNPYKk.png)

> ==公共区/缓冲区的作用:==  
>
> 1. 解耦合,降低生产者与消费者之间的耦合度,两者不需要直接的关联
> 2. 提高并发能力, 生产者消费者数量不对等时,依然能够保持通信
> 3. 缓存, 生产者、消费者数据处理速度不一致时,暂存数据
> 4. 异步通信, 双方不用同时在线  (如果是即是通信/同步通信的方式,则不需要缓冲区, **根据业务场景选择**)

```go
func Produce(ch chan <- int)  {
	for i:=0; i<10; i++ {
		ch <- i
	}
	close(ch)
}

func Consume(ch <- chan int)  {
	for {
		if num, ok := <- ch; ok {
			fmt.Println("消费", num)
		}else {
			break
		}

	}
}
func main() {
	ch := make(chan int)		// 无缓冲同步通信, 有缓冲异步通信
	go Produce(ch)
	go Consume(ch)
	for true {
		;
	}
}
```

### 2.6 定时器

#### 1.timer

![xS8MjJ](http://xwjpics.gumptlu.work/qinniu_uPic/xS8MjJ.png)

时间等待的三种方式:

```go
func main() {
	// 1. sleep
	fmt.Println("当前时间：", time.Now())
	time.Sleep(2 * time.Second)
	fmt.Println("现在时间：", time.Now())

	// 2. timer
	fmt.Println("当前时间：", time.Now())
	// 创建一个定时器，并设定等待时间
	// 到时间操作系统会将数据写入到timer的channel中
	timer := time.NewTimer(2 * time.Second)
	<- timer.C	// 读
	fmt.Println("现在时间：", time.Now())

	// 3. after
	fmt.Println("当前时间：", time.Now())
	// After直接返回channel
	afterTime := time.After(2 * time.Second)
	<- afterTime
	fmt.Println("现在时间：", time.Now())
}
```

#### 2.停止与重置

* 停止

  ```go
  func main() {
  	// 创建一个定时器
  	fmt.Println("当前时间：", time.Now())
  	timer := time.NewTimer(3 * time.Second)
  	go func() {
  		<- timer.C
  		fmt.Println("协程结束")
  		fmt.Println("现在时间：", time.Now())
  	}()
  
  	// 停止定时器
  	timer.Stop()  // 注意，当停止时读取端不会阻塞，因为写端是操作系统，一直存在只是暂停写入
  	for {
  		;
  	}
  }
  ```

* 重置

  ```go
  func main() {
  	// 创建一个定时器
  	fmt.Println("当前时间：", time.Now())
  	timer := time.NewTimer(3 * time.Second)
  	timer.Reset(1 * time.Second)		//重置时长
  	go func() {
  		<- timer.C
  		fmt.Println("协程结束")
  		fmt.Println("现在时间：", time.Now())
  	}()
  	for {
  		;
  	}
  }
  ```

#### 3.ticker周期定时

```go
// A Ticker holds a channel that delivers ``ticks'' of a clock
// at intervals.
type Ticker struct {
	C <-chan Time // The channel on which the ticks are delivered.
	r runtimeTimer
}
```

```go
func main() {
	ticker := time.NewTicker(time.Second)   // 系统每隔一秒向channel中写入
	fmt.Println("开始时间：", time.Now())
	go func() {
		// 循环的读出
		for {
			<- ticker.C
			fmt.Println("现在时间：", time.Now())
		}
	}()
	for {
		;
	}
}
```

### 2.7 select

#### 基本使用

go语言提供select关键字,==**来监听多个channel上的数据流动**==

语法与switch类似,但是不同的是<font color='red'>**select每一个case语句中必须是一个IO操作(channel可以看作是一种特殊的文件)**    </font>

![UTc4Be](http://xwjpics.gumptlu.work/qinniu_uPic/UTc4Be.png)

>  特点:
>
> 1. 当任意数量的case都同时可以执行(都不阻塞了),那么select会在可执行的case中**任意选择一条来使用**
> 2. 如果case全部阻塞,那么就有以下两种情况:
>    * 给出了default语句,那么就会**执行default**,同时程序从select语句后的语句中**恢复/继续**
>    * 没有给出default,那么select整个语句将会**阻塞**,直到有一个通信可以执行下去
> 3. <font color='green'> 一般select不会写default(会产生忙轮询), 会让select阻塞等待其中的case</font>
> 4. select**本身不循环**,所以一般配合外层for使用
> 5. case中的break只能跳出当前case,外层for无法跳出

基本使用:

```go
func main() {
	ch := make(chan int)			// 用于通信
	quit := make(chan bool)			// 用于判断结束

	go func() {
		for i:=0; i<5; i++{
			ch <- i
		}
		close(ch)
		quit <- true	// 写入结束
	}()

	exit:
	for {
		select {
		case num := <- ch:
			fmt.Println("读出：", num)
		case <-quit :
			fmt.Println("结束！")
			break exit
		}
	}
}
```

例2

```go
func main() {
	intChan := make(chan int, 10)
	stringChan := make(chan string, 5)
	for i:=0; i<10; i++ {
		intChan<- i
	}
	for i:=0; i<5; i++ {
		stringChan<- "哈喽" + fmt.Sprintf("%d", i)
	}

	exit:
	for{
		select {
			//注意：此处即使读完了管道内容，没有结束标记也不会报错死锁阻塞而是继续向下执行，这是select的特殊之处
			case i := <-intChan:
				fmt.Println("intChan", i)
			case s := <-stringChan:
				fmt.Println("stringChan", s)
			default:
				fmt.Println("两个管道都取完了！结束吧")
				break exit			//不加标记的break只是跳出select循环，所以要使用标记或者使用return
		}
	}
}
```

例3

```go
func run(done chan int) {
	for {
		select {
		case <-done:
			//一开始没有读出内容阻塞，一读出内容就结束了
			fmt.Println("exiting...")
			done <- 1		//写入，让阻塞的一方结束（主进程）
			break
		default:
		}

		time.Sleep(time.Second * 1)
		fmt.Println("do something")
	}
}

func main() {
	c := make(chan int)

	go run(c)

	fmt.Println("wait")
	time.Sleep(time.Second * 5)

	c <- 1		//写入就代表想让goroutine结束了
	<-c			//阻塞等待goroutine结束，一旦有值就结束

	fmt.Println("main exited")
}
```

#### 超时

当channel一端发生阻塞时, 另一端始终不继续,那么阻塞端需要设置最大的阻塞时间,以此来维护整个程序的运行,此时就需要使用超时.

```go
func main() {
	ch := make(chan int)			// 用于通信
	quit := make(chan bool)			// 用于判断结束


	// 读取数据
	go func() {
		for {
			select {
				case num := <- ch :
					fmt.Println(num)
				case <- time.After(3 * time.Second):		// 阻塞达到5秒就结束
					fmt.Println("超过三秒阻塞")
					quit <- true
					runtime.Goexit()
			}
		}
	}()

	// 写入数据
	go func() {
		// 写完两次就不写了，会导致读方阻塞
		for i:=0; i<2; i++{
			ch <- i
			time.Sleep(2 * time.Second)
		}
	}()

	<- quit
	fmt.Println("结束")
}
```

### 2.8 传统锁

channel内部实现了锁机制,所以不需要使用传统的锁机制.但是go语言也提供了传统的锁机制.

**注意:在go语言中尽量不要将channel与传统的锁机制一起使用 (可能会产生隐性死锁,即channel与互斥/读写锁产生交叉死锁(见下图))**

![ufddOm](http://xwjpics.gumptlu.work/qinniu_uPic/ufddOm.png)

#### 1.死锁

**死锁不是一种锁**,而是锁导致的一种现象

1. 单go程自己死锁

   <font color='green'>**无缓冲**的Channel因为需要同步通信,所以应该在两个以上的go程中通信,否则发生死锁.    (有缓冲的异步通信的单go程则不会阻塞)</font>

   ```go
   func main() {
   	ch := make(chan int)	//无缓冲channel, 如果是有缓冲的channel则可以
   	ch <- 1			//deadlock! 此时读端没有，所以写端就一直在此阻塞
   	num := <- ch
   	fmt.Println(num)
   }
   ```

2. go程间channel访问顺序导致死锁

   只读出导致了死锁,没有写入方(被阻塞了), 不管channel带不带缓冲区都会阻塞

   <font color='green'>使用channel一端读要保证另一端写操作, **两者同时有机会执行**,否则死锁    </font>

   ```go
   func main() {
   	ch := make(chan int)
   	num := <- ch			//deadlock! 读出数据，但是没有写入，所以此处就一直阻塞，下面的子go程都没有开启
   	fmt.Println(num)
   	go func() {
   		ch <- 123
   	}()
   }
   ```

3. 多go程,多channel交叉死锁

   单独简单的代码可以容易看出死锁,但是大环境下则比较难看出

   go_1程掌握M的同时尝试拿N,go_2程掌握N的同时拿M

   ```go
   func main() {
     ch1 := make(chan int, 6)
     ch2 := make(chan int, 6)
     go func() {
       for {
         select {
           case num := <- ch1:
           ch2 <- num
         }
       }
     }()
   
   	for {
   		select {
   		case num := <- ch2:
   			ch1 <- num
   		}
   	}
   }
   ```

#### 2.互斥锁

如果多个go程需要**同时对一共享资源进行访问与<u>修改</u>**,那么就需要使用互斥锁来规定访问资源的顺序

**==原理:一般锁只有一把,当有go程在访问已经加锁的资源时,该进程会阻塞等待锁的释放/解锁后,再处理共享资源.以此来规定访问共享资源的顺序==**

使用传统锁机制的打印机函数

使用: `sync.Mutex`

```go
var mutex sync.Mutex			//创建一个互斥锁,go程之间只有一把

func PrintWords(s string)  {
	mutex.Lock()				//访问共享数据s之前加上锁
	for _, c := range s {
		fmt.Printf("%c", c)
		time.Sleep(time.Second)
	}
	mutex.Unlock()				//访问共享数据s之后解锁
}

func Print1()  {
	PrintWords("Hello")
}

func Print2()  {
	PrintWords("World")
}
func main() {

	go Print1()
	go Print2()
	for {
		;
	}
}
```

互斥锁是一种建议锁, 由操作系统提供,建议在编程中使用,由编程手动写的锁都是建议锁

#### 3.读写锁

![6KqgDH](http://xwjpics.gumptlu.work/qinniu_uPic/6KqgDH.png)

使用:`sync.RWMutex`

 同一把锁,与互斥锁不同的是,**将其属性分成了读锁与写锁,**分别都有对应的加锁与解锁

**注意: **

1. **因为同一把锁,所以读加锁也会导致写加锁的阻塞,但是读之间是共享的**

2. **==读时所有读操作共享,写时单go程独占(读写不能同时),同情况下写锁的优先级比读锁高==**

3. **如果读操作拿到了锁,那么此时尽管写锁的优先级高但是也会阻塞,等待读锁解锁**

4. **加入读锁的意义在于当抢夺到了锁之后,不允许写操作的进行(也即不允许写锁的获得)**

5. **channel无法实现<u>读锁这样的共享</u>,所以看实际情况选择读写锁或者channel**

```go
var value int // 全局变量，模拟共享资源
var mutex1 sync.RWMutex	// 创建一把读写锁，分别有读和写两个属性

func Read(idx int)  {
	for {
		mutex1.RLock() 	// 读锁上锁
		fmt.Printf("----%dth 读取: %d\n", idx, value)
		mutex1.RUnlock()	// 读锁解锁
	}

}

func Write(idx int)  {
	for {
		mutex1.Lock() 	// 写锁上锁
		value = rand.Intn(1000)		// 写操作
		fmt.Printf("%dth 写入: %d\n", idx, value)
		mutex1.Unlock()	// 写锁解锁
	}
}


func main() {
	rand.Seed(time.Now().UnixNano())	// 创建随机数种子
	for i:=0; i<5; i++ {
		go Read(i)
	}
	for i:=0; i<5; i++ {
		go Write(i)
	}
	for {
		;
	}
}
```

### 2.9.条件变量

#### 基本概念

本身不是锁,但是经常与锁结合使用.

条件变量是在生产者、消费者对公共共享访问区访问加锁之==**前**==进行的判断,<font color='#39b54a'>**为了防止公共缓冲区已满或者全空而导致的即使抢到了锁也需要等待对端的阻塞状况.**</font>

> <font color='#e54d42'>为了避免在公共区数据已满或者为空情况下，各个读写goroutine盲目的抢占锁但是什么也做不了，条件变量就是体现当前公共区是值得开始抢占锁的一个变量</font>

使用步骤:

	1. 判断条件变量
	2. 加锁
	3. 访问
	4. 解锁
	5. 唤醒阻塞在条件变量上的对象

![90OqIH](http://xwjpics.gumptlu.work/qinniu_uPic/90OqIH.png)

#### 使用

`synv.Cond`类型代表了条件变量, **条件变量与锁一起使用**

```go
// Each Cond has an associated Locker L (often a *Mutex or *RWMutex),
// which must be held when changing the condition and
// when calling the Wait method.
//
// A Cond must not be copied after first use.
type Cond struct {
	noCopy noCopy

	// L is held while observing or changing the condition
	L Locker

	notify  notifyList
	checker copyChecker
}
```

条件变量本身不是锁,但是其内部属性包括锁`L Locker`,<font color='#e54d42'>所以在使用的时候要指定其中的锁</font>

 常用方法:Wait、Signal、Broadcast

wait函数注意：

* “阻塞等待条件变量满足”的意思是：**让当前协程阻塞，因为当前公共访问区内的数据已满或者已空**

* “释放互斥锁”就是解锁操作，所以我们在**调用wait函数之前需要先加锁**

  > <font color='#e54d42'>为什么要解锁？因为当前公共区域全满或为空，如果占用着锁没有任何意义，应该放给对端去获取锁处理</font>

* “两步为一个原子操作”：以上a和b两步属于一个原子操作

* “被唤醒”：调用`Signal()`或者`Broadcast()`函数的时候，此时自己被唤醒，再对公共区进行操作

 <img src="http://xwjpics.gumptlu.work/qinniu_uPic/2cidMe.png" alt="2cidMe" style="zoom:50%;" />

使用流程:

一个go程(生产者/消费者)中:

 1. 创建 条件变量

 2. 创建/指定 条件变量中的锁

 3. 公共区加锁

 4. **循环判断**公共缓冲区是否达到条件

    * 满: len(common_channel) == cap(common_channel)
    * 空: len(common_channel) == 0

    Wait()成功代表着: 1.阻塞; 2. 解锁<font color='#39b54a'>(给对端用)</font>; 3.被唤醒后重新争夺锁,拿到后上锁

	5. 访问公共区,处理事务

	6. 解锁

	7. 唤醒阻塞在条件变量上的对端<font color='#39b54a'>(以唤醒的不用管)</font>

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

var  cond sync.Cond					// 创建条件变量

func main() {
	cond.L = new(sync.Mutex)		// 给条件变量指定锁
	common := make(chan int, 5)		// 创建公共区，容量为5
	rand.Seed(time.Now().UnixNano())
	// 创建五个生产者，五个消费者
	for i:=0; i<5; i++ {
		go Production(common, i)
	}

	for i:=0; i<5; i++ {
		go Consuming(common, i)
	}
	for {
		;
	}
}

// 生产
func Production(in chan <- int, idx int)  {
	for {
		// 公共区加锁
		cond.L.Lock()
		// 循环判断公共缓冲区是否达到条件
		for len(in) == cap(in) {
			cond.Wait()
		}
		// 访问公共区, 写数据
		num := rand.Intn(1000)
		in <- num
		fmt.Printf("%dth写入%d数据, 公共区剩余%d个数据\n", idx, num, len(in))
		// 解锁
		cond.L.Unlock()
		// 唤醒
		cond.Signal()
	}
}


// 消费
func Consuming(out <- chan int, idx int)  {
	for {
		// 公共区加锁
		cond.L.Lock()
		// 循环判断公共缓冲区是否达到条件
		for len(out) == 0 {
			cond.Wait()
		}
		// 访问公共区, 读数据
		fmt.Printf("---%dth读出%d数据, 公共区剩余%d个数据\n", idx, <- out, len(out))
		// 解锁
		cond.L.Unlock()
		// 唤醒
		cond.Signal()
	}
}
```

注意:

 1. <font color='#fbbd08'>在调用Wait函数之前,需要先给cond中的锁**加锁**,这样才能解锁 </font>

 2. Wait先解锁再加锁的原因:

    <font color='#e54d42'>先解锁,例如缓冲区已满写go程1拿到了锁,但是缓冲区满了需要将锁交给读go程去消费缓冲区数据,所以解除该锁.而在上锁,是指自己的阻塞状态就已经被对端提醒唤醒了,可以去争抢锁了(抢到了就是加锁)</font>

 3. 使用for循环wait的原因:

    <font color='#e54d42'>假设,多个写go程同时写已满的公共缓冲区时,第一个线程调用wait,唤醒其他的写go程,但是其他的写go程被唤醒后是**在wait语句之后继续运行的**,此时公共区还是满状态还是要阻塞.所以,为了防止这样的情况出现,要使用for</font>

    一般都采用for**循环**去判断条件变量并wait，这样的原因是如果用if，所有协程都在阻塞wait的时候，当被唤醒时，一个协程获得了锁写入了数据，而其他协程如果拿到锁之后也执行到了wait下面也要写入就会造成写入的阻塞（因为if只判断一次），所以要用循环判断
    
    ![C9wHbN](http://xwjpics.gumptlu.work/qinniu_uPic/C9wHbN.png)

### 2.9 <font color='red'>Waitgroup</font>

#### 基本知识

Waitgroup是多协程同步的第二种方式

sync包中的Waitgroup结构，是Go语言为我们提供的多个goroutine之间同步的好刀。下面是官方文档对它的描述：

```
A WaitGroup waits for a collection of goroutines to finish. The main goroutine calls Add to set the number of goroutines to wait for. 
Then each of the goroutines runs and calls Done when finished. At the same time, Wait can be used to block until all goroutines have finished.
```

通常情况下，我们像下面这样使用waitgroup:

1. 创建一个**Waitgroup**的实例，假设此处我们叫它wg
2. **在每个goroutine启动的时候，调用wg.Add(1)**，<font color='red'>**这个操作可以在goroutine启动之前调用，也可以在goroutine里面调用**</font>。当然，也可以在**创建n个goroutine前调用wg.Add(n)** <font color='green'>建议**在协程启动之前调用**，**不然主进程跑的太快太快可能导致协程还未启动就结束**</font>
3. 当**每个goroutine完成任务后，调用wg.Done()**
4. 在**等待所有goroutine的地方调用wg.Wait()**，<font color='red'>**它在所有执行了wg.Add(1)的goroutine都调用完wg.Done()前阻塞**</font>，**当所有goroutine都调用完wg.Done()之后它会返回**。

那么，如果我们的goroutine是一匹不知疲倦的牛，一直孜孜不倦地工作的话，<u>如何在主流程中告知并等待它退出呢</u>？像下面这样做：

```go
type Service struct {
	// Other things

	ch        chan bool
	waitGroup *sync.WaitGroup
}

func NewService() *Service {
	s := &Service{
		// Init Other things
		ch:        make(chan bool),
		waitGroup: &sync.WaitGroup{},
	}
	return s
}

func (s *Service) Stop() {
	close(s.ch)
	s.waitGroup.Wait()
}

func (s *Service) Serve() {
	s.waitGroup.Add(1)
	defer s.waitGroup.Done()

	for {
		select {
			case <-s.ch:
				fmt.Println("stopping...")
				return
			default:
		}
		s.waitGroup.Add(1)
		go s.anotherServer()
	}
}

func (s *Service) anotherServer() {
	defer s.waitGroup.Done()
	for {
		select {
			case <-s.ch:
				fmt.Println("stopping...")
				return
			default:
		}

	// Do something
	}
}

func main() {
	service := NewService()
	go service.Serve()
	// Handle SIGINT and SIGTERM.
	ch := make(chan os.Signal)
	signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
	fmt.Println(<-ch)
	// Stop the service gracefully.
	service.Stop()
}
```

#### waitgroup的两个大坑

1. 在goroutine函数中使用Add并且在末尾Done，但是却goroutine还是被主程序中断了

   原因：主进程执行的太快了，先wait了，协程还没有来得及跑

   修改：<font color='red'>**Add一般不要写在goroutine函数中**</font>

2. 在goroutine开始之前ADD，使用Waitgroup作为参数传递给goroutine函数，在函数末尾Done

   原因：创建时是：wg := sync.WaitGroup{}， 传参时是值拷贝，所以在函数中Done的只是你的wg副本

   修改：**<font color='red'>创建时要使用WaitGroup指针！</font>**

### 2.10 多go程练习

#### 练习1

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201124110338.png)

```go
package main

import (
	"fmt"
	"math/big"
	"sync"
)

//多协程求阶乘

const SUM_NUM  = 5000

type Server struct {
	NumChan  chan int  			//数据channel
	ResChan chan *big.Int		//存放阶乘大数
	Wg sync.WaitGroup			//实例化一个Waitgroup实例
}

func (s *Server) SetData()  {
	//添加进程ADD
	s.Wg.Add(1)
	defer s.Wg.Done()	//延迟结束

	for i:=1; i<=SUM_NUM; i++ {
		s.NumChan<- i
	}
	//写入完毕就关闭管道
	close(s.NumChan)
}


func (s *Server) Fun()  {
	//添加进程ADD
	s.Wg.Add(1)
	defer s.Wg.Done()	//延迟结束

	for  {
		num, ok := <- s.NumChan
		if !ok {
			//遇到结束标记就结束
			break
		}
		//计算
		mulRes := big.NewInt(int64(1))
		for i:=1; i<=num; i++ {
			mulRes.Mul(mulRes, big.NewInt(int64(i)))
		}
		//写入
		s.ResChan<- mulRes
	}
}

func (s *Server) Stop()  {
	//注意这里的顺序，要先让所有的数据处理协程算完后写入完毕后，再关闭管道
	s.Wg.Wait()			//让主进程等待所有协程结束
	close(s.ResChan)	//关闭运算结果管道
}


func main() {
	//实例化一个服务
	s := Server{
		NumChan: make(chan int, SUM_NUM),
		ResChan: make(chan *big.Int, SUM_NUM),
		Wg:      sync.WaitGroup{},
	}
	go s.SetData()
	//用8个协程去取出数处理
	for i:=1; i<=200; i++ {
		fmt.Println("开启协程 ", i)
		go s.Fun()
	}
	//结束所有协程
	s.Stop()
	//输出运算结果
	for v := range s.ResChan{
		fmt.Println(v)
	}
	fmt.Println("主进程结束！")
}

```

#### 练习2

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201124110052.png)

```go
package main

import (
	"bufio"
	"bytes"
	"encoding/binary"
	"fmt"
	"math/rand"
	"os"
	"sort"
	"strconv"
	"sync"
)

type Fileserver struct {
	DataChan chan int
	FileName string
	Wg sync.WaitGroup
}

func (s *Fileserver) WriteChanData()  {
	//预先关闭协程
	defer s.Wg.Done()
	//打开文件
	file, err := os.OpenFile(s.FileName, os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil {
		fmt.Println("文件打开错误！", err)
	}
	defer file.Close()
	writer := bufio.NewWriter(file)
	defer writer.Flush()
	//生成随机数
	for i:=1; i<=1000; i++ {
		//生成数据
		data := rand.Intn(1000)
		//写入管道
		s.DataChan<- data
		//写入文件
		_, err := writer.Write([]byte(strconv.Itoa(data) + " "))
		if err != nil {
			fmt.Println("写入失败！", err)
		}
	}
	//关闭当前的数据channel
	close(s.DataChan)
	fmt.Println(s.FileName, "文件写入完成")

}

//int转Bytes
func IntToBytes(n int) []byte {
	data := int64(n)
	bytebuf := bytes.NewBuffer([]byte{})
	binary.Write(bytebuf, binary.BigEndian, data)
	return bytebuf.Bytes()
}

//读取数据，并执行操作
func (s *Fileserver) ReadChanData()  {
	//预关闭进程
	defer s.Wg.Done()
	//添加协程任务
	s.Wg.Add(1)
	defer s.Wg.Done()
	//创建数组
	intAry := make([]int, 1000)
	for{
		//阻塞读取，保证写入完毕
		data, ok := <-s.DataChan
		if !ok {
			break
		}
		intAry = append(intAry, data)
	}
	//操作函数-排序
	sort.Ints(intAry)
	//重新写入文件
	file, err := os.OpenFile(s.FileName, os.O_WRONLY|os.O_TRUNC, 0666)
	defer file.Close()
	if err != nil {
		fmt.Println("打开文件失败!", err)
	}
	writer := bufio.NewWriter(file)
	defer writer.Flush()
	for v := range intAry{
		_, err := writer.Write([]byte(strconv.Itoa(v) + " "))
		if err != nil {
			fmt.Println("写文件失败！", err)
		}
	}
	fmt.Println(s.FileName, "排序后文件写入完毕！")
}

func main() {
	//创建10个任务
	for i:=1; i<=10; i++ {
		newFileServer := &Fileserver{
			DataChan: make(chan int, 1000),
			FileName: "../data/" + "file"  + strconv.Itoa(i) + ".txt",
			Wg:       sync.WaitGroup{},
		}
		//写入文件
		newFileServer.Wg.Add(1)
		go newFileServer.WriteChanData()
		//读取文件并排序写入
		newFileServer.Wg.Add(1)
		go newFileServer.ReadChanData()
		newFileServer.Wg.Wait()
	}

}
```

### 2.11 goroutine中panic的捕获

goroutine 中使用recover，解决协程中出现panic，导致程序崩溃问题

<font color='green'>**注意：一个主进程开启的多个协程中一个协程崩溃panic，那么所有其他协程都会崩溃**</font>

=> 解决方法：使用panic捕获 defer+ recover

```go
var wg = &sync.WaitGroup{}

func test1()  {
	defer wg.Done()
	for i:=0; i<10; i++ {
		fmt.Println(i)
	}

}

func test2()  {
	defer func() {
		if err := recover(); err!=nil {
			fmt.Println(err)
		}
	}()

	defer wg.Done()
	//使用defer + recover捕获


	var newMap map[int]string
	newMap[1] = "dsad"  // 错误：没有make就赋值

}

func main() {
	wg.Add(2)
	go test1()
	go test2()	//一但一个协程发生错误，所有的协程都会崩溃
	wg.Wait()
	fmt.Println("主进程结束！")
}
```

## 3. 反射

使用反射机制，编写函数的适配器, 桥连接

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128163004.png)

应用：反射开发框架

### 3.1 基本介绍

1)反射可以在**运行时**动态获取变量的各种信息, 比如变量的类型(type)，类别(kind)

2) 如果是结构体变量，还**可以获取到结构体本身的信息**(包括结构体的**字段、方法**)
3) **通过反射，可以修改变量的值，可以调用关联的方法**。
4) 使用反射，需要<u>import (“reflect”)</u>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128163307.png)

### 3.2 应用场景

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128164613.png)

### 3.3  反射函数与概念

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128165115.png)



<font color='red'>**变量**、**interface**{} 和**reflect**.**Value** 是可以相互转换的</font>，这点在实际开发中，会经常使用到。示意图：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128170416.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128170300.png)

* reflect.TypeOf()

  ```
  func TypeOf(i interface{}) Type
  ```

  返回值类型是reflect.Type是一个丰富的接口：type [Type]()

  ```
  type Type interface {
      // Kind返回该接口的具体分类
      Kind() Kind
      // Name返回该类型在自身包内的类型名，如果是未命名类型会返回""
      Name() string
      // PkgPath返回类型的包路径，即明确指定包的import路径，如"encoding/base64"
      // 如果类型为内建类型(string, error)或未命名类型(*T, struct{}, []int)，会返回""
      PkgPath() string
      // 返回类型的字符串表示。该字符串可能会使用短包名（如用base64代替"encoding/base64"）
      // 也不保证每个类型的字符串表示不同。如果要比较两个类型是否相等，请直接用Type类型比较。
      String() string
      // 返回要保存一个该类型的值需要多少字节；类似unsafe.Sizeof
      Size() uintptr
      // 返回当从内存中申请一个该类型值时，会对齐的字节数
      Align() int
      // 返回当该类型作为结构体的字段时，会对齐的字节数
      FieldAlign() int
      // 如果该类型实现了u代表的接口，会返回真
      Implements(u Type) bool
      // 如果该类型的值可以直接赋值给u代表的类型，返回真
      AssignableTo(u Type) bool
      // 如该类型的值可以转换为u代表的类型，返回真
      ConvertibleTo(u Type) bool
      // 返回该类型的字位数。如果该类型的Kind不是Int、Uint、Float或Complex，会panic
      Bits() int
      // 返回array类型的长度，如非数组类型将panic
      Len() int
      // 返回该类型的元素类型，如果该类型的Kind不是Array、Chan、Map、Ptr或Slice，会panic
      Elem() Type
      // 返回map类型的键的类型。如非映射类型将panic
      Key() Type
      // 返回一个channel类型的方向，如非通道类型将会panic
      ChanDir() ChanDir
      // 返回struct类型的字段数（匿名字段算作一个字段），如非结构体类型将panic
      NumField() int
      // 返回struct类型的第i个字段的类型，如非结构体或者i不在[0, NumField())内将会panic
      Field(i int) StructField
      // 返回索引序列指定的嵌套字段的类型，
      // 等价于用索引中每个值链式调用本方法，如非结构体将会panic
      FieldByIndex(index []int) StructField
      // 返回该类型名为name的字段（会查找匿名字段及其子字段），
      // 布尔值说明是否找到，如非结构体将panic
      FieldByName(name string) (StructField, bool)
      // 返回该类型第一个字段名满足函数match的字段，布尔值说明是否找到，如非结构体将会panic
      FieldByNameFunc(match func(string) bool) (StructField, bool)
      // 如果函数类型的最后一个输入参数是"..."形式的参数，IsVariadic返回真
      // 如果这样，t.In(t.NumIn() - 1)返回参数的隐式的实际类型（声明类型的切片）
      // 如非函数类型将panic
      IsVariadic() bool
      // 返回func类型的参数个数，如果不是函数，将会panic
      NumIn() int
      // 返回func类型的第i个参数的类型，如非函数或者i不在[0, NumIn())内将会panic
      In(i int) Type
      // 返回func类型的返回值个数，如果不是函数，将会panic
      NumOut() int
      // 返回func类型的第i个返回值的类型，如非函数或者i不在[0, NumOut())内将会panic
      Out(i int) Type
      // 返回该类型的方法集中方法的数目
      // 匿名字段的方法会被计算；主体类型的方法会屏蔽匿名字段的同名方法；
      // 匿名字段导致的歧义方法会滤除
      NumMethod() int
      // 返回该类型方法集中的第i个方法，i不在[0, NumMethod())范围内时，将导致panic
      // 对非接口类型T或*T，返回值的Type字段和Func字段描述方法的未绑定函数状态
      // 对接口类型，返回值的Type字段描述方法的签名，Func字段为nil
      Method(int) Method
      // 根据方法名返回该类型方法集中的方法，使用一个布尔值说明是否发现该方法
      // 对非接口类型T或*T，返回值的Type字段和Func字段描述方法的未绑定函数状态
      // 对接口类型，返回值的Type字段描述方法的签名，Func字段为nil
      MethodByName(string) (Method, bool)
      // 内含隐藏或非导出方法
  }
  ```

* reflect.ValueOf()

  ```
  func ValueOf(i interface{}) Value
  ```

  返回类型reflect.Value是一个<font color='red'>**空接口**</font>，但是有非常多的方法

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128200155.png)

### 3.4  使用

#### 3.4.1 三者转化案例

请编写一个案例，演示对**(基本数据类型、interface{}、reflect.Value)**进行反射的基本操作代码演示：

```go
//请编写一个案例，演示对(基本数据类型、interface{}、reflect.Value)进行反射的基本操作代码演示，见下面的表格：
func reflectTest01(b interface{}){		//使用interface是因为可以使用任意接口进行反射
	//通过反射拿到一系列信息type类型、kind类别、value值
	//1. 先获取到reflect.Type类型   接口转reflect.Type
	rType := reflect.TypeOf(b) 	//rType是反射的类型而不是参数的类型
	fmt.Println("rType =", rType)	//rType= int  注意这个int不是基本数据类型的int，他是int的接口，可以调取函数
	fmt.Printf("rType的类型是：%T\n", rType)	//rType的类型是：*reflect.rtyperVal =  15
	//2. 获取到reflect.Value		接口转reflect.Value
	rVal := reflect.ValueOf(b)
	fmt.Println("rVal = ", rVal)	//rVal =  15 注意这个15不是普通数字15，而是reflect.Value的结构体类型，不能做算术运算。
	// 输出是100是因为底层的机制输出代表的值
	fmt.Printf("rVal的类型是%T\n", rVal)		//rVal的类型是reflect.Value

	//3. reflect.Value转变量原本类型
	val := rVal.Int()
	fmt.Printf("变量的值是：%d, 变量的类型是:%T\n", val, val)	//变量的值是：15, 变量的类型是:int64
	//但是上方的做法如果传入的参数类型是int那么就会导致错误
	//所以就要使用断言的方法，见下面

	//4. 将reflect.Value转为interface
	iV := rVal.Interface()
	//利用断言转换成为需要的类型
	num2 := iV.(int)
	fmt.Println("num2 = ", num2)
}


func main() {
	var num int  = 15
	reflectTest01(num)
}
```

请编写一个案例，演示对**(结构体类型、interface{}、reflect.Value)**进行反射的基本操作

```go
type Rap struct {
	Name string
	Age int
}

//对结构体的反射案例
func reflectStruct(i interface{})  {
	//1. 获取reflect.Type
	rType := reflect.TypeOf(i)
	fmt.Println("rType : ", rType)		//rType :  *main.Rap
	//2. 获取reflect.Value
	rVal := reflect.ValueOf(i)
	fmt.Println("rVal : ", rVal)		//rVal :  &{Gai 30}
	//3. 结构体转换成为interface
	iV := rVal.Interface()
	fmt.Printf("iV=%v, iV的类型%T\n", iV, iV)//iV=&{Gai 30}, iV的类型*main.Rap
	//注意：这个类型是底层运行的时候自动识别出来的，编译器是无法知道其类型的
	//fmt.Println("rap的名字：", iV.Name)	//这样在编译器是无法知道类型的，所以也无法取其字段,这就是要先通过断言再取值的原因
	//4. 断言获取其值
	rap := iV.(Rap)
	fmt.Printf("rap=%v, rap的类型:%T, rap.Name=%v, rap.Age=%v\n", rap, rap, rap.Name, rap.Age)	// 此时就不会错了 rap={Gai 30}, rap的类型:main.Rap, rap.Name=Gai, rap.Age=30

	//5. 多个类型断言可以用switch，或者使用返回值判断断言是否正确
	if str, ok := iV.(string); !ok{
		fmt.Println("断言失败！", str)
	}
}


func main() {
	//1. 定义一个int类型的基本数据类型
	//var num int  = 15
	//reflectTest01(num)

	//2. 定义一个结构体数据类型
	newRap := Rap{
		Name: "Gai",
		Age:  30,
	}
	reflectStruct(newRap)
}
```

注意：<font color='red'>**反射是在代码<u>运行</u>过程中起作用的，断言是为了在<u>编译期</u>让编译器知道接口类型的**</font>

**反射是运行时的反射**

#### 3.4.2 Kind类别与Type类型

**Kind是类别范围比Type大**，例如：类型是自定义类型student，而他的类别就是Struct结构体

Type 是类型, Kind 是类别， **Type 和Kind 可能是相同的，也可能是不同的.**
比如: var num int = 10 num 的Type 是int , Kind 也是int
比如: var stu Student stu 的Type 是pkg1.Student , Kind 是struct

type [Kind](https://github.com/golang/go/blob/master/src/reflect/type.go?name=release#208)

```
type Kind uint
```

Kind代表Type类型值表示的具体分类。零值表示非法分类。

```go
const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    Int16
    Int32
    Int64
    Uint
    Uint8
    Uint16
    Uint32
    Uint64
    Uintptr
    Float32
    Float64
    Complex64
    Complex128
    Array
    Chan
    Func
    Interface
    Map
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)
```

反射中获取Kind：

**reflect.Type接口具有该函数和reflect.Value接口具有该方法，都可以获得Kind**

```go
func reflectTest02(b interface{}){

	rType := reflect.TypeOf(b) 	//rType是反射的类型而不是参数的类型

	rVal := reflect.ValueOf(b)

	kind1 := rType.Kind()
	kind2 := rVal.Kind()
	fmt.Println(kind1, kind2)	//int int

}
```

#### 3.4.3 反射修改变量

在改变之前注意：**想要修改有效果（修改到原值），那么一定函数要传递值的指针而不是本身**

既然是指针直接使用set等方法是不行的，需要使用的方法：

func (Value) [Elem](https://github.com/golang/go/blob/master/src/reflect/value.go?name=release#831)

```
func (v Value) Elem() Value
```

Elem返回**v持有的接口保管的值**的Value封装，或者<font color='red'>v持有的**指针指向**的值的Value封装</font>。如果v的Kind不是Interface或Ptr会panic；如果v持有的值为nil，会返回Value零值。

<font color='green'>理解：Elem（）方法可以理解为**类似于指针取数的*号**，如果reflect.Value的类别是Ptr（指针）的话都要使用此方法才能改值，否则报错</font>

```go
//通过反射修改student的值
func reflectTest03(i interface{})  {
	//rType := reflect.TypeOf(i)
	rVal := reflect.ValueOf(i)
	//输出rVal的类型
	fmt.Printf("rVal的类型是%T,rVal的类别是%v\n", rVal, rVal.Kind()) //rVal的类型是reflect.Value,rVal的类别是ptr(指针)
	//rVal的类别是指针类别
	//rVal.SetInt(20)   //直接这样写报错
	rVal.Elem().SetInt(25) //正确的写法
}


func main() {
	num := 15
	fmt.Println(num)	//15
	reflectTest03(&num)
	fmt.Println(num)	//25
}
```

#### 3.4.4 反射调取方法

核心的两个方法：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128204728.png)

<font color='red'>**注意1： Method的参数i是按方法名的ASCII排序下来的，并不是定义时的顺序**</font>

<font color='red'>**注意2： 调用函数Call方法，参数是reflect.Value类型的切片，返回值也是该类型的切片！所以可以说调用函数时的基本类型就是Value**</font>

其他要使用的方法：

* func (Value) [Field](https://github.com/golang/go/blob/master/src/reflect/value.go?name=release#866)

```
func (v Value) Field(i int) Value
```

​	返回结构体的第i个字段（的Value封装）。如果v的Kind不是Struct或i出界会panic

* func (Value) [NumField](https://github.com/golang/go/blob/master/src/reflect/value.go?name=release#1320)

```
func (v Value) NumField() int
```

返回v持有的结构体类型值的字段数，如果v的Kind不是Struct会panic

* 结构体字段标签相关

```
type Type interface {
	......
    Field(i int) StructField
        // 返回索引序列指定的嵌套字段的类型，
        // 等价于用索引中每个值链式调用本方法，如非结构体将会panic
    ......
}
```

type [StructField](https://github.com/golang/go/blob/master/src/reflect/type.go?name=release#734)

```
type StructField struct {
    // Name是字段的名字。PkgPath是非导出字段的包路径，对导出字段该字段为""。
    // 参见http://golang.org/ref/spec#Uniqueness_of_identifiers
    Name    string
    PkgPath string
    Type      Type      // 字段的类型
    Tag       StructTag // 字段的标签
    Offset    uintptr   // 字段在结构体中的字节偏移量
    Index     []int     // 用于Type.FieldByIndex时的索引切片
    Anonymous bool      // 是否匿名字段
}
```

StructField类型描述结构体中的一个字段的信息。

func (StructTag) [Get](https://github.com/golang/go/blob/master/src/reflect/type.go?name=release#763)

```
func (tag StructTag) Get(key string) string
```

Get方法返回标签字符串中键key对应的值。如果标签中没有该键，会返回""。如果标签不符合标准格式，Get的返回值是不确定的。

案例 ： **使用反射来遍历结构体的字段，调用结构体的方法，并获取结构体标签的值**

```go
type Teacher struct {
	Name string	`json:"name"`
	Age int		`json:"age"`
	ID int		`json:"id"`
	Classroom int	`json:"classroom"`
	Salary float64	`json:"salary"`
}

//结构体的三个方法

func (t Teacher) Print()  {
	fmt.Println("-------Start--------")
	fmt.Println(t)
	fmt.Println("-------End--------")
}


func (t Teacher) GetSum(i, j int) int {
	return i + j
}

func (t Teacher) Set(name string, age int, id int, classroom int, salary float64)  {
	t.Name = name
	t.Age = age
	t.ID = id
	t.Classroom = classroom
	t.Salary = salary
}


//反射方法
//遍历结构体的字段，调用结构体的方法，并获取结构体标签的值
func Reflect(i interface{})  {
	typeOf := reflect.TypeOf(i)
	valueOf := reflect.ValueOf(i)
	kd := valueOf.Kind()

	//1. 判断是否是结构体  注意判断时要用reflect包使用
	if kd != reflect.Struct {
		fmt.Println("不是个结构体！")
		return
	}
	//2. 获取结构体字段总数
	sum := valueOf.NumField()
	//3. 遍历字段
	for i:=0; i < sum; i++ {
		fmt.Printf("字段%d: %v\n", i, valueOf.Field(i))
		//获取struct的tag字段
		tagVal := typeOf.Field(i).Tag.Get("json")	//注意json可以不是写死的，但是反序列化写死是json，所以一般不改
		if tagVal != "" {
			fmt.Printf("字段%d的标签为：%v\n", i, tagVal)
		}
	}

	//4. 获取到有多少方法
	Msum := valueOf.NumMethod()
	fmt.Printf("一共有%d个方法", Msum)
	//5. 获取到第二个方法 并执行  调用的是Print方法
	valueOf.Method(1).Call(nil)

	//6. 调用多参数方法
	var valueSlice []reflect.Value	//创建Value切片
	valueSlice = append(valueSlice, reflect.ValueOf(10), reflect.ValueOf(20))	//加入到切片中
	values := valueOf.Method(0).Call(valueSlice)	// 调用
	res := values[0].Interface().(int)   //!!!!难点；断言结果类型， 记得返回值也是一个切片，所以取下标；然后需要转换成为interface才能断言
	fmt.Printf("函数%s的运行结果是:%d\n", typeOf.Method(0).Name, res)

}


func main() {
	newTeacher := Teacher{
		Name:      "马保国",
		Age:       69,
		ID:        012,
		Classroom: 10,
		Salary:    100000,
	}
	Reflect(newTeacher)
}
```

#### 3.4.5 反射修改结构体字段案例

核心思路：<font color='red'>**传递指针，并且在反射函数中使用Elem(), 不然会报错**</font>（如果是方法则可以直接调用，这类似于*p.fun() 与 p.fun() 是相同的，来自于Go底层的优化解析）

```go
import (
	"fmt"
	"reflect"
)

//使用反射的方式来获取结构体的tag 标签, 遍历字段的值，修改字段值，调用结构体方法


type Monster struct {
	Name string	`json:"name"`
	Age int		`json:"age"`
	ID int		`json:"id"`
	Classroom int	`json:"classroom"`
	Salary float64	`json:"salary"`
}

//结构体的三个方法

func (t Monster) Print()  {
	fmt.Println("-------Start--------")
	fmt.Println(t)
	fmt.Println("-------End--------")
}


func (t Monster) GetSum(i, j int) int {
	return i + j
}

func (t Monster) Set(name string, age int, id int, classroom int, salary float64)  {
	t.Name = name
	t.Age = age
	t.ID = id
	t.Classroom = classroom
	t.Salary = salary
}


//反射方法
//遍历结构体的字段，调用结构体的方法，并获取结构体标签的值
func Reflect02(i interface{})  {
	typeOf := reflect.TypeOf(i)
	valueOf := reflect.ValueOf(i)
	kd := valueOf.Kind()

	//1. 判断是否是结构体指针  注意判断时要用reflect包使用
	if kd != reflect.Ptr {
		fmt.Println("不是个结构体！")
		return
	}
	//2. 获取结构体字段总数
	sum := valueOf.Elem().NumField()		//注意要加上ELem()
	//3. 遍历字段
	for i:=0; i < sum; i++ {
		fmt.Printf("字段%d: %v\n", i, valueOf.Elem().Field(i))
		//获取struct的tag字段
		tagVal := typeOf.Elem().Field(i).Tag.Get("json")	//注意json可以不是写死的，但是反序列化写死是json，所以一般不改
		if tagVal != "" {
			fmt.Printf("字段%d的标签为：%v\n", i, tagVal)
		}
	}

	//4. 获取到有多少方法
	Msum := valueOf.Elem().NumMethod()
	fmt.Printf("一共有%d个方法", Msum)
	//5. 获取到第二个方法 并执行  调用的是Print方法
	valueOf.Method(1).Call(nil)

	//6. 调用多参数方法
	var valueSlice []reflect.Value	//创建Value切片
	valueSlice = append(valueSlice, reflect.ValueOf(10), reflect.ValueOf(20))	//加入到切片中
	values := valueOf.Method(0).Call(valueSlice)	// 调用
	res := values[0].Interface().(int)   //!!!!难点；断言结果类型， 记得返回值也是一个切片，所以取下标；然后需要转换成为interface才能断言
	fmt.Printf("函数%s的运行结果是:%d\n", typeOf.Method(0).Name, res)


	//7. 修改字段值
	age := valueOf.Elem().FieldByName("Age")
	fmt.Println("原本的Age字段值:", age)		//69
	valueOf.Elem().FieldByName("Age").SetInt(99) 	//重新设置值
	fmt.Println("现在的Age字段值:", age)		//99
}


func main() {
	newTeacher := &Monster{
		Name:      "马保国",
		Age:       69,
		ID:        012,
		Classroom: 10,
		Salary:    100000,
	}
	Reflect02(newTeacher) 	//注意此处传递的是指针
	fmt.Println("外部输出的字段值：", newTeacher.Age)	//99
}
```

<font color='green'>**反射的理解：以前结构体执行方法是创建结构体实例执行方法，而通过反射则可以创建实例，交给反射，反射会帮你执行所有需要的步骤 ===>底层框架**</font>

很多的框架规范（比如函数名必须以xxx开头，都是因为反射机制在后面要求好的）

#### 3.4.6 函数适配器

定义了两个函数test1 和test2，定义一个适配器函数用作统一处理接口

```go
import (
	"reflect"
	"testing"
)

func TestFun(t *testing.T)  {
	//定义两个函数

	call1 := func(i, j int) {
		t.Log(i, j)
	}

	call2 := func(i, j int, s string) {
		t.Log(i, j, s)
	}

	//创建桥梁
	brige := func(call interface{}, args...interface{}) {
		//1. 获取参数个数
		n := len(args)
		//2. 创建参数Value切片
		argSlice := make([]reflect.Value, n)
		//添加进切片
		for i:=0; i<n; i++ {
			argSlice[i] = reflect.ValueOf(args[i])
		}
		//3. 执行函数
		fun := reflect.ValueOf(call)	//创建Value
		fun.Call(argSlice) 				//执行
	}

	//测试
	brige(call1, 1, 2)
	brige(call2, 3, 4, "哈哈哈")
}
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129104222.png)

可以基于此实现函数的**重写**功能

#### 3.4.7 反射操作任意结构体

```go
package test

import (
	"reflect"
	"testing"
)

type user struct {
	Name string
	Age int
}

func TestXxx(t *testing.T)  {
	newUser := &user{}		//注意创建的是指针
	nV := reflect.ValueOf(newUser)
	t.Log("nV的类别", nV.Kind())
	nV = nV.Elem()  		//关键的一步
	t.Log("nV的类别", nV.Kind())
	//修改值
	nV.FieldByName("Name").SetString("xwj")
	nV.FieldByName("Age").SetInt(18)
	t.Log("该结构体的值为", newUser)
}
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129105314.png)

#### 3.4.8 反射创建并操作结构体

**用到的方法：**

func [New](https://github.com/golang/go/blob/master/src/reflect/value.go?name=release#2302)

```
func New(typ Type) Value
```

New返回一个Value类型值，该值持有一个指向类型为typ的新申请的零值的指针，返回值的Type为PtrTo(typ)。

```go
type man struct {
	Name string
	Age int
}

func TestStruct(t *testing.T)  {
	model := &man{}  				//创建一个指向man空间的指针
	t.Logf("model指向的地址%p", model)	//0xc000004500
	mV := reflect.TypeOf(model)		//获取其Type
	t.Logf("mv的类型%T,类别%v", mV, mV.Kind())
	mV = mV.Elem()					//取这个指针指向的类别（之前本身的类别是指针）
	t.Logf("mv的类型%T,类别%v", mV, mV.Kind())

	//前面这些工作都是为了帮助newPtr创建该类型的Value
	//新建了一个man空间并指向但类型还是Value
	newPtr := reflect.New(mV)		//关键一步 New返回一个Value类型值，该值持有一个指向类型为typ的新申请的零值的指针
	t.Logf("newPtr的类型是%T, 类别是%v\n", newPtr, newPtr.Kind())

	//把newPtr的类型转换为*man, 并赋值给model，让model也指向该空间
	model = newPtr.Interface().(*man)
	t.Logf("model指向的地址%p", model)	//0xc000004580
	t.Logf("model的类型是%T, newPtr的类型是%T, 类别是%v\n", model, newPtr, newPtr.Kind())
	//修改值
	newPtr = newPtr.Elem() 	//Value转向其指向的结构体值
	newPtr.FieldByName("Name").SetString("xwj")
	newPtr.FieldByName("Age").SetInt(25)

	t.Logf("model:%v, mode.Name:%v\n", model, model.Name)
}

```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129122823.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129122811.png)

## 4. Go指针

指针能够操作内存，这是指针最大的优势之一。

go语言的指针介于java与C/C++之间，即没有像java那样取消了代码对指针的操作能力，也没有像C/C++那样滥用指针而造成安全性和可靠性的问题。

但是go语言的指针并没有达到C/C++那样的自由度。

**指针就是地址，指针变量就是存储变量地址的变量**

*p 叫做**解引用**也叫间接引用

### 1 函数运行的内存解析与栈帧

​	栈帧：用来给函数运行**提供内存空间**。取内存于stack上。当函数调用时，产生栈帧，当函数调用结束时，释放栈帧。

​	栈帧存储：1.局部变量 2.形参（与局部变量地位等同） 3. 内存字段描述值

​	==**一个函数就对应一个栈帧**== 

​	以32位的内存4G的空间为例

​	![OnTiL2](http://xwjpics.gumptlu.work/qinniu_uPic/OnTiL2.png)

**当函数调用完毕后,空间释放/回收其实就是指针的归位,但是其中的数据是不会清空删除的,也就是说系统会当作无效的旧数据覆盖处理**

![uYi8o2](http://xwjpics.gumptlu.work/qinniu_uPic/uYi8o2.png)

### 2 注意事项

#### 空指针与野指针

空指针：未被初始化的指针 var p *int 

野指针：被一片无效的空间地址初始化的指针 var p *int = 0x989798

#### 函数返回局部变量指针

go语言中函数返回局部变量的指针是安全的，函数结束后局部变量依然存在（因为go语言中局部变量是动态的生命周期，或者说发生了逃逸内存分配到了堆上）

```go
vap p = test()
func test() *int {
  a := 10
  return &a
}
fmt.Println(p)	// 每次调用test都会是一个新的值，即每次都创建了一个新的局部变量
```

### 3 变量的左值右值

变量在等式的左边时代表的含义是变量的存储空间 （存）

变量在等式的右边时代表的含义是变量的存储内容/值 （取）

```go
p := new(string)
*p = "hha"                //左值代表堆中的存储空间
fmt.Printf("%q\n", *p) //右值代表存储空间中的值，二者含义不同
```

### 4 指针的函数传参

传地址（引用）：将形参的地址值作为函数参数、返回值后传递

传值（数据）：将实参的值拷贝一份给形参

传引用：在A栈帧内部修改B栈帧中的变量值

==注意函数传参永远都是值传递==

* 传递值

![jjguqX](http://xwjpics.gumptlu.work/qinniu_uPic/jjguqX.png)

这种方法swap栈帧是在内部交换两个变量的值，所以无法修改，而==指针可以实现栈帧之间的数据修改==

*  传递指针

![oQnga5](http://xwjpics.gumptlu.work/qinniu_uPic/oQnga5.png)  

当swap栈帧结束时，此时main中的局部变量的值已经被修改了，所以凑效了。

### 5 内存申请释放 

![p1edob](http://xwjpics.gumptlu.work/qinniu_uPic/p1edob.png)

所以后来的java/python都有gc垃圾回收机制

堆空间看做无限大但是并不是能够无限的使用，我们在编程时==对于无用的指针需要设置为空nil，以此来提醒gc快速回收== 

## 5.内存与垃圾回收

### 1. 堆栈

* 堆：适合不可预知大小的内存分配，但是分配速度较慢，且可能会出现内存碎片。全局变量保存在堆上

* 栈：一次函数调用内部申请到的内存，它们会随着函数的返回把内存还给系统。特点是内存分配、操作都非常快，但是不适合分配大内存。临时变量一般保存在栈上

### 2. 逃逸分析

go语言中，**堆栈空间的分配**不依靠语法上的关键字**即由系统分配（编译器分析代码）而不是代码层面上的指定**（与java语言不同）

分配规则如下：

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

**逃逸分析**：分析一个变量是分配在栈上还是堆上

**为什么要逃逸分析**：让合适的变量存在合适的地方，节省GC内存回收的压力

**编译器逃逸分析原理：** 就是上面的分配原则，如果函数中的变量被多个引用就分配到堆上，Go中的变量只有在编译器可以证明在函数返回后不会再被引用的，才分配到栈上，其他情况下都是分配到堆上。

### 3. 查看是否逃逸

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

t逃逸我们可以理解，x逃逸是因为fmt输出语句的参数为空接口类型，编译期间很难确定其参数的具体类型，也会发生逃逸。

1. 堆上动态分配内存比栈上静态分配内存，开销大很多；
2. 变量分配在栈上需要能在编译期确定它的作用域，否则会分配到堆上；
3. Go编译器会在编译期对考察变量的作用域，并作一系列检查，如果**它的作用域**在运行期间对编译器一直是可知的，那么就会分配到栈上；

4. <font color='#e54d42'>**函数传递变量的指针可以避免复制值的开销，但是这种情况不是一定的，因为复制值的过程在栈上进行这个的开销远比变量逃逸后在堆上动态分配内存少的多**</font>

# Tips

## 1. 注意给结构体打tag的时候不要空格

（错误写法）``json: "name"``    :  后面不能有空格

正确写法：

```go
type Teacher struct {
	Name string	`json:"name"`
	Age int		`json:"age"`
	ID int		`json:"id"`
	Classroom int	`json:"classroom"`
	Salary float64	`json:"salary"`
}
```

