---
title: 使用golang写一条区块链
tags:
  - golang
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-10-04 20:54:04
---

项目来自视频教学：https://www.bilibili.com/video/BV1kE411W7aD?

文章底部有完整代码链接

# 项目功能演示

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004090620.png)

## 1. printChain 输出整条区块链
simple:
![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004091517.png)

<!-- more -->

## 2. getBalance ADDRESS 查询账户余额

参数： ADDRESS-账户地址
simple:
![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004091724.png)

## 3. send FROM TO AMOUNT MINER DATA  由FROM转AMOUNT钱给TO，由MINER挖矿，同时写入DATA
参数: FROM-转出人 TO-转入人 AMOUNT-转账金额 MINER-挖矿人 DATA-铸币交易可以自添加的数据
simple:
![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004092752.png)

## 4. getTransaction TXHASH 查询交易信息
参数 TXHASH-交易hash
simple:
![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004092924.png)

## 5. newWallet 创建一个新的钱包（公私钥对）
simple:
![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004093103.png)

## 6. listAddress 列举所有的钱包地址
simple:
![](http://xwjpics.gumptlu.work/qiniu_picGo/20201004093157.png)

# 一、Go基础

## G252o环境安装

### Go语言

去官网下载并安装配置好全局变量即可，记得配置GOROOT和GOPATH

### 编程IDE：GOLand的安装与破解

详情见：https://tech.souyunku.com/?p=16189

<!-- more -->

## Go项目的目录结构

项目目录结构如何组织，一般语言都是没有规定。但Go语言这方面做了规定，这样可以保持一致性，做到统一、规则化比较明确。

#### 1、一般的，一个Go项目在GOPATH下，会有如下三个目录：

```
|--bin

|--pkg

|--src
```

其中，bin存放编译后的可执行文件；pkg存放编译后的包文件；src存放项目源文件。

对于pkg目录，曾经有人问：我把Go中的包放入pkg下面，怎么不行啊？他直接把Go包的源文件放入了pkg中。

这显然是不对的。pkg中的文件是Go编译生成的，而不是手动放进去的。（一般文件后缀.a）

对于src目录，存放源文件，**Go中源文件以包（package）的形式组织**。通常，新建一个包就在src目录中新建一个文件夹。

# 二、项目中的数据库Blot

## 1.Blot简介与实例

简介：一个小型的key-value数据库，没有sql，轻便快捷高效。

操作demo详情见：https://blog.csdn.net/yang731227/article/details/82974575

结构：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200914183432.png)



demo：

```go
package main

import (
	"fmt"
	"itcast_Go/bolt"
	"log"
)

func main()  {
	//1. 打开数据库
	//第一个参数是名字，第二个参数是权限6代表允许读写
	db, err := bolt.Open("test.db", 0600, nil)
	defer db.Close()
	if err != nil{
		log.Panic("打开数据库失败！" , err)
	}
	//操作数据库
	db.Update(func(tx *bolt.Tx) error {
		//2. 打开抽屉(没有就创建)
		var bucketName []byte = []byte("b1")
		bucket := tx.Bucket(bucketName)
		if bucket == nil{
			//没有就创建
			bucket, err = tx.CreateBucket(bucketName)
			if err != nil{
				log.Panic(err)
			}
		}
		//操作抽屉中的数据，添加数据
		//3. 写数据
		bucket.Put([]byte("1111"), []byte("hello"))
		bucket.Put([]byte("2222"), []byte("world"))
		return nil
	})

	//4. 读数据
	db.View(func(tx *bolt.Tx) error {
		//找到抽屉
		bucket := tx.Bucket([]byte("b1"))
		if bucket != nil{
			//如果存在就读取
			v1 := bucket.Get([]byte("1111"))
			v2 := bucket.Get([]byte("2222"))
			//输出
			fmt.Printf("'1111'-> %s\n", v1)
			fmt.Printf("'2222'-> %s\n", v2)
		}
		return nil
	})
}

```

## 2.项目存储结构分析



![image-20200914200555511](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200914200555511.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20200914200743.png)



# 三、项目中go语言的区块序列化与反序列化

对于较为复杂的数据，采用序列化作为value存储在k/v数据库（blot）中，那么go语言的序列化与反序列化的基本操作是怎样的呢？

使用**gob**来简化操作，看完demo就懂：

demo：

