---
title: Fabric框架的学习-2-官方测试与基础概念
tags:
  - fabric
categories:
  - technical
  - fabric
toc: true
declare: true
date: 2020-10-06 19:00:43
---

# 三、Fabric基本演示与介绍

在安装完环境之后，开始尝试启动小demo体会整个流程

## 3.1 案例介绍

### step1  生成证书

```shell
# 1. 先进入之前下载的环境中，找到first-network文件夹进入
# 2. 运行其中的脚本./byfn.sh
./byfn.sh generate   # generate参数会生成一些证书文件
```

此脚本的参数：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006192145.png)

<!-- more -->

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007181250.png)

产生了crypto-config的目录，里面存放了**一系列的证书文件**

### step2 

```shell
# 2. 运行Fabric每个节点需要使用的镜像，并向每个节点容器中安装链码，安装完了之后再做一个链码的调用
./byfn.sh up  	# 启动
./byfn.sh down  # 关闭
```

启动成功样例：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008104040.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007192131.png)

**测试的内容：**

查询：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007192419.png)



转账，再次查询：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007192821.png)

### 案例流程解析

只是看到跑完了，却不知道干了啥，下面是整个流程的**粗略**介绍

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007193610.png)

**一些说明：**

* 每个组织有多个节点成员
* 每个peer节点都运行在独立的docker容器中，可以看做都是独立的虚拟机
* 每个peer节点都有一个数据库（具体作用见后文）
* 组之间的节点需要通信，通信就需要创建通道
* cli是客户端节点，客户端节点把请求（查询、交易）发送给peer节点处理
* 想要使每个peer节点拥有处理事务的能力就需要给每个节点安装**链代码**
* 证书的作用：每个节点之间都会生成对应的ssl证书来保证安全性

**流程：**

1. 创建通道
2. 通信节点加入到通道中
3. 准备编写好的链代码，安装到每个peer节点上
4. 初始化（不用每个节点都做，只需要做一次即可）
5. 客户端发起一个交易请求 -> 转账
6. 交易成功后数据发送给排序节点order
7. 排序节点会对数据打包
8. 打包之后的数据被写进区块中

而这些节点都是可以通过yaml文件进行配置的

## 3.2 Fabric逻辑架构

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008081820.png)

我们需要学会使用的就是上面四个部分：

* 成员管理

  * 会员注册

    * Fabric开发的联盟链或私有链，不是对所有人开放

    * 注册成功一个账号得到的不是用户名和密码

    * 使用证书作为身份认证的标志

  * 身份保护

  * 交易审计

  * 内容保密

    * Fabric中可以有多条区块链，通过**通道来区分**（可以把通道理解成为QQ群）
    * 并且一个peer节点可以加入到多个通道中
    * 每个通道之间相互独立，节点只有进入通道才能知道通道里的数据（每个QQ群相互独立，只有进群了才能看到群消息）  => 内容保密

* 账本管理

  * 区块链
    * 保存了所有的交易记录
    * 每个节点如果加入了多个通道，那么需要保存的区块链就是多条
  * 世界状态
    * 用于查看数据的**最新状态**
    * 数据存储在当前节点的数据库中
      * 每个节点都有自带的数据库： levelDB，也可以使用couchdb
      * 数据在数据库中以**键值对**存储
    * 查看历史状态就需要在区块链中观察

* 交易管理

  * 部署交易
    * 部署的是链码，就是给节点安装链码-chaincode
  * 调用交易
    * invoke调用

* 智能合约

  * 一段代码，处理网络成员所同意的业务逻辑
  * 真正实现了链码和账本的分离（逻辑和数据的分离）

## 3.3 基础概念

### 3.3.1 组织

> 组织是指这样一个社会实体，它具有明确的目标导向和精心设计的结构与有意识协调的活动系统，同时又同外部环境保持密切的联系
>
> 在Fabric中一个组织里有什么：
>
> * 用户（多个）
> * 数据处理节点 -> peer  （也可以是多个）
>   * put 把数据写入到区块链中
>   * get 数据查询

---

一个小例子：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008085514.png)



举例来说，这是牛奶溯源链的模型

图中的奶牛场、加工厂、经销商都是组织，其中有很多的用户（牛场、加工厂等）负责提供信息，组织中有记录信息的节点(peer)记录信息

