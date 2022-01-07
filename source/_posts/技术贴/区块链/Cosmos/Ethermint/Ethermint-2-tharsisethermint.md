---
title: Ethermint-2-tharsis/ethermint
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-10-04 15:51:02
---

> **记录使用tharsis/ethermint的一些操作记录与错误总结**
>
> **注意:**目前ethermint转移到了新的项目地址：https://github.com/tharsis/ethermint.git
>
> 老的项目地址是：https://github.com/cosmos/ethermint.git，官网文档地址：https://docs.ethermint.zone
>
> 需要注意的是**官网教程对应老版本（cosmos）的ethermint**, 会有所冲突。
>
> **本文使用的是新版本的ethermint，另一篇是老版本的ethermint**，可以看这篇 [Ethermint-1-cosmos/ethermint](https://blog.csdn.net/weixin_43988498/article/details/120613591)
>
> 建议先按官网教程熟悉老版本！

<!-- more -->

# 一、tharsis/ethermint的一些改动

1. `ethermintcli`与`ethermintd`两个命令行程序在新版本合并为了一个命令行`ethermintd`,所以之前使用的命令都在其中使用。
2. `init.sh`程序自动会启动rest-rpc服务，不要额外启动jrpc了，默认的jrpc端口是9000
3. 新的节点连接时会快速同步区块高度
4. 新的日志输出样式



![4z3vWU](http://xwjpics.gumptlu.work/qinniu_uPic/4z3vWU-20211005142705367.png)



# 二、遇到的问题与Bug

## 2.1 Remix上无法部署合约

问题描述：

https://github.com/tharsis/ethermint/issues/79

```shell
ERR failed to broadcast tx error="couldn't retrieve sender address ('') from the ethereum transaction\n --- at github.com/tharsis/ethermint/app/ante/eth.go:82 (EthSigVerificationDecorator.AnteHandle) ---\nCaused by: invalid chain id for signer: tx intended signer does not match the given signer" client=json-rpc t=2021-10-05T13:17:16+0800 lvl=warn msg="Served eth_sendRawTransaction" conn=192.168.31.214:59126 reqid=9136769938060 t=3.430958ms err="couldn't retrieve sender address ('') from the ethereum transaction\n --- at github.com/tharsis/ethermint/app/ante/eth.go:82 (EthSigVerificationDecorator.AnteHandle) ---\nCaused by: invalid chain id for signer: tx intended signer does not match the given signer"
```

原因：MetaMask上的设置chain ID与启动的rpc不匹配，导致签名失败。

解决：修改MetaMask上的设置与自己设置的chain ID匹配

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211005142923438.png" alt="image-20211005142923438" style="zoom:67%;" />



## 2.2 mykey.info: key not found

问题描述：

![image-20211005150526046](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211005150526046.png)

原因与解决：

在使用`ethermintd keys`命令的时候如果添加账户使用的是`--keyring-backend=test`, 那么操作`unsafe-export-eth-key`也需要加上这个flag

![image-20211005150551486](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211005150551486.png)

## 2.3 生成交易时MetaMask报错invalid nonce

问题描述：

https://github.com/tharsis/ethermint/issues/317

```shell
creation of FSCoin errored: Error: [ethjs-query] while formatting outputs from RPC '{"value":{"code":-32603,"data":{"code":-32000,"message":" --- at github.com/tharsis/ethermint/app/ante/eth.go:218 (EthNonceVerificationDecorator.AnteHandle) ---\nCaused by: invalid nonce; got 2, expected 1: invalid sequence"}}}'
```

## 2.4 生成交易时MetaMask报错Key not found

问题描述:

```shell
creation of FSCoin errored: Error: [ethjs-query] while formatting outputs from RPC '{"value":{"code":-32603,"data":{"code":-32000,"message":"rpc error: code = NotFound desc = rpc error: code = NotFound desc = account ethm10y032nunen0z6eu0msmleqk98ypjk5efdrxw9c not found: key not found"}}}'
```

当添加账户使用`--recover`时，在MetaMask上发起交易会找不到当前账户，但是如果是新倒入的账户（私钥）而不是`–recover`，则MetaMask可以正常使用

原因：未知，可能是MetaMask的问题

解决：不使用`–recover`，而是在第一次添加账户后，每次直接启动即可，不用`init.sh`：

`ethermintd start --pruning=nothing —-trace --log_level info --minimum-gas-prices=0.0001aphoton --json-rpc.api eth,txpool,personal,net,debug,web3,miner`

# 三、自定义修改

## 3.1 修改ethermint的代币aphoton为自定义代币

在`init.sh`中修改`aphoton`为自定义的名称

Vim: `%s/aphoton/[自定义名称]/g`

```shell
# Change parameter token denominations to fscoin
cat $HOME/.ethermintd/config/genesis.json | jq '.app_state["staking"]["params"]["bond_denom"]="aphoton"' > $HOME/.ethermintd/config/tmp_genesis.json && mv $HOME/.ethermintd/config/tmp_genesis.json $HOME/.ethermintd/config/genesis.json
cat $HOME/.ethermintd/config/genesis.json | jq '.app_state["crisis"]["constant_fee"]["denom"]="aphoton"' > $HOME/.ethermintd/config/tmp_genesis.json && mv $HOME/.ethermintd/config/tmp_genesis.json $HOME/.ethermintd/config/genesis.json
cat $HOME/.ethermintd/config/genesis.json | jq '.app_state["gov"]["deposit_params"]["min_deposit"][0]["denom"]="aphoton"' > $HOME/.ethermintd/config/tmp_genesis.json && mv $HOME/.ethermintd/config/tmp_genesis.json $HOME/.ethermintd/config/genesis.json
cat $HOME/.ethermintd/config/genesis.json | jq '.app_state["mint"]["params"]["mint_denom"]="aphoton"' > $HOME/.ethermintd/config/tmp_genesis.json && mv $HOME/.ethermintd/config/tmp_genesis.json $HOME/.ethermintd/config/genesis.json
# 添加这一行
cat $HOME/.ethermintd/config/genesis.json | jq '.app_state["evm"]["params"]["evm_denom"]="aphoton"' > $HOME/.ethermintd/config/tmp_genesis.json && mv $HOME/.ethermintd/config/tmp_genesis.json $HOME/.ethermintd/config/genesis.json
```



