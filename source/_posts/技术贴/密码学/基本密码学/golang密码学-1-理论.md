---
title: golang密码学-1-理论
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2020-10-29 19:09:21
---

# 一、学习目录

1. Hash算法
2. DES、3DES、AES对称加密
3. RSA非对称加密算法
4. RSA数字签名算法
5. 椭圆曲线加密算法ECC
6. 椭圆曲线数字签名算法ECDSA
7. 椭圆曲线secp256k1算法
8. 编码解码算法（base64、base58）
9. 参考网站

http://tools.jb51.net/code

http://www.fileformat.info/tool/hash.htm

<!-- more -->

# 二、密码学家族介绍

## 2.1 分类

* 不可逆的哈希算法
* 加密解密算法可逆，但是必须要有密钥
* 编码解码算法可逆，不需要密钥

## 2.2 Hash算法（消息摘要Message Digest）

MD4、MD5、Hash1、ripeMD160、SHA256（比特币）、SHA3（以太坊）、Keccak-256（以太坊）

## 2.3 加密解密算法

### 2.3.1 对称加密算法

DES、3DES、AES等

### 2.3.2 非对称加密算法

RSA、椭圆曲线加密算法

### 2.3.3 数字签名算法

RSA数字签名算法、椭圆曲线数字签名

## 2.4 编码解码算法

Base64、Base58（区块链）

# 三、Hash算法

hash（哈希或者成为散列）算法能将任意长度的二进制值（明文）转换为（映射为）较短的固定长度的二进制值（Hash值）

又被称为：数字指纹、数字摘要、消息摘要

## 3.1 特点

一个好的Hash算法应该满足的特点：

- 正向快速： 正向hash很快
- 逆向困难： 在有限时间难以逆推出明文
- 输入敏感：雪崩效应，输入有一点点的变化，hash结果都会发生巨大的改变
- 抗冲突抗碰撞
  - 冲突、碰撞：不同的明文产生了相同的Hash
  - 抗冲突：尽可能的使不同明文产生的输出不相同，<font color='red'>注意抗冲突不一定是要完全的避免冲突，而是要让攻击者找到有冲突的两个输入的代价是非常大的</font>

## 3.2 典型

MD4、MD5、SHA-1和SHA-2系列（SHA-224、SHA-256、SHA-384、SHA-512，后面的数字是位数）

SHA = Secure Hash （安全Hash）

* MD4/MD5都是由麻州理工大学的Ronald.Rivest在1990和1991年设计的，MD是Message Digest的缩写，<font color='green'>目前MD5已经被证实不具备有“强抗碰撞性”，它和SHA-1算法的都已经被证明安全性不足应用于商业场景。</font>

* SHA（Secure Hash Algorithm加密哈希算法）是一个Hash函数库
  * SHA-1在1995年面世，输出为160位，原理与MD4相同
  * 为了提高安全性，后面出现了SHA-2系列和SHA-1的原理类似。SHA-224、SHA-256、SHA-384和SHA-512并称为SHA-2
  * <font color='red'>SHA-256是区块链中使用的加密算法，输出256位二进制，使用16进制的数来表示就是64位16进制数</font>
  * SHA-3,之前称为Keccak算法，是一个加密杂凑算法，**Keccak算法是SHA3的前身，以太坊一开始使用的是Keccak算法，后来被规范为SHA-3算法**
    * 以太坊中仍然使用的是Keccak-256这样的算法
    * Keccak的输出长度分别有：512、384、256、224

* RIPEMD-160（RACE完整的原始评估信息摘要）
  * 是一个160位的加密Hash函数，旨在替代128位的MD4与MD5（**不希望像SHA256那样的长度，但是仍然希望保持安全性**）
  * 首先在EU项目RIPE的框架中开发的

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223120745.png)

## 3.3 Hash算法的攻击

**寻找碰撞法、穷举法、密码字典暴力破解**

1. 寻找碰撞法

   * 目前对于MD5和SHA1并不存在有效的寻找碰撞的方法

   * 王小云破解MD5，实现了预期步骤数从2^80降到了2^69，降低了很多的数量级，但是这仍然是一个天文数字。

     ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223122246.png)

2. 穷举法与字典破解

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223122029.png)

3. 彩虹表攻击

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223122339.png)

   彩虹表的防范：

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223122537.png)

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223122612.png)

## 3.4 Hash与加密解密的区别

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223122942.png)

**简单来说就是Hash不需要密钥，而加密解密是需要密钥的。Hash不可逆，而加密解密是可逆的**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223123013.png)

两者方式的选择：

**需要知道原始的明文，那么就使用加密解密**

**不需要知道原始的明文那就使用Hash**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223123046.png)

# 四、对称加密

又叫私钥加密算法、单密钥加密算法、传统密钥加密算法

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223162140.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223162459.png)

对称的最大**特点**就是**<font color='red'>使用相同的密钥</font>**

最大的**问题**在于<font color='red'>**密钥的共享的安全性**</font>

##  4.1 DES与TripleDES（3DES）算法

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223162740.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223162913.png)

**DES的密钥是64bit位，对应就是8字节，其中8bit位为奇偶校验位，另外56位作为密码的长度**

**3DES的密钥是DES的三倍，就是192bit位即24字节**

## 4.2 AES

### 4.2.1 基本概念

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223163240.png)

**AES是一种加密标准，其中典型的实现就是Rijndael算法**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223163427.png)

**AES可以选择密钥的长度，128、192、256bit位**

### 4.2.2 加密模式

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223163655.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223163754.png)

# 五、非对称加密

## 5.1 发展史

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223163838.png)

## 5.2 基础概念

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223163953.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223164017.png)

**非对称：加密与解密的密钥不同**

**公钥加密，私钥解密： 非对称加密**

**私钥加密，公钥解密：数字签名**

**<font color='red'>非对称加密的算法既可以作为加密解密算法也可以作为数字签名算法</font>**

## 5.3 对称与非对称的区别

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223164456.png)

## 5.4 椭圆曲线加密算法

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223164734.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223164805.png)

![image-20201223164907975](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20201223164907975.png)

## 5.5 数字签名

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223165014.png)

只验证：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223165046.png)

明文加密+验证

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223165445.png)

# 六、字符编码/解码

## 6.1 Base64

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223165858.png)

## 6.2 Base58

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223170004.png) 

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223170118.png)

## 6.3 Base58Check

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201223170153.png)

检验比特币地址是否正确

