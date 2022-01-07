---
title: cosmos-1-理论知识
tags:
  - cosmos
categories:
  - block_chain
toc: true
date: 2021-02-04 14:39:07
---

# 资料

官网：https://cosmos.network/intro

白皮书：https://cosmos.network/resources/whitepaper

个人白皮书总结（思维导图）：http://ginblog.gumptlu.work/Cosmos.pdf

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/Cosmos-20210723173235422-20210723173646734-20210723173701980.svg" alt="Cosmos" style="zoom:5%;" /> 

点击查看大图，浏览器不支持在线阅读pdf可以点此下载 <a href="http://yoursite.com/the.pdf">Download PDF</a>.</p> 


# 一、概述与简介

## 1.是什么

**Cosmos 是一个独立并行区块链的<font color='red'>去中心化网络</font>，每个区块链都由 [Tendermint](https://cosmos.network/intro#what-is-tendermint-core-and-the-abci) 共识这样的 BFT 共识算法构建**。

Cosmos 不是一个产品， 而是**建立在一套模块化、适应性强和可交互工具之上的生态系统。适合于公有链或者私有链。**

三个特点：

- **<font color='green'>模块化开发：</font>**Cosmos 通过 Tendermint BFT 和 模块化的 Cosmos SDK 使区块链易于开发。
- <font color='green'>**跨链**</font>Cosmos 使区块链能够通过 IBC 和 Peg-Zones 相互转移价值， 同时让它们保留主权。
- <font color='green'>**强拓展性：**</font>Cosmos 允许区块链应用通过水平和垂直可扩展性解决方案可支持数百万用户。

## 2.区块链层级

### 2.1. 三层模型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204152113.png)

- **应用程序：** 负责更新给定的一组交易，即处理交易的状态，**更新状态**。
- **网络：** 负责交易和共识相关**消息的传播**。
- **共识：** 使节点能够就系统的当前**状态达成一致**。

### 2.2. 耦合性

* BitCoin

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204152747.png)

  比特币系统三层耦合在一起，比特币脚本语言只支持交易处理，没有虚拟机的支持无法实现智能合约

