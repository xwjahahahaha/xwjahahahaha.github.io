---
title: 斯坦福密码学-3-分组密码block_cipher
tags:
  - null
categories:
  - knowledge
  - null
toc: true
declare: true
date: 2021-08-01 15:43:02
---

# 一、What is a block cipher?

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801160941922.png" alt="image-20210801160941922" style="zoom:67%;" />

<!-- more -->

## 1. PRPs 和 PRFs

**伪随机函数和伪随机置换**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801162905429.png" alt="image-20210801162905429" style="zoom:67%;" />

## 2. PRP和PRF安全定义

### 安全PRF

安全PRF的定义如下：

<font color='#e54d42'>伪随机函数$S_F$与$Funs[X,Y]$​（真随机函数）不可区分</font>

> <font color='#39b54a'>如果不可区分，那么就可以享受真随机函数的巨大Size，使得破解困难（无法确定映射函数）</font>

![image-20210801165113865](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801165113865.png)

> <font color='#39b54a'>将单个的X映射到Y变成了由[k, *]映射到Y，通过固定了部分输入即k降低了整个大小，同时还能保持不可区分性</font>

直观的理解：

![image-20210801170311152](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801170311152.png)

一个例子：

![image-20210801171018679](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801171018679.png)

### 安全PRP

将$Funs[X,Y]$限制是一对一函数即$Perms[X]$就可以表示安全PRP

![image-20210801171303026](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801171303026.png)

应用案例：

**通过伪随机函数生成伪随机发生器**

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210801195831517.png" alt="image-20210801195831517" style="zoom:67%;" />

# 二、DES数据加密标准

 The data encryption standard(DES)

DES主要讨论的就是对于分组密码格式中的**密钥拓展算法**和**回合函数**

## 2.1 DES的历史

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802164024167.png" alt="image-20210802164024167" style="zoom:67%;" />

## 2.2 DES的构造

### Feistel 网络

![image-20210802164906899](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802164906899.png)

可逆性的证明：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802170022767.png" alt="image-20210802170022767" style="zoom:67%;" />

进而整体的逆运算就是解密算法：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802170732659.png" alt="image-20210802170732659" style="zoom:67%;" />

进而推出的定理：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802171400378.png" alt="image-20210802171400378" style="zoom:67%;" />

> <font color='#e54d42'>**Feistel网络是从PRF走向PRP的桥梁！**</font>

### DES

#### overview

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802172111141.png" alt="image-20210802172111141" style="zoom:67%;" />

#### 回合函数

其中的**回合函数**：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802200948000.png" alt="image-20210802200948000" style="zoom:67%;" />

#### S盒

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802201101377.png" alt="image-20210802201101377" style="zoom:67%;" />

S盒的设计规范：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802203148583.png" alt="image-20210802203148583" style="zoom:67%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802203318899.png" alt="image-20210802203318899" style="zoom:67%;" />

> <font color='#e54d42'>一句话总结： **选择的S盒的输出表现不可以是非常接近一个线性函数的输出**</font>

## 2.3 穷举攻击 - Exhaustive Search Attacks

目标：根据一对输入与输出找出其中的DES密钥

问题的前提是：一对明文/密文就确定一定是对应一个密钥吗？

Dan Boneh老师给出了证明：

![image-20210803165701765](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803165701765.png)

**并集上届**在第一部分讲过：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803165859668.png" alt="image-20210803165859668" style="zoom:67%;" />

