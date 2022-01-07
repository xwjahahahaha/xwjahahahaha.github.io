---
title: DApp项目-基于truffle的简单投票合约-1
tags:
  - DApp
categories:
  - technical
  - DApp
toc: true
declare: true
date: 2020-09-03 00:58:40
---
# 简介

基于以太坊开发投票系统DApp，在基础投票功能的基础上，增加了基于自定义token进行投票的功能；另外还涉及到了以太坊开发框架truffle的使用。

通过一个完整的DApp的开发，将以太坊理论和实践紧密结合起来，可以使学习者对以太坊上的DApp开发有更加全面充分的认识，进而对整个区块链技术有更深刻的理解。

# 基础准备

需先掌握以下基础知识：

1. truffle框架环境的搭建
2. geth启动以太坊私链以及操作
3. web3js
4. solidity智能合约编程语言的掌握
5. 简单DApp开发流程

可以在博客中搜索到以上详细的知识

<!-- more -->

我的环境是windows7（Linux玩的不熟），具体学习资源来自[硅谷投票系统区块链以太坊尚硅谷](https://www.bilibili.com/video/BV1JJ411D7Ve?p=1)

# 开发步骤

## 打造自己的truffle-box

* 新建项目空文件夹，在文件夹下unbox webpack
* 清空webpack中不需要的东西

  删除掉两个自带的示例合约:

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903010953.png)

  把准备好的投票合约Voting加进去

  Voting.js:

  ```js
  pragma solidity >=0.4.0;

  contract Voting{
      bytes32[] public candidateList; //候选人数组
      mapping(bytes32 => uint) public votesReceived;  
      constructor(bytes32[] candidateListName) public{
          candidateList = candidateListName;
      }
      
      //检测是否为有效的投票地址
      function validateCandidate(bytes32 candidateName) internal view returns(bool){
          for(uint i = 0; i < candidateList.length; i++){
              if (candidateName == candidateList[i])
                  return true;
          }
          return false;
      }

      //投票函数
      function vote(bytes32 candidateName) public {
          require(validateCandidate(candidateName));
          votesReceived[candidateName] += 1;
      }

      //返回某个候选人的得票
      function totalVotesFor(bytes32 candidateName) public view returns(uint){
          require(validateCandidate(candidateName));
          return votesReceived[candidateName];
      }

  }
  ```

  然后修改migration文件夹下的`2_deploy_contracts.js`文件：

  ```js
  var Voting = artifacts.require("./Voting.sol");

  module.exports = function(deployer) {
    // 注意这里部署构造函数需要给初始值
    deployer.deploy(Voting, ['Alice', 'Bob', 'Cary'], {gas: 1000000});
  };
  ```

* 有build文件夹的话删除这个文件夹（build是输出文件夹）

上面的修改完，现在就是自己的工作空间了。

## 编译合约

`truffle compile`

注意windows下cmd直接使用此命令不可行，需采用git bash执行。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903013006.png)

warning指的是migration合约构造函数使用了老版本的同名方式，这个不用管。

完成后会生成build文件夹：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903013322.png)

其中的json就是合约对应的abi。

## 部署合约

`truffle migrate`

报错：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903014111.png)

因为连接的是自己geth启动的私链，部署合约没有先解锁账户，所以先解锁账户。

可以使用`truffle console`启动客户端来解锁账户：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903015004.png)

再次部署报错：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903015227.png)

修改构造函数中的gas，以及设置最小Gas限制（改大点）:

truffle.js:

```js
// Allows us to use ES6 in our migrations and tests.
require('babel-register')

module.exports = {
  networks: {
    development: {
      host: 'localhost',
      port: 8545,
      network_id: '*' ,// Match any network id
      gas: 470000 
    }
  }
}

```

2_deploy_contracts.js:

```js
var Voting = artifacts.require("./Voting.sol");

module.exports = function(deployer) {
  // 注意这里部署构造函数需要给初始值
  deployer.deploy(Voting, ['Alice', 'Bob', 'Cary'], {gas: 100000});
};

```
再次部署：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903015529.png)