* Ethereum

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204153003.png)

  以太坊构建了一个人们可以部署任何类型应用的区块链。 以太坊通过将应用层转换为称为[以太坊虚拟机(EVM)](https://learnblockchain.cn/2019/04/09/easy-evm/)的虚拟机来实现这一点。该虚拟机能够处理称为智能合约的程序，任何开发人员都可以以***无许可***的方式部署到以太坊区块链。 这种新的方法允许成千上万的开发人员开始[构建去中心化应用（DApps）](https://learnblockchain.cn/2018/01/12/first-dapp/)。 

  **<font color='orange'>在应用层实现了EVM虚拟机来处理智能合约，简化了去中心化应用的开发，但是整体上还是耦合的</font>**

  缺陷：

  1. **可拓展性（scalability）**

     所有的DApp都在Ethereum一条链上运行，采用PoW的以太坊效率很低，建立在[以太坊](https://learnblockchain.cn/2017/11/20/whatiseth/)之上的去中心化应用程序被每秒15交易数的共享速率所抑制。

  2. **可用性（Usability）**

     合约编程语言有限制，开发人员编程有较低的灵活性，不能实现代码的**自动执行**，[以太坊智能合约](https://learnblockchain.cn/2018/01/04/understanding-smart-contracts/)的执行需要有外部账号的触发动作。

  3. **主权（Sovereignty）**

     每个应用程序在主权方面都受到限制，因为它们都共享相同的基础环境。应用程序出现问题（例如智能合约出现漏洞The DAO事件）需要以太坊平台的改变才能解决，而以太坊平台是多个应用程序的共享平台。

* Cosmos

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204154948.png)

  1. **将网络层和共识层打包成通用引擎用于开发**，允许开发人员专注于应用程序开发，而不是复杂的底层协议。

  2. [Tendermint BFT 引擎](https://github.com/tendermint/tendermint)通过使用 [ABCI（Application Blockchain Interface）](https://github.com/tendermint/abci) 套接字（socket）协议连接到应用程序。 这个协议可以用任何编程语言进行封装，开发者可以选择适合他们适合的语言。

     ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204155527.png)

  3. 进一步对于应用层，[Cosmos SDK](https://cosmos.network/sdk) 是一个通用框架，使用**模块化**的思想简化了在 Tendermint BFT 之上构建安全区块链应用的过程。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204155711.png)

  4. 通过Tendermint BFT引擎构建的各个区块链Zone可以通过[区块链间通信协议（IBC：Inter-Blockchain Communication protocol）](https://blog.cosmos.network/developer-deep-dive-cosmos-ibc-5855aaf183fe)实现跨链的操作。

  个人看法：<font color='orange'>Cosmos并非完全解耦，共识层与网络层打包成为一个通用引擎便利于开发者***为各种应用程序创建独享的链***这样每个应用的主权就是完整的，同时在应用层上通过cosmos sdk来模块化开发从而减少重复性开发，IBC解决跨链互通问题。但是Cosmos的共识机制是不能替代的（为了保证通用性），比较于Fabric或其他主流的联盟链框架，Fabric在共识层上实现了可拔插的共识协议，共识层解耦度更高</font>

## 3.网络结构

### 3.1 Zone

Cosmos网络中各自独立的区块链，多条Zone与多个Hub组成复杂的Cosmos网络。

每一个Zone都依赖于Tendermint Core也就是Tendermint BFT引擎。

### 3.2 Cosmos Hub

在 Cosmos 网络中推出的**第一个** Hub 是 Cosmos Hub。 Cosmos Hub 是一个开放的**权益证明（POS）**的区块链，其原生 staking 代币为 ATOM，并且交易费用可以用多个 Token 支付。 Cosmos Hub 的推出也标志着 [Cosmos 主网上线](https://cosmos.network/launch)。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208153658.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208150447.png)

#### 功能职责

* 能够与其他的Zone进行拓展
* 所有的Zone之间的代币交换都需要经过Hub, Hub记录代币种类以及记录各个Zone中代币的总额记录
* Cosmos Hub区块链承载的是多资产分布式账本，其中代币可以由个体用户或空间本身持有。这些代币能够通过特殊的 IBC 包裹，即"代币包"（coin packet）从一个Zone转移到另一个Zone。Cosmos Hub可看作不同区块链之间交易的枢纽,使全网的代币总量保持恒定.
* 负责流通Atom代币，是hub唯一的可质押代币。可用于汽油费来避免垃圾交易，额外的Atom和汽油费奖励给Validator与代理人
* 当有超过三分之一的Validator投票停止系统或者有超过三分之一的Validator审查到恶意行为证据时，Hub必须通过硬分叉reorg-proposal恢复

### 3.3 light client

网络中的轻客户端，新的客户端需要对当前网络进行同步：

#### 同步过程

1. 同步当前所有Validator集合

2. 确定网络最新状态

   通过验证最新区块结果的绝大多数（>2/3）投票结果，轻型客户端可以定期与验证器设置的更改保持同步，以避免远程攻击，与以太坊类似，Tendermint允许应用程序在每个块中嵌入一个全局Merkle根Hash，根据应用程序的性质，允许对账户余额、合同中存储的价值或是否存在未使用的交易输出等事项进行容易验证的状态查询。

## 4.角色

### 4.1 Validator验证者

#### 1.概念

应用层的角色，拥有投票权重的节点， 负责**出块**与**投票**。

每个区块链都由一组验证者维护，他们的工作是**同意下一个区块提交给区块链**。

**如何选取Validator是由开发者自由决定的，每个验证器的投票权重也是**，如果采用持有的Token来选择，那么就是区块链就是权益证明POS（Proof-of-Stake）。如果只能是经过授权或许可才能成为验证者那么就是许可链或者私有链。

<font color='red'>Tendermint BFT 只处理区块链*网络*和*共识*，它帮助节点**传播交易**和**验证追加交易**到区块链。**出块共识与主链共识是分开的。**</font>

<font color='orange'>Tendermint采用确定性轮询机制的实用拜占庭容错协议。在**出块选举阶段**，不采用工作量证明来实现而是采用规定节点在主账户存入保证金（Atom）才能实现投票（成为验证者），投票权重与保证金数量（Atom）成正比。每轮轮询机制选出的出块者为Leader或proposer</font>

> 联盟链系统 Tendermint[81]采用基于轮询机制的实用拜占庭容错协议对新区块达成共识.在出块节点选举 环节,Tendermint 采用确定性轮询机制决定出块节点.由于未采用类似工作量证明的身份定价机制,为防止拜占庭节点发动女巫攻击,系统规定节点必须在账户存入保证金才能参与拜占庭容错协议的投票过程,保证金数额 与票数成正比.

#### 2.成为验证者

* 数量上的限制

  在创世日那天，验证器的最大数量将被设置为100个，这个数字将以13%的速度增长10年，并稳定在300个验证器。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204181005.png)

* 途径

  持有Atom可以通过签名和提交**BondTx**事务成为Validator，任何人任何时间只要持有Atom那么就可以抵押自己的Atom成为验证者,除非验证者的数量为当前最大，则触发替换机制。

* **替换机制**

  如果当前验证器集合数量已经是最大，那么想要成为新的验证器就需要比当前集合中的最小验证器的质押大（有效的质押Atom包括别人委托的质押Atom），被替换的验证者变成不活动的，进入unbonding状态。

* **惩罚机制**

  如果Validator违反了规定那么就会受到处罚，例如在同一区块中双重签名或者违反prevote-the-lock（Tendermint的共识协议）中的规定

  如果是因为断电或故障那么会损失ValidatorTimeoutPenalty(默认1%)的股份。

  如果恶意节点的违规是不易找到证据的，那么可以通过绝大多数投票将其强制超时

  处一般为削减Atom和代币股份

### 4.2 No-Validator/delegator

可以将自己的权益委托给Validator进行使用，从而获得部分占额的出块利息以及出块奖励，但是这个代理过程需要自己承担风险，并且自己需要支付一定的佣金给Validator，这可以由系统来决定。

## 5.协议

### 5.1 IBC

#### 1.概念

`inter-blockchain communication protocol` 区块链间通信协议, **将开发者自由定义的区块链（Zone）连接起来的协议。**

IBC 利用 Tendermint 共识的“强一致性”（其他的具有“强一致性”共识引擎也可以），以允许**异构链之间相互转移价值（如 token）或数据**。

异构链：网络、共识、应用层结构与实现方式不同

IBC 允许异构链之间转移价值（如 token）和数据，这意味着具有不同应用程序和验证人集合的区块链是可互操作的。 例如，它允许公有链和私有链间相互转移 token。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204180146.png)

#### 2.作用

Hub与Zone交流的协议， 用于**同步状态**。各个zone不断提交区块确认让Hub能够同步每个zone的状态

每个zone与Hub保持同步，同一个Hub的zone之间没有直接的同步，但是可以通过Hub间接实现同步

#### 3.包结构

* **Coin Packet**

  代币包，跨链时发送的特殊的IBC包，它必须**有发送链、Hub链、接收链的确认。**

* **IBCBlockCommitTX**

  用于提供**可证明的最近的区块Hash**，**证明区块正确性与存在性。**

  其中的**交易结构**：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208145602.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208145629.png)

  SimpleProof是针对验证区块Hash的，AppHash则是采用AVL+树保存应用程序状态

*  **IBCPacketTx**

  提供**最近区块的Merkle-proof**，**证明给定的包确实是发送方应用程序发布的**

  证明交易的正确性

  交易结构:

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208145813.png)

  **其中IBCPacket的结构：**

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208151528.png)

  Payload或PayloadHash中必须有一个存在。IBCPacket的Hash是头和负载这两个项的简单Merkle根。没有完整负载的IBCPacket称为<u>缩写包</u>。

  **其中IBCPacketHeader的结构：**

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208151642.png)

  **在整个协议传递过程中SrcChainID 和 DstChain始终不会改变**，当“Zone1”想通过“Hub”向“Zone2”发送数据包时，无论Merkle化的数据包是在“Zone1”、“Hub”还是“Zone2”上, IBCPacket数据都是相同的。**唯一可变的字段是跟踪交付的状态。**

