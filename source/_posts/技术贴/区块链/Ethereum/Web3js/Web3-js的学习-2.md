---
title: Web3.js的学习(2)
tags:
  - block_chain
  - web3js
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-11 19:37:29
---

# Web3.js的学习(2)

## 常用API-基本信息查询

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811193706.png)

<!-- more -->

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811193847.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811193933.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811203345.png)

## Web3通用工具方法

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811204458.png)

## Web账户相关

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811205257.png)
 
 ## 区块相关信息查询

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811205647.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811205945.png)

## 交易相关

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811211016.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811211513.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811211551.png)

## 消息调用

消息调用就是调用那些不改变状态的合约函数，此时不使用交易调用而应该使用消息调用

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811212829.png)

## 日志过滤

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811213151.png)

小例子：

- 检测出块-latest

  Node下

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812194347.png)

  geth控制台下，启动挖矿：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812194407.png)

  Node下收到消息：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812194514.png)

- 检测交易-pending

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812195012.png)

  geth控制台下发起一个交易：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812195335.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812195431.png)

  pending只管当前在pending的交易，至于之后交易是否进块是不管的。

## 合约相关

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812200633.png)

部署合约实例：

  合约内容（C就是一个简单的更新余额的合约,D合约可以不看，这里使用的也是c合约）：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812221357.png)


  ```js
  var Web3 = require('web3') //引入
  var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545')) //创建对象
  var abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balances","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_value","type":"uint256"}],"name":"update","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"}]
  var binData = '0x' + '608060405234801561001057600080fd5b5061015f806100206000396000f30060806040526004361061004c576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806327e235e31461005157806382ab890a146100a8575b600080fd5b34801561005d57600080fd5b50610092600480360381019080803573ffffffffffffffffffffffffffffffffffffffff1690602001909291905050506100d5565b6040518082815260200191505060405180910390f35b3480156100b457600080fd5b506100d3600480360381019080803590602001909291905050506100ed565b005b60006020528060005260406000206000915090505481565b806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002081905550505600a165627a7a72305820f53a984234b2bc55c9abf6a21d2129c609ca27a289988a1832a4513610af648d0029'
  var MyContract = web3.eth.contract(abi)
  var contractInstance = MyContract.new({data:binData, from:web3.eth.accounts[0], gas:90000})
  console.log(contractInstance)
  ```
  abi和字节码都是通过solcjs编译合约得到(详细可看第一节-Web3.js的学习(1))：

  `solcjs --bin demo.sol` 和 `solcjs --abi demo.sol`

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812203036.png)

  注意点：**二进制的字节码前面要加上0x前缀！**，**账户需要解锁，不然报错**：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812203330.png)

  gas不能太大超过区块的总gas，不然报错：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812203452.png)

  gas不能太小，不足以发布合约：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812204409.png)

  调整适当的gas后成功得到合约对象：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812204900.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812204928.png)

  检查日志文件：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812205042.png)

  与返回的交易hash是一样的。

  但是合约未入块，所以还没有地址。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812205127.png)

  挖矿后，查询地址：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812220922.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812220937.png)

调用合约

  注意： 只有被写到区块链中的合约才能够调用。否则会显示无此函数。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812220655.png)

  实例：采用第一种自动决定函数类型调用方式：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812221249.png)

  因为这个函数是view，所以是call调用，不会产生任何交易记录。与call调用无差别：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812221835.png)

  使用sendTransaction调用update函数：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812223524.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812223559.png)

  因为是改变状态，所以一定需要发起交易。可以在log中查看到发布的交易号是对应的：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812223618.png)

  注意：**函数的调用都需要指明from，否则会报错invalid address！**，**时刻检查账户是否被锁上了，有时候调用函数一直undefind，可能就是时间过长账户又被锁上了。**

  挖矿，交易写入区块。

  最后检查余额：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200813181859.png)

监听合约事件

  原来的合约中没有定义event事件，所以先修改下合约内容，然后重新部署到区块链上。

  ```js
  pragma solidity >=0.4.0;

  contract C{
      event ChangeBalance(address indexed who, uint indexed value);
      mapping(address=>uint) public balances;
      function update(uint _value) public {
          balances[msg.sender] = _value;
          emit ChangeBalance(msg.sender, _value);
      }
  }
  ```

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812214838.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814165355.png)

  监听的是latest，当挖矿开始时，就会返回数据：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814165528.png)



  
  


