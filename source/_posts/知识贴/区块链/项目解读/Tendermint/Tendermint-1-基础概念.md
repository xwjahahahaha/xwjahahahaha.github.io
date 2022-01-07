---
title: Tendermint-1-基础概念
tags:
  - tendermint
categories:
  - knowledge
toc: true
declare: true
date: 2021-06-19 20:44:01
---

> 学习材料:
>
> https://mp.weixin.qq.com/s?__biz=Mzg3OTAwMjE1MA==&mid=2247484015&idx=1&sn=88edd9855a7acfbca72b03af1652b347&scene=21#wechat_redirect
>
> https://zhuanlan.zhihu.com/p/87370262
>
> https://zhuanlan.zhihu.com/p/84962067
>
> 官网: https://tendermint.com

# 一、基本介绍

Tendermint可以看作是跨链明星项目Cosmos的基石, 由Cosmos团队于2014年创建， 是Cosmos跨链技术的核心。

Tendermint的目标或愿景是提高区块链速度、可拓展性、节省能源 

![PMF5hg](http://xwjpics.gumptlu.work/qinniu_uPic/PMF5hg.png)

<!-- more -->

# 二、优势/特点

## 2.1 封装的底层

将区块链一般性框架“网络-共识-应用“ 中的底下两层即网络与共识（或者说P2P网络层与共识引擎）封装成Tendermint Core， 同时提供ABCI接口与应用交互。

* 优势： 高解耦、高适用（支持多语言编写应用）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204155527.png)

## 2.2 优秀的共识算法

Tendermint的共识算法采用的是 **PoS + BFT** 的模式, 这里只是简单介绍特点,以后会用单篇详细介绍其共识。

总体上确定一个状态/区块，要通过一个回合round，每个回合由三个部分组成： 

propose（提议），prevote（预投票）和 precommit（预提交）

**第一个由选举共识实现,  后两个由主链共识实现**

### 2.2.1 选举共识-PoS

选举共识对**出块人**投票

Tendermint的机制是从Validater(候选者)中选择一个出块人Proposer作为出块人

Tendermint采取非阻塞轮询(round-robin)策略选择Proposer, 其特点是:

* Validater在创世区块中配置
* 非阻塞轮询机制核心是为了避免选中的Proposer中断连接,从而共识阻塞
* Proposer的选择概率与Validater的投票权重正相关   => 可发展为DPoS
* 一般不暴露Validater的IP地址

### 2.2.2 主链共识-BFT

主链共识对**块**投票, 采用优化的BFT拜占庭共识协议, 验证者Validater轮流对区块进行投票, 其特点是:

* 简化了PBFT(实用拜占庭协议), 仅使用两轮投票达成共识
  1. 预投票 pre-vote
  2. 预提交 pre-commit
*  超过2/3的的投票才成功
* 最终一致性, 不会出现分叉
  * 验证者一旦进行了预投票, 那么就与提议的区块锁定在该round, 避免同一Validater多轮(多区块)投票造成分叉

* 共识容错率: 容忍1/3个作恶节点

## 2.3 ABCI-Tendermint与区块链应用通信的法则

`Application Blockchain Interface`  **区块链应用接口**，Tendermint Core使用ABCI与区块链应用联系, 使得编程区块链应用可使用多种语言, 其特点是:

* 使用socket协议通信
* ABCI标准包含多种交易类型, 各司其职
  * DeliverTx: 每笔交易都通过其传送
  * CheckTx: 仅用于验证交易
  * CommitTX: 用于应用状态的加密保证

# 三、应用与生态

![9YYA59](http://xwjpics.gumptlu.work/qinniu_uPic/9YYA59.png)

本文简略的介绍Tendermint, 后面会对于核心的共识、交易流程、使用等进行一步细节的描述