#### 3.连接过程

类似于区块链上的Tcp/IP协议

* 发起者不需要确认

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208151815.png)

更新Hub有关于Zone1的区块，则在IBCBlockCommitTX上必须包含Zone1的块hash，这样IBCBlockCommitTx交易发布在Hub上就使Hub包含了Zone1的块Hash

* 发起者需要确认的情况

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208151925.png)

  发送方可以通过将初始包状态设置为AckPending来要求发送确认。然后，接收链有责任通过在应用程序Merkle-Hash中包含一个**缩写的**IBCPacket（没有完整负载的IBCPacket称为<u>缩写包</u>）来确认发送

  •   1. 首先，在“Hub”上发布IBCBlockCommit和IBCPacketTx，这证明了“Zone1”上存在IBCPacket

  •   2. 接下来，在“Zone2”上发布IBCBlockCommit和IBCPacketTx，这证明了“Hub”上存在IBCPacket。

  •   3. 接下来，“Zone2”必须在它的app-hash中包含一个缩写包，该包显示AckSent的新状态。

  IBCBlockCommit和IBCPacketTx被发回“Hub”上，这证明了“Zone2”上存在一个缩写的IBCPacket。

  •   4.最后，“Hub”必须更新包的状态，从AckPending到AckReceived。这个新最终确定状态的证据应返回

  到“Zone2”。

