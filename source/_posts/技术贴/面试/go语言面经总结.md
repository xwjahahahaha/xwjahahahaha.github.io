---
title: go语言面经总结
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-10-31 14:02:43
---

[TOC]

<!-- more -->

面试要点：

* 由点到面，把一个点知道的内容展开描述；如果可以的话可以多与其他语言对比

# 一、基础语法细节

## 1. make与new的区别是什么

共同点：

* 都会根据参数分配内存空间

不同点：

* `new`

  * 只接受一个参数即类型（自定义类型也可以），一般类型都可以使用`new`创建（包括`slice`、`map`、`chan`）

    ```go
    func new(Type) *Type
    ```

  * 返回值：返回的是该类型对象的引用
  * 一般很少使用，可以用`类型{}`代替

* `make`

  * 只用于三种内置的数据结构(chan、map、slice)的内存创建，会在编译期变为对应的例如`makeSlice`、`makeChannel`等函数

    ```go
    func make(t Type, size ...IntegerType) Type
    ```

  * 返回值：返回的都是其对应类型对象，而不是指针
  * 使用make创建时可以给对应对象指定len或cap，这是`new`无法做到的（`new`初始化`len`、`cap`都为0）

## 2. slice的底层原理/切片的扩容规则

相比于数据是动态的，slice本身是一个数据结构，分为data、len、cap三个字段，data指向底层数组的一个起始位置，len表示切片元素的个数，cap是底层数组的全部大小。初始化的时候省略cap时会将cap设置与len一样。

**当向切片添加元素的时候如果元素个数超出了cap**，那么底层会重新分配一个**两倍**长度的数组并将原来的值复制过来。如果当前容量达到1024，那么则后续每次扩容1/4左右

`0,1,2,4,8,16,32,64,128,256,512,1024,1280,1696,2304`

## 2.函数传slice注意事项

slice作为函数传参数可能在函数中添加元素导致底层数组的扩大，所以函数中修改无效

两种解决方法：

1. 参数中传递切片的指针
2. 函数中return返回，调用的时候用原切片接收

## 3. 类型可比较问题

### interface可以比较吗？

接口基础：（圣经P142）

* 一个接口值分为两个部分：一个动态类型T和该动态类型所对应的动态值

  > go是静态语言，类型只是编译时的概念，类型不是一个值而是一个*类型描述符*，其描述了一个类的具体信息，包括类型的名称和所包含的方法

* 接口的动态类型T就是一个类型描述符

* 一个接口未初始化时，其T和V都是零值，均为nil
* 当将一个具体类型赋值给一个接口时，首先会将该类型隐式转换为该接口类型，然后将接口的动态类型设置为该类型的类型描述符，动态值设置该类型值的副本
* 接口调用方法的*动态分发*: 编译时无法知道接口的动态类型是什么，所以通过动态类型即动态描述符中拿到该类型的对应方法实现间接的调用，效果与该类型直接的调用该方法一致

接口比较：

* 接口可以直接使用`==`或`!=`比较

* 只有两种情况才会相同：

  * 两个接口都为零值，T和V都为nil

    >  注意一种情况：
    >
    > ```go
    > func main() {
    > 	var p *int = nil
    > 	var i interface{} = p
    > 	fmt.Println(i == p) // true
    > 	fmt.Println(p == nil) // true
    > 	fmt.Println(i == nil) // false  因为
    > }
    > ```

  * 两个接口的动态类型相同，动态值相同

### 可用直接用==比较

对于go语言中的基本类型都可以比较，例如`int、string、bool`等。

可以比较的必须是*同类型*：

* 如果数组的元素类型是基本类型，那么两个数组之间也可以比较，长度、数值相同才返回true。

* 同理，如果结构体的元素基本类型，那么这个结构体的实例之间也可以直接比较

* 指针可以比较，但是比较的是指针保存的地址，而不是指向的值

### 不可以用==比较

* *不同类型*一定不能比较

* 对于**slice切片、map、函数变量**都是不能够直接比较的, 需要比较的话需要使用for循环比较元素

> slice虽然可以通过比较指向底层数组的地址、len以及cap，但是slice是可以随时扩容的，所以这样的比较不具有意义，go团队不想在这耗费精力，所以go中的slice不可以直接比较
>
> 可以使用`reflect.DeepEqual()`比较

### 总结

* interface可以比较，只有都为零值或者T、V都相同才相同
* 类型一样才可比较，类型不同一定不能比较
* 基本类型可以直接比较，基于基本类型的数组、结构体也是可以直接比较
* **slice切片、map、函数变量**都不能够直接比较

## 4. go怎么从源码编译到二进制文件

### 1. 总体过程

go语言编译的总体过程：（可以通过`go build -n xxx.go`查看整个过程）

**创建临时目录 => 查找依赖信息 => 编译 => 连接 => 生成可执行文件 => 复制到当前项目目录下**

go语言执行run的过程：

**创建临时目录 => 查找依赖信息 => 编译 => 连接 => 生成可执行文件 => 执行**

### 2. 编译过程

四个阶段：

1. **词法和语法分析**

   用词法解释器将源文件中的字符串转换成为`Token`，然后将这些`Token`再通过语法分析器分析语法生成抽象语法树`AST`，一个`go`文件就对应一个`AST`

2. **类型检查**

   对于每个抽象语法树进行类型检查，保证没有类型错误。并且还会对一些函数进行拓展，比如make就会拓展成`runtime.makeslice`

3. **中间代码的生成**

   经过语法分析和类型检查之后就不会有语法错误和类型错误了，这时候编译器就会将项目中的抽象语法树的所有函数放入编译队列中，每个goroutine将从中获取需要的函数将抽象语法树转化为中间代码。

   这个过程中还会使用*静态单赋值SSA*进行代码的一个优化，例如取出无用变量、片段等

4. **机器码的生成**

   go源代码中有对应各个cpu指令架构的包，使用这些包生成不同的架构的机器码

   在编译的过程中也可以通过`GOARCH` ` GOOS`进行交叉编译

## 5. string转[]byte内存拷贝/多个string拼接更高效的方式

会发生类型拷贝。因为go语言中string类型是不可变的，而[]byte类型是可变的，所以string转换[]byte是需要内存拷贝的。（强类型转换都会发生底层数据的内存拷贝）

优化处理：

在底层转换两者，把`StringHeader`的地址转换为`SliceHeader`。可以使用go语言中的`unsafe`包完成这个操作

```go
/*StringHeader 是字符串在go的底层结构。*/
type StringHeader struct {
 Data uintptr
 Len  int
}
/*SliceHeader 是切片在go的底层结构。*/
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```

```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	s := "xxxwwwjjj"
	// 拿到s的地址
	sAddr := unsafe.Pointer(&s)
	// 把字符串s转换为底层StringHeader
	stringHeader := (*reflect.StringHeader)(sAddr)
	var sBytes []byte
	// 拿到sBytes的指针
	sBytesAddr := unsafe.Pointer(&sBytes)
	// 把字节数组转换为底层SliceHeader
	sBytesHeader := (*reflect.SliceHeader)(sBytesAddr)

	// 赋值操作
	sBytesHeader.Data = stringHeader.Data
	sBytesHeader.Len = stringHeader.Len
	sBytesHeader.Cap = stringHeader.Len

	fmt.Printf("s = %s, sBytes = %s\n", s, sBytes)
}
```

注意：这样不会发生字符串的底层拷贝，但是这样使用需要慎重，因为**字节数组不能再改动**！

多个string拼接更加高效的方式：

* `strings.join`
* `strings.Bulider`
* `bytes.buffer`

## 6. 静态类型语言与动态类型语言

* 静态语言的变量在编译时就要确定其类型，变量要申明其类型，也叫强类型语言(java、c/c++、go)，动态类型语言则在运行时确定类型，不用申明变量类型，也叫弱类型语言（python，js）
* 静态语言编译时会进行类型匹配检查，所以不能变量的类型要确定，转换要在代码中体现。动态语言的变量在执行期间是可以改变的，**与生俱来多态性质**
* 静态语言规则要求高，代码安全性好，但是编写会复杂一点。动态语言的代码更加简洁，可以把精力放在业务上，但是排错可能会比较麻烦

