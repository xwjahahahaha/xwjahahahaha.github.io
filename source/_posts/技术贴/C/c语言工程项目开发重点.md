---
title: c语言工程项目开发重点
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-20 19:46:18
---

[TOC]



> 参考内容：
>
> * https://blog.csdn.net/guotianqing/article/details/104224439

<!-- more -->

# 1. 环境

## C程序识别头文件、库文件、动态库的顺序是什么？

### 头文件

头文件用于编译

1. 先搜索当前目录

2. 然后搜索`gcc -I`指定的目录

3. 再搜索`gcc`的环境变量`C_INCLUDE_PATH`（`C++`程序使用的是`CPLUS_INCLUDE_PATH`）

   所以可以通过将库的目录添加到此环境变量中实现查找，但是一般会写入到`CMakeLists`或者`Makefile`中

4. 最后搜索`gcc`的内定目录：

   ```shell
   /usr/include
   /usr/local/include
   /usr/lib/gcc/x86_64-linux-gnu/7/include  # gcc程序的库文件地址，各个用户的系统上可能不一样
   ```

   此内定目录与`g++`的`--prefix`有关, 具体可以通过`gcc -v` 查看，具体目录可以通过`echo | g++ -v -x c++ -E -`查看

   ![image-20220520203420108](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220520203420108.png)

当各个目录存在相同的文件时，按顺序先识别到哪个用哪个

> 注意：
>
> * `#include <>`形式的引用不会查找当前目录(而`#include ""`会)

### 库文件

库文件用于链接，是编译通过之后的下一步，链接时库文件的查找顺序如下：

1. 编译时指定的库文件目录（由`gcc -L`参数指定）
2. 环境变量`LIBRARY_PATH`指定的目录
3. 系统默认目录：`/lib; /usr/lib; /usr/local/lib`

一般用户安装的库会安装在`/usr/local/lib`，系统自带的库位于`/lib; /usr/lib`，用户自己编译的库可能就要使用`-L`参数指定了。

### 动态库

动态库时运行时加载的，所以还会有一个查找顺序：

1. 编译时指定的动态库搜索路径（通过gcc 的参数`-Wl,-rpath,`指定。当指定多个动态库搜索路径时，路径之间用冒号`:`分隔）
2. 环境变量`LD_LIBRARY_PATH`指定的动态库搜索路径（路径之间用冒号`:`分隔）
3. 配置文件`/etc/ld.so.conf`中指定的动态库搜索路径
4. 默认的动态库搜索路径`/lib:/usr/lib`

> 注意：
>
> * 库文件的查找是不会查找当前目录的（不论是静态链接库还是动态库），即时在同一个目录内也要指定
> * 

## `vscode`配置识别库文件