这样卡住了，因为是个人私链，所以要去挖矿！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903015714.png)


![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903015839.png)

挖儿挖~

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903015947.png)

出块了就停止吧。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903020145.png)

第一个成功了，第二个又失败了。。。因为给的gas太小了，所以部署合约的汽油费一定要恰当！

修改2_deploy_contracts.js中的汽油费为：400000 或者直接删掉右面的{gas：xxx}使用默认的truffle.js中设置的汽油费

成功了：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903020514.png)

注意，第一个合约部署过后就不会再部署了。

truffle 部署的好处在于：**它会检测是否修改了合约，如果合约内容没有变化，那么部署成功后以后都不会去重复部署了。**





## 控制台使用合约

truffle包装了web3麻烦的根据abi和字节码部署一系列复杂的操作，在其控制台，对于已经部署的合约可以使用其特殊语法创建实例。（部署完合约之后，truffle就会帮我们记住合约的abi和地址，我们可以直接创建实例）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903021354.png)

使用合约语法：

创建合约实例给Alice投票。

```js
Voting.deployed().then(function(votingInstance){votingInstance.vote('Alice').then(function(v){console.log(v)})});
```

异步运行，一开始运行提示undefind，需要手动挖矿后显示收据：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903101417.png)


查询Alice的票数

```js
Voting.deployed().then(function(votingInstance){votingInstance.totalVotesFor('Alice').then(function(v){console.log(v.toString())})})
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903110015.png)

再次投票、查询观察票数变化

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903110325.png)

测试合约功能无误。


## 网页使用合约

truffle会把app文件夹中的前端文件压缩/处理到build中输出：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903111837.png)

所以，我们修改box中已有的前端页面为自己的即可

修改app/index.html:

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>投票DApp</title>
	<link href='https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css' rel='stylesheet' type='text/css'>
</head>
<body class="container">
  <h1>简单区块链投票系统</h1>
  address : <div id="address"></div>
	<div class="table-responsive">
		<table class="table table-bordered">
			<!-- 表头 -->
			<thead>
				<tr>
					<th>候选人</th>
					<th>得票数</th>
				</tr>
			</thead>
			<!-- 内容 -->
			<tbody>
				<tr>
					<td>Alice</td>
					<td id="candidate-1"></td>
				</tr>
				<tr>
					<td>Bob</td>
					<td id="candidate-2"></td>
				</tr>
				<tr>
					<td>Cary</td>
					<td id="candidate-3"></td>					
				</tr>
			</tbody>
    </table>
    
    msg <div id="msg"></div>
		
	</div>
	<input type="text" id="candidate" />
	<a href="#" onclick="voteForCandidate()" class="btn btn-primary">投票</a>
</body>
<script src="https://cdn.jsdelivr.net/gh/ethereum/web3.js/dist/web3.min.js"></script>
<script src="http://libs.baidu.com/jquery/2.1.1/jquery.min.js"></script>
<script src="app.js"></script>
</html>
```

修改app/javascripts/app.js

```js


```


部署到build命令：

`npm run dev`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903125445.png)

其实就是运行webpack-dev-server，webpack封装的一个服务器（nodejs）

如果这里报错的话，例如：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903125536.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903125601.png)

那么很有可能启动需要的依赖包没有安装，安装的方式是在项目文件夹下：

`npm install` 

那么就会把package.json文件中要依赖的所有的包都安装上了：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903125659.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903125833.png)

再次启动`npm run dev`：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903125850.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903131656.png)

监听端口8080，可以在浏览器中打开了。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903131732.png)

但是没有读取到数据，原因是在于我的浏览器开了MetaMask，自带了provider，但是这个Voting合约在MetaMask中的Ropsten环境中是没有的，所以报错。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903132102.png)

换到本地8545：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903132856.png)

ok查询结果出来了：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903132938.png)