```go
package main

import (
   "bytes"
   "encoding/gob"
   "fmt"
   "log"
)

//创建“人”结构体
type Person struct {
   Name string
   Age uint
}

func main()  {
   //定义一个“人”结构
   var xiaoming Person
   xiaoming.Name = "小明"
   xiaoming.Age = 18
   //编码的数据放进buffer
   var buffle bytes.Buffer

   //使用gob序列化得到字节流
   //定义一个编码器encoder
   encoder := gob.NewEncoder(&buffle)
   //编码结构体
   err := encoder.Encode(&xiaoming)
   if err != nil{
      log.Panic(err)
   }
   fmt.Printf("小明编码后的结果为：%v\n", buffle.Bytes())

   //使用gob反序列化得到结构体
   //创建byte读input流,然后创建解码器
   decoder := gob.NewDecoder(bytes.NewReader(buffle.Bytes()))
   var daming Person
   //解码
   err = decoder.Decode(&daming)
   if err != nil{
      log.Panic(err)
   }

   fmt.Printf("解码后的小明: %v\n", daming)

}
```



# 四、Go语言迭代器原理与区块链的特殊迭代

range内部其实就是指针的迭代指向，然后赋值使用：



![](http://xwjpics.gumptlu.work/qiniu_picGo/20200915120936.png)

但是由于区块链指针的特殊性，所以迭代需要**从后向前**迭代：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200915121426.png)



# 五、Go语言命令行的使用

很简单

demo:

