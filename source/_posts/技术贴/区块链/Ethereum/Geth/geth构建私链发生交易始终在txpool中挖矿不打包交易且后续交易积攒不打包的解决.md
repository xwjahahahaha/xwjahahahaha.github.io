---
title: geth构建私链发生交易始终在txpool中挖矿不打包交易且后续交易积攒不打包的解决
tags:
  - geth
categories:
  - technical
toc: true
declare: true
date: 2020-09-03 20:45:41
---

# 错误描述
![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjAzNDM0LnBuZw?x-oss-process=image/format,png)
如图，该账户发起的交易nonce为46的交易往后一直未打包，不管怎么挖矿都会打包不进区块链上。

<!-- more -->

# 解决方案
原因所在：以太坊中交易的nonce是指：

**每个交易的发起方（账户）都有一个nonce记录自己发起交易的数量，可以通过`eth.getTransactionCount(eth.accounts[0])`查看目前的累计数。**

而区块的nonce则是代表着挖矿难度，两者要分清楚。

首先查看自己目前txpool中阻塞的交易nonce：直接看第一个编号或者上面的eth.getTransactionCount都可以，上图为例是46

然后发起一个nonce与之相同且更加高gasPrice交易去覆盖它：

你的gasPrice设置要比之前的大，之前的可以在txpool.content中看到：

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjA0MjU1LnBuZw?x-oss-process=image/format,png)
我就设置稍微大一点，出现交易hash则代表ok

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjA0MzM2LnBuZw?x-oss-process=image/format,png)

设置小了会报错：

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjA0NDExLnBuZw?x-oss-process=image/format,png)
然后再挖矿，再次查询txpool，发现之前所有阻塞的交易都被丢弃了，不会在pending了。




