---
title: go的错误处理
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-12-26 14:03:50
---

> 参考：
>
> * https://segmentfault.com/a/1190000023691221

<!-- more -->

# 一、单goroutine的错误处理



# 二、多goroutine的并发错误处理

## 2.1 特点

* 一个子协程发生了panic，其他协程**都会**因为panic而结束，整个程序挂掉
* 一个子协程发生了panic，其他所有协程**无法**捕获recover到，只有自己能捕获到
