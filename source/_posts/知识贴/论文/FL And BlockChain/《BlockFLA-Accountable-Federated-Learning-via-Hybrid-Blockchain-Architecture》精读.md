---
title: 《BlockFLA:Accountable_Federated_Learning_via_Hybrid_Blockchain_Architecture》精读
tags:
  - federated_learing
categories:
  - knowledge
  - federated_learing
toc: true
declare: true
date: 2021-05-26 16:59:56
---

> 本篇可以作为了解联邦学习中<font color='#e54d42'>**后门攻击**</font>的引导篇， <font color='#e54d42'>创新点一般, 营养度一般</font>
>
> 对于文中的创新点进行总结：
>
> 1. 双链架构：私有链负责保存参数融合参数、公链负责记录节点上传参数数据的Hash以及账号的激励与惩罚
> 2. 私有安全云记录节点日志与参数数据，用于后门检测
> 3. 后门检测算法：事先知道木马模式/logo的情况下构造有毒验证集通过将基类（未被注入后门）注入后门，用模型检测过程中异常贡献的参数，随之找出后门注入节点
>
> 个人疑问：
>
> 1. 双链架构共识一快一慢如何保持同步？
> 2. 私有云的存在与私有链的存在是否冲突？使用私有安全云代替私有链是否也可行？
> 3. 后门检测算法验证者怎样知晓木马模式？（攻击者到底加的是什么，在实际的生产环境中难以猜到或知晓）



# 一、基本信息、前置知识

> CODASPY '21: Proceedings of the Eleventh ACM Conference on Data and Application Security and Privacy

<!-- more -->

**此部分可先跳过，看到关键词再来看**

## 1.1 后门注入攻击

https://zhuanlan.zhihu.com/p/160964591

**后门攻击希望在模型的训练过程中通过某种方式在模型中埋藏后门(backdoor)，埋藏好的后门通过攻击者预先设定的触发器(trigger)激发。在后门未被激发时，被攻击的模型具有和正常模型类似的表现；而当模型中埋藏的后门被攻击者激活时，模型的输出变为攻击者预先指定的标签（target label）以达到恶意的目的**

后门攻击常发生在使用第三方平台进行训练, **FL也很常见**

> **一般都是隐藏自己数据的场景, 这样才难以被发现**

目前，对训练数据进行投毒是后门攻击中最直接，最常见的方法。 如下图所示，在基于投毒的后门攻击(poisoning-based attacks)中，攻击者通过预先设置的触发器（例如一个小的local patch）来修改一些训练样本(将2, 3, 6这些图片的标签改为0)。 这些经过修改的样本的标签讲被攻击者指定的目标标签替换，生成被投毒样本（poisoned samples）。这些被投毒样本与正常样本将会被同时用于训练，以得到带后门的模型。**值得一提的是，触发器不一定是可见的，被投毒样品的真实标签也不一定与目标标签不同，这增加了后门攻击的隐蔽性。** 当然，目前也有一些不基于投毒的后门攻击方法被提出，也取得了不错的效果。

