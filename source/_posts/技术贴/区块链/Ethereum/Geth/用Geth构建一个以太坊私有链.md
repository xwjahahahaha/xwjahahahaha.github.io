---
title: 用Geth构建一个以太坊私有链
tags:
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-07-21 21:21:49
---

# 客户端基础概念

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721194437.png)

<!-- more -->

以太坊网络：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721201804.png)

客户端：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721202053.png)

全节点：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721203053.png)

运行一个全节点的要求：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721204246.png)


# Geth（Go-Ethereum）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721204508.png)

## 安装Geth

1. 可以使用以下的国内镜像下载`https://ethfans.org/wikis/Ethereum-Geth-Mirror`
2. 安装exe文件
3. 安装完之后记得添加到环境变量

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721214120.png)

这样就是ok的。

## 克隆一份当前的go-ethereum源码

`git clone https://github.com/ethereum/go-ethereum.git`到本地即可。

查看当前交易信息：`git log`

查看版本`git tag`

下载新版本`git checkout v新版本号`

# 启动同步节点

使用命令`geth --datadir .--syncmode fast`来快速同步区块。具体快速的方式是下载每个区块头和区块体，但是不去验证所有的交易，知道所有的区块都同步完毕在去获得一个系统的当前状态。节省的就是验证交易的时间。其中的fast参数还可以换成full表示完整下载所有区块并验证交易，light表示只下载所有头结点。

如果想同步测试网络的区块可以用`geth --testnet --datadir .--syncmode fast`这个测试网络就是Ropsten。

# 搭建自己的私链

第一步：确定私链参数

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722163922.png)

json代码：
```json
{
	"config": {
		"chainId": 15
	},
	"difficulty": "2000",
	"gasLimit": "2100000",
	"alloc": {
		"0xe6e3c70727869D7ccEB4f8006827D8cB4D0b41f1": {"balance": "30000000000000"}
	}
}
```
保存为genesis.json

说明： 其中的chainId就代表了这条私链的网络号，difficulty代表的是挖矿难度，gasLimit代表规定初始块中所有交易消耗汽油的上限。alloc设置一些初始账户和其状态。这里设置了一个初始的账户，并且设置其余额为30000000000000Wei。（这个操作类似于预挖矿，引投资）

第二步：创建初始块Genesis

使用命令`geth --datadir 保存的文件路径 init genesis.json`

成功后：
![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722170302.png) 


第三步：启动私链

进入创建的目录（就是第二步保存的文件路径的文件夹），使用命令`geth --datadir . --networkid 15` 注意参数networkid要与你设置的参数相同。--datadir .表示是当前目录。

启动后与启动主网大致相同

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722195458.png)

**注意：chainId显示的是15才是启动的是自己的私链**

# 与私链交互

## 操作简介

在启动私链的时候就可以在后面加一个console启动控制台交互：
`geth --datadir . --networkid 15 console`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722200009.png)

在这里就可以用js与私链交互了。

也可以在后面添加参数，将输出重定向到外部文件(output.log):
`geth --datadir . --networkid 15 console 2>output.log`

这样界面就会清爽很多。

在linux环境下可以通过`tail -f output.log`实时的查看log文件的输出，在windows下则可以通过notepad++软件设置自动更新文件来实现。


> 输入`web3`可以看到最大的对象web3

> 输入`admin`可以看到一些节点管理员信息

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722200705.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722200829.png)

> 输入`eth`可以看到这条私链相关的一些信息：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722201025.png)

**注意：对于eth中accounts不包含先前设置参数中的账户，只有在keystore中有私钥的账户才会存储在这里。**

keystore中保存的都是加密存储的私钥。

> 输入`personal`可以看到和账户相关的一些信息

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722201223.png)

> 同样的还可以查看`txpool`、`miner`等信息

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722201435.png)

## 操作实例

### 1.查看私链中某个账户中的余额

查看初始账户的余额：

操作：`eth.getBalance("0xe6e3c70727869D7ccEB4f8006827D8cB4D0b41f1")`

结果：
![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722203218.png)

将其转换为eth为单位

操作：`web3.fromWei(eth.getBalance("0xe6e3c70727869D7ccEB4f8006827D8cB4D0b41f1"), "ether")`

结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722203903.png)

### 2. 创建一个新用户

操作：`personal.newAccount("password")`

