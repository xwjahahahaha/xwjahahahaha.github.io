---
title: fiat-shamir零知识证明合约
tags:
  - solidity
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-08-05 10:53:15
---

> 文章学习自微众银行智能合约代码征集活动第一期中的隐私计算任务
>
> 微信：由@周小周发布
>
> 代码地址：https://github.com/WeBankBlockchain/SmartDev-Contract/tree/dev/contracts/business_template/privacy_computation/Fiat-Shamir-ZK


<!-- more -->

# 一、Fiat-Shamir零知识证明协议

## 场景

Fiat-Shamir with secret password

Fiat-Shamir零知识证明协议可以**允许用户向注册的网站证明自己知道自己的密码**，而**不向网站泄露**任何关于密码的信息。

## 角色

* 登陆者：合约调用方

* 验证者：区块链合约

# 二、流程介绍

* 参与者：登陆者Peggy，验证者Victor
* 公共参数：一个素数$n$以及一个模n群的生成元$g$

## Step-1 注册

Peggy选择一个密码，使用Hash函数取摘要，然后将其转换为一个整型设为$x$​, 即$x= int(Hash(passwd))$

构造$y = g^x \ mod \ n$​  发送$y$​​​给Victor

Victor保存下$y$ （区块链环境中存储在合约中）, 表示着用户Peggy已经注册成功

## Step-2 登陆

Peggy开始登陆，自己生成一个随机数记为$v$, 计算:

$t = g^v mod \ n$

将$t$发送给Victor

## Step-3 验证端随机数挑战

Victor收到$t$之后，发送一个随机数给Peggy记为$c$

## Step-4 客户端应答

Peggy收到随机数$c$之后计算：

$r = v - cx \ mod \ (n-1)$

并将$r$发送给Victor

## Step-5 验证

Victor收到后开始验证计算：

$t' = (g^r)(y^c) \ mod \ n$

通过判断$t' == t$判断用户是否拥有该密码

## 分析

作为验证端的Victor可以获取到的信息：$y 、t、r$

根据离散对数困难，Victor无法得知 $x 、v$​​ 所以可以做到：

用户可以向验证端证明自己拥有密码却不需要透露任何除了‘拥有密码’这一知识之外的任何知识（例如密码本身）

# 三、合约详解

可以将合约看作是验证端的逻辑表达，那么核心就是验证的逻辑

整体的合约以及注解如下：

```js
pragma solidity >=0.4.16 <0.9.0;

contract FiatShamir {
    //============Phase 0: 公共共享参数===================
    // prime
    uint public n = 8269;
    // generator
    uint public g = 11;
    //=========================================================
    
    // g^x mod n
    uint y;
    // Victor's random challenge    Victor的随机数挑战
    uint public c;
    // Peggy sends random t 
    uint t;
    
    //======Phase 1: 注册：Peggy sends y to Victor, Victor store y as Peggy' token==================
    // Victor存储Peggy的密码通证即y
    // Peggy registers with y,  y = g^x mod n
    function Step1_register( uint _y) public {
        y = _y;
    }
    //=======================================================================================
    
    //======Phase 2: 登陆：Peggy wants to login , She send t to Victor=============================
    // Victor存储登陆信息t
    function Step2_login(uint _t) public {
        t = _t;
    }
    //=======================================================================================
    
    //======Phase 3: 验证端随机数挑战：Victor choose c randomly ,and sends it to Peggy=========================
    function Step3_randomchallenge() external returns (uint){
        c = randomgen();
        return c;
    }

    // TODO : NOT secure , low entropy ,change random source.  
    // 这里使用区块时间+Hash求余这样的伪随机数产生器是不安全的，PRF的熵变化太低 （可以优化）
    function randomgen() private view returns (uint) {
        return uint(keccak256(abi.encodePacked(block.timestamp))) % n;
    }
    //=======================================================================================
    
    //======Phase 4: 客户端应答: Peggy recieves c and calculate r=v-cx, sends r to Victor================
    //======Phase 5: 验证：Victor calculates (g^r)*(y^c)== t? =====================================
    function Step45_verify(uint r) public  returns (bool){
        uint256 result = 0;
        
        result = (modExp(g,r,n)*modExp(y,c,n)) % n;
        
        return t == result;
    }
    //=======================================================================================
    

    // modular algorithm : calculate  b**e mod m
    function modExp(uint256 _b, uint256 _e, uint256 _m) private returns (uint256 result) {
        assembly {
            // Free memory pointer
            let pointer := mload(0x40)

            // Define length of base, exponent and modulus. 0x20 == 32 bytes
            mstore(pointer, 0x20)
            mstore(add(pointer, 0x20), 0x20)
            mstore(add(pointer, 0x40), 0x20)

            // Define variables base, exponent and modulus
            mstore(add(pointer, 0x60), _b)
            mstore(add(pointer, 0x80), _e)
            mstore(add(pointer, 0xa0), _m)

            // Store the result
            let value := mload(0xc0)

            // Call the precompiled contract 0x05 = bigModExp
            if iszero(call(not(0), 0x05, 0, pointer, 0xc0, value, 0x20)) {
                revert(0, 0)
            }

            result := mload(value)
        }
    }
}
```

