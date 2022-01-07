---
title: Web3-js的学习(3)-实现简单转币脚本
tags:
  - block_chain
  - web3js
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-14 17:10:13
---

# 简单转以太币脚本

## 代码

```js
//创建web3对象
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));
//获取node参数
var arguments = process.argv.splice(2);
var _from = arguments[0];
var _to = arguments[1];
var _value = arguments[2];
//转币e
web3.eth.sendTransaction({from: _from, to: _to, value: _value}, (err,res)=>{
	if (err) console.log('Error: ', err);
	else console.log('Tx ID: ', res);
});
```
<!-- more -->

## 运行

先检查当前的账户余额：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814172222.png)

注意：转账之前先解锁账户

使用脚本：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814172658.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814172913.png)

生成的交易ID与log相同。

此时挖矿，注意为了不影响转币的效果，采用第三个账号挖的矿。这样挖矿奖励与转账的两个账户就无关了。

检查两账户的余额变动：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814173200.png)

检查下交易详细信息：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814173359.png)

成功转了5Gwei。

## 升级

可以添加错误检测增加鲁棒性：

```js
//创建web3对象
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));
//获取node参数
var arguments = process.argv.splice(2);
if (!arguments || arguments.length != 3){
	console.log('Parameter Error');
	return;
}
var _from = arguments[0];
var _to = arguments[1];
var _value = arguments[2];
if (!web3.isAddress(_from) || !web3.isAddress(_to)){
	console.log('Parameters not a address');
	return;
}
//转币e
web3.eth.sendTransaction({from: _from, to: _to, value: _value}, (err,res)=>{
	if (err) console.log('Error: ', err);
	else console.log('Tx ID: ', res);
});
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814174750.png)

# 简单转合约代币脚本

首先，合约要部署到私链或公链上，得到合约的地址。

合约内容：
```js
pragma solidity ^0.4.22;

contract Coin{
    mapping(address=>uint256) public balances;
    event Sent(address from, address to, uint256 amount);
    constructor(uint256 initalSupply) public{
        balances[msg.sender] = initalSupply;
    }
    
    function send(address toAdd, uint256 amount) public returns(bool success){
        require(balances[msg.sender] >= amount);
        require(balances[toAdd] + amount >= balances[toAdd]);
        balances[msg.sender] -= amount;
        balances[toAdd] += amount;
        emit Sent(msg.sender, toAdd, amount);
        return true;
    } 
}
```

部署代码：

```js
var Web3 = require('web3') //引入
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545')) //创建对象
var abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balances","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"toAdd","type":"address"},{"name":"amount","type":"uint256"}],"name":"send","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[{"name":"initalSupply","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"name":"from","type":"address"},{"indexed":false,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Sent","type":"event"}]
var binData = '0x'+ '608060405234801561001057600080fd5b506040516020806103f483398101806040528101908080519060200190929190505050806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020819055505061036e806100866000396000f30060806040526004361061004c576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806327e235e314610051578063d0679d34146100a8575b600080fd5b34801561005d57600080fd5b50610092600480360381019080803573ffffffffffffffffffffffffffffffffffffffff16906020019092919050505061010d565b6040518082815260200191505060405180910390f35b3480156100b457600080fd5b506100f3600480360381019080803573ffffffffffffffffffffffffffffffffffffffff16906020019092919080359060200190929190505050610125565b604051808215151515815260200191505060405180910390f35b60006020528060005260406000206000915090505481565b6000816000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020541015151561017457600080fd5b6000808473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002054826000808673ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002054011015151561020157600080fd5b816000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002060008282540392505081905550816000808573ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600082825401925050819055507f3990db2d31862302a685e8086b5755072a6e2b5b780af1ee81ece35ee3cd3345338484604051808473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1681526020018373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001828152602001935050505060405180910390a160019050929150505600a165627a7a72305820670dd25adb0e8896d00f58a9001a182d581a159163cbd3435581b269b137fb6a0029'
var MyContract = web3.eth.contract(abi)
//注意，构造函数有参数
var contractInstance = MyContract.new(90000, {from:web3.eth.accounts[0], data:binData, gas:999999})
```

部署后得到交易id，再挖矿上链得到合约地址。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814202127.png)

调用合约转币脚本

```js
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));

var arguments = process.argv.splice(2);
var _from = arguments[0];	//发币者也是合约函数调用者
var _to = arguments[1];
var _value = arguments[2];

//创建合约实例
var abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balances","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"toAdd","type":"address"},{"name":"amount","type":"uint256"}],"name":"send","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[{"name":"initalSupply","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"name":"from","type":"address"},{"indexed":false,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Sent","type":"event"}];
var CoinContract = web3.eth.contract(abi);
var contractAddress = '0x767b276a86d36b66830a95720513e26a0773ca0c';
var contractInstance = CoinContract.at(contractAddress);

//调用转币函数
contractInstance.send(_to, _value, {from: _from}, (err,res)=>{
	if (err)
		console.log('Error: ', err);
	else
		console.log('Result: ', res);
})
```

同样的，先检查当前各个账户的余额：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814202312.png)

账户一：90000 （构造函数的初始值）
账户二：0

这里也可以写一个简单的查询余额脚本：

```js
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));

var arguments = process.argv.splice(2);
var _who = arguments[0];	//要查询的人

//创建合约实例
var abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balances","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"toAdd","type":"address"},{"name":"amount","type":"uint256"}],"name":"send","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[{"name":"initalSupply","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"name":"from","type":"address"},{"indexed":false,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Sent","type":"event"}];
var CoinContract = web3.eth.contract(abi);
var contractAddress = '0x767b276a86d36b66830a95720513e26a0773ca0c';
var contractInstance = CoinContract.at(contractAddress);

//查询
contractInstance.balances(_who, {from:_who}, (err,res)=>{
	if (err)
		console.log(err);
	else
		console.log(res.toString(10));
})
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814202618.png)

生成交易ID，挖矿，然后查看余额：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200814202725.png)

成功转代币5万。

进一步的优化：

[自动解锁](https://myblog.gumptlu.work/2020/08/14/%E6%8A%80%E6%9C%AF%E8%B4%B4/%E5%8C%BA%E5%9D%97%E9%93%BE/Web3-js%E7%9A%84%E5%AD%A6%E4%B9%A0-4-%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%8A%A8%E8%B4%A6%E6%88%B7%E8%A7%A3%E9%94%81%E8%84%9A%E6%9C%AC/)










