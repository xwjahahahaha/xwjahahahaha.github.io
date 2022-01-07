---
title: Web3-js的学习(4)-实现自动账户解锁脚本
tags:
  - block_chain
  - web3js
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-14 20:50:53
---

# 解锁账户

每次转币都需要事先把账户解锁，不然会报错，非常的麻烦。

可以将解锁的过程加到web3js的脚本中来处理。

<!-- more -->

但是其中还是有一些细节要注意：

1. **首先在开启私链的时候需要在命令中赋予rpc调用personal这个api的权限**，这一点至关重要。
  
  因为默认不带api参数一般只提供eth、net等api，personal、db、admin等是不会向用户打开的，需要手动打开。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814223353.png)

  `geth --datadir . --networkid 15 --rpc --rpcapi "eth,personal" console 2>output.log`

2. 在需要解锁才能调用的函数之前，使用如下代码：

  `web3.personal.unlockAccount(web3.eth.accounts[0], 'XXX(密码)', (err,res)=>{(回调函数)})`

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814224421.png)

  这里需要注意的是，**解锁成功后的函数调用需要放在回调函数中才可**。因为这是异步的。


# 实例

对于转代币合约的优化：

```js
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));

var arguments = process.argv.splice(2);
var _from = arguments[0];	//发币者也是合约函数调用者
var _to = arguments[1];
var _value = arguments[2];
var _password = arguments[3];	//用户的账户密码

//创建合约实例
var abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balances","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"toAdd","type":"address"},{"name":"amount","type":"uint256"}],"name":"send","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[{"name":"initalSupply","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"name":"from","type":"address"},{"indexed":false,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Sent","type":"event"}];
var CoinContract = web3.eth.contract(abi);
var contractAddress = '0x767b276a86d36b66830a95720513e26a0773ca0c';
var contractInstance = CoinContract.at(contractAddress);

//解锁用户
web3.personal.unlockAccount(_from, _password, (err,res)=>{
	if (err)
		console.log('Error: ', err);
	else {
		//成功解锁才可转币
		//调用转币函数
		contractInstance.send(_to, _value, {from: _from}, (err,res)=>{
			if (err)
				console.log('Error: ', err);
			else
				console.log('Result: ', res);
		})
	}
		
});

```