* 确认超时的情况

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208152457.png)

  如果“Hub”没有从“Zone2”收到350高度内的AckSent状态，它将自动将状态设置为Timeout。超时的证据可以在“Zone1”上发回，并且可以返回任何代币

#### 4.工作流程

IBC 背后的原理相当简单。 我们以链 A 上的一个帐户想要发送 10 个 Token（假设是 ATOM）到链 B 为例介绍。

> Atom 是 Cosmos Hub 的原生货币。 持有 Atom 可以获得投票权，可以委托给维护 Cosmos Hub 网络的验证者。

**跟踪（Tracking）**

链 B 会不间断地接收链 A 的报头，反之亦然。 这允许每个链跟踪其他链的验证者集合。 <font color='red'>从本质上讲，每个链运行一个其他链的轻客户端。</font>

> 轻客户端是一个区块链客户端，只下载块头。<font color='red'> 它通过 Merkle Proof 来验证查询结果。 这为用户提供了一个轻量级的替代全节点又具有良好的安全性的方案</font>。

**锁定（Bonding）**

当 IBC 转移被启动时，ATOM 被锁定（Bonding）在链 A 上。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208153030.png)

**中继证明（Proof Relay）**

然后，需要一个从链 A 转移到链 B 的 10 个 ATOM 被锁定的证明。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208162427.png)

**验证（Validation）**

链 B 上针对链 A 的区块头的证明进行验证，如果有效，则在链 B 上创建 10 个 ATOM 凭证（ATOM-vouchers）。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208153158.png)

注意， **在链 B 上创建的 ATOM 不是真正的 ATOM,** 因为 ATOM 仅存在于链 A 上。**它们是链 A 中 ATOM 在 链 B 上的表示形式，** **同时还证明了这些 ATOM 被冻结在链 A 上。**

当他们回到其原始链时， 也使用类似的机制来解锁 ATOM。

<font color='orange'>**个人理解：IBC的包传递过程的核心就是传递Merkle证明，以此来证明资金的锁定情况**</font>

### 5.2 ABCI

`Application Blockchain Interface`  **区块链应用接口**，Tendermint Core使用ABCI与区块链应用联系, 使得编程区块链应用可使用多种语言

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204155527.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210204155711.png)

##### 消息类型

有三种，从核心core发向应用程序，应用程序作出对应的响应，ABCI请求/响应是简单的Protobuf消息

**<font color='red'>ABCI是底层（共识层与网络层）即Tendermint Core与应用层的交互方式</font>**

1. **AppendTx消息**

   区块链中的每个交易都和此消息一起交付，应用程序需要根据事务的当前状态、应用程序协议和加密凭据验证使用AppendTx消息接收到的每个交易。然后经过验证的交易将更新应用程序的状态，存储到k-v数据库或更新UTXO数据库

   结构：

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208154324.png)

2.  **CheckTx消息**

   类似于AppendTx，但是**只用于验证交易，不改变状态**

   •   Tendermint Core的内存池首先**通过CheckTx检查交易的有效性**，然后只将有效的交易转发给其他节点

   •   应用程序可以检查事务中不断递增的nonce，如果nonce旧，则在CheckTx时返回错误。

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208154433.png)

3.  **Commit消息**

   Commit消息用于计算**对当前应用程序状态的加密承诺**，并将其放入下一个块头。

   •   有一些方便的属性。更新状态的不一致会显示为区块链分叉，它会捕获所有类型的编程错误

   •   简化了安全轻量级客户机的开发，因为可以通过对块哈希进行检查来验证Merkle -hash证明，并且块哈希

   由法定数量的验证器(通过投票)签名。

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208154551.png)

