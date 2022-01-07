---
title: Web3-js的学习(5)-实现合约事件监听
tags:
  - block_chain
  - web3js
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-14 23:15:38
---

# 合约事件监听

latest： 监听最新出块事件

pending：监听发布未进块事件

代码很简单：

<!-- more -->

```js
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));

//创建合约实例
var abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balances","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"toAdd","type":"address"},{"name":"amount","type":"uint256"}],"name":"send","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[{"name":"initalSupply","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"name":"from","type":"address"},{"indexed":false,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Sent","type":"event"}];
var CoinContract = web3.eth.contract(abi);
var contractAddress = '0x767b276a86d36b66830a95720513e26a0773ca0c';
var contractInstance = CoinContract.at(contractAddress);

//调用监听
contractInstance.Sent('latest', (err,res)=>{
	if (err)
		console.log(err);
	else
		console.log(res);
})
```

开启后会一直监听：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814232236.png)

发布新交易：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814232450.png)

例子中使用的latest，所以进块了才会监听到，开始挖矿：

查看返回监听的结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814232721.png)

出现了交易的详细信息！并且还会一直监听。