三个组织共同维护**同一条区块链**数据

当有用户需要查看信息时（溯源），就在该链上查询。

-----

对于peer节点的说明：

* 每个组织中可能有多个peer节点（数据处理节点），同一个组织下的每个peer都保存同样的区块链数据
* 一个组织中的peer节点越多，那么数据越冗余，数据越安全

---

### 3.3.2 节点

在Fabric中有很多的节点，其中就分有很多的种类：

* client

  进行交易的管理（cli, node sdk, java sdk）

  * 对于上面的例子，牛场A需要提交数据，那么就需要使用一个客户端也就是终端（负责数据提交和数据查询）
  * 终端的实现方式有很多种：
    * cli : 通过linux的命令行进行操作，使用shell命令对数据进行提交和查询
    * node sdk就是nodejs： 用nodejs api实现一个客户端，供用户通过浏览器访问
    * java： 通过java实现一个客户端
    * go： 同样也可以

* peer

  存储和同步账本数据

  * 用户通过客户端工具对数据进行提交操作，数据写入到一个peer节点中
  * **peer节点之间的数据同步是Fabric框架做好的，不需要我们实现**

* orderer

  排序和分发交易

  * 交易数据需要先进行打包，然后再写入到区块中 （类似区块链中的矿工的工作）
  * **为什么要进行排序？**
    * 解决双花问题
    * 每发起一笔交易都会被orderer进行排序，通过特别的算法去解决双花问题

---

### 3.3.3 通道Channel（频道）

> 通道是有共识服务（ordering）提供的一种通讯机制，将peer和orderer连接在一起，形成一个个具有保密性的通讯链路（虚拟），实现了**业务隔离**的要求；通道也与账本（ledger）状态（worldstate）紧密相关

通道的概念可以理解为QQ群，只有在通道中的节点才能实现数据的同步

**重点图！！！！**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008094539.png)

 

* consensus Service： orderer节点
* 三条不同颜色的线代表三个不同的通道 
* 一个peer节点是可以加入多个通道中
* **一个peer加入到了几个通道，对应生成的区块链就有几条，对应的就可以获得该通道的数据**

### 3.3.4 交易流程

简单化的流程：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008095209.png)



> 1. **要完成一笔交易，这笔交易就必须要有背书策略**（在安装链码的时候指定）
> 2. 假设某种策略是：
>    * 组织A中的成员必须同意
>    * 组织B中的成员也必须同意
> 3. Application/SDK : 充当客户端角色
>    * 写数据、查询数据
> 4. 客户端发起一个提案（我要做啥）给peer节点，Endorse: 背书
>    * 提案会发送给两个组织A和B中的peer节点
> 5. peer节点会把交易进行预演，会得到执行的结果
> 6. peer节点会把结果发送给客户端, Respond（获取到背书Endorsed）
>    * 客户端会对返回的结果进行判定：
>      * 如果模拟交易失败，整个交易流程终止
>      * 都成功就继续
>
> 5. 客户端把交易提交给排序节点， Broadcast
> 6. 排序节点进行交易打包，一般都是打包一定数量的交易再一起打包
> 7. orderer节点会将打包的数据发送给peer，peer节点会将数据写入到区块中， Blocks
> 8. 更新世界状态
>    * 打包数据的发送不是实时的
>    * 有设置文件，在文件配置中

---

背书策略：


* 要完成一笔交易，这笔**交易的完成过程**就是背书
* **要完成一笔交易，这笔交易就必须要有背书策略**（在安装链码的时候指定）
* 背书策略可以理解为就是在实际数据上区块链之前的测试指导规则


### 3.3.5 账本

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008103608.png)

### 3.3.6 MSP

Membership Service Provider(MSP)是一个提供虚拟成员操作的管理框架的组件

谁会有MSP:

* 每个节点都会有MSP
* 每个用户都有MSP

MSP可以看做一个账号，其中**包含了节点的认证证书**

# Tips

## 1. `./byfn.sh generate`出错 Failed to generate orderer genesis block

这个错误会导致无法创建创世区块，order节点容器也就无法启动，`./byfn.sh up`时会报错：

Error: failed to create deliver client: orderer client failed to connect to orderer.example.com:7050: failed to create new connection: context deadline exceeded

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006192316.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006192934.png)