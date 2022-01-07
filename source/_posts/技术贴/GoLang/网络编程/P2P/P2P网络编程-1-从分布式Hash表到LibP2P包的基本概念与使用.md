---
title: P2P网络编程-1-从分布式Hash表到LibP2P包的基本概念与使用
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2021-07-17 18:32:16
---

> 学习资料来源：
>
> https://zhuanlan.zhihu.com/p/49062384
>
> https://colobu.com/2018/03/26/distributed-hash-table/
>
> libp2p官方文档

本篇文章涵盖内容：

* 分布式Hash表、Kademlia算法
* Libp2p的基本、核心概念
* Libp2p的一个简单使用流程（手动编写ping协议实现）
* 基本的主机配置

# 一、基本概念

万事万物都要从其概念学起...

## 1.1 前置概念

### 分布式Hash表

---

*查表式与计算式*

关于分布式多节点情景下数据如何布局，主要有两种思路：**查表式和计算式**。

所谓查表式，即通过维护**全局统一的映射表**，需要访问数据时，先查询该表定位数据所在节点。
计算式无需维护该映射表，需要访问数据时，通过**一定规则**计算出数据所在位置。

查表式和计算式各有优劣：

查表式需要一个中心服务器维护全局的映射表信息，这可能成为系统的瓶颈；
而计算式的主要问题在于存储节点的变更可能带来大量的数据迁移，增加系统复杂度。

<font color='#e54d42'>分布式Hash就是一种典型的**计算式**数据布局算法</font>

---

*官方概念：*

> **分布式哈希表（distributed hash table，缩写DHT）**是分布式计算系统中的一类，用来将一个键（key）的集合分散到所有在分布式系统中的节点。这里的节点类似哈希表中的存储位置。分布式哈希表通常是为了拥有大量节点的系统，而且系统的节点常常会加入或离开。
>
> 其主要的动机是为了开发**点对点**系统。

---

*算法过程：*

DHT算法大致可以描述为以下两个子算法：

* 建立节点的位置算法
* 确定查询节点位置算法
* 确定存储对象的位置算法 => 提供服务

下面以Dynamo使用的结构举例：