不写参数就会提示输入密码，**keystore就是根据这个密码对私钥进行加密。**

确认密码后就会新生成一个地址：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722204455.png)

再通过`eth.accounts`就可以查看到新增的账户

在keystore文件夹中可以看到新创建的文件。用此文件+密码就可以还原出账户的私钥。

看一下余额：`eth.getBalance(eth.accounts[0])`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722205144.png)

### 3. 转账

从地址1 -> 地址2

直接转账在geth上是不允许的。需要先对账户unlock解锁。
操作：`personal.unlockAccount("地址1")`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722210212.png)

操作：`eth.sendTransaction({from: "地址1", to: "地址2", value: xxx})`

注意：发起转账必须是一个本节点管理的已有的账户地址，例如初始账户没有其私钥是无法完成转账的。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722211118.png)

此外，发起转账的账户必须是要先unlock状态。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722211853.png)

账户没钱当然也发布出去
![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722212100.png)

> 对于初始账户怎样将其私钥加入到keystore中，使之成为一个可管理的账户？
>
>   使用`personal.importRawKey("xxxxx")`引入
>

如果转账金额和账户解锁都完成了，那么转账就可以实现：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200725203841.png)

生成了交易hash，但应该注意：**转账并未实现，因为还没有上链。**

经过挖矿后得到对应的转账的钱：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200725204438.png)

### 4. 挖矿

转账没钱咋办 --> 挖矿

需要指定一个账户，不指定账户默认是eth.accounts[]数组中的第一个地址来挖矿。

指定账户挖矿：`miner.setEtherbase(eth.accounts[1])`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722220033.png)

第一次启动挖矿会先生成挖矿所需的DAG文件，这个过程有点慢，等进度达到100%后，就会开始挖矿，此时屏幕会被挖矿信息刷屏。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722215719.png)


`miner.start(1)`其中的参数表示挖矿启动的线程数

开始挖矿：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722215815.png)

使用`miner.stop()`停止挖矿

挖完矿可以查看下区块的个数`eth.blockNumber` 

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722220957.png)

最重要的查下挖矿奖励！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200722235119.png)

因为以太坊刚开始的出块奖励是5ETH，创建私链时参数设置中也没有说明具体的出块奖励金额，所以出块奖励是每个块5ETH

### 5. 在Web中实现连接自己的私链网络

连接MetaMask：

MetaMask默认的可连接的localhost端口是8545，如果想实现在Web上查看就需要在启动私链的参数中加上-rpc（远程过程调用）

`geth --datadir . --networkid 15 --rpc console 2>output.log`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726170834.png)

成功后可以在控制台中看出。

> 如果想指定端口的话，可以在--rpc后面加上--rpcport xx。不加的话默认就是8545。

然后在MetaMask中选择网络，连接即可成功。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726171059.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726170958.png)

其实这里是使用了JSON RPC功能，具体详细内容在下面可以看到。

#### geth新版本的一些其他功能



##### Dev模式

启动加上参数--dev，使用POA权威证明模式。此模式不用自己创建的私链，直接会帮私链创建好，默认会创建好开发者账户。所以，可以在新建一个文件夹使用dev。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726172416.png)

`geth --datadir . --dev console 2>dev_output.log`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726172843.png)

对于初始账户有非常多的以太币
![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726173230.png)

并且初始账户不需要解锁就可以直接向其他账户发送以太币。

此外，**DEV模式下不需要手动挖矿，有交易生成会自动挖矿。**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726182843.png)

这种模式是为了测试所用，省去了繁琐的操作。

##### 查看一些详细信息

`eth.get`按下table：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726183232.png)


查看交易信息：

使用`eth.getTransaction("txid")`即可查看

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726183324.png)

查看块信息:

块中有几个交易：`eth.getBlockTransactionCount([blockNumber])`参数可以写区块的编号

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726183808.png)

查看区块完整信息：`eth.getBlock([blockNumber])`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726184027.png)

可以看到区块上的miner是没有的，测试环境下的POA是虚拟的。

## geth常用命令

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200725201258.png)


## 外部访问私链数据 

[JSON-RPC](https://myblog.gumptlu.work/2020/07/26/%E6%8A%80%E6%9C%AF%E8%B4%B4/%E5%8C%BA%E5%9D%97%E9%93%BE/JSON-RPC%E4%BD%BF%E7%94%A8/)