## 7. go语言的特点/与其他语言相比go有哪些好处

* 语言层面支持并发，而不是程序设计上支持并发。（一个`go`即可开启协程不用关系细节，而其他例如`java`还需要进程池等代码操作）
* 自动垃圾回收比其他java、python更加有效，因为是并行GC
* 生态：主要在于web中间件以及容器化等都很丰富

## 8. go组合的好处

* 结构体的组合类似于java的继承，可以实现对象的再封装
* 匿名的组合可以直接调用组合内结构体的变量、方法而不需要不断的用`.`符号取元素
* 将多个小对象组合成一个大对象，直接创建大对象的同时小对象也会被创建出来。这样可以减少代码的编写同时减少GC的压力

## 9. go的新版本功能

go 1.18新特性：

* 支持泛型

  解决了函数多变量参数的通用性问题，函数的更加抽象统一

  ```go
  //  comparable 表示K必须是一个可比较的任意类型， V必须是int或float64的一种否则编译会报错
  func SumIntsOrFloats[K comparable, V int | float64](m map[K]V) V {
      var s V
      for _, v := range m {
          s += v
      }
      return s
  }
  ```

* 工作区模式

  https://mp.weixin.qq.com/s/Aa9s_ORDVfzbj915aJD5_w

  * 问题：解决`import`导入本地`module`包或者依赖未发布到`github`的包时需要`replace`本地路径从而导致上线不适配（别人可能没有这个本地包）

  * `go work init`支持创建一个工作区，将自己本地依赖的包放在工作区中，这样这些包就会被优先依赖，从而就避免了`replace`

    项目目录如下：

    ```go
    awesomeProject
    ├── mod
    │   ├── go.mod      // 子模块
    │   └── main.go
    ├── go.work         // 工作区
    └── tools
        ├── fish.go
        └── go.mod      // 子模块
    ```

    生成的 go.work 文件内容：

    ```go
    go 1.18			# 表明使用go的版本
    
    use (				# 表明使用的模块
        ./mod 
        ./tools
    )
    ```

## 10. uint类型溢出

1. 显性类型转换
2. 用`big.Int`

## 11. 怎样停止一个goroutine

* 自己停止：
  * `runtime.Gosched()`: 暂停当前G，让G放弃此次抢占的时间片，让其他协程先执行，未来恢复执行
  * `runtime.Goexit()`: 立即停止当前G的运行

* 外部停止：向goroutine发送信号来停止它

  ```go
  func main() {
  	over := make(chan bool, 1)
  	go func() {
  		for {
  			select {
  			case <-over:
  				runtime.Goexit()
  			default:
  				fmt.Println("do some thing")
  			}
  		}
  	}()
  	time.Sleep(2 * time.Second)
  	over <- true
  	fmt.Println("close goroutine.")
  }
  ```

* `context`包停止

## 12. Type switch - 运行时检查变量类型

```go
func main() {
	var s interface{}
	s = "string"
	switch s.(type) {
	case string:
		fmt.Println("is string type")
	case int:
		fmt.Println("is int type")
	default:
		fmt.Errorf("error")
	}
}
```

## 13. defer函数执行位置

defer一般用于处理成对的操作：打开、关闭连接；获得、释放锁；打开、关闭文件等。主要作用在于可以避免忘记资源的释放

* 在return、goexit或panic宕机之后执行
* 在调用栈被销毁之前执行。（所以可以在defer函数中输出调用栈，以及通过defer recover分类处理宕机）

## 14. rune、byte类型

* rune就是`int32`类型的同义词，值表示一个utf-8的编码值(Unicode编码是ASCII编码的超集)，适合表示中文字符串
* byte就是`uint8`类型的同义词，值就表示一个字符

## 15. 断言x.(Type)的使用场景(圣经p164)

* 断言方法：判断一个变量是否有某个方法
  1. 创建一个接口，包含需要判断的方法
  2. 对类型进行断言，检测其是否有该方法
* 断言类型：判断一个变量是否是一个类型，直接在括号中写入类型即可

## 16. Map底层实现/扩容规则/查找

https://i6448038.github.io/2018/08/26/map-secret/

### Map数据结构

Map的底层实现就是一个散列表，在这个散列表中主要的数据结构有两个：

* hmap

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220309155949681.png" alt="image-20220309155949681" style="zoom:30%;" />

  >  红色重点：hmap的指针指向一个bucket数组

* bmap：用于存储实际的数据

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220309160210020.png" alt="image-20220309160210020" style="zoom:30%;" />

  >  字节数组同时存储了真正的数据key和value，key和value在同一个字节数组中，顺序如下图所示：
  >
  > <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220309161358118.png" alt="image-20220309161358118" style="zoom:67%;" />
  >
  > 这样设计的原因在于：这样可以避免k与v长度不同而需要填充造成空间浪费（TODO）

bmap数据结构中的指针让多个bucket链接成为一个链表，所以整体结构hmap与bmap的整体结构如下图：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220209220412276.png" alt="image-20220209220412276" style="zoom: 67%;" />

### Map映射数据/查找的过程

1. 对key用Hash函数取Hash值，然后将此Hash值分为两个部分，前半部分（蓝色）为高位地址，后半部分（红色）为低地址
2. 通过低位地址查找当前key属于hmap中的哪个bucket（根据hmap中指向bucket数组的指针找到bucket数组，然后低位地址作为数组index对应的bucket就是）
3. 再通过高位地址查找当前bucket中对应的高位Hash值的bucket，从而读取value数据

### Map扩容的两种方式

扩容方式：当当前hmap的bucket数组长度不够用时，就会触发扩容，go会将此数组的长度扩张一倍，产生一个新的两倍长度的bucket数组，然后复制旧数据到新数组

判断扩充的条件，就是哈希表中的`加载因子`(即loadFactor)。

`加载因子`是一个阈值，一般表示为：散列包含的元素数除以位置总数。

是一种“产生冲突机会”和“空间使用”的平衡与折中：

* `加载因子`越小，说明空间空置率高，空间使用率小，冲突率比较低
* `加载因子`越大，说明空间利用率上去了，但是“产生冲突机会”高了

go中的计算方式：

```go
map长度 / 2^B   // B：当前已经扩容的次数
```

扩容过程：

* 当当前`map`的大小超过加载因子所需的`map`大小时，会产生扩容
* 首先会创建一个两倍大小的`bucket`数组，然后把旧的`bucket`数组中的所有`bucket`移动到一个属性字段`oldbucket`
* **惰性处理**：当需要访问旧的`bucket`时，才会把这个旧的`bucket`复制到新的数组，hmap的引用也转移到新的`bucket`
* 旧的`bucket`不会手动删除掉，而是会因为没有被引用而被GC回收

### go中Map如何解决hash冲突

很明显，当有多个哈希映射到相同的的桶中时，使用链表解决哈希冲突（链地址法）

## 17. CSP并发模型

* 不同于传统的多线程通过共享内存来通信，CSP 讲究的是“**以通信的方式来共享内存**”
* 用于描述两个独立的并发实体通过共享的通讯` channel` (管道)进行通信的并发模型。`CSP` 中 `channel `是第一类对象，它不关注发送消息的实体，而关注与发送消息时使用的 `channel`

## 18. init在程序中的顺序/一个项目init函数执行的顺序

**核心顺序：**`import --> const --> var --> init()`

![image-20220309004242658](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220309004242658.png)

总结：

* 按照`import`导入时的顺序从最里层的包的`init`函数开始向外执行 （如果一个包导入了其他包，则首先初始化导入的包）
* 每个包只初始化一次，即使该包被多个其他包导入
* 一个包可以有多个 init 函数 (在单个文件中或分布在多个文件中)，并且按照它们呈现给编译器的顺序被调用(文本的顺序)

## 19. go语言的内存对齐

https://geektutu.com/post/hpg-struct-alignment.html

### 什么是内存对齐？

* cpu根据自己的位数一次性读取对应字节的数据（例如16位一次读取1个字节），如果结构体的设计不得体（内存不对齐）那么就会导致一个结构体的读取次数过多效率变低，内存对齐就是期望避免这样的事发生提高效率

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220309002433958.png" alt="image-20220309002433958" style="zoom:50%;" />

