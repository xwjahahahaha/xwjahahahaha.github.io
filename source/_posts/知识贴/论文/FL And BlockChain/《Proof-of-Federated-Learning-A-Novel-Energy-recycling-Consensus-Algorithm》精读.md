---
title: '《Proof_of_Federated_Learning:A_Novel_Energy-recycling_Consensus_Algorithm》精读'
tags:
categories:
  - knowledge
  - null
toc: true
declare: true
date: 2021-04-15 06:09:48
---

# Proof of Federated Learning: A Novel Energy-recycling Consensus Algorithm

## 摘要

将Pow的能源浪费问题与Federal Learning结合起来

提出了一个基于反向博弈(reverse game-based)的数据交易机制和隐私保护模型验证机制

<!-- more -->

## Introduction

1. POW:
   * 指出目前POW的电力消耗/资源浪费的问题(对比于澳大利亚、美国家庭),偏离环境友好
   * 因此后面提出了能源保护的POS共识
   * 以及能源循环: 通过计算一些有意义的任务(最长链、矩阵计算、图像切分、深度学习等)达成共识
2. 能源循环共识-PoFL
   * 联邦学习:分布式机器学习的一种方法 => 带着代码去找数据(数据不动模型动)
   * 提出PoFL的原因: **区块链与联邦学习部分组织架构相同** => 区块链PoW矿池原理与联邦学习都有一样的集群架构<font color='#39b54a'>(区块链矿池分配工作给各个矿工, 然后各个矿工根据工作量获得对应的奖励,这与联邦学习的分布式架构很像)</font>
   * PoW中的加密算法谜底 替换为联邦学习, 集群管理者(cluster head)关联每个矿工(pool members)本地高质量模型的计算结果的更新
   * **结合主要挑战**: 联邦学习本地数据的所有权与使用权是集成在一起的,而区块链是数据的所有权与使用权分离的模型<font color='#39b54a'>(链上的数据都会公开,这可能会导致数据隐私的泄漏)</font>

3. 核心工作

   1. 一个通用的PoFL架构的提出

      (设计了一种新的支持区块验证的区块结构)

   2. 一个基于**激励相容(incentive-compatible)反向博弈(reverse game-based)**的数据交易机制

      利用市场力量来防范训练数据的泄露。(确定最优的交易概率, <font color='#e54d42'>**将节点/pool数据隐私泄露概率与其交易机会以及购买支付金额挂钩**,抑制数据的泄露</font>)

   3. 一个隐私保护模型验证机制. 

      (验证训练模型的准确性,同时保护测试数据以及pool训练的模型的隐私 => **在不披露测试数据或模型本身的情况下计算模型的准确性**)  

## Framework of PoFL

### 角色分布:

1. 请求者requesters : 发布请求任务tasks(例如图像识别、语义分析)并且提供相应的任务奖励去激励矿工
2. Pool manager : 矿池管理者, 处理/分配任务数据(聚合各个miner模型实现高质量聚合ML模型)
3. Pool member(miners): 矿工, 计算子任务(更新本地模型)
4. Data provider: FL的数据提供者

### 框架:

task优先级判定: 奖励高、时间早

以当前区块链矿池为与FL相像的特点设计了如下架构:

