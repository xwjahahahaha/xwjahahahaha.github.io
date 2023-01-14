---
title: runC源码学习-1
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-04 19:40:13
---

[TOC]


<!-- more -->

> 参考：
>
> * runc地址：https://github.com/opencontainers/runc

# 一、源码学习思路

拟定初步的线路是：

* 先实现[libcontainer](https://github.com/opencontainers/runc/tree/main/libcontainer)的重点包依赖，然后在编写上层建筑
* 只记录重点、困难、没看懂的、有意思的、不会的代码编写技巧