# 四、使用测试

为了快速的测试，测试使用的python生成，手动将参数输入到测试合约中，就没有使用web3js了

首先将合约部署到remix平台：

![image-20210805144656738](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805144656738.png)

## step-1 注册

```python
import libnum
import hashlib
n=8269
g=11

password = "Hello"

print("Password:\t\t",password)
x = int(hashlib.sha256(password.encode()).hexdigest()[:8], 16) % n
print("Password hash(x):\t",x,"\t (last 8 bits)")

print('\n======Phase 1: Peggy sends y to Victor,Victor store y as Peggy\' token==================')
y= pow(g,x,n)
print('y= g^x mod P=\t\t',y)
```

![image-20210805143955475](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805143955475.png)

remix中输入2889:

![image-20210805144911987](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805144911987.png)

注册成功

## Step-2 登陆

```python
import libnum
import hashlib
import random
n=8269
g=11

v = random.randint(1,n)

print('\n======Phase 2: Peggy wants to login , She send t to Victor==================')
v = random.randint(1,n)
t = pow(g,v,n)
print('v=',v,'\t(Peggy\'s random value)')
print('t=g**v % n =\t\t',t)
```

![image-20210805144102085](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805144102085.png)

remix中输入5888

![image-20210805145107800](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805145107800.png)

合约储存t

## Step-3、4 随机数挑战

生成随机数：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805145211890.png" alt="image-20210805145211890" style="zoom: 67%;" />

将1125输入到py测试程序：

```python
import libnum
import hashlib
import random
n=8269
g=11

password = "Hello"
x = int(hashlib.sha256(password.encode()).hexdigest()[:8], 16) % n


print('\n======Phase 4: Peggy recieves c and calculate r=v-cx, sends r to Victor==================')
c = input("c= ")
v = input("v= ")
r = (int(v) - int(c) * x) % (n-1)

print('r=v-cx =\t\t',r)
```

输入c=1125, v=3596, 计算得出r:

![image-20210805145612654](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805145612654.png)

## Step-5 验证

### 正确验证

remix中输入7748:

![image-20210805145801834](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805145801834.png)

验证正确

### 错误验证

remix随便输入一个数：1234

![image-20210805145856128](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210805145856128.png)

验证失败！

> <font color='#39b54a'>注意：这里的手动输入输出通过web3js完全可以自动化，所以不需要担心</font>

# 五、总结

## 优势

零知识证明，完美的保护了密码的安全性！

## 短处

1. 落地安全性可能会受限于智能合约的伪随机函数生成，所以这是优化的下一步
2. 通信、计算的复杂度提升：零知识证明双方的通信次数多了两次，计算复杂度也要求更高
3. 合约的gas费，如果每次登陆都需要gas那成本是很大的
