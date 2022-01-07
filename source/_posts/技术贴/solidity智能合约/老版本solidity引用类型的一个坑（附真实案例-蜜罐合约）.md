---
title: 老版本solidity引用类型的一个坑（附真实案例-蜜罐合约）
tags:
  - solidity
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-09 20:34:39
---

# 都是引用类型不赋初值惹的祸

## 一个例子：

注意看变量a的初始值为0：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809195043.png)

<!-- more -->

当调用f（）函数后：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200808225422.png)

a的值居然变成了1，而且多次调用f函数会发现每次a的值都会增加1。

这也太坑了！

原因在于：创建数组x的操作`uint[] x`没有添加初始的值，所以x保存的地址指向是默认指向**整个合约的开始位置**，也就是变量a的位置/地址。而Solidity中数组的机制是会使用初始的位置保存整个数组的长度值，而数组的值则是用这个长度和下标哈希映射到某个地方存储。**这个记录数组长度的位置就是a变量的位置，a等同于在记录局部数组x的长度！**所以当使用push操作的时候，数组的长度变为1，那么a的值也就变成了1。

解决方法：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200808230402.png)

## 真实案例 - 蜜罐合约

猜数游戏：猜中了就双倍返回金额，GuessHistory用来记录每次猜的信息

合约内容：

```js
pragma solidity >=0.4.0;

contract A{
    uint public luckNum = 52;
    struct Guess{
        address player;
        uint num;
    }
    Guess[] public guessHistory;
    
    function guess(uint _num) public payable{
        Guess newGuess;
        newGuess.player = msg.sender;
        newGuess.num = _num;
        guessHistory.push(newGuess);
        if(_num == luckNum){
            msg.sender.transfer(msg.value * 2);
        }
    }
    
}
```
因为以太坊上的合约的内容都是可见的，所以luckNum别人是可以直接看到的，并且在这例子中还设置了public更加方便知道数字了。（这不傻子吗？我还能猜不中？）

猜52，下注10ether

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809204740.png)

结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809204955.png)

记录上也啥都没有

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809205014.png)

明明猜对了啊？为什么？

问题所在：`Guess newGuess`这一步

struct和上面的uint[]一样都是引用类型，这里没有给其赋初始值或者说指定到某个空间，所以这里的newGuess指针就会指向合约的开头，也就是`uint public luckNum = 52;`！！

再次查看luckNum的值：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809205315.png)

居然变成了这样，这个数就是msg.sender地址形成的数！与num无关是因为luckNum只够存struct中的一个参数-地址

如果把结构体中的num换到第一个吗，那么就会变成怎么猜都是对的了，因为_num会修改luckNum的值，每次都猜对！把自己坑了。

把这个数复制，再猜这个数：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809210530.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809210613.png)


为啥这次luckNum不改变了呢，因为msg.sender没有改变。

但是，真实的情况下，luckNum是不会public的，所以钱没法拿回来，自以为是反而会被坑！

## 总结

**在老版本的solidity中，局部变量引用类型要赋初值，指向某个区域！**