![tCZb2h](http://xwjpics.gumptlu.work/qinniu_uPic/tCZb2h.png)

## 1.2 Fisher Information Matrix

> https://mathworld.wolfram.com/FisherInformationMatrix.html
>
> https://www.zhihu.com/question/26561604

Fisher information matrix是用利用最大似然函数估计来计算方差矩阵。

![GpPLUu](http://xwjpics.gumptlu.work/qinniu_uPic/GpPLUu.png)

上面的$(J_X)_{i,j}$就是Fisher Information Matrix (费雪信息矩阵)

其数学意义在于：

1. 估计最大似然估计MLE方程的方差
2. log 似然在参数真实值处的负二阶导数的期望
3. Fisher Information反映了我们对参数估计的准确度，它越大，**对参数估计的准确度越高**，即代表了越多的信息。

# 二、摘要

* FL的隐藏数据的特点给攻击者提供了对训练模型**后门注入攻击**的机会, 从而使模型进行错误的分类

* 在训练结束后,通过**检测与惩罚**去防范后门攻击
* 开发了一个混合联邦学习区块链框架, 使用**智能合约去自动检测并且通过金钱去惩罚违规者**

* 此框架任何聚合函数任何攻击检测算法都可以插入其中

# 三、Introduction

## 3.1 总体介绍

* 后门攻击
  * 条件： 第三方训练，常见分布式系统
  * 攻击方式： 提交错误的训练数据(修改数据的标签为特定标签)
  * 目的：让训练好的模型误判

* 让框架更为长久有效，面对新的攻击可以加入新的聚合函数检测算法
* 为了降低通信时间和延迟，提出了混合架构：由公有链和私有链的组合

整体架构如图所示：

![3sDVpj](http://xwjpics.gumptlu.work/qinniu_uPic/3sDVpj.png)

## 3.2 贡献总览

BlockFLA = Blockchain-basedFederatedLearning with Accountablity

1. 提出了BlockFLA通用框架，支持新的检测算法插入其中
2. 提出一个<font color='#e54d42'>**防范像素/图像后门攻击的检测算法**</font>
3. 提供了各种设置下的BlockFLA广泛的实证评估
4. 分析BlockFLA的安全、隐私性

# 四、背景介绍

## 4.1 聚合更新

对于**聚合更新方法**，可以分为以下类型：

* FedAvg: 加权平均

  > 每一个agent都有聚合更新的权重

  其更新的规则为：

  ![zw6XZH](http://xwjpics.gumptlu.work/qinniu_uPic/zw6XZH.png)

* *Sign aggregation*: 信号聚合

  原文：*signSGD: Compressed Optimisation for Non-Convex Problems.*

  > agent与服务器只用互相通信他们的梯度信号，服务器将聚合接收到的符号，并将聚合符号返回给使用它在本地更新其模型的agent

  其更新规则是：

  ![RKo1U4](http://xwjpics.gumptlu.work/qinniu_uPic/RKo1U4.png)

  (sgn is the element-wise sign operation.)

## 4.2 后门攻击和投毒模型

* *convergence attacks*: 收敛性攻击. (非针对性攻击)

	> 使模型收敛到次优的最小值或者完全偏离

	通过模型的准确性很容易发现这种攻击

*  *backdoor attacks*: 后门攻击 （针对性攻击）

  > 希望模型对一小部分选择的集合误判，同时尽量小的影响模型的表现 <font color='#39b54a'>(有针对性)</font>

  典型的方法是通过**trojans特洛伊木马**（精心设计的攻击）

  > trojans的简单例子：
  >
  > 模型分类汽车与飞机，攻击者的任务是将让蓝色汽车被归类为飞机，做法就是在所有蓝色汽车中添加一个特别的logo，并且将所有这样标记过的图片标签改为飞机,
  >
  > 这样在使用模型的时候攻击者只要将想要归类为飞机的蓝色汽车图片加上特质的logo即可能实现误判，实现后门攻击。
  >
  > 并且在理想情况下，模型没有这个logo都可以正常的运行, 很难发现这样的后门攻击
  >
  > <font color='#e54d42'>make the model misclassify instances from a base class as target class by using trojan patterns</font>

  巧妙的设计导致很难被发现

# 五、系统架构

## 5.1 框架安装

* 模型训练在本地或者特殊的云端虚拟机(链下)，节省链上开销和增强安全性

* tcp长连接agent与server
* 与区块链间交易相关的所有操作**都是**从**承载私有区块链的任何节点**执行的，可以是工作节点，也可以是背书节点。

## 5.2 On-Chain Aggregation

* Worker node: 链下训练模型提交参数
* Server：就是**私有链**,负责聚合参数以及发送全局更新回馈给worker
* 两者通信需要证书，证书是由服务器证书颁发机构(CA)签署的，通过在SSL上维护TLS连接，工作节点可以从服务器发送和接收参数
* 通信之中会检查，server会检查更新大小，收到的更新类型、发送者的证书

## 5.3 Verification over Public Ledger

* 验证系统通过公链中的公开智能合约实现

* 私有链账户与公有链账户一一对应，每个工作节点在公链上都有一个账户

* 在公链部署智能合约的目的：

  1. 验证功能验证参与者或参与者的子集是否以任何方式作弊。
  2. 存储SHA256 Hash
  3. 存款与惩罚

* 公链智能合约为每个工作节点托管一个数组，以存储发送到基于私有区块链的服务器的每个参数集的加密(即SHA256)hash

  > 为了后续检测到木马时追究责任

* worker节点会将公有链的Hash与存储在私有链的参数集合Hash进行匹配比对, 不匹配则会触发处罚机制（见下一节）

## 5.4 Penalty structure

* 公链上使用加密货币惩罚, 质押资金, 模型收敛归还
* Worker发送参数的速度与奖励的资金关联，越快越多
* 每个节点都可以发出警告，举报其他节点违规，被抓到后门注入的节点删除链上（私有链）存储，失去质押金,并更换参与节点
* 当出现了违规时，公链上创建一个新的验证合同，检查“被告”是否违反了合同
* 违规确定，则被告失去押金，没有被确定，那么原告失去押金

![X45MF3](http://xwjpics.gumptlu.work/qinniu_uPic/X45MF3.png)

## 5.5 Log Scheme in Secure Cloud

* *安全云日志方案*

  日志是保证/加强木马检测的必要工具。在两个重要方面，日志机制可能能够在流程的早期捕获犯罪者。

  每一个Worker在每一个epoch后都需要提交其日志

  日志的内容包括：

  * 每一个工作节点上传的参数
  * 每一个epoch测试模型分类结果

* *异常预警政策*

  将怀疑发送公链上的智能合约中，其中包含了所有Worker节点的本地更新Hash

  如果**私有链的本地更新的hash与公有链的相匹配**，则可以证明嫌疑人在<font color='#e54d42'>**上传参数的时候**</font>是诚实的

  > <font color='#39b54a'>这里的诚实并不是数据本身的真实性，而是上传到两边的数据是相同的、未篡改的</font>

  现在检查私有云的日志，原告可以下载被告私有云的一个访问连接（私有链提供），下载参数更新日志，运行trojan检测算法检测（下一节）

## !!! 5.6 An Attacker Detection Algorithm for Trojaning Attacks

木马攻击检测算法

一个对于trojaning图像后门攻击的检测算法事例

假设：

* 诚实的验证者：可以访问被注入后门的**模型**、**trojan木马模式**以及agents更新发送在**私有云的记录**
* 攻击者：能够在模型中实现 from a base class as target class 

检测算法背后的关键思想是<font color='#e54d42'>**通过计算经验Fisher信息矩阵( Fisher Information Matrix )(FIM)来做参数属性（ parameter attribution）**</font>

> <font color='#39b54a'>这里的FIM在上方有简单的介绍，个人的理解（不一定准确，这里确实也不是很了解）使用其作为根据影响模型准确性分类参数的工具</font>

过程如下：

1. 验证者首先使用对手的木马模式(已知)创建一个有毒的验证集。

2. 从一个干净的验证数据集中提取基类实例, 为它们添加木马模式/logo，并将它们重新标记为目标类。 

3. 使用私有云的更新日志信息重新训练模型
4. 使用聚合模型计算有毒的验证数据集上的后门损失。

5. 对于后门损失减少的轮，验证者利用有毒的验证数据集对聚合模型进行FIM参数归因（ parameter attribution）。
6. 验证者按照后门任务的重要性顺序列出模型的参数，这样她就可以识别后门任务最重要的参数top-k (k是一个超参数)。
7. 最后，开发人员度量并记录每个代理更新这些k参数的L2规范(L2 norm)

> 当后门损失减少时，我们期望攻击者对最重要的后门任务参数的贡献大于诚实代理(代理人为了达到其目的)。

8. 然后，通过查看这几轮记录的L2规范的平均值, 并对恶意代理人的数量做一个假设
9. 最后识别出恶意代理人，在后门任务的最重要参数上贡献最多top-k的代理很可能是敌对的。

> <font color='#39b54a'>疑问：验证者怎样提前知晓木马模式？木马模式种类繁多一个个测试？</font>

# 六、实现

* 使用Fabric作为私有链，golang编写链码逻辑实现接受Deep Learning的批次更新的参数，并存储到kv数据库中
* 使用Ethereum Ropsten测试网作为公链,其中每个节点都用MateMask创建了钱包账户，其上使用合约实现存储Hash

* 使用了两种联邦学习平均算法：1.signSGD 2.FedAvg

# 七、提升措施/优化技术

> 此部分多为技术上的细节优化，不是很重要，简记

1. 使用base64进行二进制压缩

   Fabric的链码不支持输入二进制的参数, 必须是字符串的字符. 使用Go实现Base64编码进行压缩

2. 防止键碰撞

3. 一些体系结构的改进
4. 缓存链码

# 八、实验

公链消耗数据结果分析：

![hYEov7](http://xwjpics.gumptlu.work/qinniu_uPic/hYEov7.png)

参数变化与合约变化在上传时间上的分析：

![sji2Zi](http://xwjpics.gumptlu.work/qinniu_uPic/sji2Zi.png)

两种模式下的参数融合效果分析：

![1Pmlcz](http://xwjpics.gumptlu.work/qinniu_uPic/1Pmlcz.png)

（貌似signSGD更加快速）

后门注入检测算法：

![C6V1nl](http://xwjpics.gumptlu.work/qinniu_uPic/C6V1nl.png)

![xZSX39](http://xwjpics.gumptlu.work/qinniu_uPic/xZSX39.png)

