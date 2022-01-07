---
title: Uniswap合约的学习-0-基本概念
tags:
  - solidity
categories:
  - technical
  - solidity
toc: true
declare: true
date: 2021-06-08 11:26:50
---

> 参考资料:
>
> https://uniswap.org/docs/v2/protocol-overview/how-uniswap-works/
>
> https://blog.csdn.net/weixin_39430411/article/details/108665694

# 一、什么是Uniswap

Uniswap is an automated liquidity protocol powered by a **constant product formula** and implemented in a system of non-upgradeable smart contracts on the Ethereum blockchain.

> Uniswap是一种自动流动性协议，由一个恒定乘积算法提供动力，并在以太坊区块链上不可升级的智能合约系统中实现。

关键点:

* 自动的流动性协议
* 基于恒定乘积算法
* 智能合约不可升级
* 运行在以太坊上

<!-- more -->

而一般来讲，我们可以认为Uniswap是一个以太坊上的去中心化的数字货币交易所(DEX)，也可以认为是一种Defi。

## 1.1 恒定乘积算法

