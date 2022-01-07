---
title: Web3.js的学习(1)
tags:
  - block_chain
  - web3js
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-10 21:19:55
---

# Web3.js的学习(1)

## Web3js的简介与安装

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810212353.png)

<!-- more -->

geth控制台已经内嵌了一个web3，所以我们可以在控制台使用其命令。

那么我们不能永远在geth控制台上使用web3，所以还是需要自己安装：

`npm install web3@0.20.1`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810215030.png)


例如MetaMask启动后就会在浏览器上创建一个Web3实例的provider，在控制面板中可以看到：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810214409.png)

并且在remix中选择环境时的选项也是有含义的：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810214840.png)

所以我们**在创建provider的时候需要先检测环境中是否已经有了provider，以免覆盖**

## 异步回调

最初时，web3jsAPI在本地使用，所以是同步调用，而现在很多都是异步调用。

  - 同步调用会阻塞进程直到返回结果。
  - 异步迪奥不会阻塞进程，但是需要写回调函数逻辑。

**同步调用与异步调用现在都可以使用，区分点在于是否在后面加上回调函数**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810220531.png)

例子：

同步调用：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810220704.png)

异步调用：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810221036.png)

注意到内容是一样的。

**web应用大部分都需要的是异步调用，因为同步调用导致阻塞，这对Web来说往往是不好的，例如因为一部分内容的阻塞而导致整个网页完全显示不出来**

目前高版本的Web3js基本上都会使用异步调用。

## 回调promise时事件

执行了一个请求会产生不同的阶段，例如发起了一个交易，其后会发生：

  - 产生交易ID
  - 交易写入区块
  - 区块被确认
  - 区块成为主链

这样不同的阶段需要执行不同的功能，那么promise就可以很方便的实现

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810222221.png)

## 应用二进制接口(ABI)

还记得智能合约的工作流吗：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200727215450.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200727220429.png)


在js上仅仅有合约的地址是无法知道合约的内容的

所以ABI其实就是智能合约的json描述，智能合约的标准接口，使用这个就可以将智能合约转换成为应用程序中的对象实例来使用。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810224110.png)

例子：

编译准备好的合约文件：

`solcjs --abi demo.sol`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810224754.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810224745.png)

我这个文件中有两个合约编译后就变成了两个abi。

打开其中一个abi文件查看内容:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810230417.png)

这就是json描述，有了这些js就可以把合约转换为js对象，在js中使用了。

我们也可以看看写到区块链中的字节码是什么样子的：

`solcjs --bin demo.sol`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200810231212.png)

这些字节码就是要提交到以太坊的内容，以太坊再用这些字节码将合约创建出来。

## web3的小实践（版本0.20.1）

之前都是在geth控制台上使用其自带的web3，现在我们用js文件来使用web3.

前提：安装web3模块。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811004831.png)

node: `node web3_test.js`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811005023.png)

出现web3对象的内容即成功。

看看是否连接到私链

`web3.isConnected()`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811010929.png)

注意，geth客户端开启的时候要加rpc参数

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811011156.png)

测试：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811011216.png)


## 批处理请求

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811185702.png)

## 大数处理

因为以太坊中没有小数，采用wei来处理，所以经常会有很大的数，而js面对大数会采用科学计数法来表示，会损失一定的精度，这对数字货币来说是绝对不允许的。所以，以太坊中的大数都需要使用BigNumber包来处理。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811193505.png)

例子：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811193029.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811193421.png)