4. **附加的ABCI消息**

   允许应用程序跟踪和更改验证器集，并让应用程序接收块信息，如高度和提交投票。

## 6.共识机制

### 6.1 Tendermint BFT

部分同步的BFT共识算法，衍生于DLS共识算法

* **与PBFT的比较：**

1. Tendermint**区块按顺序提交**（只有第N提交了>N的块才能后续提交）这就避免了与PBFT的视图更改相关的复杂性和沟通开销

2. TBFT**将一些交易打包成块同时采用Merkle哈希应用程序的状态**，这比PBFT以检查点的方式周期性的摘要能够更快的确认交易和提高通信速度

3. 采用类似于LibSwift的方式将区块进行部分划分提高通信能力

4. 在弱点多点通信中也可以正常工作

* **投票过程**

  **两个阶段**

  1. **PreVote预投票**

  2. **PreCommit预确认**

  **投票过程**

  •   1. 每个Validator可以对当前的区块进行投票或者投票为空（nil）

  •   2. 当对于当前区块有大于2/3的PreVote时则将其称之为**Polka**

  •   3. 对于当前区块有大于2/3的PreCommit则成为**区块的确认**

  •   如果本轮对于单一的区块有大于2/3的投票为空则进入下一轮

  **timeoutproposal**

  •   每一轮都需要对当前的leader进行检测，并且需要防止始终达成nil不提交块的共识

  •   所以每个Validator在PreVote之前都会等待一个随机的时间=>**timeoutproposal**  

  •   并且timeoutproposal随着每轮投票的增加而增加

  •   在此期间整个系统的进程是异步的，只有Validator收到了>2/3的确认才会继续进行

  **同一高度只提交一个区块（保证强一致性）**

  1. 首先预提交/确认的区块必须是**Polka**状态的区块

  2. <font color='red'>如果Validator已经在R_1轮预提交了一个块，我们说他们被锁定在那个块上，并且用于在R_2轮验证新的预提交的Polka必须来自`R_1 < R_polka <= R_2`</font>

  3. 验证器必须提议并且预先投票它们锁定的块

  **保证已经预提交的验证器不能提供证据来预提交其他内容**

## 7.安全性

### 7.1 拜占庭机制

确保只有大于三分之一的拜占庭节点才会破坏网络安全

·    PBFT的安全保障（已证明）

·    如果Hub有三分之一以上机器宕机或者为恶意节点那么可以通过硬分叉恢复

### 7.2 锁定机制

·    任何一组Validator违反安全或者试图攻击网络都会被协议识别

·    例如投票给冲突区块，广播不公正投票等

### 7.3 分叉问责制

当共识失败，法律系统会进行识别和惩罚，当法律系统不可靠或调用成本过高时，验证者可能被迫缴纳保证金才能 

参与，而当检测到恶意行为时，这些保证金可能会被撤销或削减。

## 8.数据结构

### 8.1 Merkle树

#### 1. 简单树

·    此Merkle树用于对**块的交易和应用程序状态根**的顶级元素进行Merkle化。

#### 2.AVL+树

·    AVL+树类似于以太坊的Patricia tries

·    作为平衡二叉树，梅克尔证明平均较短。

·     使用AVL算法的一种变体来平衡树，**所有操作都是O(log(n))**。

·    在AVL树中，任意节点的两个子树的高度相差最多为1。每当更新时违反了这个条件，就会通过**创建O(log(n))个新节点来重新平衡树**，**这些新节点<u>指向</u>旧树中未修改的节点**。

·    原始AVL内部节点也可以持有键值对，与原始的AVL算法不同的是，<font color='red'>AVL+算法采用的是所有的值保留在叶子节点上，而只使用分支节点存储键  => **搜索快速，验证快速**</font>

·    键在插入IAVL+树之前不需要取Hash，因此这提供了键空间中更快的有序迭代，这可能有利于某些应用程序。

## 9.链上规章制度

实行一套实现约定好的规章制度为今后的系统问题作出解决  => **防止出现重大问题系统分叉**

Cosmos的Validator和delegators可以通过提案的方式修改系统参数、协调升级系统、回滚政策、调整规章制度等

