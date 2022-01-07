---
title: solidity编程智能合约(3)
tags:
  - solidity
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-08-06 21:18:42
---

# 语法大全

## Solidity源文件布局

### pragma(版本杂注)

- 源文件可以被版本 杂注 pragma 所注解，表明要求的编译器版本

- 例如： pragma solidity ^0.4.0;

- 源文件将既不允许低于 0.4.0 版本的编译器编译， 也不允许高于（包含） 0.5.0 版本的编译器编译（第二个条件因使用^被添加）

<!-- more -->

### import（导入其它源文件）

import（导入其它源文件）

- Solidity 所支持的导入语句import，语法同 JavaScript（从ES6 起）非常类似

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151043.png)

### 值类型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151105.png)

### 引用类型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151128.png)

### 地址类型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151147.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151213.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151226.png)

### 字符数组

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151300.png)

### 枚举

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200812151308.png)

### 数组

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200806214220.png)


![](http://xwjpics.gumptlu.work/qiniu_picGo/20200806220054.png)

### 结构

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807210406.png)

### mapping

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807214232.png)


### 调用关系

以下几种情况的比对：

注意：**update函数更新的是D地址的余额！**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807215401.png)

分清楚两个msg.sender的区别：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807215632.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807220400.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807220705.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807220927.png)

猜猜D的余额？

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807221020.png)

**记住：简单的直接调用中，谁调用我，谁就是msg.sender**

以上都是简单的直接调用，当使用delegatecall代理调用则情况又不同了。

具体的合约之间的调用可见博客[北大肖臻-第22讲-智能合约](https://myblog.gumptlu.work/2020/07/13/%E7%9F%A5%E8%AF%86%E8%B4%B4/%E5%8C%BA%E5%9D%97%E9%93%BE/%E5%8C%97%E5%A4%A7%E8%82%96%E8%87%BB%E3%80%8A%E5%8C%BA%E5%9D%97%E9%93%BE%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%BA%94%E7%94%A8%E3%80%8B%E7%AC%94%E8%AE%B0/%E5%8C%97%E5%A4%A7%E8%82%96%E8%87%BB-%E7%AC%AC22%E8%AE%B2-%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6/)


### 数据位置

注意：Solidity局部类型与平时学习的情况不太一样！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807222135.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807222818.png)

- 为什么public函数的参数一定要是memory类型呢？
  - 因为公开的函数会有大量人调用，使用storage会造成资源的浪费，并且使用memory使用完就释放掉了会比较高效。
- 为什么引用类型一定要用Storage类型？
  - 使用Storage是想用一个非常的空间来防止hash碰撞，其实现原理采用的是hash映射。由此也可知道为什么mapping不支持遍历。 

一些例子:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200808220628.png)


### 值传递与引用传递

**基础类型都是值传递，引用类型都是指针/地址传递或者说引用传递。其实都是值传递，但是引用引用传递传递的是地址。所以，值传递的后果就是产生副本，不会对原来的值造成影响，而指针传递则会直接修改原来地址的值。**

**函数的参数如果是memory，那么调用函数一定是指传递，修改不会对原值产生影响；如果是storage，那么一定是指针传递，会对原来的值产生影响。**

- memory->memory 是引用copy一份，指向同一块区域。

  例子：

  ```js
   function g(uint[] memoryArray) public returns(uint){
        uint[] memory u = memoryArray;
        memoryArray[1] = 99;
        return u[1];
    }
  ```
  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809201100.png)

- memory->storage 是将memory的值copy一份存到storage中。

- storage->storage 是引用copy一份，然后都指向同一个storage值。

  - 状态变量storge -> 局部变量storage ： 引用拷贝，同一块区域

  - 局部变量storage -> 状态变量storage ： 内容拷贝，不同的区域

  - **相同存储类型的情况下只要是局部变量 -> 状态变量都是新开区域，将所有的值copy一份即二者无关。**

- storage->memory是值copy一份到memory

### solidity函数申明和类型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809212954.png)

### 函数可见性

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809214944.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809214801.png)

### 函数状态可变性

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809215653.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809215953.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809220109.png)

### 函数修饰器

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809220456.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809221350.png)

### 回退函数

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809222143.png)

### 事件

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809222815.png)

### 异常处理

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809223305.png)

### 单位

!(http://xwjpics.gumptlu.work/qiniu_picGo/20200809223447.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200809223612.png)