### 怎么实现内存对齐优化？

* 通过`unsafe.Alignof`方法获取一个类型的对齐值/对齐系数，根据此系数调整（最好整数倍）
* 合理布局减少内存占用，可以按照字节数（可以通过`unsafe.Sizeof`获取类型字节数）由小到大排列结构体中的字段，这样占用的字节数会更少
  * 原因在于`unsafe.Alignof`的计算方式是当前已知字段的最大值，如果较大的值放在前面会一开始就设置较大的对齐倍数从而导致内存空间的浪费（具体可见链接中的例子）
  * go编译器不会自动帮我们调整顺序，因为这不利于跨平台的数据传输（序列化之后结构体字段顺序对不上）
* 空结构体`struct{}`不应该放在结构体的最后一个字段，因为空结构体本身不占字节，但是如果放在最后一个字段并且有指针指向其时，返回的地址在结构体之外（因为结构体本身不占字节），所以可能会有内存泄漏的问题，所以go会给其创建额外的内存保证安全，会浪费内存。

## 20. go的context

[]: /Users/xwj/blog/source/_posts/技术贴/GoLang/开发基础模块/go的context使用.md

## 21. 什么是协程泄露(Goroutine Leak)

https://hoverzheng.github.io/post/technology-blog/go/goroutine-leak%E5%92%8C%E8%A7%A3%E5%86%B3%E4%B9%8B%E9%81%93/

### 什么是协程泄漏？

几种场景：

* 协程不断创建协程，直到资源耗尽
* 生产者-消费者模型中：多个生成者不断的生产产品，而消费者只消费几次就不会再消费了，那么这些生产者就会产生协程泄漏
* master work场景：将大任务切分为小任务给多个子协程处理，但是主协程只接收部分结果，导致部分协程一直阻塞

特点：

* 主协程永远不会结束
* 子协程永远在阻塞，占用大量资源
* 如果不断检测协程的数量，会发现协程的数量一直在线性递增最后资源耗尽崩溃

### 怎么解决？

* 在创建一个协程的时候就应该考虑这个协程如何被中止
* 主线程在结束前或者任务结束前告知各个子协程终止，而不是让其忙等，可以用channel也可以传递ctx

## 22. go的指针与c的指针的区别









# 二、进阶语法特点

## 1. goroutine协程

### 1.1 进程的基本工作状态

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211106141557437.png" alt="image-20211106141557437" style="zoom: 33%;" />

### 1.2 进程、线程、协程三者对比

* 进程和线程是操作系统层面的概念，切换调度维护在内核态，而协程是在用户态由runtime维护，通过runtime的调度模型避免线程的等待阻塞，挂载在线程上执行，是一种“用户态线程”
* 三者依次更加轻量、资源消耗更少 / 为什么更加轻量？资源消耗少？
  * 线程资源消耗小的原因：相对于单进程来说，一个进程下的多个线程共享该进程的代码段以及堆（映射到同一块物理内存页帧），开启一个新的线程只需要开启一块新的栈
  * 线程切换快的原因：
    * 同一个进程下的多个线程的切换虽然还是需要进入内核态由操作系统调度，但是不需要再切换页表，只需要类似于进程的保留部分现场到TCB中（现场包括一些寄存器和栈地址）
    * 不同进程的线程切换等价于进程切换，需要切换页表
  * 协程资源消耗小的原因：只需要在进程地址空间中创建很小的栈（2KB），共享堆和代码段
  * 协程切换快的原因：
    * 切换策略上：不寄托于操作系统调度切换（可能是时间片），而是由用户/runtime实现切换调度算法
    * 切换过程上：因为不要操作系统的管理，所以不需要从用户态到内核态的切换
  * 协程高效的原因：
    * 协程的切换无需用户、内核态的转换，调度的优劣决定于用户（在go中决定于runtime）
    * 创建后挂载在线程上执行，通过高效的用户态调度策略让阻塞等待的线程得以执行其他协程任务，提高线程的执行效率
  * 协程的缺点：
    * 最终还是需要由系统层面的进程、线程执行，无法实现标准线程的多并发能力，性能还是受限于系统物理CPU核心数

## 2. 协程调度模型GMP

### 2.1 M:N模型

在内核态中创建了N个进程，在用户态中有M个携程，并且M>>N，这M个协程共享内核中的N个进程

对于同一个时刻下，内核中的一个线程只执行一个协程，如果协程遇到了I/O或者其他阻塞（锁、网络），那么就会使用另一个协程挂在这个线程上，让这个线程继续执行

### 2.2 谈谈go进程调度模型

#### GM模型

go语言调度模型最早是go 1.2前采用的是GM模型，G是协程，M就是内核线程

**实现逻辑/流程：**把所有的G都放到一个全局队列中，这个队列的访问需要锁机制，每个线程M从其中获得一个G然后执行。一旦G阻塞（IO、channel等待或Syscall）就会进入parked状态返回到全局队列中，在随后重新被争夺执行。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211106161429689.png" style="zoom: 33%;" />

**缺点：**1. 全局的互斥大锁限制了执行效率，每个线程都可能会阻塞于这个全局队列。所有 goroutine 相关操作，比如：创建、结束、重新调度等都要上锁。2. 亲缘性问题，新创建的G就放在全局而不是本地运行，可能会被其他M执行，亲缘性不好。3. Mcache缓存浪费。M需要为执行的G创建mcache缓存（2MB），而这个缓存在M执行syscall系统调用的时候是不需要的，造成了浪费。4.在GM模型中，当M没有任务后会休眠，一旦有新的G再唤醒，这样的不断的休眠唤醒效率很低（后面就添加了M自旋）

#### GMP模型

随后就出现了新的GMP调度机制：P是一个上下文环境，它为每个在其上的M创建了一个本地队列，P与M之间是一一绑定的，数量可使用`runtime.GOMAXPROCS`来设定。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/llOn2F-20211106165131708.png" style="zoom: 33%;" />

**改进：**1. 为每个M维护一个本地队列，类似于本地缓冲区，这样就避免频繁的访问全局队列。2.将`mcache`等一些资源浪费放到了P中，防止了资源浪费。 3. `work-stealing`调度算法。可以从别的M的本地队列中偷取G执行。

### 2.3 goroutine的生命周期

0. 整个程序始于一段汇编，而在随后的`runtime·rt0_go`（也是汇编程序）中，会执行很多初始化工作。

   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/MLBbwa.png" alt="MLBbwa" style="zoom:50%;" />

1. 绑定 M0 和 g0，M0就是程序的主线程，程序启动必然会拥有一个主线程(运行主函数`main()`)，这个就是 M0。g0 负责调度，即 shedule() 函数。

2. 创建P0：首先会创建**逻辑 CPU （不是物理CPU核数）核数个 P ，存储在 sched 的 空闲链表(pidle)。**然后绑定P0与M0。

3. 新建任务G（就是main）到 P0 本地队列，M0 的 g0 会创建一个指向`runtime.main() `的 G ，并放到 P0 的本地队列。

4. `runtime.main()`: 启动 sysmon 线程M1；启动 GC 协程；执行 init，即代码中的各种 init 函数；执行 `main.main` 函数。

### 2.4 g0的作用

g0是每个系统线程创建的第一个协程就是g0，是一个特殊的goroutine，它与一般的goroutine不同有一个较大的固定的栈

1. **Goroutine创建**。当调用`go func(){ ... }()`或`go myFunction()`时，Go会将函数创建委托给`g0`，然后再将其<u>放置在本地队列中</u>。<u>**新创建的goroutine优先运行，并放置在本地队列的顶部。**</u>

   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/G7WkRM.png" alt="G7WkRM" style="zoom:50%;" />

2. **找寻调度新的goroutine，负责G之间的切换**

3. **Defer方法分配。**

4. **GC操作**，例如stw，扫描goroutine的栈以及一些标清操作。

5. **栈增长**。在需要时，Go会增加goroutine的大小。该操作由`g0`在prolog方法中完成。

### 2.5 G的切换过程/协程切换的过程

G切换原因：