来快速改善系统bug，每个zone也可以制定自己的政策

## 10.交易费

### 1. 个人交易

·    每一个Hub Validator都可以接受**任意的代币组合**来支付交易的汽油费

·    并且由**自己来主观决定汇率是多少**

·    当交易结束后就会按照对应的费用扣除

### 2. 系统

·    系统收取的所有交易费的**百分之二**用户储备，作为系统安全性和价值的依据，这些资金也可以根据治理系统的决策进行分配。

## 11.性能

恶劣条件下每秒数千交易，提交延迟大约在1~2秒

取自夏青论文中的对比结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210208161307.png)

## 12.应用

### 1.分布式交易所

Cosmos DEX, 也就是去中心化的交易所，实现跨虚拟代币系统的交易,交易双方不需要同时在线,交易者可以提交限价指令，进行交易。在Cosmos中，**Hub的职责就是一个分布式跨链交易所.**

### 2.桥接其他加密货币

#### 1.2.1.  概念

·    桥接的区块链必须同步保持最新的区块，以此来实现Merkel Proof

·    Cosmos需要和加入的其他虚拟货币保持同步

·    桥接Zone的方式简单并且不用知道其他链的共识模式

#### 1.2.2.  一般过程

**发送代币到Cosmos Hub**

•   1. bridge-zone运行一个Tendermint-core的区块链并且生成一个特殊的桥接应用，同时**在原链（原加密货币链）上运行一个全节点**

•   2. 当原链有新区块出现时，bridge-zone的所有的Validator通过签名和分享他们各自的本地视图来实现对当前提交区块正确性的判定一致结果

•   3. 当运行的全节点收到付款后（原链是pow机制的话需要足够多的确认），在bridge-zone中创建相关的账户并且有对应的余额

**从Cosmos Hub收回代币到原链**

•   1.在原链上将原链代币转移到**特定的地址**

•   2. IBC包能够证明在bridge_zone上发生了代币销毁交易（币由zone转向Hub）

•   3. 原链上确认bridge_zone被销毁后（以太坊就是发布交易到合约）就可以允许原链代币被撤回

#### 1.2.3.  连接以太坊

**原链发送币到Cosmos**

•   1. 发送以太币到bridge_contract的账户上

•   2. bridge_contract会记录当前bridge_zone上对应的Validator集合，这个集合可能和Hub的Validator相同

•   3. bridge_zone确认后创建对应的账户和余额

**Cosmos返回币**

•   通过向以太坊特定的取款地址交易销毁

•   证明交易发生在bridge-zone的IBC包发布到以太坊的桥接合同上，从而允许以太坊被撤回。

#### 1.2.4.  连接比特币

**原链发送币到Cosmos**

•   1. 类似于以太坊但是没有合约

•   2. 将UTXO使用P2SH的多重签名进行控制 

•   3.由于P2SH的限制，一般与Hub的Validator集合不同

**Cosmos返回币**

•   因为P2SH的多重签名的签名人集合会发生变化，所以一旦变换就需要迁移UTXO到新的UTXO，以此来适应签名人集合的改变

•   一种方法是压缩和解压缩UTXO的集合，以此来降低频繁改变UTXO所带来的UTXO集合的过大

## 13.激励措施

创世纪上atom代币和验证器的初始分发将流向Cosmos资金筹集者(75%)、主要捐赠者(5%)、Cosmos网络基金会

(10%)和ALL IN BITS, Inc(10%)。从创世纪开始，每年1/3的原子总数将奖励给绑定验证者和授权者。

#### 黑客漏洞奖励

为了鼓励和尽早的发现漏洞，Cosmos鼓励黑客可以通过**ReportHackTx** 交易向系统发布漏洞，如果无误的话则每个人会拿出5%的atom奖励给黑客地址

## 14.优缺点

优点：拓展性强，通过Hub实现跨链支持去中心化的跨链，能够拓展当前主流公链；轻客户端同步当前网络状态迅速高效；解决了公钥认证问题。

缺点：对接比特币/以太坊等已有链的跨链资产返回还未设计完全；没有合约引擎无法部署和使用合约（但是可以用组件实现：**ETHERMINT**）；法律监管、pos、规章制度是否是中心化的隐患

## 15.其他

Cosmos与波卡的区别：https://xiaozhuanlan.com/topic/0567839241