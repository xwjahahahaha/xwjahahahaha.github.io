---
title: solidity编程智能合约(2)
tags:
  - solidity
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-07-27 21:17:37
---

# 正式开始

## Solidity是什么
• Solidity 是一门**面向合约**的、为实现智能合约而创建的高级编程语言。这门语言受到了 C++ Python 和 Javascript 语言的影响，设计的目的是能在以太坊虚拟机（ EVM ）上运行。

• Solidity 是静态类型语言，支持继承、库和复杂的用户定义类型等特性。

• 内含的类型除了常见编程语言中的标准类型，还包括 address等以太坊独有的类型， Solidity 源码文件通常以 .sol 作为扩展名

• 目前尝试 Solidity 编程的最好的方式是使用 Remix 。 Remix是一个基于 Web 浏览器的 IDE ，它可以让你编写 Solidity 智能合约，然后部署并运行该智能合约。
<!-- more -->

## Solidity语言特性
Solidity的语法接近于 JavaScript ，是一种面向对象的语言。但作为一种真正意义上运行在网络上的去中心合约，它又有很多的不同：

•以太坊底层基于帐户，而不是 UTXO ，所以增加了一个**特殊的address 的数据类型**用于定位用户和合约账户。

•语言内嵌框架支持支付。**提供了 payable 等关键字**，可以在语言层面直接支持支付。

•**使用区块链进行数据存储。数据的每一个状态都可以永久存储，所以在使用时需要确定变量使用内存，还是区块链存储。**

•运行环境是在去中心化的网络上，所以需要**强调合约或函数执行的调用的方式。** 

•不同的异常机制。一旦出现异常，所有的执行都将会被回撤，这主要是为了保证合约执行的原子性，以避免中间状态出现的数据不一致。

## solidity工作流

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200727215450.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200727220429.png)

## Solidity编译器

**Remix** (ps：成熟的IDE)

•Remix 是一个基于 Web 浏览器的 Solidity IDE ；可在线使用而无需安装任
何东西

•http://remix.ethereum.org

**solcjs** (ps:初始的命令行)

•solc 是 Solidity 源码库的构建目标之一，它是 Solidity 的命令行编译器

•使用 npm 可以便捷地安装 Solidity 编译器 solcjs

•`npm install g solc`

## 一些简单的合约

### 1. 存取数据

```js
pragma solidity ^0.4.22;

contract SimpleStorage{
    uint256 storageData;
    function setData(uint256 newData) public{
        storageData = newData;
    }
    
    function getData() public view returns(uint256){
        return storageData;
    } 
}
```

### 2. 汽车合约
```js
pragma solidity ^0.4.22;

contract Car{
    bytes32 carName;
    uint carPrice;
    constructor(bytes32 name, uint price) public{
        carName = name;
        carPrice = price;
    }
    function setcarName(bytes32 newName) public{
        carName = newName;
    }
    function getcarName() public view returns(bytes32){
        return carName;
    }
    function setcarPrice(uint newPrice) public{
        carPrice = newPrice;
    }
    function getcarPrice() public view returns(uint){
        return carPrice;
    }
    function getcarInfo() public view returns(bytes32 name, uint price){
        return (carName, carPrice);
    }
    function () payable public{
        
    }
}
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200728203733.png)

注意: 建议不要直接使用string来表示字符串

因为string表示的字符很大，恶意攻击会导致消耗很多的gas。**一般采用bytes来代替string。**

使用了bytes输入时直接用字符串就会报错：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200728205351.png)

因为使用的是字符串类型而不是字符数组。

**bytes与string转换方式也是一般的使用方式：**

使用Web3中的两个函数： 

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200728210052.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200728211514.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200728211758.png)

注意：新版本的remix中输入bytes32不能省略0，即使后面都是0也要把32位补全，否则会报错。

> 一些修饰符解释：
> 1. view : 简单来说就是表示只可读，从函数中读取某些值
> 2. pure : 表示纯计算

constructor 表示初始化函数，在合约部署的时候就可以设置参数。注意，此函数编译器的版本需要>0.4.22。

### 3. 子货币合约

```js
pragma solidity ^0.4.22;

contract Coin{
    //发币人，币的创始人
    address public minter;
    //类似于map数据结构，指定从地址->余额(int)的映射
    mapping(address=>uint256) public balances;
    //定义一个log事件
    event Sent(address from, address to, uint256 amount);
    //构造函数，将合约的部署者设置为发币人
    constructor() public{
        minter = msg.sender;
    }
    //挖矿函数，只能发币人挖矿，参数：给谁钱，多少钱
    function mint(address toAdd, uint256 amount) public{
        //异常判断，指明身份
        require(msg.sender == minter);
        //注意这里不是交易，而是在合约中修改数据，并且这样直接+是有风险的
        balances[toAdd] += amount;
    }
    //转账函数，函数的调用者就是转账的发起人，参数：给谁钱，多少钱
    function send(address toAdd, uint256 amount) public{
        //先判断余额
        require(balances[msg.sender] >= amount);
        //在修改条件
        balances[msg.sender] -= amount;
        //发钱
        balances[toAdd] += amount;
        //打log
        emit Sent(msg.sender, toAdd, amount);
    }
}
```

对于这样的合约的问题：发币人的权利太大，可以无限发币，这样会破坏整个代币的使用。

改进的版本：

```js
pragma solidity ^0.4.22;

contract Coin{
    //不要miner了
    mapping(address=>uint256) public balances;
    event Sent(address from, address to, uint256 amount);
    constructor(uint256 initalSupply) public{
        //在部署时就将初始的代币发行量设置好
        balances[msg.sender] = initalSupply;
    }
    //不需要挖矿了
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

关于代币合约，有专门的一个合约设计标准：ERC20
详情可看：[ERC20接口下USDT代币的深入解析](https://blog.csdn.net/weixin_43988498/article/details/107750882)