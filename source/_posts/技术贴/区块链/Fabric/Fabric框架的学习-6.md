---
title: Fabric框架的学习-6
tags:
  - fabric
categories:
  - technical
  - fabric
toc: true
declare: true
date: 2020-10-16 15:10:46
---

# 十、Fabric核心梳理

## 10.1 fabric网络小结

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201016150737.png)

<!-- more -->

### 10.1.1 客户端

- 连接peer需要用户身份的账号信息, 可以连接到<u>同组的</u>peer节点上（想连接哪个组就连哪个组）
- 客户端发起一笔交易
  - 会发送到参与背书的各个节点上
  - 参加背书的节点进行模拟交易
  - 背书节点将处理结果发送给客户端
  - 如果提案的结果都没有问题, 客户端将交易提交给orderer节点
  - orderer节点将交易打包
  - leader节点将打包数据同步到当前组织
  - 当前组织的提交节点将打包数据写入到区块中

### 10.1.2 Fabric-ca-sever

- 可以通过它<u>动态</u>创建用户
- 网络中可以没有这个角色

### 10.1.3 组织

- peer节点 -> 存储账本
- 用户Users

### 10.1.4 排序节点orderer

- 对交易进行排序
  - 解决双花问题
- 对交易打包
  - configtx.yaml （打包参数设置的配置文件）

### 10.1.5 peer节点

- 背书节点
  - 进行交易的模拟, 将节点返回给客户端
  - 客户端选择的, 客户端指定谁去进行模拟交易谁就是背书节点
- 提交节点
  - 将orderer节点打包的数据, 加入到区块链中
  - <font color='orange'>只要是peer节点, 就具有提交数据的能力</font>
- 主节点
  - 和orderer排序节点直接通信的节点
    - 从orderer节点处获取到打包数据
    - 将数据同步到当前组织的各个节点中
  - 只能有一个
    - 可以自己指定
    - 也可以通过fabric框架选择 -> 推荐
- 锚节点
  - 代表当前组织和其他组织通信的节点
  - 只能有一个

# 十一、Fabric共识机制

> 交易必须按照发生的顺序写入分类帐，尽管它们可能位于网络中不同的参与者组之间。为了实现这一点，必须建立交易的顺序，并且必须建立一种拒绝错误（或恶意）插入分类帐的坏交易的方法。
>
> 在分布式分类帐技术中，共识渐渐已成为单一功能中特定算法的代名词。然而，共识不仅仅是简单地同意交易顺序，而是通过在整个交易流程中的基本作用，从提案和认可到订购，验证和承诺，在Hyperledger Fabric中强调了这种差异化。简而言之，<font color='red'>共识被定义为对包含块的一组交易的正确性的全面验证</font>。
>
> Hyperledger Fabric共识机制，目前包括SOLO，Kafka，以及未来可能要使用的PBFT（实践拜占庭容错）1.4已经有了、SBFT（简化拜占庭容错）

## 2.1 Solo

> <font color='red'>SOLO机制是一个非常容易部署的<u>非生产环境</u>的共识排序节点</font>。它由一个为所有客户服务的单一节点组成，所以不需要“共识”，因为只有一个中央权威机构。相应地没有高可用性或可扩展性。这使得独立开发和测试很理想，但不适合生产环境部署。orderer-solo模式作为单节点通信模式，所有从peer收到的消息都在本节点进行排序与生成数据块。
>
> 客户端通过GRPC发起通信，与Orderer连接成功之后，便可以向Orderer发送消息。Orderer通过Recv接口接收Peer发送过来的消息，Orderer将接收到的消息生成数据块，并将数据块存入ledger，peer通过deliver接口从orderer中的ledger获取数据块。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201017101435.png)

## 2.2 Kafka

> <font color='red'>Kafka是一个分布式消息系统</font>，由LinkedIn使用scala编写，用作LinkedIn的活动流（Activitystream)和运营数据处理管道（Pipeline）的基础。具有高水平扩展和高吞吐量。
>
> 在Fabric网络中，数据是由Peer节点提交到Orderer排序服务，而Orderer相对于Kafka来说相当于上游模块，且Orderer还兼具提供了对数据进行排序及生成符合配置规范及要求的区块。<u>而使用上游模块的数据计算、统计、分析，这个时候就可以使用类似于Kafka这样的分布式消息系统来协助业务流程。</u>
>
> 有人说Kafka是一种共识模式，也就是说平等信任，所有的HyperLedger Fabric网络加盟方都是可信方，因为消息总是均匀地分布在各处。但具体生产使用的时候是依赖于背书来做到确权，相对而言，Kafka应该只能是一种启动Fabric网络的模式或类型。
>
> Zookeeper一种在分布式系统中被广泛用来作为分布式状态管理、分布式协调管理、分布式配置管理和分布式锁服务的集群。Kafka增加和减少服务器都会在Zookeeper节点上触发相应的事件，Kafka系统会捕获这些事件，进行新一轮的<u>负载均衡</u>，客户端也会捕获这些事件来进行新一轮的处理。
>
> Orderer排序服务是Fabric网络事务流中的最重要的环节，也是所有请求的点，它并不会立刻对请求给予回馈，一是因为生成区块的条件所限，二是因为依托下游集群的消息处理需要等待结果。c  

kafka集群的关系：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201017103306.png)

kafka是一个分布式消息系统，在大数据Hadoop等框架中都有使用。

## 2.3 多机多节点配置

## 2.4 多机之间的通信工具SCP

Linux下的scp命令可以实现多机之间文件的拷贝

对于构建fabric网络来说，在一台peer节点主机上创建了通道后，生成对应的block文件，其他节点的加入就需要这个文件，所以需要使用到scp

```shell
# 命令的格式
usage: scp [-346BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
           [-l limit] [-o ssh_option] [-P port] [-S program]
           [[user@]host1:]file1 ... [[user@]host2:]file2

# 使用
scp [-r(拷贝目录)] 要拷贝文件路径 目标主机用户名@目标主机ip地址:目标主机被拷贝的路径
#example
scp node-v8.11.4-linux-x64.tar.xz root@172.17.0.2:/root
```

输入密码则可以开始拷贝。成功如下:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201017142254.png)

**多次输入正确的密码都显示[<font color='red'>Permission denied,please try again</font>]的解决办法**

http://www.cnblogs.com/xuliangxing/p/7428737.html

当scp的时候我们发现错误，被拒绝，是因为ssh的权限问题，需要修改权限，进入**要访问的主机（目标主机）**到/etc/ssh文件夹下，用root用户修改文件sshd_config

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201017142144.png)

将PermitRootLogin no / without-password 改为 PermitRootLogin yes，然后重启sshd服务。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201017142151.png)

**重启命令：**sudo service ssh restart