并集上届直观上很好理解，但是对于$Pr[DES(k, m)=DES(k',m)] <= 1/ 2^{64}$​是如何计算的呢？

下面一个例子希望能让你明白：

DES假设是理想的随机函数，我们认为是一个6面骰子，那么问题就转换为同时掷两个6面骰子，两个骰子数字相同概率的计算方法，我想应该很好计算：

两个骰子总工会出现6*6=36种结果，其中相同面的结果占6个，概率就是$\frac{1}{6}$

同理，DES理想随机函数单个分组加密的输出是64位，两个相同的概率就是:

$\frac{2^{64}}{2^{64}*2^{64}} = \frac{1}{2^{64}}$

现在问题的前提是已经被证明是正确的了即：**一对明文、密文对可以大概率的确定唯一对应一个密钥加密**

事实上，**如果有两对明文密文对用key作为密钥，则通过类似的推算这个key唯一的概率将会基本为1**

顺带一提，这一点对AES也成立, 如果你看AES-128，给定两个明文密文对以很高的概率只有一个密钥

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803171832696.png" alt="image-20210803171832696" style="zoom:67%;" />

---

现在的问题是怎样得到这个密钥

最暴力的方法-穷举：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803193313332.png" alt="image-20210803193313332" style="zoom:67%;" />

> <font color='#e54d42'>56位的DES已经不安全了，不可再使用</font>

那么，安全性问题该如何解决呢？

简单直接的方式 => 增加DES的密钥长度

### 3DES

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803193900412.png" alt="image-20210803193900412" style="zoom:67%;" />

> <font color='#39b54a'>需要注意的是，3DES并不是直接加密三次，中间的一次是解密. 并且三次都使用**独立的密钥**</font>
>
> 为什么这样设计？
>
> 原因：这仅仅只是一个技巧，为了当k1=k2=k3时，先加密再解密3DES退化成DES而已

2的168次方的时间，就算地球上所有机器一起工作10年也是不能破解的

> 事实上，<font color='#e54d42'>**任何大于2的90次方的都可被认为是充分安全的**</font>

大家可能认为3DES的安全性是2的168次方的,但事实上，有一个简单的攻击，只需2的118次方(下面会说到)

2的118次方被认为是对穷举攻击充分安全的, 是个足够高的安全级别了

*为什么不能直接加密两次？*

### 中间相遇攻击

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803200108942.png" alt="image-20210803200108942" style="zoom: 60%;" />

核心在于，**可以将解密/加密算法用于两边制造碰撞（判断相等），然后建表查询**

虽然3DES也可以用此方法实现$2^{118}$时间内查找，但是还是安全的

3DES实际上是NIST标准,所以3DES实际上应用广泛，而DES不应再被使用了,如果出于某些原因，你一定要用DES的某种变体，用3DES，别用DES

----

增强DES抵抗穷举攻击能力的方法

这个方法没有被NIST标准化，因为它**不能抵抗针对DES更为复杂的攻击**，不过如果只考虑穷举攻击, 又不想承担3DES的性能开销，这就是个有趣的想法:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803202552025.png" alt="image-20210803202552025" style="zoom:60%;" />

## 2.4 更多对分组密码的攻击

### 侧道攻击和故障攻击

**旁/侧道攻击**： 通过统计测量加密**时间**或**功耗**的角度来破解密码

**故障攻击**：人为的让其计算错误，暴露密钥

侧信道攻击(Side-channel attacks, SCA)是一类攻击者尝试通过观察侧信道泄露来推测目标计算的信息。（例如，时间，功率消耗，电磁辐射，噪声等。）

故障攻击（Fault attacks, FA）是一类攻击，在这种攻击中，攻击者尝试利用错误计算的结果。错误可能导致程序或者设计错误（例如 Intel著名的FDIV错误），或者被攻击者诱导的错误（例如，能量故障，时钟故障，温度变化，离子束注入等）。

![image-20210804170441730](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804170441730.png)

> <font color='#e54d42'>**使用标准库中的密码而不是自己去编写实现或创造！**</font>

### 线性密码分析

发现了一条定理, 明文与密文对之间存在某种依赖:

根据这个定理对于有一点点线性关系的密码算法就可以进行攻击

例如DES，其第五个S盒设计时有点接近线性函数也就导致了被这种分析降低了攻击时间

![image-20210804192217457](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804192217457.png)

DES的这一点点偏差在较大量的（具体数量关系见上图）明文/密文对的线性分析下就可以很轻松的得到其密钥和

![image-20210804192752789](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804192752789.png)

在第五个S盒的输入输出上运用这个方法就可以将整个密钥的时间复杂度降下来

> <font color='#e54d42'>不要自己设计密码，按标准的来。因为一点点的线性关系都会导致严重的后果</font>
>
> <font color='#39b54a'>这个定理的线性分析类似于机器学习/深度学习的过程，在大量的数据集的支持下就可以将数据之间的某种线性关系分析出来</font>

### 量子攻击

量子计算机的由来：

20世纪七八十年代，Richard Feynman对于当前经典计算机无法完成量子实验的计算提出了自己的设想：

量子物理问题对于经典计算机来说就是无法计算的，需要特定的计算机来实现也即量子计算机

因为量子计算机有着与经典计算机不同强大的计算能力，所以可以作为对传统密码学算法的一种攻击

但是目前是否能够真正的制造出这样的计算机仍然是一个迷

![IjZ3sH](http://xwjpics.gumptlu.work/qinniu_uPic/IjZ3sH.png)

> <font color='#39b54a'>量子计算对与密码学的影响是巨大的，因为整个社会、网络安全等各方面都基于密码学难题</font>

> <font color='#39b54a'>量子计算机能否制造出来未知，但是理论上使用其进行计算已有了明确的算法，即Grover</font>

![image-20210804232939465](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804232939465.png)

# 三、AES

## 3.1 AES历史

因为DES或者3DES对于硬件来说都时间都太慢，所以NIST启动了一个新的分组加密标准叫做**高级加密标准AES**

![image-20210807155528632](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807155528632.png)

## 3.2 Subs-Perm网络

AES不基于Feistel网络而是基于Subs-Perm（substitution permutation 代置换）网络

最大的区别在于位数的改变，在Feistel中每回合有一半的位是不变的，而Subs-Perm网络中每回合每一位都要改变

![image-20210807160120442](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807160120442.png)

> <font color='#e54d42'>DES通过Feistel整体提供可逆性，对于DES其中的S盒是不可逆的，其输入6位，输出4位； 而AES则是每一个步骤都是可逆的，这样才保证了整体上的可逆</font>

## 3.3 构造

![image-20210807160636652](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807160636652.png)

对于上方的每个轮函数(蓝色区域), 其具体过程如下： 

![image-20210807162006057](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807162006057.png)

关于AES字节代换、行置换、列混淆的一些补充:

![image-20210807174357429](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807174357429.png)

![HYsN5E](http://xwjpics.gumptlu.work/qinniu_uPic/HYsN5E.png)

## 3.4 代码大小与时间的平衡

![image-20210807163144640](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807163144640.png)

## 3.5 应用

网络上的应用：

![image-20210807163551013](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807163551013.png)

> <font color='#39b54a'>先将AES计算库发送给客户端，客户端预先计算所有的表，随后每次交互只要将基本数据发送给客户端即可</font>

![image-20210807164040223](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807164040223.png)

## 3.6 攻击

![image-20210807164540591](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807164540591.png)

尽管已有一些攻击，但是这些攻击的效果在如今看来依旧是安全的

# 四、用PRGs构建分组密码

<font color='#e54d42'>构建分组密码，实际上就是构建一个PRP伪随机置换</font>

我们的思路: $PRG => PRF => PRP$

从伪随机数发生器构建伪随机函数

从一位的安全PRG开始推导：

![image-20210807165925563](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807165925563.png)

> <font color='#39b54a'>根据$G(k)[.]$中的 . 选择左右</font>

拓展到两位$G_1(k)$：

![image-20210807171310777](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807171310777.png)

证明两位的$G_1(k)$是安全的：

![image-20210807171608843](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807171608843.png)

再次拓展:

![image-20210807171947646](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807171947646.png)

拓展n次：

![image-20210807173249028](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807173249028.png)

> <font color='#39b54a'>虽然构造过程很美，但是不实用！</font>

可以从PRF到PRP吗？可以！通过Feistel网络的结构即可！

![image-20210807173548330](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210807173548330.png)
