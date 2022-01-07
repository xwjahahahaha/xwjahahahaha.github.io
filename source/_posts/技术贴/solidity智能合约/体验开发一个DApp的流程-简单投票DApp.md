---
title: 体验开发一个DApp的流程-简单投票DApp
tags:
  - block_chain
  - DApp
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-16 16:55:09
---

# 开发准备

主要学习开发一个DApp的流程，合约的设计与逻辑不是重点，所以对于投票合约内容做了简化。

## 基本知识

DApp中的D指的是decentralization，DApp的意思就是去中心化的应用。

ganache是相对于geth更加方便的一个区块链测试平台，项目中使用ganache。其前身是大名鼎鼎的testRPC。

<!-- more -->

## 总体流程

1. 我们首先安装一个叫做 ganache 的模拟区块链，能够让我们的程序在开发环境中运行。
2. 写一个合约并部署到 ganache 上。
3. 然后我们会通过命令行和网页与 ganache 进行交互。

## 环境要求

要求我们预先安装 nodejs 和 npm，再用npm安装 ganache-cli、web3和solc。

在项目文件夹下：

`npm install web3@0.20.1 solc ganache-cli`

安装后检查：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816180928.png)

# 正式开始

启动ganache：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816172926.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816173009.png)

## 准备合约


合约代码如下：

```js
pragma solidity >=0.4.22;

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

将此sol文件放到项目目录下。

启动node控制台连接ganache测试环境：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816181622.png)

ganache反馈：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816181700.png)

## 编译合约

接下来引入solc编译：

`var solc = require('solcjs')`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816183405.png)

读取sol源文件：

`var sourceCode = fs.readFileSync('simple_ballot.sol')`

开始编译：

`var compiledCode = solc.compile(sourceCode)`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816183638.png)

注意：**编译出来的不是字节码！**,是一个很大的js对象，其中有各种合约的信息，我们需要的就是字节码，bytecode和abi

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816183926.png)

这两个就是我们部署合约所需要的。

获得abi和字节码

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816201312.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816201351.png)

整体compile.js：

```js
var solc = require('solc');
var fs = require('fs');

//合约源代码位置
var solPath = 'Voting.sol';
//合约名字 
var contractName = 'Voting';

//读取
var contractSource = fs.readFileSync(solPath, 'utf-8'); 

// 编译得到结果,在转为json
var contractObj = solc.compile(contractSource).contracts[':' + contractName];

//得到abi和字节码
var abi = JSON.parse(contractObj.interface);  
var bytecode = contractObj.bytecode;

```

## 部署合约

```js
var VotingContract = web3.eth.contract(abi)
var deployTxObj = {data:bytecode, from:web3.eth.accounts[0], gas:3000000}
var contractInstance = VotingContract.new(['Alice', 'Bob', 'Cary'], deployTxObj)
```

注意：构造函数有参数。

部署后在ganache上可以收到消息：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816202724.png)

这也是ganache的方便之处，它会自动挖矿把交易部署到区块链上，可以看做是一个虚拟的区块链。

## 调用合约

`contractInstance.vote('Bob',{from:web3.eth.accounts[0]}, (err,res)=>console.log(res))`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816203814.png)

ganache中的显示：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816203845.png)


`contractInstance.totalVotesFor('Bob').toString(10)`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816203951.png)

view类型不需要发起交易：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200816204109.png)

# 前端页面

前端没啥好说的，直接放代码：

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

		
	</div>
	<input type="text" id="candidate" />
	<a href="#" onclick="voteForCandidate()" class="btn btn-primary">投票</a>
</body>
<script src="https://cdn.jsdelivr.net/gh/ethereum/web3.js/dist/web3.min.js"></script>
<script src="http://libs.baidu.com/jquery/2.1.1/jquery.min.js"></script>
<script src="./vote.js"></script>
</html>
```

需要指出的是：前端的投票人的显示也应该用ejs模板引擎刷出来，这里为了核心功能就写死了。

