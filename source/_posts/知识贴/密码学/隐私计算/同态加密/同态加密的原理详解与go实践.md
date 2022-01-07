---
Hotitle: 同态加密的原理详解与go实践
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2021-07-16 11:01:23
---

> 学习资料来源：
>
> 知乎VenusBlockChain: https://zhuanlan.zhihu.com/p/110210315
>
> 知乎刘巍然：https://www.zhihu.com/question/27645858/answer/37598506
>
> https://blog.csdn.net/Gouph/article/details/106179325

# 一、基本概念

## 1.1 同态加密

***什么是同态加密？***

提出第一个构造出全同态加密（Fully Homomorphic Encryption）[Gen09]的Craig Gentry给出的直观定义最好：

> A way to delegate processing of your data, without giving away access to it.
>
> 一种委托数据处理的方法，但是让你不丧失对数据的所有权

<font color='#e54d42'>同态加密对于数据安全来说，不像一般的加密方案只关注**数据存储安全**，攻击者无法从密文中获得任何信息, 对加密数据的任何改动操作都会造成解密的错误。而同态加密关注于**数据的处理安全**，其提供了一种对加密数据处理的功能，且处理过程中无法得知原始内容，同时数据经过操作后还能够解密得到处理好的结果。</font>

同态加密（Homomorphic Encryption）允许对密文处理后仍然是加密的结果。即对密文直接进行处理，跟对明文进行处理后再对处理结果加密，得到的结果相同。从抽象代数的角度讲，保持了同态性。

同态加密是**基于数学难题**的计算复杂性理论的密码学技术，它的概念可以简单的解释为：对经过同态加密的数据进行密文运算处理得到一个输出，这一输出解密结果与用同一方法处理未加密的原始数据得到的输出结果是一样的。

<!-- more -->

可以定一个运算符$\Delta$ , 对应的加密算法E和解密算法D, 同态加密满足: $E(x\Delta y) = E(x) \Delta E(y)$

***一个例子：***

(刘老师的例子非常贴切)

Alice想让工人加工自己的金子但是不信任工人，害怕其在操作过程中偷取自己的金子，那么就相出了以下的办法：

- Alice将金子锁在一个密闭的盒子里面，这个盒子安装了一个手套。
- 工人可以带着这个手套，对盒子内部的金子进行处理。但是盒子是锁着的，所以工人不仅拿不到金块，连处理过程中掉下的任何金子都拿不到。
- 加工完成后。Alice拿回这个盒子，把锁打开，就得到了金子。

这个盒子的样子大概是这样的：

