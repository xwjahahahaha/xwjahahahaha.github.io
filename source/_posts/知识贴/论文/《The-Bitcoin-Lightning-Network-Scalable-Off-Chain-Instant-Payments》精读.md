---
title: 《The_Bitcoin_Lightning_Network:Scalable_Off-Chain_Instant_Payments》精读
tags: 
categories:
  - knowledge
toc: true
declare: true
date: 2021-05-07 20:40:16
---

# 基本信息

闪电网络白皮书

<!-- more -->

# 1. The Bitcoin Blockchain Scalability Problem

当前比特币区块链的问题:

* TPS太小, 大量的小微支付交易以比特币网络的块大小将会造成大量的存储需求, 从而导致只有能够承担的节点只有中心化的节点逐渐导致网络成为中心化的网络

* 增加区块大小的方式会造成手续费的升高, 提高了交易的成本

# 2. A Network of Micropayment Channels Can Solve Scalability

微支付通道闪电网络可以解决可拓展性问题

> If a tree falls in the forest and no one is around to hear it, does it make a sound ? 

区块链中的交易双方的一些交易可以不需要让其他人/节点知道这些交易的发生, 相反的,只需要区块链知道一些(最少的)重要的信息即可

<font color='#e54d42'>**通过将每笔交易的信息延迟告知整个世界，在晚些时候对他们的关系进行净结算，**</font>使比特币用户能够进行许多交易，而不会膨胀区块链或在一个集中的对手方中造成中心化信任

通过**时间锁(time locks)**作为全局共识机制的组件, 实现中一个高效的不可信的结构

微支付渠道使用的是真实的比特币交易，只是选择将交易延迟广播到区块链，以保证双方在区块链上的当前余额;

这不是一个可信的覆盖网络，小额支付渠道是真正的比特币在<font color='#e54d42'>**链下**</font>通信和交换。

### 2.1 Micropayments Channels Do Not Require Trust

当有区块链中出现不一致的行为时, 区块链的时间戳系统可以避免, 例如相同的交易双方只有最新的交易有效.

微支付通道大致流程:

1. Alice 和 Bob 都可以创建fund交易将资金存储到一个2-2多重签名的地址(双方都签名才能支出)中

   fundTx : Alice 0.05btc,  Bob 0.05btc  => 2-2 account

2. 双方各自都可以签名一个refund交易(2-2多重签名的交易)返还金额给自己,并且不广播其到区块上(在合适的时候可以广播)

   refundTx1: 2-2 account => Alice 0.05btc, Bob 0.05btc	(双方都互相签过名)

3. 更新余额

   refundTx2: 2-2 account => Alice 0.07btc, Bob 0.03btc	(双方都互相签过名)

问题: refundTx1 和 refundTx2 怎样防止作恶,知道哪一个交易是正确的?

要保证当前只有一个正确的余额, 并且旧的余额会被丢弃

**比特币脚本**: 在比特币中可以设计一个比特币脚本，让所有旧的交易失效，只有新交易有效。无效是通过比特币输出脚本和依赖的交易强制执行的，这些交易迫使另一方将他们所有的资金交给渠道对手方。通过将所有资金作为一种惩罚给予对方，所有旧的交易因此失效。

> 论文中并没有详细的说明这样的脚本如何设计,可能后面会详细讨论

这种无效过程可以通过渠道共识的过程来存在，如果双方同意当前的余额状态(并建立新的状态)，那么实际余额就会更新。只有当有一方不同意时，余额才会反映在区块链上。从概念上讲，这个系统不是一个独立的覆盖网络;这更像是当前系统的一种状态延迟，因为执行仍发生在区块链本身(尽管推迟到未来的日期和交易)。

### 2.2 A Network of Channels 

建立一个庞大的微支付网络来解决比特币拓展性问题

每个用户只要有一个通道能够接入到这个微支付网络,那么他就可以实现微支付

通过**哈希锁hashlock**和**时间锁timelock**阻碍比特币交易输出,能够确保交易金额不被偷窃

通过使用**交错超时staggered timeouts**，可以通过网络中的多个中介发送资金，而不会有中介窃取资金的风险。

# 3. Bidirectional Payment Channels 双向微支付通道

目前，Hub-and-Spoke微支付渠道[7][8][9](以及可信的支付渠道网络[10][11])已经开始着手构建一个现在的Hub-and-Spoke网络。

闪电网络的双向微支付通道需要附录A中描述的**可延展性软分叉**，以实现近乎无限的可扩展性，同时降低中间节点违约的风险。

通过将多个微支付渠道链接在一起，就有可能创建一个交易路径网络

路径选择可以类似于使用BGP路由协议,发送方可以指定到接收方的特定路径

**输出脚本被接受方的hash所阻碍, 通过向该hash披露输入，接收方的交易对手将能够沿着该路线提取资金。**

## 3.1 The Problem of Blame in Channel Creation 渠道创建过程中的过错问题

### 3.1.1 Creating an Unsigned Funding Transaction

==Initial Channel Funding Tx==

初始渠道资金交易(Initial Channel Funding Tx)是由**一个或双方**的交易双方为该交易的输入提供资金而产生的。

输出为一个关于通道双方的2-2多重签名脚本, <font color='#e54d42'>使用这个output需要双方的签名</font>

双方为该交易创建输入与输出,但是<font color='#e54d42'>**不签署交易**目前该交易还是无效的</font>