* G 阻塞时：系统调用、互斥锁或 chan。阻塞的 G 进入睡眠模式/进入队列，并允许Go 安排和运行等待其他的 G。
* 在函数调用期间，如果 G 必须扩展其堆栈。这个断点允许 Go 调度另一个 G 并避免运行 G 占用CPU。

在这些情况下，当前G变为Parked，当前G会切换到g0，运行调度程序的g0会替换当前G在M上执行，寻找下一个*优先级*的G然后切换到该G运行

在 Go 中，G 的切换相当轻便，其中需要保存的状态仅仅涉及以下两个：

* **Goroutine 在停止运行前执行的指令**，程序当前要运行的指令是记录在程序计数器（PC）中的， G 稍后将在同一指令处恢复运行；
* **G 的堆栈**，以便在再次运行时还原局部变量；在切换之前，堆栈将被保存，以便在 G 再次运行时进行恢复

**从 G 到 g0 或从 g0 到 G 的切换是相当迅速的，它们只包含少量固定的指令。**相反，对于调度阶段，调度程序需要检查许多资源以便确定下一个要运行的 G。

### 2.6 Schedule优先级

M 绑定的 P 没有可执行的 goroutine 时，它会去**按照优先级**去抢占任务`runtime.schedule`：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/BzHxuL-20211109102510047.png" alt="BzHxuL" style="zoom:33%;" />

* 1/61的概率直接去全局队列中寻找新的G
* 如果没有找到，检查本地队列有没有G
* 如果还没有找到：
  * 尝试向其他P的本地队列中偷取G（work-stealing）
  * 还没找到，检查全局队列获取G
  * 还没找到，找寻一些网络处理的G

> <font color='#e54d42'>核心思路：避免饥饿，首先防止全局队列饥饿，然后是本地队列等等</font>

### 2.7 work-stealing

为了保证公平性，**从随机位置上的 P 开始**，而且遍历的顺序也**随机化**了(选择一个小于 GOMAXPROCS，且和它互为质数的步长)，**保证遍历的顺序也随机化了。**

### 2.8 M线程的自旋

自旋简单来说就是当M没有挂载G时，会一直不断的寻找新的G（采用上面的优先级调度算法）。虽然会消耗少量的CPU资源，但是降低了 M 的上下文切换成本，提高了性能。

如果当前有空闲的P但是没有自旋的M，那么P会在M的空闲列表中唤醒一个M，如果空闲列表中也没有，那么就会新创建一个M然后自旋寻找新的G执行（保证至少有个M在自旋）

### 2.9 syscall系统调用与sysmon系统监视

go有自己封装的系统调用，使用封装的系统调用让P才会触发新的调度（因为系统调用会阻塞），触发系统调用会让M处于syscall状态。

sysmon是一个系统级别的goroutine，不需要挂载P，运行期间始终执行。通过一个循环去监控各个M执行的G，如果时间过长（阻塞），那么就会主动发送信号将M与P断开（信号式抢占）。sysmon的循环周期会不断缩减，为了避免空转，如果一段时间未发现长时间的调用，那么就会睡眠一段时间然后在循环。

Sysmon的主要作用：

* 释放闲置超过5分钟的 span 物理内存；
* 如果超过2分钟没有垃圾回收，强制执行；
* <u>*将长时间未处理的 netpoll 添加到全局队列；*</u>
* *向长时间运行的 G 任务发出抢占调度；* （信号抢占）
* *收回因 syscall 长时间阻塞的 P*；

### 2.10 communicate-and-wait问题

使用channel机制的时候，本方G阻塞在channel的一端，等待读取；当另一端写入channel后，本方G就应该立马执行。但是因为goroutine的调度模型，P维护了一个本地队列顺序，阻塞的G活跃后不会直接立即挂载执行，所以就会产生延迟（要依靠别的P偷取）

解决方式就是Go 1.5 在 P 中引入了` runnext` 特殊的一个字段，可以高优先级执行 unblock G （插队）。加速channel通信机制的时间效率。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/t9uU29.png" alt="t9uU29" style="zoom: 25%;" />

## 3. 内存管理

###  1. 堆栈的区别

* 栈：栈是一次函数调用申请到的内存空间，会随着函数的结束而释放，占用小空间。函数的局部变量以及行参等都会放在栈上。其特点是分配和操作都很快。在内存中从高位向低位拓展。

* 堆：堆适合分配不可预知大小的内存，分配速度较慢。在go语言中由gc回收内存碎片，程序中的全局变量都保存在堆上，部分局部变量也可能逃逸到堆上。堆在内存中由低位向高位拓展。**栈分配廉价，堆分配昂贵。**

​	以32位的内存4G的空间为例：

​	<img src="http://xwjpics.gumptlu.work/qinniu_uPic/OnTiL2-20211111092200699.png" alt="OnTiL2" style="zoom: 33%;" />

### 2. 描述逃逸分析过程

逃逸就是指一个**局部变量的作用域超出其函数栈帧的继续引用**，在go语言中就会将其存储由栈逃逸到堆上，防止空指针的影响

go语言中的逃逸分析是由编译器来处理，这与代码的语义没有直接的关联，程序员也无法直接的控制逃逸，只能够避免一些不必要的逃逸

go语言中的逃逸分析的基本逻辑就是：

* 如果一个局部变量在函数结束后不再引用，那么这个变量就放在栈上使用与回收

* 如果一个局部变量在函数结束后继续引用（return返回了引用或赋值给全局变量）那么在编译的过程中**go的编译器就会将该变量在堆上分配空间，该变量也会在栈上分配，但是包含了一个指向堆的指针，返回时也返回这个指针，这样就可以继续引用**

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/OqdYP8.png" alt="OqdYP8" style="zoom: 33%;" />

逃逸分析可以让合适的变量存在合适的地方，节省GC内存回收的压力

在实践中检查逃逸分析的方法：`$ go build -gcflags '-m -l' main.go`

逃逸分析需要注意的点：

**函数传递变量的指针可以避免复制值的开销，但是这种情况不是一定的，因为复制值的过程在栈上进行这个的开销远比变量逃逸后在堆上动态分配内存少的多，所以如果是很小的结构体可以直接传递本身而不是一定传递指针**

### 3. Goroutine的内存栈扩容与收缩

每个goroutine都会维护一个自己的内存栈（2kb），相互之间隔离，在运行的过程中会扩充与收缩

最开始go开发团队采用的是**分段栈**的机制，就是当需要扩容的时候并不会复制的当前的内容，而是创建一个更大的空间，然后指向那个空间，实现扩容。

但是这样的机制的缺点就是会出现热分裂的问题，如果在一个循环中调用了一个函数，因为当前栈空间不足就需要创建额外的栈空间，然后指向，随后执行完毕后又将其回收，在循环中往复这个过程就会不断的重复创建、回收，效率低下。

所有后面就出现了**连续栈**的思想，连续栈的思想中当需要扩容时就会重新分配一个大内存空间，然后将之前内存数据、指针等全部复制到新的内存空间中，虽然拷贝需要时间但不会出现分段栈的热分裂的问题

* `runtime.newstack` 分配更大的栈内存空间
* `runtime.copystack `将旧栈中的内容复制到新栈中
* *<u>将指向旧栈对应变量的指针重新指向新栈</u>*
* `runtime.stackfree` 销毁并回收旧栈的内存空间

如果之前扩容到了很大的空间，但是目前程序的使用率却不高，那么GC也会**栈缩容**：

如果栈区的空间使用率不超过1/4，那么在垃圾回收的时候使用 `runtime.shrinkstack` 进行**栈缩容**，同样使用 copystack拷贝到一个小栈空间继续使用

### 4. 内存管理机制