![6wo1wc](http://xwjpics.gumptlu.work/qinniu_uPic/6wo1wc.png)

### Federated mining 进程

池成员(miner)根据他们的优先数据单独训练机器学习(ML)模型，以获得本地计算的更新，这些更新将由池管理者(pool manager)聚合，以实现**高质量的模型**

在使用请求者的测试数据进行精度计算之后，每个池管理器打包交易并生成一个包含模型验证所需信息的新区块。

一旦接收到块，全节点将通过验证模型的准确性来识别赢家池,赢家池将他的模型发送给请求者，从而获得记账权(accounting right)和相应的奖励。

> <font color='#e54d42'>一个任务多个pool同时执行,根据pool的FL模型的准确度来决定出块权</font>
>
> * 多任务是否可以多点执行
> * 数据源保证问题,数据源多的pool更为有利,能够适应更多的task数据要求

### 水平联邦学习过程举例

水平联邦学习: 特征相同,样例不同 

![vSk1Fw](http://xwjpics.gumptlu.work/qinniu_uPic/vSk1Fw.png)

1. Manager初始化模型与公钥分发
2. Miner不断本地更新梯度或者中途使用公钥接收manager信息
3. 加密发送更新值给manager, manager进行聚合形成公共模型
4. 根据pool中的模型准确度基准决定是否继续训练,如果继续则返回公共模型参数重复2, 3两步骤

第4部如果不断聚合的情况下返回miner的过程中可能会造成数据的隐私泄露(<font color='#39b54a'>暴露了对方的数据,即使已加密但是也有可能</font>)

### 反向博弈思想与最终过程隐私保护概述

**反向博弈**的思想:

利用市场的力量，pool泄露数据隐私的风险越低，pool可以购买敏感数据的概率就越高，pool需要支付的价格就越低，从而激励pool表现良好。

----

pool中的模型到达最终的模型基准后,将会测试`requester`的测试集合,这个过程中可能会有两种数据隐私的暴露:

1. 测试集隐私暴露
2. manager的最终模型 (暴露可能会被其他竞争者如其他pool的manager发现从而争夺区块)

---

提出了一种<font color='#e54d42'>隐私保护**模型(精准度)验证**机制</font>

在requester的验证集下对接收到的models都可以进行验证

设计了新的区块结构:

<font color='#e54d42'>在区块头中添加Task、Vm、Accuracy三个字段</font>来方便其他节点/pool进行验证模型的准确度:

* task: 所有矿工当前执行的任务,由平台选择
* Vm: 验证模型准确度的信息
* Accuracy: 准确度

![NhaFBu](http://xwjpics.gumptlu.work/qinniu_uPic/NhaFBu.png)

## REVERSE GAME-BASED DATA TRADING MECHANISM 反向博弈数据交易机制

**对象: Data provider 与 Manager之间**

<font color='#e54d42'>传统数据的网络传递就意味着数据的使用权与所有权的分离</font>

解决方法:

* 敏感数据加密传输 ==> 加大训练时间,影响区块生成速率与整体性能  

  作者观点: <font color='#39b54a'>在区块链中使用加密数据训练ML模型达成共识是不现实的。</font>

* 反向博弈数据交易机制

在数据提供者provider与数据使用者pool之间实现反向博弈的机制

provider 害怕数据泄漏, pool manager害怕挖矿隐私净利润….等自身隐私

<font color='#e54d42'>数据提供者可以在不知道pool的私人信息的情况下确定<u>**最优交易概率和相应的价格**</u>,从而利用市场作为工具来保护训练数据的隐私。</font>

<font color='#e54d42'>**将pool的名誉与其数据交易概率和支付价格相挂钩**</font>

名誉的计算与其被纰漏出的数据隐私泄漏次数相对应

Data provider与Manager之间的数据交易记录记录在区块链上,当有数据泄漏时就可以方便查询到源头

----

由Manager与Data Provider交易的好处:

* 防止同一个pool中过的memeber重复获取数据造成数据的浪费
* Manager掌握整个Pool的数据总量,能够更好的(平均)分配给Miner,并实现平摊奖励

数据交易过程完成后，数据提供者直接将数据发送给相应的矿机，避免了池管理器传输造成的通信开销和可能的隐私泄露风险

---

数据提供者需要提前设计好游戏规则(game rule), 方便pool根据自身的隐私信息制定策略

设计游戏规则的原则: **IC即incentive compatibility 激励相融原则**

提高pool策略的现实效率

* Pool期待的利用率公式如下:

![PF8tMY](http://xwjpics.gumptlu.work/qinniu_uPic/PF8tMY.png)

> ·<font color='#e54d42'>**利用率U可以看作一段时期的经济收益**</font>

Q: 交易的纯利润, $\tilde{c}(r, V)$交易敏感数据的预估利润, 其中V = $\eta D_s$数据越敏感,加价Ds越高,V敏感数据价值越高

> $\tilde{x}$ 代表等价无穷小? 往往表示在x周围加了一个小扰动后的另一个值

其中p 为

![YGbloO](http://xwjpics.gumptlu.work/qinniu_uPic/YGbloO.png)

**由上式可知，池的信誉越高(上式中的r)或敏感数据的最终价格越高(数据越敏感,provider加价Ds越高,防止交易敏感数据)，数据提供者越愿意出售数据.(p越大)**

在数据交易开始时,**计算p结果给交易的双方**:Manager与Provider

* Provider期待的利用率如下(变量与(1)相同):

![Dh5OFO](http://xwjpics.gumptlu.work/qinniu_uPic/Dh5OFO.png)

其中 c(r, V)代表交易敏感数据的预期损失

![pChRjM](http://xwjpics.gumptlu.work/qinniu_uPic/pChRjM.png)

* m : pool的策略集合

* $ D_s(m)$ : data provider的策略集合

数据提供者的策略不是一个值，而是一个规则，也就是**一个函数**

**这使得数据提供者能够强迫pool根据他的真实隐私信息做出最优出价, 例如Q和$\tilde{c}(r, V)$**

上图4表示了激励相容反向博弈的主要的是三个阶段:

1. 数据提供者设计一个最优的游戏规则$D^*_s(m)$, 当然是为了能够**最大化他的收益**
2. pool接收后决定是否接受这个规则,是的话就根据游戏规则计算最优的出价$m^*$,**最大化其收益**;否则就忽略
3. 在限期内数据提供者未收到出价$m^*$那么就终止谈判; 否则计算$D_s^*$和此轮交易的p(交易成功可能性). 最终双方达成一个协定后的数据交易费用就是$m^* + D^*_s$

> 双方都会最大化自己的收益,这就是博弈论

----

作者通过数学上的**变分法**与**拉格朗日法**推倒出了三个理论:

![tNjJal](http://xwjpics.gumptlu.work/qinniu_uPic/tNjJal.png)

![W8kxLy](http://xwjpics.gumptlu.work/qinniu_uPic/W8kxLy.png)

equilibrium : 均衡

**结论:**

<font color='#e54d42'>满足激励兼容原则的游戏规则驱使池根据真实隐私信息进行计算$m^*$。由于数据提供者的策略是$m^*$的函数，即$D_s(m^*)$，它也是基于真实的私有信息衍生出来的。**这相当于池和数据提供者都根据双方已知的全局信息制定最大化其效用的最优策略**</font>

此外，游戏规则的激励兼容性使得$m^*$和$D_s(m^*)$都可以揭示这轮数据交易中数据隐私泄露的风险。

(2)中成功交易概率p的计算可以减少甚至防止隐私泄露行为，因为它不仅取决于池的**历史数据隐私泄露行为**，**还取决于这一轮的隐私泄露风险。**

> <font color='#e54d42'>反向博弈论 => 双方制定真实的数据交易定价</font>
>
> <font color='#e54d42'>激励相容 => 保证双方不揭露数据的隐私给其他人</font>

## Privacy-preserving Model Verifcation Mechanism 

**对象: requester 与 Pool之间**

隐私保护的模型验证机制

pool中的全节点也可以在这里验证

模型的验证分为两个部分:

* 基于**同态加密**的标签预测
* 基于**2PC-based**的标签比对

作者以深度反馈学习为例



## Related Work

<font color='#e54d42'>代替Pow的对社会有实际价值的难题, 其关键的一点是**实现困难,验证简单**并且**计算的花费与实际的效用收益要有很好的性价比**</font>

## Conclusion

* PoFL : puzzles => FL task
* data provider && pool manager :   reverse game-based data trading mechanism
* Verify, requester && pool : 1. homomrphic encryption label prediction 2. 2PC-based label comparison