```go
package main

//go命令行测试

import (
   "fmt"
   "os"
)

//go命令行练习
func main()  {
   len1 := len(os.Args)
   fmt.Printf("命令长度为：%d\n", len1)
   for i, cmd := range os.Args{
      fmt.Printf("arg[%d]: %s\n", i, cmd)
   }

}
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200915135850.png)

# 六、项目添加转账功能 

转账重要的两点：

1. 每一笔交易能支配的钱都来自于上一个交易的输出
2. 每一个花费的输出要一次性花完，有剩余的都要转给自己

首先需要熟悉下比特币交易脚本的三种模式介绍：

[北大肖臻-第9讲-比特币脚本](https://myblog.gumptlu.work/2020/06/24/%E7%9F%A5%E8%AF%86%E8%B4%B4/%E5%8C%BA%E5%9D%97%E9%93%BE/%E5%8C%97%E5%A4%A7%E8%82%96%E8%87%BB%E3%80%8A%E5%8C%BA%E5%9D%97%E9%93%BE%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%BA%94%E7%94%A8%E3%80%8B%E7%AC%94%E8%AE%B0/%E5%8C%97%E5%A4%A7%E8%82%96%E8%87%BB-%E7%AC%AC9%E8%AE%B2-%E6%AF%94%E7%89%B9%E5%B8%81%E8%84%9A%E6%9C%AC/)

项目交易的结构：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200916100049.png)



# 七、账户余额UTXO计算细节

### 1. 计算账户余额时统计UTXO

遍历UTXO去统计某个账户的余额，如果只是简单的遍历则效率太低，而区块链中的交易都是相关联的，所以利用这一特点可以使用一些小技巧：



![](http://xwjpics.gumptlu.work/qiniu_picGo/20200926001757.png)

图中黄色是已消费的输出，蓝色是还未消费的输出

### 2. 转账时计算UTXO中的账户余额

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200926151535.png)

项目中并没有优化计算最合适的将零钱拼装，而是简单的遍历逐步统计，满足要求了就转账



# 八、blotDB数据库的可视化

1. 下载工具：运行`go get github.com/boltdb/boltd`

2. 就会下载到GOPATH目录中，查看GOPATH方法：`go env`

3. 找到cmd文件夹中的main文件编译其成为可执行文件，编译`go build main.go`

4. 把可执行文件放到和blot的db类文件相同的目录下，运行：`main.exe -- xxx.db`

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200927164131.png)

5. 显示结果：

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200927164205.png)

   



# 九、公私钥

比特币公私钥与地址的关系

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200928100122.png)

公钥生成地址的流程：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200928150723.png) 



最后一步要用到base58算法，一般都没有这个包，可以通过以下命令引入比特币源码官方的提供的包：

`go get github.com/btcsuite/btcutil/base58`

# 十、P2PKH的检验方式

## 1. 比特币的几种校验方式

https://blog.csdn.net/weixin_43988498/article/details/107958185  三种比特币的校验方式

贴出P2PKH的校验流程：

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTgzMTU2LnBuZw?x-oss-process=image/format,png)

这一种是较为常见的一种形式，**输出脚本中输出的是公钥的Hash，而输入脚本中要除了签名还要包含公钥**

除了这些，其他的DUP、HASH160都是一些验证操作。

脚本执行过程：

同样的为了方便看，将输入与输出拼接到一起，从上往下执行。

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTgzODM4LnBuZw?x-oss-process=image/format,png)

前两步操作相同，将输入中的签名和公钥压入栈

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTgzOTMyLnBuZw?x-oss-process=image/format,png)

第三步操作DUP是将栈顶的公钥复制一份

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTg0MDIwLnBuZw?x-oss-process=image/format,png)

第四步操作HASH160是将复制的公钥取HASH值，然后压入栈中。

第五步，将输出脚本里面的公钥Hash压入栈，这时栈里面出现了两个公钥的Hash值

搞清楚这个Hash值的来源：

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTg0NDIxLnBuZw?x-oss-process=image/format,png)

第六步，EQUALVERIFY是弹出栈顶的两个Hash值，比较两者是否相等。

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTg0NjQxLnBuZw?x-oss-process=image/format,png)

最后一步，和之前一样，分别弹出，检查公钥与签名是否配对（正确）。

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTg0NzM2LnBuZw?x-oss-process=image/format,png)

整个过程如果两个Hash对不上，或者公钥与私钥签名对不上，那么这个交易就是错误的，非法的

实例：

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwNjI2MTg0OTAyLnBuZw?x-oss-process=image/format,png)

**重点：**

> **两个保证：**
>
> **1. 输入中的公钥和上一个输出的公钥的hash进行校验，使input与output连接起来，保证使用者的身份的统一**
>
> **2. 输出入中的私钥签名与输出的公钥进行验证，保证使用者使用此笔钱的权利，必须本人签名了这个input才能被使用**

## 2. 项目中的逻辑

采用P2PKH的校验方式，输入要包含公钥和私钥签名，而输出则需要包含公钥的hash

公钥的hash可以通过地址倒推：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002110121.png)

这里的公钥hash并不是简单的最原始公钥做一次SHA256，从图中可以看出还经过了RIPEMD160的加密

代码：

```go
//地址转其公钥的hash函数
//（地址是由公钥计算过来的, 可以逆推回去到公钥的hash，但是无法逆推到原公钥，原公钥无法逆推到私钥，因为hash函数不可逆）
func (Output *TxOutput)Lock(address string)  {
   // 1.base58函数的解码
   bytes25Data := base58.Decode(address)
   // 2.去除尾部添加的4byte校验码和首部添加的1byte版本号
   addressHash := bytes25Data[1: len(bytes25Data)-4]
   // 3.赋值给Output
   Output.PubKeyHash = addressHash
}
```

# 十一、地址String到公钥hash的分场景安全策略

1. 在NewTransaction函数中，我们使用地址，不采用反推公钥Hash，因为涉及转账，此地址不一定是自己钱包中管理的地址，所以需要通过在本地钱包中读取公私钥，并且读取的另一个目的就是接下来要使用私钥
2. 在getBalance函数中，获取某个账户的余额，需要使用地址查询UTXO，此时我们可以使用函数反推的方式去查询，而不需要查询本地钱包。因为对于区块链系统来说，查询余额功能是全网都可以使用的，不仅限于本地钱包账户。



# 十二、对于地址使用之前的校验细节

不论是getBalacne还是Send转账，都需要对用户输入的地址进行校验。

因为通过string逆推得到公钥hash的方式会截去尾部的4字节还有前面的四字节，所以即使是后面几位不同的地址不做检测就查询的话可能会查出一样的结果

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002110121.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002143318.png)

所以在查询之前一定要做校验

**校验的原理来自于那个四字节的分支！**

校验流程思路:  （先反向走在正向走岔路回来）

1. 根据地址反推出25byte的数据。截断后4字节得到21字节的数据
2. 将这21byte数据进行两次SHA256，再截取4字节的校验码
3. 通过校验码与原本的25字节数据后四位比对

代码：

```go
//校验地址
//校验流程思路:  （先反向走在正向走岔路回来）
func checkAddress(address string) bool {
   //1. 根据地址反推出25byte的数据。截断后4字节得到21字节的数据
   bytes25Data := base58.Decode(address)
   if len(bytes25Data) < 4 {     //地址长度不够直接返回
      return false
   }
   this4Bytes := bytes25Data[: 4]
   //2. 将这21byte数据进行两次SHA256，再截取4字节的校验码
   org4Bytes := CheckSum(adsToPubKeyHash(address))
   //3. 通过校验码与原本的25字节数据后四位比对
   return bytes.Equal(this4Bytes, org4Bytes)
}
```

加上校验后效果:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002151153.png)



# 十三、 签名验证

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002155201.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002155409.png)

## 1. 签名需要的内容：

* 被签名的数据
* 私钥

## 2. 验证需要的内容 :

* 已签名的数据
* 公钥
* 数字签名

## 3. 注意事项

1.  一个交易中同一个人的每一个input都需要签名

2. ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002160334.png)

   具体签名的数据需要能够包含整个交易详细内容

3. 验证时也是每个input都要验证一次

4. **签名是由创建交易的节点完成，而校验是验证交易的节点完成**

## 4. 对于每个input的签名过程

每个input都有其对应的唯一的output，复制一份input，获取其output的pubKeyHash赋值到input的pubKey中

对于每个交易中的input其自己生成的output就在同交易中并且其中自带pubKeyHash和转账金额

对这个整体交易做hash，赋值到input的签名中

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201002213923.png)

**1. 为什么不直接用创建新交易时打开的钱包中的公钥去计算签名hash，而是使用如此复杂的过程去寻找前一个hash？**

因为直接用那个公钥签名的话那么验证一定是成功的，因为公私钥是一起那出来的。根据input中的前一个交易hash和outputindex找出来其关联的output的pubKeyhash才是正确的做法，也是为了后面的验证。

**2. 注意对于每个input的签名，在当前交易中与其无关的其他input的pubKey和ScriptSig都应该是空**

代码：

```go
//签名的实现
//参数：账户的私钥
func (tx *Transaction) Signature(privateKey *ecdsa.PrivateKey, bc *BlockChain)  {
   // 1.复制一份input，获取其output的pubKeyHash赋值到input的pubKey中(只是为了计算签名)
   trimmedCopyTx := TrimmedCopy(tx, bc)
   //对于每个交易中的input其自己生成的output就在同交易中并且其中自带pubKeyHash和转账金额
   // 2.对这个整体交易做hash，赋值到input的签名中
   for i, input := range trimmedCopyTx.Vin{
      //2.1 找到每个input关联的上一个output的公钥hash，并添加到当前的input的pubKey中
      //获取上一个交易
      preTx, err := FindTxByTxHash(input.TxHash, bc)
      if err != nil{
         log.Panic("签名时查找相关输出交易出错！", err)
      }
      //获取该交易中的output中的公钥hash
      prePubKeyHash := preTx.Vout[input.OutputIndex].PubKeyHash
      //注意！在这里直接对input赋值是无效的！！！
      trimmedCopyTx.Vin[i].PubKey = prePubKeyHash
      //2.2 签名需要的数据都具备了,做hash处理
      trimmedCopyTx.SetTxHash()  //交易的hash就是需要的签名数据
      signDataHash := trimmedCopyTx.TxHash
      //2.3 重要的一步！把当前交易中的这个input的pubKey还原为空，保证不影响其他input的签名
      trimmedCopyTx.Vin[i].PubKey = nil
      //2.4 执行签名动作得到r，s字节流
      r, s, err := ecdsa.Sign(rand.Reader, privateKey, signDataHash)
      if err != nil{
         log.Panic(err)
      }
      //2.5 把签名放到原本交易的ScriptSig中
      signnature := append(r.Bytes(), s.Bytes()...)
      tx.Vin[i].ScriptSig = signnature
   }
}
```

签名的验证

```go
//验证交易
func (tx *Transaction) Verify(bc *BlockChain) bool {
   if tx.isCoinbaseTx(){
      return true          //铸币交易无需验证
   }
   //1. 获取验证所需要的数据
   // 1.1 Data
   trimmedCopy := tx.TrimmedCopy(bc)
   for i, input := range tx.Vin{  //注意，遍历的是原本的交易
      preTx, err := FindTxByTxHash(input.TxHash, bc)
      if err != nil{
         log.Panic(err)
      }
      trimmedCopy.Vin[i].PubKey = preTx.Vout[input.OutputIndex].PubKeyHash
      //计算hash
      trimmedCopy.SetTxHash()
      //a. Data得到
      dataHash := trimmedCopy.TxHash
      // 还原
      trimmedCopy.Vin[i].PubKey = nil
      //b. 签名得到
      signature := input.ScriptSig
      //c. 公钥
      //拆解PubKey， X, Y得到原生公钥
      PubKey := input.PubKey

      //拆开签名，得到r和s
      r1 := big.Int{}
      s1 := big.Int{}
      //r是前半部分，s是后半部分
      r1.SetBytes(signature[:len(signature)/2])
      s1.SetBytes(signature[len(signature)/2:])

      //拆开公钥，得到x和y
      x := big.Int{}
      y := big.Int{}
      //r是前半部分，s是后半部分
      x.SetBytes(PubKey[:len(PubKey)/2])
      y.SetBytes(PubKey[len(PubKey)/2:])
      //得到公钥原型
      pubKeyOrigin := ecdsa.PublicKey{elliptic.P256(), &x, &y}

      //1.2 verify
      if !ecdsa.Verify(&pubKeyOrigin, dataHash, &r1, &s1){
         return false         //一旦有一个input验证错误就失败
      }
   }
   return true
}
```

# 十四、项目总结

## 1. 项目完成的功能

* 区块的创建
* 交易的打包
* 用户余额的查询
* pow挖矿算法
* 钱包功能，增加账户、管理账户等
* 转账功能
* 公私钥签名以及验证

## 2. 项目的待完善地方（缺点）

* 每个区块都是只能打包一个交易就直接发布了，没有区块链网络体系去获取交易
* 分布式网络共识协议没有实现
* 梅克尔树root的计算，目前项目只是简单的拼接字节
* 签名机制待完善，项目使用的签名验证方式是P2PKH，还有P2SH、多重签名等可以完善
* 远程访问rpc调用，类似于geth的远程访问
* 客户端的构建

**目前的项目只完成到数据层**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005100923.png)





# 十五、项目代码地址

https://github.com/xwjahahahaha/simpleBitCoin/tree/main/version9

# Tips

## 1.Go语言语法

1. 在同一个包下的go文件不需要使用import导入
2. 使用命令`go run xxx.go`命令运行main主函数的时候，如果同个包下的go文件互相调用函数，单独`go build main.go`（编译），`go run main.go`（运行）是不对的：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200913205823.png)

解决办法：

1. linux下：`go build *.go` `go run *.go`
2. windows下: `go build./` `go run ./`

## 2.Goland的使用技巧

`ctrl + b` 进入查看函数实现

## 3.Blot中要注意的细节

bucket.Put()方法如果bucket不存在那么就是直接的添加，但是如果已经存在了，那么就是**更新**，所以不存在Key重复的问题

## 4.go结构体内字段命名不规范导致使用gob出错！！！！Golang大坑！

注意：go语言中的结构体内的字段都必须**首字母大写**，不然会报如下错误：

```go
gob: type main.Person has no exported fields
```

**记住，Golang的结构体命名字段必须要大写，不然<u>外部无法无法调用</u>这个字段，序列化都可能不行！！！！！**

## 5. go循环修改数组值无效

注意Go语言和python一样循环修改数据的值必须使用索引！！！！

例: 错误示范:

```go
for _, output := range outputArray{
   output.TxHash = newTxHash
}
```

正确做法:

```go
for i := range outputArray{
   outputArray[i].TxHash = newTxHash
}
```

## 6. 关于golang.org/x包问题

由于谷歌被墙，跟谷歌相关的模块无法通过go get来下载，解决方法：

```shell
git clone https://github.com/golang/net.git $GOPATH/src/github.com/golang/net

git clone https://github.com/golang/sys.git $GOPATH/src/github.com/golang/sys

git clone https://github.com/golang/tools.git $GOPATH/src/github.com/golang/tools

ln -s $GOPATH/src/github.com/golang $GOPATH/src/golang.org/x
```

上面三条命令会把所要用到的官方辅助包都下载到`$GOPATH/src/github.com/golang中，在windows下软连接不好弄，我的方法就是下载好了以后把这些复制一份到``$GOPATH/src/golang.org/x`下，使用的时候就优先使用golang.org/x下的包。

## 7. 使用gob进行编码

使用gob进行编码的时候，如果字节流中或者自定义的结构有interface（）对象那么需要提前注册

编码解码的时候都需要添加！

## 8.panic: crypto: requested hash function #9 is unavailable

类似于这样的错误无非两个错误导致:

* 没初始化crypto对应的包

  详细解决: https://blog.csdn.net/test1280/article/details/106678075

* 包使用错误

  一些加密算法只有在`golang.org/x/crypto`中有,例如RIPEMD60

  解决: 安装: `go get -t golang.org/x/crypto/ripemd160`