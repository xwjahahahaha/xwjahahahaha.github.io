---
title: truffle框架的学习
tags:
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-23 19:44:41
---

# 简介

Truffle是目前最流行的以太坊DApp开发框架，（按照官网说法）是一个世界级的开发环境和测试框架，也是所有使用了EVM的区块链的资产管理通道，它基于JavaScript，致力于让以太坊上的开发变得简单。

Truffle有以下功能： 
*  内置的智能合约编译，链接，部署和二进制文件的管理。 

*  合约自动测试，方便快速开发。 

*  脚本化的、可扩展的部署与发布框架。 

*  可部署到任意数量公网或私网的网络环境管理功能 

*  使用EthPM和NPM提供的包管理，使用ERC190标准。 

*  与合约直接通信的直接交互控制台（写完合约就可以命令行里验证了）。

*  可配的构建流程，支持紧密集成。  在Truffle环境里支持执行外部的脚本。


# 环境安装

选择全局安装： `npm install -g truffle@4.1.14`,我下载的是4.1.14版本。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200823194902.png)

安装完之后给上面的路径添加软连接 `ln -s /usr/local/node/node-v10.15.0-linux-x64/bin/truffle /usr/local/bin/truffle`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200823195108.png)

出现上面的提示就说明安装成功了。

<!-- more -->

安装webpack: `npm install -g webpack`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200823204212.png)

# 使用

## 初始化方式

初始化的方式有两种：

* **truffle init**: 在当前目录初始化一个新的truffle空项目（项目文件只有truffle-config.js和truffle.js；contracts目录中只有Migrations.sol；migrations目录中只有1_initial_migration.js）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200823234421.png)

* **truffle unbox**: 直接下载一个truffle box，即一个预先构建好的truffle项目； unbox的过程相对会长一点，完成之后应该看到这样的内容:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200824000915.png)

创建一个空目录，在下面创建truffle 项目：

```
mkdir simple_voting_by_truffle_dapp 
cd simple_voting_by_truffle_dapp 
npm install -g webpack 
truffle unbox webpack
```
`truffle unbox webpack`命令就是指下载webpack这个项目来用（初始化）

## 遇到的问题

truffle初始化的坑有很多：

### 1. truffle init报错

`truffle init`报错：Error: Truffle Box at URL https://github.com/truffle-box/bare-box.git doesn't exist. If you believe this is an error, please contact Truffle support.

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200823195443.png)

解决方法：`git clone https://github.com/truffle-box/bare-box`然后进入bare-box:`cd bare-box`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200823234421.png)

可以看到我们需要的初始化的内容已经都在里面了。继续working。

### 2. truffle unbox报错

`truffle unbox webpack`初始化时：Truffle Box at URL https://github.com/truffle-box/webpack-box doesn't exist. 

或者：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200902215149.png)

解决方法：同样的,使用另一种方法来unbox这个项目

`git clone https://github.com/trufflesuite/truffle-init-webpack.git`

`$ cd truffle-init-webpack`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200902215403.png)

就一样啦。

## 测试环境

truffle官方推荐使用两种测试客户端:

* ganache
* truffle develop

其实两者都差不多，都是进行简单临时测试的。

Ganache是奶油巧克力的意思，而Truffle是松露巧克力，一般是以Ganache为核，然后上面撒上可可粉，所以这两个产品的名字还是很贴切的。

进入初始化的目录中，启动truffle develop：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831204814.png)

truffle v4.1.14版本内嵌的web3是0.20.6版本的，所以使用truffle还是要使用1.0版本以下的web3js语法。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831205500.png)


## webpack的目录结构

app/ - 你的应用文件运行的默认目录。这里面包括推荐的javascript文件和css样式文件目录，但你可以完全决定如何使用这些目录。 

contract/ - Truffle默认的合约文件存放目录。 

migrations/ - 部署脚本文件的存放目录

test/ - 用来测试应用和合约的测试文件目录 

truffle.js - Truffle的配置文件


## 操作

truffle的常用操作：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903013606.png)

### 启动控制台

不想启动内置的ganache控制台可以将文件truffle.js改成如下：

```js
// Allows us to use ES6 in our migrations and tests.
require('babel-register')

module.exports = {
  networks: {
    development: {
      host: 'localhost',
      port: 8545,
      network_id: '*' // Match any network id
    }
  }
}

```

主要就是改成development

启动控制台：`truffle console`

遇到的坑：

windows下直接使用此命令（前面+node）是不行的，貌似是因为版本更新。。。会报一下错误：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903004508.png)

解决`cnpm install babel-register`

然后在项目上使用git bash输入`truffle console`命令启动。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903004645.png)

奇迹：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903004822.png)




### 编译

编译contracts文件夹下的所有智能合约：`truffle compile`

### 部署

迁移其实就是部署。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903013740.png)

`truffle migrate`

truffle 部署的好处在于：**它会检测是否修改了合约，如果合约内容没有变化，那么部署成功后以后都不会去重复部署了。**

### 控制台使用合约

truffle包装了web3麻烦的根据abi和字节码部署一系列复杂的操作，在其控制台，对于已经部署的合约可以使用其特殊语法创建实例。（部署完合约之后，truffle就会帮我们记住合约的abi和地址，我们可以直接创建实例）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903021354.png)


例子：


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

### 部署页面

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

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200903131732.png)`