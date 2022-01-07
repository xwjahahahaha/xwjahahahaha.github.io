---
title: FISCO_BCOS联盟链架构-星型架构的区块链工作证系统-1
tags:
  - bcos
categories:
  - technical
  - bcos
toc: true
declare: true
date: 2020-11-29 21:06:49
---

# 一、部署多群组星型网络

官网文档很详细，**本片主要记录心得与重点**

[多群组部署 — FISCO BCOS v2.7.0 文档 (fisco-bcos-documentation.readthedocs.io)](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/manual/group_use_cases.html#id10)

<!-- more -->

## 1.1 集群规划

| 云服务器IP     | 名称   | 节点  | nodeID前五位 | channel_listen_portonnect | connnect_point |
| -------------- | ------ | ----- | ------------ | ------------------------- | -------------- |
| 47.103.203.133 | Wenjie | node0 | e9553        | 20200                     | 30300          |
|                | Wenjie | node1 | ee4f8        | 20201                     | 30301          |
| 121.196.41.126 | rz     | node2 | 840ee        | 20200                     | 30300          |
|                | rz     | node3 | 9efa1        | 20201                     | 30301          |
| 121.196.149.14 | sy     | node4 | 8a36a        | 20200                     | 30300          |
|                | sy     | node5 | a9fa9        | 20201                     | 30301          |
| 123.56.163.105 | Daniel | node6 | bce46        | 20200                     | 30300          |
|                | Daniel | node7 | c5e43        | 20201                     | 30301          |



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129211656.png)

## 1.2 部署重点



# 二、Java-SDK重点

### 2.1 使用Maven引入SDK

创建maven项目，修改文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
      .....
    <dependencies>
        <dependency>
            <groupId>org.fisco-bcos.java-sdk</groupId>
            <artifactId>java-sdk</artifactId>
            <version>2.7.0</version>
        </dependency>
    </dependencies>
</project>
```

然后按照官网文档走就ok：[快速入门 — FISCO BCOS v2.7.0 文档 (fisco-bcos-documentation.readthedocs.io)](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/sdk/java_sdk/quick_start.html#gradle)

=> 安装证书，编译合约.............

下一步：把编译后的.java文件放入应用中：

* 坑/重点：要把**编译得到的HelloWorld.java放入应用中。**

  注意：<font color='red'>在应用中所放的位置要与我们设定的包名相同。</font>

[官方文档gif](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/_images/prepare_contract.gif)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201130120817.png)

`sol2java.sh`脚本的使用方法

控制台`v2.6+`提供了`sol2java.sh`脚本可将`solidity`转换为`java`代码, `sol2java.sh`使用方法如下：

```shell
$ bash sol2java.sh -h
# Compile Solidity Tool
./sol2java.sh [packageName] [solidityFilePath] [javaCodeOutputDir]
 	 packageName:
 		 the package name of the generated Java class file
 	 solidityFilePath:
 		 (optional) the solidity file path or the directory where solidity files located, default: contracts/solidity
 	 javaCodeOutputDir:
 		 (optional) the directory where the generated Java files located, default: contracts/sdk/java
```

# 三、AMOP(链上信使协议)重点

链上信使协议AMOP（Advanced Messages Onchain Protocol）系统旨在为联盟链提供一个**安全高效的消息信道**，联盟链中的各个机构，只要部署了区块链节点，**无论是共识节点还是观察节点，均可使用AMOP进行通讯，**AMOP有如下优势：

- 实时：AMOP消息不依赖区块链交易和共识，消息在节点间实时传输，延时在毫秒级。
- 可靠：AMOP消息传输时，自动寻找区块链网络中所有可行的链路进行通讯，只要收发双方至少有一个链路可用，消息就保证可达。
- 高效：AMOP消息结构简洁、处理逻辑高效，仅需少量cpu占用，能充分利用网络带宽。
- 安全：AMOP的所有通讯链路使用SSL加密，加密算法可配置,支持身份认证机制。
- 易用：使用AMOP时，无需在SDK做任何额外配置。

**分类：普通话题、私有话题**

## 3.1 私有话题认证流程

假定链外系统1是话题消息发送者（消息发送端），链外系统2是话题订阅者（消息接收端）。私有话题的认证流程如下：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201201103026.png)

- 1：链外系统2连接Node2,宣称订阅T1,Node2将T1加入到话题列表，并将seq加1。同时每5秒同步seq到其他节点。
- 2：Node1收到seq之后，对比本地seq和同步过来的seq，发现不一致，则从Node2获取最新的话题列表，并将话题列表更新到p2p的话题列表，对于还没认证的私有话题，状态置为待认证。Node1遍历列表。针对每一个待认证的私有话题,进行如下操作：
  - 2.1：Node1往Node1推送消息(消息类型0x37)，请求链外系统1发起私有话题认证流程。
  - 2.2：链外系统1接收到消息之后，生成随机数，并使用amop消息(消息类型0x30)将消息发送出去，并监听回包。
  - 2.3：消息经过 链外系统1–>Node1–>Node2–>链外系统2的路由，链外系统2接收到消息后解析出随机数并使用私钥签名随机数。
  - 2.4：签名包(消息类型0x31)经过 链外系统2–>Node2–>Node1->链外系统1的路由，链外系统1接收到签名包之后，解析出签名并使用公钥验证签名。
  - 2.5：链外系统1验证签名后，发送消息(消息类型0x38)，请求节点更新topic状态（认证成功或者认证失败）。
- 3：如果认证成功，链外系统的一条消息到达Node1之后，Node1会将这条消息转发给Node2,Node2会将消息推送给链外系统2。

<font color='green'>理解：其实整个过程就是通过私钥签名，公钥认证在两个节点间形成快速通路</font>