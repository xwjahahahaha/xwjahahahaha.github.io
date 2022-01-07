---
title: 《Decentralized_Privacy_Using_Blockchain-Enabled_Federated_Learning_in_Fog_Computing》精读
tags: 
  - null
categories:
  - knowledge
  - null
toc: true
declare: true
date: 2021-07-20 15:40:34
---

> 论文地址：https://ieeexplore.ieee.org/document/9019859

> <font color='#e54d42'>糟心的一篇文章，整段照抄引用文章、放公式不解释变量字母含义、题目起攻击的防御不写防御，不知这文章怎么上的一区期刊</font>

# 一、基本信息、前置知识

## 1.1 基本信息

《Decentralized Privacy Using Blockchain-Enabled Federated Learning in Fog Computing》

作者：[Youyang Qu](https://ieeexplore.ieee.org/author/37086113748); [Longxiang Gao](https://ieeexplore.ieee.org/author/37400376100); [Tom H. Luan](https://ieeexplore.ieee.org/author/37085345130); [Yong Xiang](https://ieeexplore.ieee.org/author/37286573200); [Shui Yu](https://ieeexplore.ieee.org/author/37405530700); [Bai Li](https://ieeexplore.ieee.org/author/37088420381); [Gavin Zheng](https://ieeexplore.ieee.org/author/37088420573)

出版刊物：[IEEE Internet of Things Journal](https://ieeexplore.ieee.org/xpl/RecentIssue.jsp?punumber=6488907) ( Volume: 7, [Issue: 6](https://ieeexplore.ieee.org/xpl/tocresult.jsp?isnumber=9115800))

年份：June 2020

期刊影响因子/分区：2021年9.47/**Q1**

<!-- more -->

## 1.2 前置知识

### 雾计算

> 百度百科：
>
> 雾计算（Fog Computing），在该模式中数据、（数据）处理和[应用程序集](https://baike.baidu.com/item/应用程序集/3491931)中在**网络边缘的设备**中，而不是几乎全部保存在云中，是[云计算](https://baike.baidu.com/item/云计算/9969353)（Cloud Computing）的延伸概念，由[思科](https://baike.baidu.com/item/思科/454822)（Cisco）提出的。这个因“云”而“雾”的命名源自“雾是更贴近地面的云”这一名句。
>
> 雾计算和云计算一样，十分形象。云在天空飘浮，高高在上，遥不可及，刻意抽象；而雾却现实可及，贴近地面，就在你我身边。雾计算并非由性能强大的服务器组成，而是由**性能较弱、更为分散的各类功能计算机组成**，渗入工厂、汽车、电器、街灯及人们物质生活中的各类用品。

雾计算不是具体的一种算法，而是偏向一种新型的应用概念

# 二、解决的问题

1. 分布式隐私：融合区块链和联邦学习框架解决雾计算的单点隐私问题, 通过区块链解决隐私保护 

   > <font color='#39b54a'>去中心化实现的隐私保护</font>

2. 投毒攻击证明：区块链系统提供non-tempering特点实现投毒攻击的评估

3. 高效率：一方面联邦学习只交换梯度参数，第二方面区块链只存储指针，数据通过**链下的分布式Hash表存储**

# 三、创新的方法

## 3.1 FL-Block体系框架

![P6vXDd](http://xwjpics.gumptlu.work/qinniu_uPic/P6vXDd.png)

* 在FL-Block的框架中，区块的区块体中保存所有本地设备的模型更新, 对于每个设备来说包括：

  * 在每个epoch的$(w_i^{(l)}, \{ \nabla f_k(w^{(l)}) \}_{s_k \in S_i})$​
  * 本地计算时间 ![2NSrSL](http://xwjpics.gumptlu.work/qinniu_uPic/2NSrSL.png)

  > <font color='#39b54a'>后面的式子可以简单理解: (此次参数， 此次参数的变化量)</font>

* 每个区块的大小被定义为$h + \delta_m N_V $
  * $h$代表区块头大小
  * $\delta_m$​代表模型更新大小
  
* **每一个矿工Miner拥有一个与其相关联的设备或者其他Miner的充满了本地模型更新数据的候选区块(未上链区块)**,写入区块数据的过程直到达到区块的最大数据量或者达到等待时间$T_{wait}$​

* 区块的生成速度$\lambda$（在区块头）可以被POW的难度控制,即POW的难度越大/区块目标值越小,区块生成速率$\lambda$​越小

* 系统奖励分为数据奖励和挖掘奖励

* 一个初步的验证是：通过比较样本大小$N_i$与其相关联的计算时间$T_{local,i}^{(l)}$ <font color='green'> (样本数量与其对应的计算时间是相关的,所以两者不匹配就会错误)   </font>这在实际中可以由英特尔的软件保护扩展(Intel’s  software  guard  extensions)来保证

> <font color='#e54d42'>体系框架这部分区块链设计可以看《Blockchained_On-Device_Federated_Learning》论文，因为和它一模一样…….emmm, 估计后面的时间效率分析也是一摸一样了….**抄袭过于明显了.**</font>

算法过程如下：

![hYm71k](http://xwjpics.gumptlu.work/qinniu_uPic/hYm71k.png)

## 3.2 去中心化隐私机制

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/di4JKS.png" alt="di4JKS" style="zoom:50%;" />

雾服务器（fog server）的网络被称为分布式Hash表(DHT)

为建立区块，提出了**混合身份、区块链内存、策略、辅助功能**

### 混合身份(Hybrid identity)

传统公私钥机制的区块链身份可以大量的制造假身份

私人化的定制一套身份来实现访问控制

* $u_0$代表唯一终端设备标识者

* $u_g$代表客户端设备即身份接受者

身份构建过程：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210727103127867.png" alt="image-20210727103127867" style="zoom:50%;" />

混合身份**公开**的部分：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210727103402394.png" alt="image-20210727103402394" style="zoom:50%;" />

混合身份总体:

(如果分别写g1,g2则是一个十元组，现在用$i, i=1,2$来简写为五元组)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/UP1Xu7.png" alt="UP1Xu7" style="zoom:50%;" />

> <font color='#39b54a'>貌似就是将公私钥体系用元组表示了起来，代替了中心化的PKI认证</font>

### 区块链内存(memory of blockchain)与Policy

根据区块链内存，设BM为内存空间。我们有$BM: \{0,1\}^{256}→\{0,1\}^n$，其中$N >> 256$，这足以存储大数据文件。

模型更新中的前两个输出编码为256位内存地址指针以及一些辅助元数据。其他输出被用来构建序列化的文档。

如果查询L[k]，则返回具有最新时间戳的模型更新。此设置允许插入、删除和更新操作。

> <font color='#39b54a'>kv数据库？key的大小为256位 -> value就是模型数据</font>

我们将策略$p_u$定义为终端设备v可以从特定服务获得的一系列权限。

 if v needs to read, update, and delete a dataset, then $P_v = read, update, delete.$​ 

### 辅助功能(auxiliary functions)

两个关键函数: 

##### $Parse(x)$​​​

不断的将参数传递给特定的交易

---

$Verify(pk^k_{sig}, x_p)$​​：

帮助验证终端设备的权限

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210727112859527.png" alt="image-20210727112859527" style="zoom:50%;" />

> <font color='#e54d42'>这个逻辑判断是否有问题？？？如果前面的true则后面的and $x_p$都不需要判断了。此外，原文缺少必要的符号解释，H是什么？$P_{u_0,u_{g_i}}$又是啥？（规定的政策？）</font>

当一个模型更新$A_{access}$被记录时，Protocol. 3由网络内的节点执行。类似地，当记录模型更新$A_{data}$时，Protocol. 4由节点执行。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210727113540533.png" alt="image-20210727113540533" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/4o9t6n.png" alt="4o9t6n" style="zoom:50%;" />

> <font color='#39b54a'>顶不住了，介绍太少看不懂。。</font>

## 3.3 毒害攻击与防御

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210727135504089.png" alt="image-20210727135504089" style="zoom:50%;" />

攻击者想要在训练时注入毒药，他的做法就是将上传的模型改为：

这种增强攻击使马里模型MM的权值增加$\eta = n/r$，以保证全局模型GM被MM替代。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/9hQgZQ.png" alt="9hQgZQ" style="zoom:50%;" />

这样的攻击在全局模型接近收敛的时候最好, 并且攻击者可以随意的更改学习率$\eta$的比值

> <font color='#e54d42'>然后“防御”呢？？？文章此节就说了攻击就结束了后面居然还有防御的实验评估，绝绝子，不看了。。。。再见</font>

# 四、总结

无