![GiDNiP](http://xwjpics.gumptlu.work/qinniu_uPic/GiDNiP.png)

这里面的对应关系是：

* Alice：数据持有方
* 工人： 不可信的服务提供第三方

- 盒子：加密算法
- 盒子上的锁：用户密钥
- 将金块放在盒子里面并且用锁锁上：将数据用同态加密方案进行加密
- 加工：应用同态特性，在无法取得数据的条件下直接对加密结果进行处理
- 开锁：对结果进行解密，直接得到处理后的结果

## 1.2 分类

同态性来自代数领域，一般包括四种类型：加法同态、乘法同态、减法同态和除法同态。

同时满足加法同态和乘法同态，则意味着是代数同态，称为**全同态**(Full Homomorphic)

同时满足四种同态性，则称为**算数同态**。

<font color='#e54d42'>对于计算机来说，实现了全同态就可以实现所有操作的同态性</font>

只实现部分特定操作的同态性称为**特定同态**, 对于特定同态特性的算法，如RSA，Elgamal，Paillier、Pedersen Commitment等等。

目前业界使用的较多的还是特定同态/部分同态，而斯坦福大学的博士生Craig Gentry基于**理想格**提出一个全同态加密方案。Fully Homomorphic Encryption Using Ideal Lattices. In the 41st ACM Symposium on Theory of Computing (STOC), 2009.

所谓的格（Lattice）就是整系数基的线性组合构成的点，也就是一个空间中的一些离散有规律的点。离散的点之间的距离产生了一些困难问题，例如：最短向量问题(SVP)。

如果是一个二维平面，那么寻找在格上寻找最短向量问题是简单的，但是当维数变大的时候，例如200多维，寻找格上的最短向量问题就变的异常困难，称之为格上标准困难问题，是一个指数级的困难问题。 

Gentry首次设计出一个真正的全同态加密体制，即可以在不解密的条件下对加密数据进行任何可以在明文上进行的运算，使得对加密信息仍能进行深入和无限的分析，而不会影响其保密性。

> IBM公布的开源代码：FHE ：https://github.com/IBM/fhe-toolkit-linux

## 1.3 应用

***云计算***

![rb1HwZ](http://xwjpics.gumptlu.work/qinniu_uPic/rb1HwZ.png)

Alice通过Cloud，以Homomorphic Encryption（以下简称HE）处理数据的整个处理过程大致是这样的：

1. Alice对数据进行加密。并把加密后的数据发送给Cloud；
2. Alice向Cloud提交数据的处理方法，这里用函数f来表示；
3. Cloud在函数f下对数据进行处理，并且将处理后的结果发送给Alice；
4. Alice对数据进行解密，得到结果。

可想而之，在未来如果实现了真正可以实用的全同态技术(加强环节中的f()的操作空间)，那么一款完全不会泄漏个人数据隐私的云计算服务的市场竞争有多大！

> 对于Function f()是否也可以进行加密呢，这样云服务器连操作都不会知晓

***区块链***

使用同态加密技术，运行在区块链上的**智能合约可以处理密文**，**而无法获知真实数据**，极大地提高了隐私安全性。

这样的优点是，用户将交易数据提交到区块链网络之前，可使用相应的加密算法对交易数据进行加密，数据以密文的形式存在，即使被攻击者获取，也不会泄露用户的任何隐私信息，同时密文运算结果与明文运算结果一致。

数据的操作还可以与区块链的**智能合约**相关联

![TZ7T29](http://xwjpics.gumptlu.work/qinniu_uPic/TZ7T29.png)

华为区块链提供同态加密库，对用户的交易数据用其公钥进行加密保护，交易的时候都是密文运算，最终账本中加密保存，即使节点被攻破，获取到账本记录也无法解密。

趣链Hyperchain通过同态加密（采用Paillier同态加密算法）的加密思想实现区块中交易金额和账户余额的加密。其白皮书声称经过同态加密的交易验证时间约为10微秒，可以满足Hyperchain每秒上万笔交易的需求。

BCOS也采用了Paillier同态加密算法，并开源出了加法同态解说使用说明[1]，以及Paillier同态加密算法JAVA版实现[2]。当然还有其他产品，此处不再一一列举。

# 二、实现原理

同态加密也有很多的实现算法

## 2.1 Paillier

类别:  <font color='#fbbd08'>**加法**同态加密算法</font>

困难：分解两个大质数

Paillier加密算法[3]是1999年Paillier发明的基于复合剩余类的困难问题的加法同态加密算法

### 算法生成过程：

![dQbHXB](http://xwjpics.gumptlu.work/qinniu_uPic/dQbHXB.png)

### 加密

* 明文m, 0<=m<=n
* 随机数r, 0<r<n, 且gcd(r, n) =1
* 加密结果: $c = g^m * r ^n mod n^2$

### 解密

* 解密结果：$m = L(c^\lambda)modn^2 * \mu  mod n$

### 同态性的证明

![0idXJv](http://xwjpics.gumptlu.work/qinniu_uPic/0idXJv.png)

## 2.2 ElGamal

类别：<font color='#fbbd08'>**乘法**同态加密算法</font>

困难：离散对数问题

ELGamal密码是除了RSA之外最有代表性的公开密钥密码之一，是一种公认安全的公钥密码。

### 离散对数问题

设p为素数，若存在一个正整数α，使得 $a,a^2,a^3,…,a^{p-1}$ 关于模p互不同余，则称α为模p的一个**原根**。于是有如下运算：

a的幂乘运算： $y = a^xmod p , 1 \leqslant x \leqslant{p-1}$

a的对数运算： $x = log_a^y, \ 1 \leqslant y \leqslant {p-1}$

只要p足够大，求解离散对数问题时相当复杂的。离散对数问题具有较好的单向性。

### 密钥生成

- 随机地选择一个大素数p，且要求p-1有大素数因子，将p公开。
- 选择一个模p的原根α，并将α公开。
- 随机地选择一个整数d（1＜d＜p-1）作为私钥，并对d保密。
- 计算公钥$y=a^d(modp)$ ，并将y公开。

### 加密

- 对于待加密明文M，随机地选取一个整数k（1＜k＜p-1）。
- 计算$U= a^kmodp 、C_1=a^kmodp 、C_2=U*Mmodp$

- 取(C1,C2)作为密文。

#### 解密

计算$V = C_1^dmodp$

解密结果: $M = C_2V^{-1}modp$

### 同态性的证明

![OmOuqx](http://xwjpics.gumptlu.work/qinniu_uPic/OmOuqx.png)

# 三、golang代码实现

## 3.1 Paillier

代码库：https://github.com/Roasbeef/go-go-gadget-paillier

采用Go语言实现的加法同态加密算法Paillier 

其主要可以实现以下操作：

* 可以将加密的整数加在一起 

* 加密的整数可以与未加密的整数相**乘** 

* 加密的整数和未加密的整数可以加在一起

### 安装

`go get github.com/roasbeef/go-go-gadget-paillier`

### 测试Demo

```go
func Test_homomorphicCrypto(t *testing.T)  {

	// 生成一个128位的私钥
	privKey, _ := paillier.GenerateKey(rand.Reader, 128)

	// 公钥加密明文：数字15
	m15 := new(big.Int).SetInt64(15)
	c15, _ := paillier.Encrypt(&privKey.PublicKey, m15.Bytes())

	// 私钥解密密文：15
	d, _ := paillier.Decrypt(privKey, c15)
	plainText := new(big.Int).SetBytes(d)
	fmt.Println("Decryption Result of 15: ", plainText.String()) // 15


	// 公钥加密数字20
	m20 := new(big.Int).SetInt64(20)
	c20, _ := paillier.Encrypt(&privKey.PublicKey, m20.Bytes())

	// 将15的密文与20的密文相加（注意都是同一公钥）
	plusM16M20 := paillier.AddCipher(&privKey.PublicKey, c15, c20)
	// 使用私钥解密和的明文结果
	decryptedAddition, _ := paillier.Decrypt(privKey, plusM16M20)
	fmt.Println("Result of 15+20 after decryption: ",
		new(big.Int).SetBytes(decryptedAddition).String()) // 35!

	// 将15密文与10明文常数相加
	plusE15and10 := paillier.Add(&privKey.PublicKey, c15, new(big.Int).SetInt64(10).Bytes())
	decryptedAddition, _ = paillier.Decrypt(privKey, plusE15and10)
	fmt.Println("Result of 15+10 after decryption: ",
		new(big.Int).SetBytes(decryptedAddition).String()) // 25!

	// 将15密文与10明文常数相乘
	mulE15and10 := paillier.Mul(&privKey.PublicKey, c15, new(big.Int).SetInt64(10).Bytes())
	decryptedMul, _ := paillier.Decrypt(privKey, mulE15and10)
	fmt.Println("Result of 15*10 after decryption: ",
		new(big.Int).SetBytes(decryptedMul).String()) // 150!
}
```

```shell
# 结果显示
=== RUN   Test_homomorphicCrypto
Decryption Result of 15:  15
Result of 15+20 after decryption:  35
Result of 15+10 after decryption:  25
Result of 15*10 after decryption:  150
--- PASS: Test_homomorphicCrypto (0.00s)
PASS
```

值得注意的是该包的作者说其用于课程教学，实际的生产环境则不太适合

# 四、当下挑战

同态加密最大的问题是效率

效率。效率一词包含两个方面，一个是**加密数据的处理速度**，一个是这个**加密方案的数据存储量**。

* 工人戴着手套加工金子，肯定没有直接加工来得快嘛~ 也就是说，隔着手套处理，精准度会变差（**现有构造会有误差传递问题**），加工的时间也会变得更长（**密文的操作花费更长的时间**），工人需要隔着操作，因此也需要更专业（**会正确调用算法**）。

* 金子放在盒子里面，为了操作，总得做一个稍微大一点的盒子吧，要不然手操作不开啊（**存储空间问题**）。里面也要放各种工具吧，什么电钻啦，锉刀啦，也需要空间吧？

2011年，Gentry和Halevi在IBM尝试实现了两个HE方案：Smart-Vercauteren的SWHE方案[SV10]以及Gentry的FHE方案[Gen09]，并公布了效率。结果如何呢？我们给出Gentry公布的数据（原始数据可以在[2nd Bar-Ilan Winter School on Cryptography](https://link.zhihu.com/?target=http%3A//crypto.biu.ac.il/winterschool2012/)找到）Smart-Vercauteren的SWHE方案效率如下：

![IsCUut](http://xwjpics.gumptlu.work/qinniu_uPic/IsCUut.png)

看着好像还行，不过这Dimension有点夸张啊…也就是说公钥很长…那么，Gentry的FHE方案如何呢？效率如下：

![69jQG9](http://xwjpics.gumptlu.work/qinniu_uPic/69jQG9.png)
