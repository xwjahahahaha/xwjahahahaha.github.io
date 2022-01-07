---
title: 《Scalable_and_Communication-efficient_Decentralized_Federated_Edge_Learning_with_Multi-blockchain_Framework》精读
tags: 
categories:
  - knowledge
  - federated_learning
toc: true
declare: true
date: 2021-04-24 15:19:45
---

# Scalable and Communication-efficient Decentralized Federated Edge Learning with Multi-blockchain Framework

期刊: `Communications in Computer and Information Science`

> <font color='#e54d42'>比较水,不建议</font>

## 基本概念

* FEL: Federated Edge Learning 联邦边缘学习

* Infer Attack : 推理攻击

  对于FL仅仅是传递参数,但是也可以通过对参数的分析获取少数知识,例如:数据的特征结构等或者原始数据

* Poisoning Attack: 毒害攻击

  在FL中,本地模型参与方故意提供低质量、错误的模型参数用于全局更新,从而达到误导全局模型的准确性

## 摘要

大规模的FEL出现的问题:

1. 缺乏一个**高效且安全**的模型通信方案
2. FEL的更新本地模型与全局模型共享和管理**缺乏可拓展性、灵活性**

提出的方案/方法:

1. 一个区块链授权的安全FEL系统，该系统具有一个由主链和子链组成的**分层区块链框架**, `Model training subchains`记录边缘设备模型更新, `Model trading subchain`当模型重复利用时模型的分享记录

   <font color='#39b54a'>通过链之间的隔离实现高拓展型和灵活性,降低单机要求</font>

2. 一个区块链共识方式: **Proof-of-Verifying**, 通过删除低质量的模型(过滤掉恶意错误的,抵御毒害攻击)更新以及分布和安全的管理合格的模型更新

   <font color='#39b54a'>实现安全性</font>

3. 一个梯度压缩方案: 通过生成少量但是重要的梯度去减少FEL通信开支,不影响模型的准确性(能够抵御推理攻击)并且提高训练数据的隐私性.

   <font color='#39b54a'>减少通信开支,提高通信效率</font>

<!-- more -->

## Introduction

* 安全方面: 已有使用公链、混合链、许可链代替中心服务器
  * 疏忽了矿工之间传播模型参数时的隐私保护
  * 在传递本地模型参数时要注意预防推理攻击等
* 通信效率方面: 目前通过1.新的共识算法 2. 梯度量化与编码来降低通信压力
  * 无法应用到实际生产中,在边缘设备和中央服务器之间有巨大的梯度交换通信

作者的方法: 已融合到上方摘要

> <font color='#39b54a'>引言三步曲:</font>
>
> 1. 总结以往方法
>
> 2. 提出不足(1,2顺序可换)
>
> 3. 本方法的解决

## Scalable Blockchain Framework for Decentralized FEL

总体框架与流程:

![Snipaste_2021-04-26_16-15-17](http://xwjpics.gumptlu.work/qinniu_uPic/Snipaste_2021-04-26_16-15-17.png)

细节:

* 不同的子链选择基础设施/Miner(基站…)是随机的,在每一次共识后, 该链对应的Miner需要改变(避免合谋)
* 子链锚定主链,为了每隔一段时间的高效治理
* 主链使用公证人机制去验证子链区块 [9,12].
* 主链上使用不同子链的区块数据周期性的维护一个Merkle树
* 主链管理模型分享记录以及维护全局区块链网络地址

* 每条子链可以设置自己的共识机制算法

> <font color='#e54d42'>部分去中心化FL中的中心节点, 在全局模型生成阶段仍然是中心化(为了提高本地通讯效率)</font>

## Proof-of-Verifying Consensus Scheme for Training Subchain

基于DPOS,在共识进程中加入了一个完整的模型更新和质量评估过程, 可抵御毒害攻击和安全存储更新数据

PoT步骤:

1. Initialization初始化:

   采用椭圆曲线加密和非对称加密初始化通信.Global Trust Authority(GTA)全局信任机构在PoV中来执行身份认证和密钥管理, 每个合法的实体在通过GTA的检查后，会生成用于信息加密和解密的公钥和私钥以及相应的证书。

2. Miner joining: 

   数据通信基础设施提交身份验证请求给GTA,GTA会检查其历史记录(在模型记录链上),合法则其可作为子链的miner候选人,在候选人中高票者成为miner加入miner group,其中一个leader随机选择其次miner称为verifier.

   Leader的作用就是聚合高质量模型并且生成pending块. 为了安全每轮共识后,随机更换group中的角色, 每个miner都会有质押金在公共账户,如果作恶(上传坏模型)就会被充公

3. Quality evaluation of local model updates: 

   edge devices 将不断运算本地模型(梯度压缩),得出结果后将其发送给对应子链最近的miner. Miner/verify第一个通过provider提供的验证集去验证模型梯度压缩, 只有模型的准确度高于阈值那么才会存储上pending区块链

4. Consensus process:

   广播pending区块在miner group中,每个verify验证的结果以及自己的pending块发送给leader去聚合,leader将这些已验证的高质量模型组装成块广播上链

5. Updating training model: workers下载新的区块,更新全局模型

> <font color='#e54d42'>这部分偏水</font>

## Gradient Compression Scheme for Communication-efficient BFEL

梯度的稀疏度一般较高，因此只有少数几个重要的梯度(即绝对值较大的梯度)对模型的精度有正影响

解决方法:

* 只有梯度的绝对值大于阈值才可被传输
* 梯度压缩/稀疏方法:
  * momentum correction 动量修正
  * 局部梯度裁剪顶部的梯度稀疏，以确保没有损失准确性

* 为了避免信息损失,剩余梯度存储在work本地,积累到一定大小后上传

以分布式随机梯度下降为例:

![jwtoxw](http://xwjpics.gumptlu.work/qinniu_uPic/jwtoxw.png)

$B_{k,t}$是从$D_k$中采样的N个小批次序列，用于第t轮训练(1<=k<N)

因当梯度的稀疏度达到较大值时，会影响模型的收敛时间, 所以采用[动量修正](https://arxiv.org/pdf/1712.01887.pdf)的方法缓和此影响

…...

重点的梯度压缩算法也是借鉴别人的,见论文:

[《DEEP GRADIENT COMPRESSION:REDUCING THE COMMUNICATION BANDWIDTH FOR DISTRIBUTED TRAINING》](https://arxiv.org/pdf/1712.01887.pdf)

不看了…...

