---
title: Iptables
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-03-25 13:29:51
---

# 一、iptables是什么？

iptables 是一个简单、灵活、实用的命令行工具，可以用来配置、控制 linux 防火墙。

<!-- more -->

iptables中总共有5张表还有5条链，我们可以在链上加不同的规则。

五张表：`filter表、nat表、mangle表、raw表、security表`

五条链：`prerouting、input、output、forward、postrouting`

## 表（tables）

提供特定的功能，iptables内置了4个表，即：

* filter表（实现包过滤）
* nat表(网络地址转换)
* mangle表(修改数据标记位规则表)
* raw表（跟踪数据表规则表）
* securit(强制访问控制网络规则表)

## 链（chains）

当一个数据包到达一个链时，iptables就会从链中第一条规则开始检查，看该数据包是否满足规则所定义的条件。如果满足，系统就会根据该条规则所定义的方法处理该数据包；否则iptables将继续检查下一条规则，如果该数据包不符合链中任一条规则，iptables就会根据该链预先定义的默认策略来处理数据包。

(这 5 条链实际上就是 netfilter 提供的在数据包流转路径上的 HOOK)

* INPUT(入站数据过滤)
* OUTPUT（出站数据过滤）
* FORWARD（转发数据过滤）
* PREROUTING（路由前过滤）
* POSTROUTING（路由后过滤）

## 关系

![image-20220325135324241](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220325135324241.png)

# 二、转发过程/流量走向

![图片](http://xwjpics.gumptlu.work/qinniu_uPic/640.png)

![image-20220325135423736](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220325135423736.png)
