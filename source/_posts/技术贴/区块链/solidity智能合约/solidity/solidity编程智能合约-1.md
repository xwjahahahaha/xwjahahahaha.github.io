---
title: solidity编程智能合约(1)
tags:
  - solidity
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-07-19 22:48:53
---
# 基础概念

## 以太币单位

wei 是以太坊最小的单位

1 ether = 10^18 wei  

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720082955.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720083518.png)

转钱时的金额不能太小，不然连手续费都比转账贵。

<!-- more -->

## 以太坊钱包

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720083732.png)

公钥、私钥、地址：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720084301.png)

注意：平时用到的简单的随机数生成器都是伪随机的，尽量不要自己使用随机数作为私钥来生成账户。

助记词：

有的钱包为你的账户私钥给出对应的若干个助记词，助记词的作用就与私钥相同，所以也一定要保管好。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720153716.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720154547.png)

## 切换网络

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720155225.png)

一些常用的网址：

查询以太坊交易gas price标准的网站：https://ethgasstation.info/

要合理的设置交易的汽油费单价才能够使得交易被快速的打包。

查交易信息的平台：https://cn.etherscan.com/

# remix编辑器

在线编译智能合约的平台remix：https://remix.ethereum.org/

## MetaMask + remix在测试网络中部署合约

1. 首先登陆MetaMask账号，确保账号中有一定的以太币（用来支付合约部署的汽油费）
2. 合约写完了以后部署时选择injected Web3
  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200721165828.png)
3. remix会自动检测到你使用的测试网络类型，以及你的账户，自己再核对一下。
4. 合约无误的话就点击部署吧。
5. 部署成功后会在Deployed Contracts中显示已部署的合约

ps：remix还会贴心的把合约中的函数给显示成按钮，且参数可以通过输入框输入，真是良心测试，太好用了。


# 智能合约

## 智能合约语法知识点

Solidity是面向对象的语言。

1. 开头一般都需要指定编译器的版本，具体代码：`pragma solidity 版本要求;`

2. `contract 合约名{}`类似于java的类声明，声明一个合约。

### 一些固定写法/关键字

`msg.sender`是指调用这个合约的人发来的消息中的发送者地址，也就是指这个合约的调用者地址。


转币的方式和特点：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200718162208.png)


## 第一个合约Helloword

导入

`pragma solidity ^0.4.0;`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200719225158.png)

solidity中四种函数权限的标识符：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200719230032.png)

代码内容：

```
contract helloworld {
    string myword = "helloworld";
    
    function show() public view returns(string){
        return myword;
    }
}
```

发布成功后的信息：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200720184014.png)


## 自己写一个水龙头合约

1. 合约内容：

```
pragma solidity ^0.4.0;

contract Faucet{

    function getOne(uint amount) public {
        require(amount <= 10000000000000000000);

        msg.sender.transfer(amount);
    }
    
    function () public payable{
        
    }
}
```
2. 部署后要向这个余额账户先转钱，然后才能提取出来钱

注意： 记得写上fallback函数且有payable，不然向这个合约转钱会报错。