![K7w6uc](http://xwjpics.gumptlu.work/qinniu_uPic/K7w6uc-20211111142228647.png)

**一般小对象通过 mspan 分配内存；大对象则直接由 mheap 分配内存。**

* Go 在程序启动时，会向操作系统申请一大块内存，由 mheap 结构全局管理(现在 Go 版本不需要连续地址了，所以不会申请一大堆地址)
* Go 内存管理的基本单元是 mspan，每种 mspan 可以分配特定大小的 object。
* **mcache, mcentral, mheap 是 Go 内存管理的三大组件，mcache 管理线程在本地缓存的 mspan；mcentral 管理全局的 mspan 供给所有线程**

## 4. GC机制

### 1. 主流的垃圾回收算法

#### 引用计数（python，PHP）

* 一次赋值语句中，将新的对象的引用计数+1，老的对象引用计数-1
* 一旦一个对象的引用计数变为0则可以回收该对象，并把该对象引用的所有对象引用计数-1

特点：

* 可以立即回收垃圾
* 不需要stw单独起其他线程用于GC回收，业务线程自己可以实现

缺点：

* 每次赋值语句的计算量较大

* 循环引用问题导致两个对象始终无法被消除

  A、B两个对象互相引用，引用计数都是1，没有其他引用永远不会被消除

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220310150717190.png" alt="image-20220310150717190" style="zoom:33%;" />

#### 追踪式垃圾回收/标记清除法（go）

就是go的实现方式

### 2. STW与Root什么意思？

垃圾回收的基本概念：

* STW(Stop The World)：为了确定程序的引用关系，GC需要将整个程序暂停然后开始扫描。这也是GC机制优化的关键点

  当前运行的所有程序都被暂停（给goroutine发送信号量抢占），扫描内存和添加写屏障，所有的goroutine都被放到一个全局队列中等待，M也会与P分离放到idle列表中

* Root(根对象)：不需要其他对象就可以扫描到的对象，例如全局对象、栈对象中的数据等。是回收扫描时的出发点 

### 3. go垃圾回收机制的版本变迁

* go的**v1.1**早期版本：Mark&Sweep标记清除法，分为标记和清除两个阶段，其步骤如下：

  1. STW暂停程序
  2. Mark：标记引用关系
  3. Sweep：对于未标记的对象加入到free list，用于再分配
  4. Start the world

  缺点：STW需要秒级别，对整个程序暂停扫描，对象多分配速度慢、内存碎片较多

* go的**v1.3**版本后：只有标记过程需要STW（也就是上图的第二阶段）并在实现了并发GC
  * 并发GC： 1. 标记和清除本身是多个线程/协程执行的（已实现） 2. 在GC标记Mark、Sweep清扫的过程中应用程序和内存分配也同时运行 （部分实现）
  * 缺点：v1.3版本只实现了Sweep清扫过程的并发而没有实现标记过程的并发

* go的**v1.5**版本后（三色标记法）：实现了标记、清扫的过程都可以并发的执行应用程序与内存分配，只有在标记阶段的前后需要STW一定的时间来做GC的准备工作和栈的重新扫描re-scan

* go的**v1.8**版本后：实现了混合写屏障（结合了删除写屏障和插入写屏障），满足弱三色不变性，只需要标记阶段一开始的时候扫描STW小段时间后续就不需要STW了，实现了更好的并发

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/pnpbKC.png" alt="pnpbKC" style="zoom:67%;" />

### 4. 三色标记法

**三色的分类：**

* 白色对象：未被扫描的潜在垃圾
* 灰色对象：活跃的对象。<u>存在指向白色对象的外部引用</u>。所有的根对象Root一开始都会被设置为灰色
* 黑色对象：活跃的对象。被扫描过后成为黑色，已经<u>不存在任何外部引用的对象</u>，已经到达引用的终点，GC不会再扫描这些对象。

**标记/染色的过程：**

1. 开始扫描前，所有的对象都为白色
2. 将所有的根对象Root都标记为灰色，然后随引用递归标记。如果是在mcache的span中分配于标记noscan的对象，那么扫描后直接将其标记为黑色
3. 将所有灰色对象的所有下一个引用标记为灰色，并将此灰色对象变为黑色
4. 不断重复3，直到所有对象都遍历完

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/nT4XUm.png" alt="nT4XUm" style="zoom:50%;" />

* 标色过程中，所有的灰色对象都会放到工作池中，由多个GC的goroutine进行扫描
* 染色结束后，所有黑色对象都是还需要使用的对象，而白色对象就是要回收的对象
* 染色的原理：每个scan都有个名为`gcmarkBits`的位图属性，将被扫描的对象标记为1，未被标记的为0

**整体的过程：**

1. 初始化GC任务，包括开启写屏障和开启辅助GC，统计根对象Root数量等，这个过程需要一小段的STW
2. 将所有根对象标记为灰色，然后扫描所有的灰色对象，扫描包括全局指针和goroutine栈上的指针（扫描到对应的goroutine的栈需要将其停止），将所有灰色对象放入工作池/灰色队列中。
3. 从工作池中不断取出灰色对象，将其所有下一层引用加入工作池/灰色队列，将其本身变为黑色
4. 如果一个对象存储的span标记了noscan，那么扫描后直接将其标记为黑色
5. 工作池的读取扫描是后台多个goroutine并行的，不断的循环扫描，直到工作池为空
6. 完成标记工作后，需要重新扫描re-scan整个全局指针和栈，因为在扫描于内存分配并行，可能会有新的对象分配和指针赋值，这就需要用写屏障记录下来，这个阶段也需要STW。
7. 按照标记结果并行回收白色对象

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/oBUQRD.png" alt="oBUQRD" style="zoom:67%;" />

### 5. 强三色不变性与弱三色不变性

这两种都是对标记算法的一种规则/约束描述：

* 强三色不变性：黑色对象不会指向白色对象，只会指向黑色或者灰色对象
* 弱三色不变性：如果要有一条黑色指向白色的引用，那么必须同时要存在一个灰色指向这个白色对象的引用的可达路径（不然这个白色就永远不会被扫描到）

### 6. 屏障技术

**主要处理在标记过程中标记与内存分配、应用程序的并行执行出现的问题：标记的过程中可能有<u>新的对象创建</u>或者<u>旧的对象运行代码时修改指针</u>对现有的标记造成破坏（对象丢失问题）。是一种增量标记的解决算法**

破坏的现象：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211112162556193.png" alt="image-20211112162556193" style="zoom: 33%;" />

如果这个白色对象下游还有引用且唯一依靠白色，那么整个路径都会出现对象的丢失，因为没扫描就会被直接清理掉了。

解决的办法最简单的是标记时不让程序运行即STW，但是效率低。下面两种写屏障算法就实现了并行的处理并避免这个问题：

#### 1. Dijkstra写屏障

* 原理：实现强三色不变性，拦截黑色对象指向白色对象的过程，在指向时将白色对象变为灰色对象：

  （图：当f指向时，将c变为灰色）

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/puBFlQ.png" alt="puBFlQ" style="zoom: 33%;" />

* 实现：在代码的编译期间分析然后强制插入一段代码让白色对象变为灰色
* **注意：**go1.5的写屏障**只对堆上的对象**实现，而为了避免栈上对象的效率不对栈对象进行拦截，栈对象有重扫描保底
* STW：在开始阶段需要一小段的STW开启，但是结束时需要STW来重新扫描栈
* 缺点：结束时需要重新STW扫描栈

#### 2. Yuasa删屏障

* 原理：实现弱三色不变性，删屏障也是拦截写操作，当我们删除一个灰色对象指向白色对象的引用时，将白色对象先标记为灰色然后再删除引用。如下图删除e的时候需要将c变为灰色

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Wapt6e.png" alt="Wapt6e" style="zoom: 33%;" />

* 实现：同样也是在编译期间插入代码实现，在灰色对象与白色对象断开代码之前插入一段代码让白色变为灰色
* STW：在GC开始时需要STW扫描堆栈来记录初始快照，结束时不用重新扫描即不用STW
* 缺点：这样的方式回收精度低，一个本来就需要删除的白色对象需要在下一轮才会被回收

#### 3. 混合屏障

go1.8之后引入了混合屏障，结合了写屏障和删屏障

* 满足弱三色不变性：

  * 写屏障：对于堆上的黑色对象，如果已经变为黑色再引用一个白色对象，那么就将这个白色对象变为灰色保护起来。（写屏障对堆上的强三色不变性处理）
  * 删屏障：对于栈上的灰色对象，如果其和一个黑色对象同时引用一个白色对象，当运行时删除了灰色对象对白色对象的引用的时候就将这个白色对象变为灰色。（这保证了栈上所有对象不会出错，弥补了写屏障对栈不处理的缺点）

  * 为了不再重新扫描，所以当运行时有新的对象被创建时立即扫描将其变为黑色。（处理新的对象创建问题）

### 7. Sweep回收机制

go为每一个内存块mspan中的每个对象创建了对应的`bitmap allocBits`位图，他表示上一次GC过后每一个Objec他的分配情况，这个值由扫描过程中的`gcmarkBits`来赋值

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/NicGa3.png" alt="NicGa3" style="zoom:50%;" />

内存清理的两种策略：

* 在后台一直启动一个worker协程等到清理内存，一个一个mspan的处理
* 当申请分配内存的时候lazy触发，即一块待被清理的内存等待重新被分配时才将其内容置为零值。清理的延迟被分配到每一次的新的内存分配过程中

### 8. GC调优的策略

通过 `go tool pprof` 和 `go tool trace` 等工具：

* 控制内存分配的速度，限制 Goroutine 的数量，从而提高赋值器对 CPU的利用率。

* 减少并复用内存，例如使用 `sync.Pool `来复用需要频繁创建临时对象，例如提前分配足够的内存来降低多余的拷贝。

* 需要时，增大 GOGC 的值，降低 GC 的运行频率。

## 5. Channel

### 1. 介绍一下channel

channel是go语言中用于多个协程goroutine之间通信、同步的一个数据结构。正是如此，channel在内存中一定会被分配到**堆**上。

channel的特点是：

1. 线程访问安全，可以支持多个goroutine访问
2. 存储并且可以在多个goroutine之间传递值
3. 先进先出FIFO
4. 能够自动让goroutine锁住或解锁，代码层面不需要考虑加解锁问题
5. 设计特点：通过通信实现共享内存而不是通过共享内存实现通信

### 2. channel类型

* 有缓冲channel ： `ch := make(chan int, 3)`

  通信的同时传输数据，一般在**异步**的时候使用，不需要同时在线

* 无缓冲channel：`ch := make(chan int)`

  不存储数据，仅做通信功能，一般在**同步**时使用，需要读写方同时在线

### 3. 使用注意事项

* channel的创建只可以使用`make`关键字，并且直接返回的是channel的指针，可直接在函数间传递使用。

* 为`nil`的channel（例如chan []string，但是没有给其赋值一个string结构体，所以就是nil）不论发送还是接收数据会造成永久阻塞
* 向一个已经关闭的channel发送数据，会造成panic
* 从一个已经关闭的channel接收数据，接收到的都是零值

### 4. channel底层结构

channel的底层的数据结构叫做`hchan`, 核心是一个**环形队列**`buff`字段是一个缓冲数组保存数据，同时还分别使用了两个指针(`sendx`、`recvx`)，来控制数据的出入对, 同时还有一个锁，使得线程访问安全。此外，还通过两个指向`sudog`结构体的字段`sendq`、`recvq`来实现读写两端goroutine阻塞记录和快速启动。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211120142649000.png" alt="image-20211120142649000" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/gdZ2bc.png" alt="gdZ2bc" style="zoom:33%;" />

读取端/写入端从channel中拿出一个元素的过程：

* 获得锁
* 拷贝值（读取端：从channel -> 变量， 写入端：从变量 -> channel）
* 释放锁

### 5. 优化设计：直接发送机制

对于有缓冲的channel：

* 发送端阻塞等待的过程（channel满还要写入）：
  1. 将等待阻塞的发送端goroutine G1放入到`hchan`中`sendq`指向的`sudog`等待队列中，保存G1和其数据
  2. 然后G1阻塞变为`gopark`状态，与M分离，切换g0根据优先级进行进程调度
  3. 等到channel中有数据被取走时，唤醒`sendq`中`sudog`保存的G1，让G1变为`goready`状态
  4. 为了亲缘性调度，将G1放在P的`runNext`字段上，让其快速被挂载执行
  5. G1复制自己的数据值到channel中

* 接收端阻塞等待过程（channel空还要读取）：

  1. 将接收端的G2协程放入到`hchan`中`recvq`指向的`sudog`等待队列中，保存G2和其**接收变量的栈指针**

  2. 然后G2变为`gopark`状态，与M分离，切换g0进行进程调度

  3. 等到channel中写入端G1将数据写入时，此时，**直接将G1的值写入到G2接收变量的栈指针所指向的位置，而不经过channel，即G1直接操作G2的栈指针内容** （优点：避免了再赋值到channel拿锁、释放锁的流程，更加快。并且此时G2处于阻塞状态，所以这样也是安全的）

     <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211119213528624.png" alt="image-20211119213528624" style="zoom: 25%;" />

  4. 唤醒G2，让G2变为`goready`状态，启动G2，变量上已经有内容。

对于无缓冲的channel（循环队列长度为0）：

* 如果是发送者先行，那么接受者直接发送（修改栈）
* 如果是接受者先行，那么发送者直接发送（修改栈）

### 3. 怎样防止channel关闭之后再写

解决方法：

1. 始终由发送者关闭，读取方读取完之后返回一个ok即可。但是这样的要求挺高，例如多个写的情况等

   对于多个写的情况让最后一个关闭是较好的选择

2. 使用两个channel，另一个channel专门负责接收关闭信号。但是对于传送数据的channel双方并不知道何时会有数据

## 6. 锁

### 1. 死锁、读写锁、互斥/同步锁、悲观锁、乐观锁、自旋锁

go语言中的锁：

* **死锁：** 死锁不是锁而是一种现象，即所有的线程都在等待，程序无法结束。例如从空channel中读取数据等

* **互斥/同步锁**：`Mutex`，对于某个公共临界区数据，在某一时刻只有一个协程能够获取到锁，使用完之后释放锁，其他协程才可以抢夺

* **读写锁**：`RWMutex`，将锁分为了读锁与写锁。读锁占用的情况下会阻止写锁，但是不阻止读锁（共享读锁），但是写锁占用的时候会阻止其他任何锁（无论读/写锁，独占）

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/6KqgDH.png" alt="6KqgDH" style="zoom: 50%;" />

  * 写锁被解锁后，所有因操作锁定读锁而被阻塞的 goroutine 会被唤醒，并都可以成功锁定读锁

  * 读锁被解锁后，在没有被其他读锁锁定的前提下，所有因操作锁定写锁而

  被阻塞的 Goroutine，其中等待时间最长的一个 Goroutine 会被唤醒

思想（乐观锁和悲观锁是两种思想，用于解决并发场景下的数据竞争问题）：

* **悲观锁**：操作数据的时候非常悲观，认为别人也会同时修改数据，因此操作的时候直接将数据锁住，直到操作结束才释放。 **（go的`Mutex`是悲观锁）**
* **乐观锁：**操作数据的时候非常乐观，认为别人不会同时修改数据，只是在执行更新的时候判断一下别人是否修改了数据，如果修改了就放弃此次修改操作，没修改就继续修改

go实现乐观与悲观锁：

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

var (
	value1 int32
	value2 int32
	mutex  sync.Mutex
)

// 乐观锁
func optimisticAdd() {
	atomic.AddInt32(&value1, 1)
}

// 悲观锁
func pessimisticAdd() {
	mutex.Lock()
	value2++
	mutex.Unlock()
}

func main() {
	// 分别开启100个协程，自增操作
	for i := 0; i < 100; i++ {
		go optimisticAdd()
		go pessimisticAdd()
	}
	time.Sleep(time.Second * 5)
	fmt.Println("乐观锁: ", value1) // 100
	fmt.Println("悲观锁: ", value2) // 100
}
```

乐观锁函数：`func AddInt32(addr *int32, delta int32) (new int32)`第一个参数是指针，是因为要获得被操作数载内存中的存放位置来施加特殊的CPU原子指令。

* **自旋锁：**当一个资源的锁被其他线程获取的时候，本程序一直循环等待其释放锁，不断判断是否能够成功获取，直到获取操作后才结束。

  优点：反应快，缺点：占用CPU资源。适合等待时间较短的场景

 **go实现乐观自旋锁**

```go
package main

import (
	"runtime"
	"sync"
	"sync/atomic"
)

type spinLock uint32 // 实现锁接口 1表示加上了锁，0表示释放了锁

// 加锁
func (s *spinLock) Lock() {
	// 不断循环获取锁, 将数据从0改为1
	for !atomic.CompareAndSwapUint32((*uint32)(s), 0, 1) {
		// 如果获取失败就放弃时间片
		runtime.Gosched()
	}
}

// 解锁
func (s *spinLock) Unlock() {
	// 归还为0
	atomic.StoreUint32((*uint32)(s), 0)
}

func NewSpinLock() sync.Locker {
	var lock spinLock
	return &lock
}
```

* **互斥锁：**一旦一个资源的锁被其他线程获取，自己就休眠，等到系统的唤醒后再次争夺

  优点：占用资源少，缺点：反应慢。适合等待时间较长的场景

### 2. CAS算法（compare and swap）

CAS算法是一种有名的无锁算法。无锁编程，即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步（Non-blocking Synchronization）。CAS算法涉及到三个操作数

- 需要读写的内存值V
- 进行比较的值A
- 拟写入的新值B

 **当且仅当 V 的值等于 A时，CAS通过原子方式用新值B来更新V的值，否则不会执行任何操作**（比较和替换是一个原子操作）。一般情况下是一个自旋操作，即不断的重试。 （其实也就是乐观锁的实现机理）

### 3. Mutex的几种状态/几种模式

mutex的几种状态：

* `mutexLocked` — 表示互斥锁的锁定状态;
* `mutexWoken` — 表示从正常模式被从唤醒;
* `mutexStarving` — 当前的互斥锁进入饥饿状态;
* `waitersCount` — 当前互斥锁上等待的 Goroutine 个数

模式：

* 正常模式（非公平锁）：

  正常模式下，**所有等待锁的 goroutine 按照` FIFO`(先进先出)顺序等待**。**唤醒的 goroutine 不会直接拥有锁，而是会和新请求 goroutine 竞争锁**。**新请求的 goroutine 更容易抢占**:因为它正在 CPU 上执行，所以刚刚唤醒的 goroutine有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的 goroutine 会加入 到等待队列的前面。

* 饥饿模式（公平锁）：

  为了解决了等待 goroutine 队列的长尾问题

  饥饿模式下，直接由` unlock `把锁交给等待队列中排在第一位的 goroutine (队头)，同时，饥饿模式下，**新进来的 goroutine 不会参与抢锁也不会进入自旋状态，会直接进入等待队列的尾部**。这样很好的解决了老的 goroutine 一直抢不到锁的场景。

  饥饿模式的触发条件:**当一个 goroutine 等待锁时间超过 1 毫秒时，或者当前队列只剩下一个 goroutine 的时候，Mutex 切换到饥饿模式。**

对于两种模式，**正常模式下的性能是最好的，goroutine 可以连续多次获取锁，饥饿模式解决了取锁公平的问题，但是性能会下降**，这其实是性能和公平的一个平衡模式。

### 4. Mutex允许自旋的条件

* 锁已被占用，并且锁不处于饥饿模式。
* 积累的自旋次数小于最大自旋次数(`active_spin=4`)。
* CPU核数大于1。
* 有空闲的P。
* 当前Goroutine所挂载的P下，本地待运行队列为空。

### 5. 条件变量Cond

条件变量本身并不是锁，但是经常会结合锁一起使用。条件变量是在goroutine访问锁**之前**进行的判断，**为了防止公共临界区数据已满或者全空而导致的即使抢到了锁也需要等待对端的情况**

一般的访问步骤：

1. 判断条件变量
2. 加锁
3. 访问/处理
4. 解锁
5. 唤醒阻塞在条件变量上的对象

`sync.Cond`类型代表了条件变量，条件变量与锁一起使用

```go
type Cond struct {
	noCopy noCopy

	// L is held while observing or changing the condition
	L Locker

	notify  notifyList
	checker copyChecker
}
```

条件变量本身不是锁，但是其内部属性包括锁，所以在使用的时候需要指定锁

常见的方法：

* `func (c *Cond) Wait()`: 相当于以下两步：
  1. 因为当前的公共变量区已满或者为空，让前goroutine阻塞，并解开当前goroutine获取到的公共区的锁 （这是一个原子操作）（解锁是为了让对端获取到锁，解开条件变量）
  2. 等待被唤醒，一旦被唤醒，那么就重新获取互斥锁，然后对数据进行操作
* `func (c *Cond) Signal()`： 单发通知，给**一个**正在等待/阻塞的goroutine发送通知让其唤醒
* `func (c *Cond) Broadcast()`：广播通知，给正在等待/阻塞在该环境变量上的**所有**goroutine发送通知让它们唤醒

注意点：

* 因为wait需要解锁，所以在创建完cond时，需要先对公共资源加锁，这样才可以解锁

* 一般都采用for**循环**去判断条件变量并wait，这样的原因是如果用if，所有协程都在阻塞wait的时候，当被唤醒时，一个协程获得了锁写入了数据，而其他协程如果拿到锁之后也执行到了wait下面也要写入就会造成写入的阻塞（因为if只判断一次），所以要用循环判断

一般结构：

```go
c.L.Lock()
for !condition() {
	c.Wait() 
}
... make use of condition ...
c.L.Unlock()
```

示例：生产者消费者

```go
package main

import (
	"fmt"
	"sync"
)

var cond sync.Cond
var ch chan string

func main() {
	ch = make(chan string, 5)
	cond.L = new(sync.Mutex)
	// 开启生产者与消费者
	for i := 0; i < 5; i++ {
		go send(ch, 5)
		go receive(ch)
	}
	for {
	}
}

func send(ch chan<- string, i int) {
	for {
		// 先锁上
		cond.L.Lock()
		// 1. 循环判断条件变量
		for len(ch) == i {
			cond.Wait()
		}
		// 2. 未满就继续写入
		ch <- "hello"
		// 3. 解锁
		cond.L.Unlock()
	}
}

func receive(ch <-chan string) {
	for {
		// 先锁上
		cond.L.Lock()
		// 1. 循环判断条件变量
		for len(ch) == 0 {
			cond.Wait()
		}
		// 2. 未空就继续读出
		fmt.Println(<-ch)
		// 3. 解锁
		cond.L.Unlock()
	}
}
```

### 6. Wait group

多协程同步的工具

使用方法：

1. 创建一个**Waitgroup**的实例，假设此处我们叫它wg
2. **在每个goroutine启动的时候，调用wg.Add(1)**，这个操作可以在goroutine启动之前调用，也可以在goroutine里面调用。当然，也可以在**创建n个goroutine前调用wg.Add(n)**建议**在协程启动之前调用**，**不然主进程跑的太快太快可能导致协程还未启动就结束**
3. 当**每个goroutine完成任务后，调用wg.Done()**
4. 在**等待所有goroutine的地方调用wg.Wait()**，它在所有执行了wg.Add(1)的goroutine都调用完wg.Done()前阻塞，**当所有goroutine都调用完wg.Done()之后它会返回**。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			fmt.Println("hello")
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println("over.")
}
```

注意点：

* 不能在goroutine函数内部Add，可能主进程跑的太快导致goroutine直接结束
* 如果想在多个goroutine函数之间传递WaitGroup请传递指针确保是同一个

原理：

* WaitGroup 主要**维护了 2 个计数器**，一个是请求计数器 v，一个是等待计数器 w，二者组成一个64bit 的值，请求计数器占高 32bit，等待计数器占低 32bit。
* **每次Add执行，请求计数器v加1**，**Done方法执行，等待计数器减1**，**v为 0 时通过信号量唤醒 Wait()。**

### 7. Sync.Once

* Once 可以用来执行且仅仅执行一次动作，常常用于**单例对象的初始化场景**。
* Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。
* sync.Once 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第 一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	f := func() {
		fmt.Println("hello")
	}
	once.Do(f)		// 只有这一个会输出
	once.Do(f)
	once.Do(f)
	once.Do(f)
}
```

### 8. Sync.Pool

对于很多**需要重复分配、回收内存**的地方，sync.Pool 是一个很好的选择。频繁地分配、回收内存会给 GC 带来一定的负担，严重的时候会引起 CPU 的毛刺。而 sync.Pool 可以将暂时将不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力，提升系统的性能。

```go
package main

