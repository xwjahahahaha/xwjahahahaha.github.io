---
title: 计算理论基础-3-图灵机与DFA最小化
tags:
  - null
categories:
  - knowledge
  - null
toc: true
declare: true
date: 2021-06-26 14:04:39
---

# 七、Turing Machine 图灵机

## 定义：

$TM = (Q,\Sigma,\Gamma,\delta，q_0，q_{accept}， q_{reject})$

1. Q ：有限状态集合
2. $\Sigma$ : 输入字母表, 不包括空白字符
3. $\Gamma$ ：磁带字母表 (tape alphabet),    $\_ \in \Gamma, \Sigma \subseteq \Gamma$ （$\_$就是空格）
4. $\delta$ : 转移函数
5.  $q_0$ : 起始状态  $q_{0} \in Q$
6. $q_{accept}$ : 接受状态    $q_{accept} \in Q$
7. $q_{reject}$ : 拒绝状态， $q_{reject} \in Q,  \ q _{reject} \neq q_{accept}$

<!-- more -->

## Configuration of a TM (格局)

### 定义：

格局 = 状态 + 已处理部分 - 今后任务

组成：

1. 当前状态 $q \in Q$
2. 当前带内容 $\in \Gamma^*$， 符号表示为$uv$  ($uv$由读取头分隔开)
3. 读写头当前位置 $\in \{0,1,2,3,...\}$，也就是读取头当前位置即$v$的第一个符号

所以格局的表示方法可以为: $uqv$

### 例子：![E2GxpY](http://xwjpics.gumptlu.work/qinniu_uPic/E2GxpY.png)

![ZF52Ys](http://xwjpics.gumptlu.work/qinniu_uPic/ZF52Ys.png)

### 格局演进/格局转换

设$u,v \in \Gamma^*; a,b,c \in \Gamma; q_i,q_j \in Q, and \ M_a \ TM$

> <font color='#39b54a'>解释： uv是当前带内容是字符串，abc是带字母表中的字母，两个q是两个状态，总体是一个图灵机$M_a$</font>

两个格局$C_1 = uaq_ibv, \  C_2=uq_jacv$

> <font color='#39b54a'>解释：$C_1 = uaq_ibv$ 将a并入u，b并入v其实就是$uqv$</font>

$C_1$转换为$C_2$的转移函数为：

$Q \times \Gamma \rightarrow Q \times \Gamma \times \{L, R\}$

$\delta(q_i, b) = (q_j,c,L)$

> <font color='#39b54a'>解释：</font>
>
> <font color='#39b54a'>1.状态改变：$q_i => q_j$  </font>
>
> <font color='#39b54a'>2.内容改变:$b => c$ </font>
>
> <font color='#39b54a'>3.L代表left，读取头左移</font>

图示：

![9zhHaV](http://xwjpics.gumptlu.work/qinniu_uPic/9zhHaV.png)

根据状态转移图，输入格局序列，写出运行结果

例

![NB2UyV](http://xwjpics.gumptlu.work/qinniu_uPic/NB2UyV.jpg)

# 八、DFA的最小化（补充）

## 概念与意义

*DFA的*最小化就是寻求状态数最小的与原*DFA*等价的 *DFA*

最小化*DFA*能够降低编译器构造的复杂度、提高编译速度

## 方法：

* 消除多余状态
  * 不可到达终态的点
  * 不可被到达的点
* 等价合并
  * 一致性条件： q和t同为终态或非终态
  * 蔓延性条件：q和t同条件到达同状态

## 例子：

对下图的DFA进行最小化

![u7CPsi](http://xwjpics.gumptlu.work/qinniu_uPic/u7CPsi.png)

（**图中红圈的是终态**）

1. 将所有的状态划分为2个集合
   * 终态集合： $k_1 = \{q_1,q_2,q_3,q_7\}$
   * 非终态集合： $k_2 = \{q_4,q_5,q_6\}$

2. 先划分非终态集合k2

   **划分规则：根据字母表判断元素是否达到同一状态集合**

   * 字母表中的0：

     $q_4 \xrightarrow{0} q_7 \in k_1$

     $q_5 \xrightarrow{0} q_2 \in k_1$

     $q_6 \xrightarrow{0} q_2 \in k_1$

     计算得出三者都到达同一个状态集合k1，故不划分

   * 字母表中的1：

     $q_4 \xrightarrow{1} q_5 \in k_2$

     $q_5 \xrightarrow{1} \varnothing \in \varnothing$

     $q_6 \xrightarrow{1} \varnothing \in \varnothing$

     计算的出$q_4$到达状态集合k2, $q_5,q_6$到达$\varnothing$

     因此划分为$k_3 = \{q_4\}, k_4=\{q_5,q_6\}$ (原来的$k_2$不存在了，现在是$k_1,k_3,k_4$三个集合)

   $k_3,k_4$仍然是非终态集合，如果还可以划分的话继续这样划分

3. 再划分终态集合k1

   划分规则同理：

   ![GkCr3H](http://xwjpics.gumptlu.work/qinniu_uPic/GkCr3H.png)

   划分k8

   $q_3 \xrightarrow{0} q_2 \in k_6$

   $q_7 \xrightarrow{0} q_7 \in k_5$

   所以，$k_8 = k_9 + k_{10}, k_9 = \{q_3\}, k_{10} = \{q_7\}$

4. 划分完毕，拉伸合并得到最小化DFA图

   最终的划分结果是：

   ![oygVRZ](http://xwjpics.gumptlu.work/qinniu_uPic/oygVRZ.png)

   将原DFA图拉伸合并得到最小DFA图：

   ![PSX8bo](http://xwjpics.gumptlu.work/qinniu_uPic/PSX8bo.png)

