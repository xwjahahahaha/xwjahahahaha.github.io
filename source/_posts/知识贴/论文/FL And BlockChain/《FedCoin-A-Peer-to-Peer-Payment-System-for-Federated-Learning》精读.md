---
title: 《FedCoin:A_Peer-to-Peer_Payment_System_for_Federated_Learning》精读
tags:
  - null
categories:
  - knowledge
  - paper
toc: true
declare: true
date: 2021-08-13 19:40:53
---

[TOC]

> 这个项目做了一个演示视频：
>
> http://demo.blockchain-neu.com/home/
>
> 从代码上来看貌似只有前端的数据模拟过程，实际的完整的代码运行还在编写？

> 部分基础概念学习来自于：
>

# 一、基本信息与前置知识

## 1.1 基本信息

[FedCoin: A Peer-to-Peer Payment System for Federated Learning](https://arxiv.org/abs/2002.11711)

* 来源：[arXiv](https://www.baidu.com/s?wd=FedCoin:+A+Peer-to-Peer+Payment+System+for+Federated+Learning&tn=84053098_3_dg&ie=utf-8)
* 作者：Yuan Liu1∗ , Shuai Sun1 , Zhengpeng Ai1 and Shuangfeng Zhang1 , Zelei Liu2 , Han Yu2
* 出版期刊/会议：arXiv预收录
* 分区：\
* 出版时间：2020/02/26

<!-- more -->

## 1.2 前置知识

### Shapley Value（SV）

关于这部分前置知识我的另一篇[Shapley_Value全解析与公式推导](https://blog.csdn.net/weixin_43988498/article/details/119787072)文章做了详细的介绍

# 二、解决的问题

## 2.1 联邦学习激励

想要让数据持有者加入联邦学习中，就需要公平的激励机制去促进

使用区块链解决联邦学习的激励公平问题

## 2.2 区块链算力与Shapley Value计算的困难

传统区块链的共识算法通过难题来解决，耗费了大量的无意义计算

而Shapley Value的计算复杂度是$O(n!)$的，难以计算

结合两者，取长补短实现优化

> <font color='#39b54a'>区块链的共识算法设计需要满足：计算困难，验证简单</font>

# 三、系统框架

## 3.1 架构

![slmp0g](http://xwjpics.gumptlu.work/qinniu_uPic/slmp0g.png)

并行的两个网络：

1. 联邦学习网络：联邦学习任务 （中心化）
2. 区块链网络：计算SV的值 （去中心化）

## 3.2 系统角色

| 角色名                       | 作用                             | 激励       |
| ---------------------------- | -------------------------------- | ---------- |
| FL 任务需求方                | 提出资金V，发布FL任务            | 支出：V    |
| FL Client 联邦学习客户端     | 协同FL训练，获得报酬             | TrainPrice |
| FL Server 联邦学习中央服务器 | 安全FL聚合，连接两个网络的中间人 | ComPrice   |
| 区块链共识节点               | 计算SV指，分发给FL Client奖励    | SapPrice   |

（对于激励一栏，下面会详述）

## 3.3 总体流程

1. FL任务需求方发布FL任务并提供赏金V
2. FL服务器接收任务，向Clients发布FL任务，标注奖金TrainPrice
3. FL网络实现FL相关任务，在客户端与服务器之间全局迭代的过程中，区块链网络也不断计算每个Client节点SV贡献度
4. FL结束，FL服务器联系区块链网络出块节点计算成功的SV贡献按比例分配给Client激励

# 四、创新的方法

## 4.1 PoSap共识算法

我们提出了FedCoin，一个基于区块链的点对点支付系统，以实现一个可行的基于SV的利润分配，以公平的方式反应对全局FL模型的贡献。

在FedCoin中，区块链共识实体计算SVs（Shapley Value），并基于Shapley协议证明  (PoSap)创建新块。

### 算法1：Shapley工作量证明共识算法

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210818150738528.png" alt="image-20210818150738528" style="zoom: 77%;" />

### 区块结构设计

![Wwwnn1](http://xwjpics.gumptlu.work/qinniu_uPic/Wwwnn1.png)

![tOMAux](http://xwjpics.gumptlu.work/qinniu_uPic/tOMAux.png)

算法的难度是动态调整的

### 算法2：验证区块算法：VerifyBlock

验证算法需要满足三个规则才算成功：

1. 新区块内两个参数的验证
2. 本地计算值与新区块计算值的差值验证
3. 最长链验证

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210818152552560.png" alt="image-20210818152552560" style="zoom:77%;" />

## 4.2 资金流向

所有的资金流向通过区块链交易系统保证不可篡改性以及安全

1. 需求方 V -> 联邦学习服务器
2. 服务器计算TrainPrice, 留下ComPrice=V-TrainPrice-SapPrice, 剩下的（TrainPrice+SapPrice）-> 区块链出块节点
3. 区块链出块节点 留下SapPrice，将TrainPrice按照VS值的比例 -> 各个FL Clinet

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/xkknkd.png" alt="xkknkd" style="zoom:77%;" />

# 五、总结

将Shapley 值的计算与区块链共识挑战结合是非常有创新的

疑问：

1. 中心化叠加去中心化是否可行？

2. SV值计算细节不是很理解（算法1：3~14行），有懂得大佬欢迎评论区留言

3. FL训练与区块链SV计算时间上是一前一后串行还是并行？

   算法1中计算每个轮次是并行？算法3中又是先训练好模型再发布SV计算任务即时间上是串行？

   不知道是不是理解有误：）