![ZTQGjf](http://xwjpics.gumptlu.work/qinniu_uPic/ZTQGjf.png)

> **注意：这里的Ring结构只是举个例子，并不是所有的DHT都使用此结构**

所有的分布式节点都遵循这样的一套规则进行数据的存储与查询，整体向外提供高容灾性的服务。

---

*特性：*

分布式Hash表本质上强调以下特性：

- **离散性**：构成系统的节点并没有任何中央式的协调机制
- **伸缩性**：即使有成千上万个节点，系统仍然应该十分有效率
- **容错性**：即使节点不断地加入、离开或是停止工作，系统仍然必须达到一定的可靠度

对于分布式Hash表有一个基本的了解即可，这里不再继续展开。。。

---

### Kademlia算法

*维基官方定义如下：*

> <font color='#e54d42'>**Kademlia**（简称kad）是一种通过 DHT 的协议算法</font>，它是由Petar和David在2002年为P2P网络而设计的。Kademlia规定了网络的结构，也规定了通过节点查询进行信息交换的方式。
> Kademlia网络节点之间使用**UDP**进行通讯。参与通讯的所有节点形成一张虚拟网（或者叫做覆盖网）。这些节点通过一组数字（或称为节点ID）来进行身份标识。节点ID不仅可以用来做身份标识，还可以用来进行值定位（值通常是文件的散列或者关键词）。
>
> 当我们在网络中搜索某些值（即通常搜索存储文件散列或关键词的节点）的时候，Kademlia算法需要知道与这些值相关的键，然后逐步在网络中开始搜索。每一步都会找到一些节点，这些节点的ID与键更为接近，如果有节点直接返回搜索的值或者再也无法找到与键更为接近的节点ID的时候搜索便会停止。

---

*节点距离计算规则：*

距离是指节点之间的**跳数**

精妙的与**异或算法**结合：

- `(A ⊕ B) == (B ⊕ A)`: XOR 符合“交换律”，具备对称性。A和B的距离从哪一个节点计算都是相同的。
- `(A ⊕ A) == 0`: 反身性，自己和自己的距离为零。
- `(A ⊕ B) > 0`: 两个不同的 key 之间的距离必大于零。
- `(A ⊕ B) + (B ⊕ C) >= (A ⊕ C)`: 三角不等式, A经过B到C的距离总是大于A直接到C的距离。

---

*映射规则 / 节点位置计算方法：*

Kad使用160位的Hash算法，完整的Key有160二进制位，所以最多可以容纳$2^{160}$个节点

Kad将所有的Key都映射到一个二叉树，每一个**Key都是二叉树的叶子**

> <font color='#e54d42'>**一个Key对应于一个节点**，当然也可能会有虚拟节点</font>
>
> <font color='#e54d42'>**注意:**上方距离DHT的时候使用的是Ring结构，这里Kad算法使用的二叉树而不是Ring</font>

将Key看作160位的二进制，二叉树的第n层就对应了第n位，可以按照左0右1的规则如下分割下去：

![k2P4PD](http://xwjpics.gumptlu.work/qinniu_uPic/k2P4PD.png)



分割完之后**叶子节点到根节点的路径**就对应于一个完整的Key，代表着该系统分布式的一个节点

---

*拆子树与K桶：*

* 拆子树

  * 每个节点按照**自己的角度**去拆分子树，从根节点开始看，如果其右子树不包含自己那么就将其左子树拆分出来，依此类推往下拆分直到只有自己

  * 因为Kad的Key一共是160位，其二叉树就是160层，那么对于一个节点来说最多拆分出来的子树有160个（每层拆一个）（当然实际情况节点数远小于$2^{160}$个，节点拆分子树的个数也不会是160个）

  * 对于一个节点的n个拆分子树，如果都知道里面的一个节点，那么就可以利用这n个节点进行**递归路由**从而找到整个二叉树的所有节点(也即到达每一个节点)

* K桶`（K-bucket）`
  * 仅知道子树中的一个节点不够健壮(考虑到意外宕机等), 多个才安全
  * K就是指知道K个节点来保证健壮性(考虑实际情况K只是一个**上限**)，其K为一个系统级别的常量
  * K桶的概念其实就是**路由表**，节点需要知道n个子树就需要维护n个路由表/K桶，每个路由表/桶的上限大小是K
  * 选择K个节点：选择**长时间在线**的节点，如果当前K桶满了就将新的节点放入**缓存**，待有节点断连就将新节点更换之

---

*Kad协议消息类型：*

共四种：

- **PING**消息: 用来测试节点<u>是否仍然在线</u>。
- **STORE**消息: 在某个节点中<u>存储</u>一个键值对。
- **FIND_NODE**消息: 消息请求的接收者将返回自己桶中<u>离请求键值最**近**</u>的K个节点。
- **FIND_VALUE**消息: 与FIND_NODE一样，不过当请求的接收者存有请求者所请求的键的时候，它将返回相应键的值。

每一个RPC消息中都包含一个发起者加入的**随机值**，这一点确保响应消息在收到的时候能够与前面发送的请求消息**匹配**

---

*定位最近节点:*

查询可以异步也可同步

1. 查询发起者节点从自己的K桶中筛选出离目标Id最近的一些节点，并发起异步查询请求
2. 被查询节点收到请求后，从自己的K桶中找出自己知道的与查询ID最近的若干个节点返回给发起者
3. 发起者收到后更新自己的结果列表，再次筛选出离目标最近的若干未请求过的节点重复步骤一
4. 直到找不到最近的未请求过的节点为止
5. 查询过程中未响应的节点会被立即排除；查询者需要最终获得的K个节点都是活动的

---

*定位资源：*

定位资源时，如果一个节点存有相应的资源的值的时候，它就返回该资源，搜索便结束了，除了该点以外，**定位资源与定位离键最近的节点的过程相似。**

考虑到节点未必都在线的情况，**资源的值被存在多个节点上（节点中的K个）**，并且，为了提供冗余，还有可能在更多的节点上储存值。储存值的节点将定期搜索网络中与储存值所对应的键接近的K个节点并且把值复制到这些节点上，这些节点可作为那些下线的节点的补充。另外还有缓存技术。

---

*加入网络：*

1. 新节点A必须知道某个引导节点B，并把它加入到自己相应的K-桶中
2. 生成一个随机的节点ID,直到离开网络，该节点会一直使用该ID号
3. 向B（A目前知道的唯一节点）发起一个查询请求（FIND_NODE），请求的ID是自己（就是查询自己）
4. B收到该请求之后，会先把A的ID加入自己的相应的 K-桶中。并且根据 FIND_NODE 请求的约定，B会找到K个最接近 A 的节点，并返回给 A
5. A收到这K个节点的ID之后，把他们加入自己的 K-桶
6. 然后A会继续向刚刚拿到的这批节点(还未发送过请求的节点)发送查询请求（协议类型 FIND_NODE），如此往复，直至A建立了足够详细的路由表。
7. 这种<font color='#e54d42'>“**自我定位**”</font>将使得Kad的其他节点（收到请求的节点）能够使用A的ID填充他们的K-桶，同时也能够使用那些查询过程的中间节点来填充A的K-桶。这已过程既让A获得了详细的路由表，也让其它节点知道了A节点的加入

## 1.2 LibP2P基本概念

Libp2p是Protocol Labs旗下的五个明星项目之一，五个项目彼此独立而又相互联系，旨在建立一个更安全、高效、开放的网络。

![QBObqG](http://xwjpics.gumptlu.work/qinniu_uPic/QBObqG.png)

Libp2p是一个**<font color='#e54d42'>模块化</font>的网络栈**，通过将各种传输协议和P2P协议结合在一起，开发人员能够构建大型、健壮的P2P网络。

<!-- more -->

**Libp2p是IPFS的网络层,主要负责发现节点、连接节点、发现数据、传输数据**

![H2RaTm](http://xwjpics.gumptlu.work/qinniu_uPic/H2RaTm.png)

> Libp2p官方网站: https://libp2p.io
>
> github(go版本): https://github.com/libp2p/go-libp2p

<font color='#e54d42'>**LibP2P实现了基于Kademlia-base的分布式Hash表**</font>

# 二、LibP2P核心概念

此章介绍LibP2P中的一些核心概念，此部分大多来自官网翻译，知道大意即可

## 2.1 Transport 传输

网络中不同电脑之间的数据传播很可能使用的就是TCP/IP协议, 当然要求快速但不可靠的服务可能会用到UDP

虽然TCP和UDP(连同IP)是目前最常用的协议，但它们绝不是唯一的选择。

备选方案存在于较低的级别(例如发送原始以太网数据包或蓝牙帧)和较高的级别(例如QUIC，它是在UDP之上分层的)。

在libp2p中，我们将这些围绕传输移动比特的基本协议称为基础协议，libp2p的核心需求之一是**与传输无关**。这意味着**使用什么传输协议取决于开发人员**，事实上**一个应用程序可以同时支持许多不同的传输**。

传输具有两个核心的操作实现：**监听**和**拨号**

libp2p 实现中的**每个传输都将共享相同的编程接口。**

监听和拨号都需要知道如何联系他们，libp2p 使用一种称为“`multiaddr`多地址”的约定或对许多不同的寻址方案进行编码。

例子：`/ip4/7.7.7.7/tcp/6543` 其表示`7.7.7.7`属于IPv4协议, 6543属于tcp

包含`PeerId`的例子：`/ip4/1.2.3.4/tcp/4321/p2p/QmcEPrat8ShnCph8WjkREzt5CPXF2RwhYxYBALDcLC1iV6`

其添加了公钥的Hash唯一标识远程对等方

<font color='#39b54a'>当**对等路由**启动后，只需要使用PeerId即可拨打给对等方，而无需事先知道他们的传输地址</font>

## 2.2 NAT穿透

NAT(网络地址转换)简单来说就是一个局域网内的主机共享一个对外的公网IP，当需要数据传出时将公共IP替换成内部IP，当数据从另一端返回时，路由器将转换回内部IP

NAT的转出一般是透明的，转入则需要一些配置，指定到一些特定的端口中, 通常是通过将一个或多个 TCP 或 UDP 端口从公共 IP 映射到内部端口。

**自动路由器配置**：许多路由器支持端口转发的自动配置协议，最常见的是 [UPnP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play) 或者 [nat-pmp。](https://en.wikipedia.org/wiki/NAT_Port_Mapping_Protocol)如果你的路由器支持这些协议之一，那么libp2p将会自动配置端口映射，无需其他操作。

当使用 IP 支持的传输时，libp2p 将尝试通过**使用相同的端口进行拨号和侦听**，使用名为的套接字选项 [`SO_REUSEPORT`](https://lwn.net/Articles/542629/).

libp2p 的核心协议之一是[identify protocol (识别协议)](https://github.com/libp2p/specs/pull/97)，这允许一个对等点向另一个点询问一些识别信息。当发送他们的[公钥](https://docs.libp2p.io/concepts/peer-id/) 和一些其他有用的信息，被识别的对等方包括它为提出问题的对等方观察到的地址集。

虽然 [识别协议](https://github.com/libp2p/specs/pull/97) 上面描述的让对等点相互通知他们观察到的网络地址，并非所有网络都允许在用于拨出的同一端口上进行传入连接。

其他对等点可以帮助我们观察我们的情况，这一次是通过尝试通过我们观察到的地址拨打我们的电话。如果这成功了，我们就可以**依靠其他节点也可以拨打我们的电话**，然后我们就可以开始宣传我们的收听地址了。

一个名为 AutoNAT 的 libp2p 协议允许对等方从提供 AutoNAT 服务的对等方请求回拨。

在某些情况下，对等方将无法以可公开访问的方式遍历其 NAT。

libp2p 提供了一个 [电路中继协议](https://docs.libp2p.io/concepts/circuit-relay/) 这允许**对等点通过有用的中间对等点进行间接通信。**

## 2.3 电路继电器(中继节点)

电路继电器是一种 [传输协议](https://docs.libp2p.io/concepts/transport/) 通过第三方“中继”对等体在两个对等体之间路由流量。

中继连接是**端到端加密**的，这意味着充当中继的对等方无法读取或篡改流经连接的任何流量。

中继协议的一个重要方面是它不是“透明的”。换句话说，源和目的地都知道正在中继流量。这很有用，因为目的地可以看到用于打开连接的中继地址，并且可以潜在地使用它来构造返回源的路径。它也不是匿名的——所有参与者都使用他们的对等 ID 进行识别，包括中继节点。

假设我有一个 peer 的 peer id `QmAlice`。我想把我的地址给我的朋友`QmBob`，但我(Alice)在一个不允许任何人直接给我拨号的 NAT 后面。

`p2p-circuit`我可以构造的最基本的地址如下所示：

`/p2p-circuit/p2p/QmAlice`

上面的地址很有趣，因为**它不包含任何 [运输](https://docs.libp2p.io/concepts/transport/)我们要联系的对等点 ( `QmAlice`) 或将传送流量的中继对等点的地址**。如果没有这些信息，对等方拨打我的唯一机会就是**发现中继并希望他们与我建立联系**。

更好的地址应该是`/p2p/QmRelay/p2p-circuit/p2p/QmAlice`. 这包括**特定中继对等体**的身份`QmRelay`。如果对等方已经知道如何打开与 连接`QmRelay`，他们将能够联系到我们。

> <font color='#39b54a'>指定特定的中继</font>

**更好的是在地址中包含中继对等方的传输地址**。假设我已经使用 peer id 建立了到特定中继的连接`QmRelay`。他们通过识别协议告诉我，他们正在侦听`7.7.7.7`IPv4 地址`55555`端口上的TCP 连接。我可以构建一个地址，描述通过该传输的特定中继到达我的路径：

`/ip4/7.7.7.7/tcp/55555/p2p/QmRelay/p2p-circuit/p2p/QmAlice`

在`/p2p-circuit/`**之前的所有内容都是中继对等体的地址**，其中包括传输地址和它们的peer id `QmRelay`。在`/p2p-circuit/`之后的是我在线路另一端的peer的peer ID，即`QmAlice`.

通过将完整的中继路径提供给我的朋友`QmBob`，他们能够快速建立中继连接，而无需“四处询问”具有到`QmAlice`.

Autorelay 是一项功能（目前在 go-libp2p 中实现），peer 可以启用它以尝试使用 libp2p 发现中继 peer [内容路由](https://docs.libp2p.io/concepts/content-routing/) 界面。

启用自动中继后，对等方将尝试发现一个或多个公共中继并打开中继连接。如果成功，对等方将使用 libp2p 的[对等路由](https://docs.libp2p.io/concepts/peer-routing/) 系统。

> <font color='#39b54a'>中继节点其实简单来说就是**代理**的作用</font>

## 2.4 协议

当您编写网络应用程序时，到处都有协议，而 libp2p 中的协议尤其丰富。

本文所关注的协议类型是使用 libp2p 本身构建的协议，使用核心 libp2p 抽象，如 [运输](https://docs.libp2p.io/concepts/transport), [对等身份](https://docs.libp2p.io/concepts/peer-id/), [寻址](https://docs.libp2p.io/concepts/addressing/)， 等等。

在本文中，我们将这种使用 libp2p**构建的协议**称为**libp2p 协议**，但您也可能将它们称为“线路协议”或“应用程序协议”。

libp2p 协议具有以下关键特性：

### 协议ID

按照惯例，协议 ID 具有类似路径的结构，版本号作为最终组成部分：

`/my-app/amazing-protocol/1.0.1`

### 核心libp2p协议

除了您在开发 libp2p 应用程序时编写的协议之外，libp2p 本身还定义了几个用于核心功能的基础协议。

![mYLLgf](http://xwjpics.gumptlu.work/qinniu_uPic/mYLLgf.png)

#### ping

| **Protocol id**    | spec |                                                              |                                                | implementations                                              |
| :----------------- | :--- | :----------------------------------------------------------- | :--------------------------------------------- | :----------------------------------------------------------- |
| `/ipfs/ping/1.0.0` | N/A  | [go](https://github.com/libp2p/go-libp2p/tree/master/p2p/protocol/ping) | [js](https://github.com/libp2p/js-libp2p-ping) | [rust](https://github.com/libp2p/rust-libp2p/blob/master/protocols/ping/src/lib.rs) |

ping 协议是一个简单的活动检查，点可以用来快速查看另一个点是否在线。

#### Identify

| **Protocol id**  | spec                                                         |                                                              |                                                    | implementations                                              |
| :--------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :------------------------------------------------- | :----------------------------------------------------------- |
| `/ipfs/id/1.0.0` | [identify spec](https://github.com/libp2p/specs/pull/97/files) | [go](https://github.com/libp2p/go-libp2p/tree/master/p2p/protocol/identify) | [js](https://github.com/libp2p/js-libp2p-identify) | [rust](https://github.com/libp2p/rust-libp2p/tree/master/protocols/identify/src) |

该`identify`协议允许对等方交换有关彼此的信息，尤其是他们的**公钥和已知网络地址**。

#### identify/push

| **Protocol id**       | spec & implementations                                       |
| :-------------------- | :----------------------------------------------------------- |
| `/ipfs/id/push/1.0.0` | same as [identify above](https://docs.libp2p.io/concepts/protocols/#identify) |

与`identify`稍有变化，`identify/push`协议发送相同的`Identify`消息，但它是主动发送的，而不是响应请求。

#### secio

| **Protocol id** | spec                                                   |                                                 |                                                 | implementations                                              |
| :-------------- | :----------------------------------------------------- | :---------------------------------------------- | :---------------------------------------------- | :----------------------------------------------------------- |
| `/secio/1.0.0`  | [secio spec](https://github.com/libp2p/specs/pull/106) | [go](https://github.com/libp2p/go-libp2p-secio) | [js](https://github.com/libp2p/js-libp2p-secio) | [rust](https://github.com/libp2p/rust-libp2p/tree/master/protocols/secio) |

`secio`（安全输入/输出的缩写）是一种类似于 TLS 1.2 的加密通信协议，但没有证书颁发机构的要求。因为每个 libp2p peer 都有一个PeerID这是从他们的公钥派生的，通过**使用他们的公钥来验证签名的消息**，**可以在不需要证书颁发机构的情况下验证对等方的身份。**

> 虽然 secio 是目前 libp2p 使用的默认加密协议，但将 TLS 1.3 集成到 libp2p 的工作正在进行中，预计一旦完成，它将成为默认加密协议。

#### kad-dht

| **Protocol id**   | spec                                                     |                                                   |                                                   | implementations                                              |
| :---------------- | :------------------------------------------------------- | :------------------------------------------------ | :------------------------------------------------ | :----------------------------------------------------------- |
| `/ipfs/kad/1.0.0` | [kad-dht spec](https://github.com/libp2p/specs/pull/108) | [go](https://github.com/libp2p/go-libp2p-kad-dht) | [js](https://github.com/libp2p/js-libp2p-kad-dht) | [rust](https://github.com/libp2p/rust-libp2p/tree/master/protocols/kad) |

`kad-dht` 是一个 [分布式哈希表DHT](https://en.wikipedia.org/wiki/Distributed_hash_table) 基于 [Kademlia](https://en.wikipedia.org/wiki/Kademlia) 路由算法，**有一些修改**。

libp2p 使用 DHT 作为其基础 [对等路由](https://docs.libp2p.io/concepts/peer-routing/) 和 [内容路由](https://docs.libp2p.io/concepts/content-routing/) 功能。

#### Circuit Relay

| **Protocol id**               | spec                                                         |                                                   | implementations                                   |
| :---------------------------- | :----------------------------------------------------------- | :------------------------------------------------ | :------------------------------------------------ |
| `/libp2p/circuit/relay/0.1.0` | [circuit relay spec](https://github.com/libp2p/specs/tree/master/relay) | [go](https://github.com/libp2p/go-libp2p-circuit) | [js](https://github.com/libp2p/js-libp2p-circuit) |

如中所述 [电路继电器篇](https://docs.libp2p.io/concepts/circuit-relay/)，libp2p 提供了一个协议，**当两个对等点无法直接相互连接时，通过中继对等点传输流量。**

## 2.5 PeerID

对等身份（通常写作`PeerId`）是对整个对等网络中特定对等的唯一引用。

除了充当每个对等点的唯一标识符外，PeerId 还是**对等点与其公共加密密钥之间的可验证链接**。

每个 libp2p peer 控制一个私钥，它对所有其他 peer 保密。每个私钥都有一个对应的公钥，与其他对等方共享。

公钥和私钥（或“密钥对”）一起允许对等方建立 [安全通讯](https://docs.libp2p.io/concepts/secure-comms/) 通道相互。

从概念上讲，PeerId 是一个 [加密散列](https://en.wikipedia.org/wiki/Cryptographic_hash_function)对等方的公钥。当对等方建立安全通道时，散列可用于验证用于保护通道的公钥是否与用于识别对等方的公钥相同。

PeerIds 使用 [多重散列](https://docs.libp2p.io/reference/glossary/#multihash) 格式，**它向散列本身添加一个小标头，用于标识用于生成它的散列算法**。

PeerId被编码成Base-58然后表示为字符串（这与比特币类似）:	`QmYyQSo1c1Ym7orWxLYvCrM2EmxFTANf8wXmmE7DWjhx5N`

> <font color='#e54d42'>PeerID生成的过程：创建私钥 => 导出公钥 => hash函数 => Base58编码</font>

## 2.6 内容路由

针对查找的**内容**进行路由

## 2.7 节点路由

针对**节点**发现进行路由

## 2.8 发布与订阅

节点可以发布topic，在网络中不断传递，当有一些人订阅之后就会收到消息 （类似mqtt）

##  2.9 数据流的多路复用

A节点与B节点之间建立流通道，不断的传输数据流，但是A、B之间的应用可能是多个，那么在此数据流通道Libp2p就实现了数据流的多路复用

使用**协议号**来标识不同应用的流

# 三、快速开始(Go)

## 2.1 前置要求

* go语言最低要求：>= 1.11

* 使用Go module模式编写代码

```shell
# 创建空文件夹，名字自定
mkdir libp2p && cd libp2p
go mod init
# 下载包环境
go get -u github.com/libp2p/go-libp2p 
```

## 2.2 开启libp2p节点

编写main.go

```go
package main

import (
	"context"
	"fmt"
	"github.com/libp2p/go-libp2p"
)

func main() {
	// 创建一个背景上下文环境
	ctx := context.Background()

	// 开启一个默认配置的libp2p节点
	node, err := libp2p.New(ctx)
	if err != nil {
		panic(err)
	}

	// 输出节点监听地址
	fmt.Println("Listen Address: ", node.Addrs())

	// 停止节点
	if err := node.Close(); err != nil {
		panic(err)
	}
}
```

```shell
# 启动
go run main.go                       
Listen Address:  [/ip4/192.168.3.30/tcp/58048 /ip4/127.0.0.1/tcp/58048 /ip6/::1/tcp/58049]
```

## 2.3 配置节点

新建节点时，添加配置，例如：第二个参数设置监听

```go
node, err := libp2p.New(ctx, libp2p.ListenAddrStrings("/ip4/127.0.0.1/tcp/2000"))
```

重新运行：

```shell
go run main.go
Listen Address:  [/ip4/127.0.0.1/tcp/2000]
```

## 2.4 等待信号

给节点设置等待OS的信号再停止

```go
....

// 等待SIGINT或SIGTERM信号
ch := make(chan os.Signal, 1)
// 当收到ctrl + c时将信号写入通道
signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
<- ch		// 等待阻塞
fmt.Println("Received signal, shutting down...")

// 停止节点
if err := node.Close(); err != nil {
  panic(err)
}
```

```shell
go run main.go
Listen Address:  [/ip4/127.0.0.1/tcp/2000]
^CReceived signal, shutting down...
```

## 2.5 运行ping协议

下面开始通信, 以go-libp2p启动的节点将默认运行自己的ping协议，但让我们禁用它并手动设置它，通过**注册流处理程序**来演示运行协议的过程。

> <font color='#39b54a'>**将原本的ping协议取消，下面手动演示手动编写一个ping协议**</font>

### 2.5.1 设置Steam handler

从`libp2p.New`返回的对象实现了Host接口，我们将使用`SetStreamHandler`方法为我们的ping协议设置一个处理程序。

配置禁止自带的Ping协议，并设置tcp监听的端口为随机（为了在单机上模拟开启多个节点时端口不冲突）

```go
// 开启一个默认配置的libp2p节点
	node, err := libp2p.New(ctx,
		libp2p.ListenAddrStrings("/ip4/127.0.0.1/tcp/0"),				// 0表示随机
		libp2p.Ping(false),
		)
	if err != nil {
		panic(err)
	}

	// 配置我们自己的ping协议
	pingService := &ping.PingService{Host: node}
	node.SetStreamHandler(ping.ID, pingService.PingHandler)
```

### 2.5.2 连接peer节点

运行Ping协议，首先要连接上其他节点

首先替换节点打印日志的内容：

```go
...
// 更改节点打印内容
// 构建打印Info
peerInfo := peer.AddrInfo{
  ID: node.ID(),
  Addrs: node.Addrs(),
}
// 增加日志输出
addrs, err := peer.AddrInfoToP2pAddrs(&peerInfo)
fmt.Println("libp2p node address: ", addrs[0])
...
```

```shell
go run main.go
libp2p node address:  /ip4/127.0.0.1/tcp/63726/p2p/Qma3pW4kM46cELahGAQrN23mM4zBbrjyq2GEQnNfLAThze
```

然后，设置命令行操作(发送ping和监听)，其中使用`github.com/multiformats/go-multiaddr`包来解析收到的multiaddr格式地址

安装依赖包：`go get -u github.com/multiformats/go-multiaddr `

```go
package main

import (
	"context"
	"fmt"
	"github.com/libp2p/go-libp2p"
	"github.com/libp2p/go-libp2p-core/peer"
	"github.com/libp2p/go-libp2p/p2p/protocol/ping"
	multiaddr "github.com/multiformats/go-multiaddr"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	// 创建一个背景上下文环境
	ctx := context.Background()

	// 开启一个默认配置的libp2p节点
	node, err := libp2p.New(ctx,
		libp2p.ListenAddrStrings("/ip4/127.0.0.1/tcp/0"),
		libp2p.Ping(false),
		)
	if err != nil {
		panic(err)
	}

	// 配置我们自己的ping协议
	pingService := &ping.PingService{Host: node}
	node.SetStreamHandler(ping.ID, pingService.PingHandler)

	// 更改节点打印内容
	// 构建打印Info
	peerInfo := peer.AddrInfo{
		ID: node.ID(),
		Addrs: node.Addrs(),
	}
	// 增加日志输出
	addrs, err := peer.AddrInfoToP2pAddrs(&peerInfo)
	fmt.Println("libp2p node address: ", addrs[0])

	// 如果命令行中包含需要连接的节点就连接，并发送5个ping消息，否则监听OS中断消息
	if len(os.Args) > 1 {
		// 解析参数
		addr, err := multiaddr.NewMultiaddr(os.Args[1])
		if err != nil {
			panic(err)
		}
		// 把multiaddr格式转换为AddrInfo格式
		peer, err := peer.AddrInfoFromP2pAddr(addr)
		if err != nil {
			panic(err)
		}
		// 连接节点
		if err := node.Connect(ctx, *peer); err != nil {
			panic(err)
		}
		fmt.Println("sending 5 ping messages to", addr)
		// 发起Ping服务，将结果写入channel
		ch := pingService.Ping(ctx, peer.ID)
		for i:=0; i<5; i++ {
			res := <-ch
			fmt.Println("got ping response!", "RTT:", res.RTT)
		}
	}else {
		// 等待SIGINT或SIGTERM信号
		ch := make(chan os.Signal, 1)
		// 当收到ctrl + c时将信号写入通道
		signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
		<- ch		// 等待阻塞
		fmt.Println("Received signal, shutting down...")
	}
	
	// 停止节点
	if err := node.Close(); err != nil {
		panic(err)
	}
}
```

### 2.5.3 运行

第一个节点：

```shell
go build ./ 
./libp2p
libp2p node address:  /ip4/127.0.0.1/tcp/49904/p2p/Qmaeyg7CEBpe7TE6eCGsmomBLHotU6GergoX2kz3cCmbGr
```

第二个节点(新开一个命令行窗口):

```shell
./libp2p /ip4/127.0.0.1/tcp/49904/p2p/Qmaeyg7CEBpe7TE6eCGsmomBLHotU6GergoX2kz3cCmbGr
sending 5 ping messages to /ip4/127.0.0.1/tcp/49904/p2p/Qmaeyg7CEBpe7TE6eCGsmomBLHotU6GergoX2kz3cCmbGr
got ping response! RTT: 120µs
got ping response! RTT: 120.875µs
got ping response! RTT: 120.167µs
got ping response! RTT: 62.125µs
got ping response! RTT: 135.792µs
```

更多的演示案例：https://github.com/libp2p/go-libp2p/tree/master/examples

> <font color='#39b54a'>这里虽然说是我们手动实现了Ping协议，但是其实应该明白的是服务的创建和处理函数都是调用原来写好的，所以真正完全的实现一个协议还需要学习后续的案例， 后续也会持续的更新。。。</font>

# 四、主机配置

主机配置即当前节点的一般配置，如果不使用默认的配置则一般会用到以下的配置：

所有配置见；[see the different options in the docs](https://godoc.org/github.com/libp2p/go-libp2p).

```go
package main

import (
	"context"
	"fmt"
	"github.com/libp2p/go-libp2p"
	connmgr "github.com/libp2p/go-libp2p-connmgr"
	"github.com/libp2p/go-libp2p-core/crypto"
	"github.com/libp2p/go-libp2p-core/host"
	"github.com/libp2p/go-libp2p-core/peer"
	"github.com/libp2p/go-libp2p-core/routing"
	dht "github.com/libp2p/go-libp2p-kad-dht"
	libp2pquic "github.com/libp2p/go-libp2p-quic-transport"
	secio "github.com/libp2p/go-libp2p-secio"
	libp2ptls "github.com/libp2p/go-libp2p-tls"
	"log"
	"time"
)

func main() {
	run()
}

func run()  {
	// 上下文管理libp2p节点的生命周期。
	ctx, cancel := context.WithCancel(context.Background())
	// 延迟停止node
	defer cancel()

	// 创建一个自己配置的节点

	// 设置自己的公私钥对
	priv, _, err := crypto.GenerateKeyPair(
		crypto.Ed25519,
		-1,
	)
	if err != nil {
		panic(err)
	}

	var idht *dht.IpfsDHT

	node, err := libp2p.New(ctx,
		// 使用自己生成的私钥
		libp2p.Identity(priv),
		// 设置Multiple格式监听地址
		libp2p.ListenAddrStrings(
			"/ip4/0.0.0.0/tcp/9000",	// tcp连接
			"/ip4/0.0.0.0/udp/9000/quic",	// 用于QUIC传输的UDP端点
		),
		// 支持TLS连接
		libp2p.Security(libp2ptls.ID, libp2ptls.New),
		// 支持secio连接
		libp2p.Security(secio.ID, secio.New),
		// 支持QUIC（该功能还在实验中）
		libp2p.Transport(libp2pquic.NewTransport),
		// 支持其他默认传输协议（tcp）
		libp2p.DefaultTransports,
		// 防止peer节点连接过多的其他对等节点，设置连接管理器
		libp2p.ConnectionManager(connmgr.NewConnManager(
			100, 		// 下限
			400,			// 上限
			time.Minute,	// 连接新连接之前的设置的宽限期
		)),
		// 尝试为NAT主机使用uPNP打开端口（自动路由）
		libp2p.NATPortMap(),
		// 为该节点使用DHT去找到其他节点
		libp2p.Routing(func(h host.Host) (routing.PeerRouting, error) {
			idht, err = dht.New(ctx, h)
			return idht, err
		}),
		// 开启中继防止节点处于NAT网络下
		libp2p.EnableRelay(),
		// 节点可以帮助其他在NAT下的节点中继，开启自己的AutoNAT中继服务端 (这个服务本身带有限制，不会产生很大的开销)
		libp2p.EnableNATService(),
	)
	if err != nil {
		panic(err)
	}

	// 连接网络中的引导节点或者其他节点
	// 连接公共引导节点
	for _, addr := range dht.DefaultBootstrapPeers{
		fmt.Println("defaultBootstrapPeer : ", addr)
		// 格式转换为AddrInfo
		p1, _ := peer.AddrInfoFromP2pAddr(addr)
		// 连接	(忽略连接不上的引导节点)
		node.Connect(ctx, *p1)
	}
	log.Printf("My Host ID is %s\n", node.ID())
}
```