import (
    "fmt"
    "sync"
)

var strPool = sync.Pool{
    New: func() interface{} {
        return "test str"
    },
}

func main() {
    str := strPool.Get()
    fmt.Println(str)
    strPool.Put(str)
}
```

* 通过`New`去定义你这个池子里面放的究竟是什么东西，在这个池子里面你只能放一种类型的东西。比如在上面的例子中我就在池子里面放了字符串。

* 我们随时可以通过`Get`方法从池子里面获取我们之前在New里面定义类型的数据。

* 当我们用完了之后可以通过`Put`方法放回去，或者放别的同类型的数据进去。

一开始这个池子会初始化一些对象供你使用，如果不够了呢，自己会通过new产生一些，当你放回去了之后这些对象会被别人进行复用，当对象特别大并且使用非常频繁的时候可以大大的减少对象的创建和回收的时间。

## 7. 反射

### 基本概念

两个重要的类型：

* `reflect.Type`接口，该接口用于表达一个类型，有很多的方法包括用于识别类型以及透视该类型的组成部分,`Type`接口的只存在一个实现实例就是 *类型描述符(interface的动态类型就是一个类型描述符)*；
* `reflect.Value`类型/结构体，可以包含任何类型的值即表示任意类型的值。

两个重要方法：

* `reflect.TypeOf()` 返回一个`reflect.Type`接口，返回一个接口值对应的动态类型/类型描述符

  ```go
  func main() {
  	t := reflect.TypeOf(3)  // 首先会将3的int类型隐式转换为Type接口，该接口动态类型变为int，动态值变为3, 返回的是该动态类型即类型描述符
  	fmt.Println(t.String()) // "int"
  	fmt.Println(t)          // "int"  直接输出使用的是其动态类型所表示的具体类型即int，而不是Type接口
  }
  ```

  `reflect.Type`的方法`Kind()`返回一个`Kind`, 其表示类型的分类(或者理解为类型的类型), 不同于类型的无限（因为有自定义类型），`Kind`只有有限的几种

* `reflect.ValueOf()` 返回一个`reflect.Value`类型/结构体，此函数接收任意的`interface{}`并将接口的动态值以`reflect.Value`返回

  ```go
  func main() {
  	v := reflect.ValueOf(3) // 返回reflect.Value，是接口值的动态值
  	fmt.Println(v)          // "3"  返回值是动态值的具体值
  	fmt.Println(v.String()) // “<int Value>”
  }
  ```

  `Value`的`Type()`方法可以实现转换，`Interface()`可以实现向`interface{}`的转换

  ```go
  func main() {
  	v := reflect.ValueOf(3) // 返回reflect.Value，是接口值的动态值
  	t := v.Type()           // 可以通过此方式将其类型以reflect.Type返回
  	fmt.Println(t)          // "int"
  	fmt.Println(t.String()) // "int"
  	i := v.Interface()
  	x := i.(int)
  	fmt.Println(x) // "3"
  }
  ```

空接口`interface{}`与`reflect.Value`的异同：

* 相同点：都可以包含任意的值

* 不同点：空接口必须断言为具体类型后才可以使用（断言就意味着提前知道其类型），而`reflect.Value`包含了很多的方法来直接分析和使用所包含的值而不需要断言

### 反射的一般作用

[]: /Users/xwj/blog/source/_posts/技术贴/GoLang/开发基础模块/Go语言-2-进阶.md

在不知道类型的情况下，可以更新变量、在运行时查看其值、调用该类型方法或者直接对齐布局进行操作

* 用于实现适配器函数/桥连接：该函数具有统一处理各种值类型的能力，即使这些值无法共享一个接口或者布局未知或者这个类型还不存在（最典型的例子就是`fmt.Printf`可以打印任意类型）

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

* 用于根据结构体的`tag`字段，修改结构体的值

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

# 三、工程类、实践类

## 1. 生产者消费者模型

略（见Cond）

## 2. 实现一个进程安全的Set

```go
type securitySet struct {
	data map[interface{}]bool
	sync.RWMutex
}

