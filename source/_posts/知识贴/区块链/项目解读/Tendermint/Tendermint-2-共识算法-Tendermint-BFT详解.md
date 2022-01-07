---
title: Tendermint-2-共识算法:Tendermint-BFT详解
tags:
  - tendermint
categories:
  - knowledge
  - null
toc: true
declare: true
date: 2021-07-22 16:22:38
---

> Tendermint项目目录：
>
> * [Tendermint-1-基础概念](https://blog.csdn.net/weixin_43988498/article/details/118060169)

> 上一篇已经简单的介绍了Tendermint的基础概念，包括优势与特点、应用与生态等。下面将会详细的介绍Tendermint的共识算法，你将会学习到一下几个方面：
>
> * Tendermint-PBFT详解（high-level view -> detail）
> * TBFT与传统PBFT的比较分析

# 一、Tendermint-BFT详解

## 1.1 共识环境分析

* 角色：Validator（预先配置的网络中的一般验证者账户们）、Proposer（选举出的出块人）

* 阶段：Propose阶段、Prevote阶段、Precommit阶段
* 投票种类：prevote、precommit、commit

> <font color='#39b54a'>这些名词下面会一一介绍到</font>

<!-- more -->

## 1.2 Round-based协议

### 1.2.1 共识的开始

整个Tendermint区块链网络需要通过Round-based协议来决定下一个区块，在区块链中共识的直接目的就是**确定下一个区块内容、链接下一个区块**

共识的决定需要所有角色进行投票，vote的基本结构如图（来自于PPIO）：

![gtpHpr](http://xwjpics.gumptlu.work/qinniu_uPic/gtpHpr.png)

通过这些字段指定以下内容：

1. 当前投票的目标区块 <= Height、Hash
2. 当前投票的轮次 <= round
3. 当前投票的发起者 <= 签名

### 1.2.2 总体流程

Round-based协议的流程总的来说有五个步骤不断**循环**执行：

<font color='#e54d42'>状态机流程:  NewHeight => Propose => Prevote => Precommit => Commit </font>

* NewHeight、Commit都是特殊的流程，后面会详述
* 中间Propose、Prevote、Precommit称之为一个Round
* 一个Round不一定就一定会产生一个区块，因为其中会有共识失败的可能，一旦失败就会重新开始轮次

以下就是整个流程图(图片来自PPIO):

![CYYuL5](http://xwjpics.gumptlu.work/qinniu_uPic/CYYuL5.png) 

1. 进入新的高度阶段，等待时间后，进入Propose阶段
2. Propose阶段，选举Proposer, 然后Proposer提交一个Proposal, 进入Prevote阶段
3. Prevote阶段Validators对收到的Proposal进行prevote投票, 当达到2/3的prevote投票后进入Precommit阶段
4. Precommit阶段进行precommit投票，当达到2/3的投票后进入Commit阶段；未达到重新进入Propose阶段即Step 2

5. Commit阶段准备好新的高度以及广播了新的状态给节点，进入Step 1

以上是整个过程，下面对于每个过程进行一一详细的介绍

### 1.2.3 Proposal

选举Proposer，也就是常见共识算法中的Leader，从Validators中选举，最初的Validator的注册通过编写配置文件genesis.json在区块链创世区块中配置（后期也可通过命令行添加）

* Validator: 所有参与共识验证的注册节点
* Proposer: 每轮从Validators中选举出的出块节点

选择的规则：Round-robin

规则其实很简单：

* 初始化配置Validators后，全网的节点都会将所有的Validator数据备份到本地
* 一个区块高度一般对应一次选举即可结束，但有时单节点障碍或网络异常等原因会造成一个高度需要多个轮次
* 全网从已构建好的**Validator循环排序数组**的0位置开始指定Validator为Proposer，本次高度确定后，新的高度后依次向后指定（选择位置1的Validator为Proposer）, 达到最后时从0重新开始
* 当出现Proposer无法连接或异常时会跳过该节点继续向下指定，算法得以自动不断执行

> <font color='#e54d42'>这里也就解释了“**非阻塞轮询机制**核心是为了避免选中的Proposer中断连接,从而共识阻塞”的意思</font>

*那么，Validator循环排序数组是怎样挑选Validator的呢？*

根据Validator的votingPower来决定顺序，其votingPower越大被选中的概率也就越大。

每个Validator的**初始化的**votingPower由其质押的资金stake1:1恒定 （PoS机制需要质押资金，质押资金也是在配置文件中配置）

*怎样防止votingPower很大的Validator选举垄断？*

votingPower更新算法:

初始化后每轮votingPower都会更新：

* 本轮未被选中的Validator本轮后其votingPower增加其初始化的stake，即$votingPower= votingPower+stake$​​

* 本轮被选中的Validator本轮后其votingPower减少为Validator循环数组中其他Validator的stake之和，即
  $$
  votingPower = votingPower - \sum_{i}^Nstake_i \ , \ i \neq thisValidator
  $$

举例：初始化stake配置：A:1, B:2, C:3

(图片引自知乎：吴寿鹤) 深色表示Proposer

![image-20210723102822577](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210723102822577.png)

这张图片理解无障碍则已了解选举过程

总结：

1. 可以结合初始化配置将Pos设计成为DPoS
2. Round-robin策略的问题在于下一个Proposer可以被预测到容易引发对于特点单点主机的DDos攻击，所以还需要隐藏Validator节点的Ip地址

---

在选举出Proposer之后该Proposer提交一个Proposal，**全网广播**，其数据结构如下:

![proposal](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210723103522580.png)

> <font color='#e54d42'>Proposal最为重要的字段就是Block，其核心目的就是将自己打包的区块广播更新区块链状态</font>

### 1.2.4 Prevote

Validator 不断监听是否有新的Proposal区块…….

在每个Validator进行Prevote阶段的投票之前，需要先判断自己是否锁定在上一轮的Proposal区块上：

* 是则继续签名广播上一轮锁定的Proposal区块并广播prevote投票
* 否则签名广播当前轮Proposal区块并广播prevote投票

* 某些原因导致Validator没有锁定/收到任一Proposal区块，那么就签名广播一个**空**的prevote投票

> <font color='#39b54a'>通过不同轮次将统一网络中的Validator区分开避免多次prevote投票，prevote投票与Proposal区块一一对应</font>

### 1.2.5 Precommit

对于每一个Validator不断接收网络中的Prevote投票…….

在Precommit开始阶段，Validator会收集prevote投票:

* 如果达到了总数的2/3个prevote投票，则为这个区块签名广播precommit投票，并且当前**Validator锁定在此Proposal区块上，同时释放之前锁定的区块** , **一个Validator只能锁定在一个区块中**

* 如果其收到了超过2/3的空nil区块prevote投票, 那么释放之前锁定的所有区块
* 如果没有收集超过任何2/3的prevote投票，那么其不会锁定在任何区块

处于锁定状态的 Validator 会为锁定的区块收集 prevote 投票，并把这些投票打成包放入 **proof-of-lock** 中，proof-of-lock 会在之后的propose阶段用到。

> 对于每个Validator需要维护一个**PoLC（Proof of Lock Change）**的投票集结构:
>
> PoLC表示在一个高度h和轮次r构成的元祖$(h, r)$下, 对于一个区块b（可能为空nil）prevote投票数超过2/3的集合

### 1.2.6 Commit

对于每一个Validator不断接收网络中的precommit投票…….

在 precommit 阶段后期，如果 Validator 收集到超过2/3的 precommit 投票，那么 Validator 进入到 **Commit 阶段。**

否则进入下一轮的 **Proposal 阶段**。

**Commit 阶段分为两个并行的步骤：**

1. Validator 收到了被全网 commit 的区块，Validator 会为这个区块广播一个commit 投票。
2. Validator 需要为被全网络precommit的区块，收集到超过 ⅔ commit投票。

一旦两个条件全部满足了，节点会将 commitTime 设置到当前时间上，并且会**进入NewHeight 阶段, 开启新的一轮**

在整个共识过程的任何阶段，一旦节点收到超过⅔ commit 投票，那么它会立刻进入到commit 阶段。

# 二、共识讨论

## 2.1 为什么Tendermint-BFT可以不分叉

### 分叉的诞生

更详细的阶段流程图：

![image-20210723114007223](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210723114007223.png)

1. 在propose阶段，proposer节点广播出了新块$Block_x$ 
2. A在超时时间内没有收到这个新块，向外广播pre-vote nil，B,C,D都收到了，向外广播pre-vote投给$Block_x$ 
3. 现在四个节点进入了pre-commit阶段，**A处于红色内圈，B,C,D处于蓝色外圈**
4. 假设A由于自身网络不好，又没有在规定时间内收到超过2/3个对$Block_x$ 的投票，于是只能发出 pre-commit nil投票消息投给空块
5. D收到了B和C的pre-vote消息，加上自己的，就超过了2/3了，于是D在**本地区块链**里commit了$Block_x$​ 
6. B和C网络出现问题，收不到D在pre-commit消息，这是B和C只能看到2票投给了$Block_x$ ，一票投给了空块，全部不足2/3，于是B和C都只能 commit空块，高度不变，进人R+1轮，A也只能看到2票投给了$Block_x$，一票投给了空块，也只能commit空块，高度不变，进人R+1轮
7. 在R+1轮，由于新换了一个proposer, 提议了新的区块blockY，A,B,C 三个个可能会在达成共识，提交$Block_y$，<font color='#e54d42'>于是对于节点D来说在同样的高度，就有$Block_x$和$Block_y$两个块，产生了分叉。</font>

Tendermint加上了<font color='#e54d42'>**锁的机制**</font>，具体就是，在第7步，即使proposer出了新块$Block_y$​，A,B,C只能被锁定在第6步他们的pre-commit块上:

* A在第6步投给了空块，那么在第R+1轮，只能继续投给空块
* B、C在第6步投给了$Block_x$，那么在新一轮，永远只能投给$Block_x$

**这样在R+1轮，就会有1票投给空块，两票投给$Block_x$，最终达成共识$Block_x$，A,B,C三人都会commit $Block_x$，与D一致，没有产生冲突。** 

### 原因

首先PBFT的要求是全网有小于1/3的拜占庭节点，这是共识的基本前提条件，也是讨论分叉的前提条件。

当一个区块B在第R轮被Commit，那么就表明有大于2/3的节点在R轮投了precommit投票，进而就表明至少有**大于1/3**的节点锁定在R'>R轮

> <font color='#39b54a'>拜占庭共识的环境就是假设节点都不一定是诚实的，所以投precommit的2/3中的节点并不一定都是诚实的，所以可能会有大于1/3的节点锁定的更高轮次</font>

对于**不同轮次的同一高度的区块确定**来说也就不会出现两个2/3的prevote投票（因为有1/3在R’轮锁定）, 所以就不会分叉

> <font color='#e54d42'>对于同一个高度，想要确定此高度的最终区块，可能会产生多个轮次的投票</font>

**总结：其不可分叉来自于其BFT共识的锁定以及PoLC共同完成**

## 2.2 Tendermint-BFT与PBFT的比较

***相同点：***

- 都属于BFT类型的算法，最多容忍不超过1/3的恶意节点
- 都是三阶段提交，Tendermint的propose->pre-vote->pre-commit三个阶段，跟PBFT的三个阶段，pre-prepare, prepare, commit 三阶段是一一对应的
- 都在超时的时候，换掉proposer/primary （都是leader）

***不同点：***

不够Tendermint相对于PBFT有两处简化。 

* Tendermint没有PBFT那种View Change阶段

  Tendermint很巧妙的**把超时的情况跟普通情况融合成了统一的形式**，都是 propose->pre-vote->pre-commit 三阶段，只是<font color='#e54d42'>超时的时候新块是一个特殊的空块</font>。**切换proposer是通过提交commit空块**来触发的，而PBFT是有一个单独的view change过程来触发primary轮换。 

* Tendermint的所有信息都存储在区块链

  因为PBFT是1999年提出来的，那时候还没有blockchain这个东西(blockchain是2009年比特币出现之后才有的)，因此PBFT的所有节点虽有有一致的数据，但**数据是分散存放**的。PBFT的每个节点的数据包括: 

  > The state of each replica includes the state of the service, a message log containing messages the replica has accepted, and an integer denoting the replica’s current view. 
  >
  > 每个副本的状态包括服务状态、包含副本已接受消息的消息日志和一个表示副本当前视图的整数。

  Blockchain就是一个分布式数据库，好比在MySQL这类DBMS数据库没出现之前，人们都是把数据写入文件然后存在硬盘上，发明出各种奇怪的文件格式和组织方式。有了MySQL后，管理数据就方便多了。同理，Tendermint 把数据全部存入blockchain, PBFT没有blockchain这样一个分布式数据库，所有全节点需要自己在硬盘上管理数据，比如为了压缩消息日志，丢弃老的消息，节省硬盘空间，引入了checkpoint的概念，光是**数据管理这一块就多了很多繁琐的步骤。** 

总结：

<font color='#e54d42'>Tendermint和PBFT关系类似于Raft和Paxos的关系，Tendermint是PBFT的简化版，是针对blockchain这个场景下的简化版PBFT 。</font>

# 三、总结

**==区块链技术元素提取：==**

==改进传统PBFT方法：==

* ==Pos+BFT选举共识与主链共识的解耦合搭配==
* ==对区块施加锁机制以及PoFL，将节点绑定在一个区块的共识中防止分叉==
* ==提交nil块的投票可以省略view change流程==

