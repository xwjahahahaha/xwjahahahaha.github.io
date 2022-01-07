---
title: 《Mitigating_Data_Poisoning_Attacks_On_a_Federated_Learning-Edge_Computing_Network》精读
tags: 
  - null
categories:
  - knowledge
  - paper
toc: true
declare: true
date: 2021-07-27 16:38:28
---

# 一、基本信息与前置知识

## 1.1 基本信息

**Mitigating Data Poisoning Attacks On a Federated Learning-Edge Computing Network**

* 来源：https://ieeexplore.ieee.org/abstract/document/9369581

* 期刊/会议： [2021 IEEE 18th Annual Consumer Communications & Networking Conference (CCNC)](https://ieeexplore.ieee.org/xpl/conhome/9369428/proceeding)
* 作者：[Ronald Doku](https://ieeexplore.ieee.org/author/37086853836); [Danda B. Rawat](https://ieeexplore.ieee.org/author/37394227500)

<!-- more -->

## 1.2 前置知识

### Label-flipping 标签翻转攻击

投毒攻击中的一种，一般是将二分类问题中的标签变为其反面标签，例如0变为1，1变为0

# 二、解决的问题

**缓解**联邦学习中的毒药攻击

# 三、创新的方法

## 思路框架

前提:

* FL去中心化的训练对于每个数据的来源提前可知

要求：

* 保障客户端节点数据安全性，与边缘计算服务器交互的隐私保护

做法：

* 对数据源在FL之**前**进行审查

* 在客户端节点和边缘计算服务器之间引入中间人facilitator
* 检测的重点是**标签翻转攻击**(label-flipping attack)

* 将有毒数据集和无毒数据集分别通过分类模型计算错误率差异来判断是否被毒害

* 由于FL不支持直接访问客户端的数据，所以采用一个**中间人去访问**

* FL的优势在于可以依靠**主动**的方式进行检测（原因见前提）

  > <font color='#39b54a'>集中式的ML数据来源不确定不可预知，而FL则是确定了数据持有人进行的训练</font>

* 中间人使用SVM支持向量机模型作为检测工具

过程图：

![流程图](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803211120977.png)

过程并不复杂, 整个思路复杂的设计在于**SVM审查模型的设计**

## SVM审查模型的设计





# 四、总结