1. 输入: 来自于单方或双方

2. 输出: 2-2双方的多重签名

   ![Lfj7Db](http://xwjpics.gumptlu.work/qinniu_uPic/Lfj7Db.png)

两个参与者都**不交换资金交易的签名**，直到他们从这2-2的输出中创建新的输出——refunding 将原始金额退还给各自的资助者。

<font color='#e54d42'>不签署交易的目的是允许**一个人从一个还不存在的交易中消费。**</font>

如果他们不合作(交换funding Tx的签名),那么这个交易就会永远的被锁定

**确定通道资金总额:** 该交易中的来自于Alice和Bob的输入需要双方**交换各自的input,** 这样就能够确定整个通道中的资金总额为多少

**交换密钥**: 交换一对密钥用于后续的签名, 这个密钥用于funding Tx中2-2多重签名output的解锁(并非Funding Tx)

### 3.1.2 Spending from an Unsigned Transaction

==SIGHASH_NOINPUT Tx==

从未签名的交易中消费

闪电网络使用`SIGHASH_NOINPUT`交易去花费2-2 Funding Tx output, <font color='#e54d42'>前提是**Funding Tx还没有交换签名即Funding Tx还没有生效**</font>

`SIGHASH_NOINPUT`使用**软分叉**确保交易能够在其各方签名之前进行消费, 因为交易都需要签名才能够计算出交易的ID/Hash**(对于Funding Tx来说双方没有签名就无法计算其Hash)**

没有`SIGHASH NOINPUT`，比特币交易在可能被广播之前就不能被消费——就好像一个人不能在没有先支付给另一方的情况下起草合同一样。`SIGHASH NOINPUT`的实现细节见附录A

没有`SIGHASH NOINPUT`，就不可能在不交换签名的情况下从交易中生成支出, 因为使用Funding Transaction需要一个交易ID作为子输入中的签名的一部分。

> <font color='#39b54a'>使用Funding Tx的子交易的输入的签名需要对整个这个子交易整个交易签名,但是Funding Tx是没有被签名的,没有Tx Hash, 所以子交易中前一个交易的Hash字段为空,所以无法签名</font>

交易ID的一个组件是父交易(Funding Tx)的签名，所以双方需要交换父交易的签名，然后才能花掉子交易。

> <font color='#39b54a'>子交易的签名需要父交易的ID,父交易的ID需要父交易的签名,所以双方需要先交换父交易的签名</font>

因为一方或双方必须知道父方的签名才能使用它，这意味着一方或双方能够在子方存在之前广播父方(funding Tx)。

> <font color='#39b54a'>双方必须互相在创建子交易之前知道父交易的(对方的)签名,所以双方都能够在知道后任何时间广播父交易, **这就违背了闪电网络的要求**</font>

`SIGHASH NOINPUT`通过允许子交易**在没有对输入进行签名的情况下消费**来解决这个问题

> <font color='#e54d42'>`SIGHASH_NOINOUT`实现了即使在父交易没有签名的情况下,子交易也可以先签名. 即花费未签名的交易</font>

操作流程如下:

1. 创建父交易(funding Tx)
2. 创建子交易(Commitment Tx 和所有花费Commitment Tx的交易)
3. 签名子交易(自己签自己)
4. 交换子交易的签名
5. 签名父交易(自己签自己)
6. **交换父交易的签名**
7. **广播父交易**

> 注意: 
>
> 1. 子交易的签名与父交易的签名所用的公私钥对不是相同的
> 2.  在步骤6完成之前，不能广播父交易(步骤7)
> 3. 直到步骤6双方都不交换他们的funding Tx签名

### 3.1.3 Commitment Transactions : Unenforcible Construction

==Commitment Transaction==

在未签署(和未广播)的funding Tx创建后，双方签署并交换初始Commitment Transaction。

这些借助于Funding Tx outputs的Commitment Txs不会被广播(其实最后一个也会被广播),最后只有Funding Tx会被广播

当Funding Transaction已经进入区块链，其output是2-2的多签名交易，需要双方同意支出，因此<font color='#e54d42'>**使用最新的Commitment transactions来表示当前余额。**</font>

参与者们只需要交换一个Commitment Tx时其余额就代表着参与者们应拿回的钱,当然,需要在founding Tx广播上链之后

**一方广播交换中最新的Commitment Tx就代表着微支付通道的关闭(结束当前余额变动/清算)**

---

一种简单的实现方式:

![RWvZ4f](http://xwjpics.gumptlu.work/qinniu_uPic/RWvZ4f.png)

> ​	图中只有Funding Tx被广播上链了, Commitment Tx还没上链

创建两个输出分别将初始资金返回给Alice和Bob

**Commitment Tx首先被签名，密钥被交换，因此双方都能够在任何时间广播Commitment Tx，前提是Funding Tx已在区块链。基于这一点,Funding Tx的签名能够被放心的交换, 因为每个人都可以赎回自己的金额**

---

改变交易状态中的余额 => 修改Commitment Tx的输出金额

当双方同意一个新的Commitment Tx并交换新的Commitment Tx的签名时，**任何一个Commitment Tx都可以广播。** 

<font color='#e54d42'>**双方都有一份相同的Commitment Tx但是最终只会有一个上链**</font>

![3CRZlF](http://xwjpics.gumptlu.work/qinniu_uPic/3CRZlF.png)