func NewSecuritySet() *securitySet {
	return &securitySet{
		data: make(map[interface{}]bool),
	}
}

func (s *securitySet) Add(v interface{}) {
	s.Lock()
	s.data[v] = true
	s.Unlock()
}

func (s *securitySet) Remove(v interface{}) {
	s.Lock()
	delete(s.data, v)
	s.Unlock()
}

func (s *securitySet) List() []interface{} {
	s.RLock()
	res := make([]interface{}, 0)
	for k := range s.data {
		res = append(res, k)
	}
	s.RUnlock()
	return res
}
```

## 3. 协程访问共享资源与循环变量捕获问题

以下程序有什么问题：

```go
total := 0
for i := 1; i <= 10; i++ {
    sum += i
    go func() {
        total += i
    }()
}
fmt.Printf("total:%d sum %d", total, sum)
```

问题：

1. 协程访问共享资源要通信协调、主进程要等到所有子进程
2. 延迟函数执行需要捕获循环变量

```go
func main() {
	total, sum := 0, 0
	mtx := new(sync.Mutex)
	wg := sync.WaitGroup{}
	for i := 1; i <= 1000; i++ {
		sum += i
		j := i
		wg.Add(1)
		go func() {
			mtx.Lock() // 加锁就会导致延迟执行，此时的i已经在循环中遍历到最后的10了
			total += j
			mtx.Unlock()
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Printf("total:%d sum %d\n", total, sum)
}
```

## 4. 实现一个简单的客户端/服务器模型

## 5. 实现一个tcp连接传输数据

## 6. 实现一个闭包

```go
func hasPrefix() func(s string) {
	prefix, i := "xxx", 1
	return func(s string) {
		if strings.HasPrefix(s, prefix) {
			i++
			fmt.Println(s, strconv.Itoa(i))
		}
	}
}

func main() {
	f := hasPrefix()
	f("xxxhello.jpg")
	f("hello1111.rpg")
	f("xxxx.jpg")
	// xxxhello.jpg 2
	// xxxx.jpg 3
}
```

## 7. goroutine的交替打印

例如：启动三个协程，交替按顺序打印`cat`、`dog`、`fish`

(第二种)

```go
// 变为串行
func main() {
	ch1 := make(chan int, 1)
	ch2 := make(chan int, 1)
	ch3 := make(chan int, 1)
	ch3 <- 0
	go func() {
		for {
			t := <- ch3
			fmt.Println("cat")
			ch1 <- t
		}
	}()
	go func() {
		for {
			t := <- ch1
			fmt.Println("dog")
			ch2 <- t
		}
	}()
	go func() {
		for {
			t := <- ch2
			fmt.Println("fish")
			if t == 100 {
				os.Exit(1)
			}
			t++
			ch3 <- t
		}
	}()
	for{}
}


// 分别工作，主线程负责打印
func main() {
	ch1, ch2, ch3 := make(chan string, 1), make(chan string, 1), make(chan string, 1)
	go func() {
		for {
			ch1 <- "cat"
		}
	}()

	go func() {
		for {
			ch2 <- "dog"
		}
	}()

	go func() {
		for {
			ch3 <- "fish"
		}
	}()

	for i:=0; i<100; i++ {
		s1, s2, s3 := <-ch1, <-ch2, <-ch3
		fmt.Println(s1)
		fmt.Println(s2)
		fmt.Println(s3)
	}
}
```

## 8. 拷贝一个文件

## 9. 三个协程交替打印100

```go
package main

import "fmt"

// 三个协程交替打印100
func main() {
	over := make(chan bool)
	// 创建三个channel
	chans := make([]chan bool, 3)
	for i := 0; i < 3; i++ {
		chans[i] = make(chan bool, 1)
	}
	num := 1
	chans[2] <- true // 给最后一个channel先写入一个数防止第一个阻塞
	// 打印
	for i := 0; i < 3; i++ {
		go func(i int) {
			for { // 循环不断执行
				if num > 100 {
					over <- true
				}
				// 等待上一个channel
				if i == 0 {
					<-chans[2]
				} else {
					<-chans[i-1]
				}
				// 打印自己
				fmt.Printf("Thread %d: %d\n", i, num)
				num++
				// 写入到自己的channel
				chans[i] <- true
			}
		}(i)
	}
	<-over
	fmt.Println("over")
}
```