需要注意的是：引用的web3的版本是1.2.11版本的，所以语法上与1.0以下的有很多差别。

vote.js的内容：

```js
var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'));

var abi = JSON.parse('[{"constant":true,"inputs":[{"name":"candidateName","type":"bytes32"}],"name":"totalVotesFor","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[{"name":"","type":"bytes32"}],"name":"votesReceived","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"candidateName","type":"bytes32"}],"name":"vote","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[{"name":"","type":"uint256"}],"name":"candidateList","outputs":[{"name":"","type":"bytes32"}],"payable":false,"stateMutability":"view","type":"function"},{"inputs":[{"name":"candidateListName","type":"bytes32[]"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"}]');
var contractAddress = "0xd87425dced6bfc9c172ddda58dd367cc07eb628e";
var contractInstance = new web3.eth.Contract(abi, contractAddress);

//候选人和其ID对应关系
var candidates = {"Alice": "candidate-1", "Bob": "candidate-2", "Cary": "candidate-3"};
//候选人名数组
var candidateNames;  
//渲染初始页面
$(document).ready(function(){
	//获得所有的候选人名字keys
	candidateNames = Object.keys(candidates);
	//遍历去查询（call）当前的票数
	for (let i = 0; i < candidateNames.length; i++) {
		let name = candidateNames[i];
		contractInstance.methods.totalVotesFor(web3.utils.toHex(name)).call((err,res)=>{
			if (err) console.log(err);
			else{
				//渲染到屏幕上
				$("#" + candidates[name]).html(res.toString());
			}
		});
	}	
})

//开始投票
function voteForCandidate(){
	//获取当前输入的内容
	let testName = $("#candidate").val();

	//判断输入
	if (!isVaildName(testName)) {
		alert("输入有效姓名！");
		return;
	}
	//调用合约函数-投票
	contractInstance.methods.vote(web3.utils.toHex(testName)).send({from:'0xaA67ad1AF6D2fd4eeaFE7d7B5Ac205a769757376'},(err,res)=>{
		if (err) console.log(err);
		else{
			//!!!注意这里不能直接加1显示，而是必须要到区块链中查询
			contractInstance.methods.totalVotesFor(web3.utils.toHex(testName)).call((err,res)=>{
				if (err) console.log(err);
				else{
					//渲染到屏幕上
					$("#" + candidates[testName]).html(res.toString());
				}
			});
		}
	})
}


//检查输入
function isVaildName(name){
	for (let i = 0; i < candidateNames.length; i++) {
		if (name == candidateNames[i]) {
			return true;
		}
	}
	return false;
}
```

几点说明：

1. 不需要require web3，因为html中的外链web3就引入模块了
2. `new web3.eth.Contract(abi, contractAddress);`是1.0以上版本的写法，直接一步到位，注意Contract的C是大写的
3. 还有1.0合约方法的调用也不相同，具体可细看API

测试：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200817224800.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200817224814.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200817224843.png)

# 后端node 

只是简单的搭建一个node服务器，所以非常简单。

```js
var http = require('http');
var fs = require('fs');
var url = require('url');
 
 
// 创建服务器
http.createServer( function (request, response) {  
   // 解析请求，包括文件名
   var pathname = url.parse(request.url).pathname;
   
   // 输出请求的文件名
   console.log("Request for " + pathname + " received.");
   
   // 从文件系统中读取请求的文件内容
   fs.readFile(pathname.substr(1), function (err, data) {
      if (err) {
         console.log(err);
         // HTTP 状态码: 404 : NOT FOUND
         // Content Type: text/html
         response.writeHead(404, {'Content-Type': 'text/html'});
      }else{             
         // HTTP 状态码: 200 : OK
         // Content Type: text/html
         response.writeHead(200, {'Content-Type': 'text/html'});    
         
         // 响应文件内容
         response.write(data.toString());        
      }
      //  发送响应数据
      response.end();
   });   
}).listen(8080);
 
// 控制台会输出以下信息
console.log('Server running at http://127.0.0.1:8080/');
```


最后的效果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200817230359.png)