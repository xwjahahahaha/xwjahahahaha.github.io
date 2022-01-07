---
title: cosmos-2-开发基础
tags:
  - cosmos
categories:
  - block_chain
toc: true
date: 2021-02-28 14:39:07
---

# 二、开发基础

## 2.1. Cosmos SDK

Cosmos  SDK是方便区块链应用开发的基础框架,方便程序员实现基于Tendermint的安全状态机.它将实现多重数据的持久化存储以及交易处理的路由功能.

### 2.1.1 application-specific blockchains 

就是尽量让一个区块链应用使用单独的一条链,不会与其他的应用共享资源,拥有对该条链的完全主权.

```js
                ^  +-------------------------------+  ^
                |  |                               |  |   Built with Cosmos SDK
                |  |  State-machine = Application  |  |
                |  |                               |  v
                |  +-------------------------------+
                |  |                               |  ^
Blockchain node |  |           Consensus           |  |
                |  |                               |  |
                |  +-------------------------------+  |   Tendermint Core
                |  |                               |  |
                |  |           Networking          |  |
                |  |                               |  |
                v  +-------------------------------+  v

```

<!-- more -->

### 2.1.2 BlockChain Architecture

区块链是重复储存的确定性状态机

Given a state S and a block of transactions B, the state machine will return a new state S'.

```js
+--------+                              +--------+
|        |                              |        |
|   S    +----------------------------> |   S'   |
|        |   For each T in B: apply(T)  |        |
+--------+                              +--------+
```

### 2.1.3 Main Components of the Cosmos SDK

SDK是使用Golang执行ABCI的基本样板程序框架

Cosmos SDK 交易处理的一般简化过程:

1. 解码来自Tendermint共识引擎的交易
2. 提取交易信息并且对交易做初步的合法检查
3. 将各个消息发送给适合的模块进行执行
4. 确认状态改变

#### baseapp

作用: 执行ABCI连接基础共识引擎的样板文件

cosmos SDK继承baseapp实现了app.go

baseapp的目标是在**数据存储store和可扩展状态机之间提供安全的接口**，同时尽可能少地定义状态机。

baseapp的工作

- Decode transactions received from the Tendermint consensus engine.

  **解码接受的交易**

- Extract messages from transactions and do basic sanity checks.

  **在交易中提取消息,并且做一些简单的验证**

- Route the message to the appropriate module so that it can be processed. Note that `baseapp` has no knowledge of the specific modules you want to use. It is your job to declare such modules in `app.go`, as you will see later in this tutorial. `baseapp` only implements the core routing logic that can be applied to any module.

  **路由消息到适合的模块, ==如果有自定义的特殊模块需要在app.go中申明,否则baseapp是无法自动识别的==,baseap只是实现了核心的路由逻辑**

- Commit if the ABCI message is [`DeliverTx` (opens new window)](https://docs.tendermint.com/master/spec/abci/abci.html#delivertx)([`CheckTx` (opens new window)](https://docs.tendermint.com/master/spec/abci/abci.html#checktx)changes are not persistent).

  

- Help set up [`BeginBlock` (opens new window)](https://docs.tendermint.com/master/spec/abci/abci.html#beginblock)and [`Endblock` (opens new window)](https://docs.tendermint.com/master/spec/abci/abci.html#endblock), two messages that enable you to define logic executed at the beginning and end of each block. In practice, each module implements its own `BeginBlock` and `EndBlock` sub-logic, and the role of the app is to aggregate everything together (*Note: you won't be using these messages in your application*).

  **根据两个消息的定义从区块链的开始区块执行到末尾区块,每一个模块都有自己的执行子逻辑,app的作用就是将其汇总**==(这些消息在你的应用程序中不会使用)==

- Help initialize your state.

  **帮助初始化状态**

- Help set up queries.

  **帮助建立查询**·

#### Multistore

多样存储Multistore允许开发人员声明任意数量的kvstore数据库,但是kvstore的value只能是[]byte,在存储时一些特定的结构可能还需要先序列化

**Multistore抽象的理解就是分离各个状态,每一种状态都由自己的模块管理**

#### Modules

cosmos的强大依赖于模块化,SDK应用就是基于一系列集合的互操作模块,每一个模块都定义了状态的子集和包含自己交易/消息的处理逻辑.而SDK的责任就在于作为转发各种消息的路由

每个模块可以看作一个小型的状态机,开发人员需要定义模块处理的状态的子集以及更改状态的特殊消息类型,**每个模块在多存储Multistore中声明自己的KVStore，以持久化它定义的状态子集。**

开发者开发时会用到其他人开发的模块即第三方模块,但是安全性是不一定能够得到保证的,所以在 [object-capabilities](https://docs.cosmos.network/v0.41/core/ocap.html)上做了规则,这意味着，与让每个模块为其他模块保留访问控制列表不同，**每个模块实现了称为keepers的特殊对象**，**可以将其传递给其他模块以授予预定义的功能集**。

Here is a simplified view of how a transaction is processed by the application of each full-node when it is received in a valid block:

```js                                      +
                                      |
                                      |  Transaction relayed from the full-node's Tendermint engine
                                      |  to the node's application via DeliverTx
                                      |
                                      |
                                      |
                +---------------------v--------------------------+
                |                 APPLICATION                    |
                |                                                |
                |     Using baseapp's methods: Decode the Tx,    |
                |     extract and route the message(s)           |
                |                                                |
                +---------------------+--------------------------+
                                      |
                                      |
                                      |
                                      +---------------------------+
                                                                  |
                                                                  |
                                                                  |
                                                                  |  Message routed to the correct
                                                                  |  module to be processed
                                                                  |
                                                                  |
+----------------+  +---------------+  +----------------+  +------v----------+
|                |  |               |  |                |  |                 |
|  AUTH MODULE   |  |  BANK MODULE  |  | STAKING MODULE |  |   GOV MODULE    |
|                |  |               |  |                |  |                 |
|                |  |               |  |                |  | Handles message,|
|                |  |               |  |                |  | Updates state   |
|                |  |               |  |                |  |                 |
+----------------+  +---------------+  +----------------+  +------+----------+
                                                                  |
                                                                  |
                                                                  |
                                                                  |
                                       +--------------------------+
                                       |
                                       | Return result to Tendermint
                                       | (0=Ok, 1=Err)
                                       v
```

SDK modules are defined in the `x/` folder of the SDK. Some core modules include:

- `x/auth`: Used to manage accounts and signatures.
- `x/bank`: Used to enable tokens and token transfers.
- `x/staking` + `x/slashing`: Used to build Proof-Of-Stake blockchains.

## 2.2 starport tool

### 2.2.1 Introduction

#### 什么是starport

搭建cosmos的脚手架工具

只需几个命令，您就可以创建区块链、启动区块链、在云上提供服务，并准备好一个 GUI 开始测试您的应用程序。

There are many projects already showcasing that the Tendermint BFT Consensus Engine and the Cosmos SDK.

The following projects are using the technology:

- [Cosmos](https://github.com/cosmos/gaia) (Main IBC Hub and "Rolemodel" of the Cosmos SDK)
- [Binance Chain](https://github.com/binance-chain) (DEX and utility token)
- [Crypto.com Chain](https://github.com/crypto-com/chain-main) (Payments, DeFi, and utility token)
- [IRIS](https://github.com/irisnet) (IBC Hub and developer oriented)
- [Kava](https://github.com/Kava-Labs/kava) (DeFi and Stable Coins)
- [Aragon](https://docs.chain.aragon.org/) (DAO catalyst)
- [CosmWasm](https://cosmwasm.com/) (smart contracts using WASM)
- ==[Ethermint](https://ethermint.zone/) (Ethereum virtual machine)==

[See the full list here](https://cosmonauts.world/).

#### 安装

https://github.com/tendermint/starport/blob/develop/docs/1%20Introduction/2%20Install.md

`git clone https://github.com/tendermint/starport && cd starport && make`

#### 快速开始

With `starport` installed on your machine, you can now build your very first blockchain!

**创建项目的命令格式:**

```
starport app github.com/username/myapp && cd myapp:
```

This command will create a directory `myapp` and scaffold a Cosmos SDK blockchain.

---

**下面的命令开启服务:**

```
starport serve
```

`serve` will install dependencies, build, initialise and run your blokchain.

---

**创建类型 , starport会自动对应的创建CRUD的操作**

```
starport type post title body
```

`type` scaffolds functionality to create, read, update and delete for a custom type.

#### 构造

##### config.yaml

When creating a new app with starport, you will see a `config.yml` file in your blockchain folder. This file defines the genesis file, the network you will be using and the first validators of your blockchain.

`config.yaml`文件定义了创世文件, 需要使用的网络以及网络中的第一个Validator

```yaml
# 当前的版本
version: 1
# 定义区块链上代币的初始分布, 这些账户被定义在创世块中, 并且这些账户会在区块链中生成对应的公私钥对,在命令行中可以访问

accounts:
  - name: user1
    coins: ["1000token", "100000000stake"]
  - name: user2
  # coins在区块链上标明coin的数量和面额。你可以在这里列出你的区块链上使用的各种面值的硬币和它们各自的金额。
    coins: ["500token"]
# 定义验证者的时候可以直接使用上面的账户,并且绑定权益
# 绑定权益即stake必须等于或小于accounts参数中给出的权益。
validator:
  name: user1
  staked: "100000000stake"
```

`config.yaml`可选字段:

* genesis

  可以直接定义创世纪文件中的参数，例如链ID：

  ```
  genesis:
    chain_id: "foobar"
  ```

  您还可以操作不同模块的参数。例如，如果您想要更改模块，其中包含标注参数，例如哪个令牌被押注，您将向"config.yml"添加以下内容`staking`

  ```
  genesis:
    app_state:
      staking:
        params:
          bond_denom: "denom"
  ```

  运行`starport serve`后, 这些配置参数将会创建于创世文件中, 其创建（或覆盖）位于用户目录下的文件夹

This will create (or override) the folder located at your user homefolder `~/.myappd` (the name of your application with a `d` for `daemon` attached) and initiate your blockchain with the genesis file. 

创世文件的目录结构:

![pYgbSb](http://xwjpics.gumptlu.work/qinniu_uPic/pYgbSb.png)

其他的一些文件创建在`~/.[你的app名]cli`下,其中包含当前命令行接口的配置文件,如`chain-id`，一些输出参数,如`json`或`indent`模式(排版模式)。

如果您想确保从区块链设置中删除所有数据，请务必删除该文件夹和文件夹。`~/.myappd` `~/.myappcli`

##### 账户地址前缀

您可以更改区块链应用程序中的地址外观。即他们在开头附加其他的内容。在cosmos Main Hub地址前面有一个地址前缀，例如`cosmos12fjzdtqfrrve7zyg9sv8j25azw2ua6tvu07ypf`

您可以通过更改创建的application的` /app/prefix.go`中的`AccountAddressPrefix`变量来更改第一个前缀。建议您保留其他变量，因为它们是Cosmos SDK链中使用的标准，因此可以被识别。这有安全隐患，如不发送到可能无法使用它的地址。

要让您的前端正确地使用新命名，您需要更改`/vue/.env`中的`VUE_APP_ADDRESS_PREFIX`变量。

为了自动完成这一切，**在使用命令`starport app github.com/foo/bar`创建应用程序时，只需添加`--address-prefix`前缀参数。**

##### 总结

- 定义您的起源帐户和验证器。`config.yml`
- 它允许您使用不同的代币激励区块链，并指定第一个区块中每个初始帐户的金额。
- 更改地址的前缀可以在文件`/app/prefix.go`中完成。或者在启动时添加前缀参数

#### 创世文件genesis.json

创世块是区块链的第一个区块,创世区块通常是区块链中唯一无法在您即将启动的同一 P2P 网络上找到的块，它必须以不同的方式共享 - 我们将查看在另一个教程中共享生成文件的方法

因为它是区块链的起点，尤其是在风险证明POS区块链中，它包含初始地址和余额列表。此外，大多数情况下，创世块定义了区块链使用的区块链网络。

使用 Starport，您将从您的文件中创建一个创世纪文件`.myappd/config/genesis.json`，它通常看起来类似于此：`config.yml`

具体文件见项目;

仔细观察创世纪文件，您可以观察到它包含区块链应用程序的初始状态参数，此外还包含您正在使用的模块的定义和参数。

除了模块定义和配置外，**生成文件还保留区块链初始利益相关者和验证者的地址**。**这些位于`gentx`参数中，这是`genutil`的一部分**。启动区块链时，**这些验证器应是网络的一部分，以便让网络运行。或者至少 66% 的验证器应该可用**，以便启动 BFT 共识。

为了正确设置您的起源文件，重要的是要了解`config.yml`，下面将会详细的介绍.

#### Starport IBC

区块链之间的通行协议是IBC, IBC 允许两个链条之间可靠且安全的连接，可用于传输令牌、多链智能合约、原子交换或数据以及任何类型的代码分片。

为了在链条之间进行通信，使用 Starport 引导两个区块链将让我们了解其工作原理以及通信过程中每个区块链应用程序上发生的情况。在此教程中，我们将创建两个区块链，连接这些区块链并通过 IBC 传输令牌。

###### Scaffolding chain `foo`

**在在线环境创建第一个项目foo**

To start using IBC with Starport open up a [web-based development environment](https://gitpod.io/#https://github.com/tendermint/starport/), then scaffold and launch a Stargate chain:

```
starport app github.com/foo/foo

cd foo

starport serve
```

You now have a blockchain `foo` running, but it's not connected to anything yet.

###### Scaffolding chain `bar`

要将此区块链连接到另一个区块链，请打开另一个[基于 Web 的开发环境](https://gitpod.io/#https://github.com/tendermint/starport/)实例，并按照上面的步骤设置脚手架并启动另一个链条（让我们称之为）`bar`

为了连接我们的区块链，我们将使用[中继器](https://github.com/cosmos/ics/tree/master/spec/ics-018-relayer-algorithms)。中继器是我们两个区块链之间的"物理"连接。它负责监控两个区块链，在它们之间中继数据，构建适当的图表，并在两个区块链上相应地执行它们。链条运行后，您将在终端输出中看到"中继器信息"字符串（您的字符串将有所不同）：

![bCt32L](http://xwjpics.gumptlu.work/qinniu_uPic/bCt32L.png)

这是一个`base64`编码的 JSON，包含有关链 ID、中继器帐户 mnemonic 和 RPC URL 的信息。 

==如果没有Relayer输出, 原因是relayer的版本太低! (官方文档中给的在线环境版本是开发版,如果relayer版本太低不会报错!), 检查starport的版本(使用0.14.0), 如果是本地环境,那么就手动下载rly==

问题详细见: https://github.com/tendermint/starport/pull/385

安装rly: https://stackoverflow.com/questions/64651914/how-to-install-relayer-for-cosmos-sdk-starport-chain

![zlJ78C](http://xwjpics.gumptlu.work/qinniu_uPic/zlJ78C.png)

```go
✨ Relayer info: eyJDaGFpbklEIjoiYmFyIiwiTW5lbW9uaWMiOiJhbW9uZyBibHVyIG5lc3QgYm9keSBmcm9udCBicmlkZ2Ugc25ha2UgYmFsYW5jZSB5b3VuZyBzeW1ib2wgdHJpbSBjaGVmIHByb3BlcnR5IGZyb3plbiB3ZWVrZW5kIG1pZG5pZ2h0IGJyYXZlIHdpbmcgZGVwYXJ0IGh1YiBoYW1zdGVyIHN1cGVyIHZlbmRvciByZWFsIiwiUlBDQWRkcmVzcyI6Imh0dHBzOi8vMjY2NTctY3JpbXNvbi1zcXVpZC1janFkeWQ0YS53cy11czAzLmdpdHBvZC5pbzo0NDMifQ
```

###### Connecting `foo` with `bar`

To connect these chains together, copy the relayer info of chain `bar`, switch to the terminal of chain `foo` and run the following command (use your own string):

```
starport chain add eyJDaGFpbklEIjoiYmFyIiwiTW5lbW9uaWMiOiJhbW9uZyBibHVyIG5lc3QgYm9keSBmcm9udCBicmlkZ2Ugc25ha2UgYmFsYW5jZSB5b3VuZyBzeW1ib2wgdHJpbSBjaGVmIHByb3BlcnR5IGZyb3plbiB3ZWVrZW5kIG1pZG5pZ2h0IGJyYXZlIHdpbmcgZGVwYXJ0IGh1YiBoYW1zdGVyIHN1cGVyIHZlbmRvciByZWFsIiwiUlBDQWRkcmVzcyI6Imh0dHBzOi8vMjY2NTctY3JpbXNvbi1zcXVpZC1janFkeWQ0YS53cy11czAzLmdpdHBvZC5pbzo0NDMifQ
```

![yaarO9](http://xwjpics.gumptlu.work/qinniu_uPic/yaarO9.png)

Chain `foo` will now restart and you should see information about two being connected:

```
Detected chains, linking them...
Linked foo <--> bar
```

The two chains are now connected via IBC and you have successfully created a relayer.

![6b9lkS](http://xwjpics.gumptlu.work/qinniu_uPic/6b9lkS.png)

###### Sending tokens from `foo` to `bar`

**跨链发送代币foo => bar**, 使用`rly`客户端创建一个IBC Token

Once the chains are connected, you can use a [relayer](https://github.com/cosmos/relayer) CLI `rly` to create an IBC token send transaction:

```
rly tx transfer foo bar 5token $(rly chains address bar)
```

After a transaction is successfully created, you can now relay it to a connected chain:

```
rly tx relay foo-bar
```

![3bXHTB](http://xwjpics.gumptlu.work/qinniu_uPic/3bXHTB.png)

![asR993](http://xwjpics.gumptlu.work/qinniu_uPic/asR993.png)

###### Checking token balances on chain `bar`

To verify that an IBC transaction was relayed correctly, let's check the balances of our relayer account:

**要验证 IBC 交易的中继正确，让我们检查我们的中继器帐户的余额：**

```
bard q bank balances $(bard keys show bar -a --keyring-backend test)
```

This command will output token balances for the relayer account and you should see 5 token transferred with IBC.

此命令将输出中继器帐户的令牌余额，您应该看到5个token通过IBC传输。

<u>*提示没有bard命令???*</u>

#### Configuration

Inside your chain's project directory you will see `secret.yml`. This file contains information about the local chain's relayer account (under `accounts` property) and relayer accounts of connected chains (under `relayer` property).

此文件包含有关本地链的中继器帐户和连接链中继器帐户的信息。

foo下的secret.yaml文件

```yaml
accounts:
- name: foo
  coins:
  - 800token
  mnemonic: fly nerve endless pistol spread harsh derive assume grass hybrid pink ancient number monster dice error diagram discover ribbon hold drink vast toy animal
relayer:
  accounts:
  - name: bar
    mnemonic: among blur nest body front bridge snake balance young symbol trim chef property frozen weekend midnight brave wing depart hub hamster super vendor real
    rpc_address: https://26657-crimson-squid-cjqdyd4a.ws-us03.gitpod.io:443
```

bar下的secret.yaml文件

```yaml
accounts:
- name: bar
  coins:
  - 800token
  mnemonic: among blur nest body front bridge snake balance young symbol trim chef property frozen weekend midnight brave wing depart hub hamster super vendor real
relayer:
  accounts: []
```

Once the chain is launched with `starport serve`, Starport uses information from `secret.yml` to create a relayer config in `~/.relayer/`. Every time the chain is restarted relayer config is reset, and connections are re-established.

### 2.2.2 Architecture

#### 介绍

**需要提前安装golang和starport, 对于运行的机器没有特别的要求, Linux或者Mac操作系统或者树莓派均可**

Starport creates a blockchain for you in Golang. Requirements for this is to have Golang installed. You can get all the information here https://golang.org/doc/install.

Starport installation instructions can be found here: https://github.com/tendermint/starport#install

To the machine you are executing on there are not many requirements. It runs on Linux or Mac Operating Systems and can be run from a Raspberry Pi.

##### Your blockchain application

To create a blockchain application we use the command `app`

```
starport app github.com/username/myapp
```

| Flag               | Default    | Description                                                  |
| ------------------ | ---------- | ------------------------------------------------------------ |
| `--address-prefix` | `cosmos`   | Prefix, used for addresses **指定地址前缀**                  |
| `--sdk-version`    | `stargate` | Version of Cosmos SDK: `launchpad` or `stargate`**指定CosmosSDK** |

This will create the folder `myapp` and is a usable blockchain blueprint. If you want to dive directly into looking at the details of your blockchain you can run it with entering your `myapp` folder and use the command `serve` to initialise your blockchain and start it.

##### Serve

`starport serve`

**以上命令会安装依赖,编译以及初始化**

To start the server, go into you application's directory and run `starport serve`. This commands installs dependencies, builds and initializes the app and runs both Tendermint RPC server (by default on `localhost:26657`) as well as LCD (by default on `localhost:1317`) with hot reloading enabled.

**`starport serve` uses `config.yml` to initialize your application, make sure you have it in your project directory (see [Configure](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/1 Introduction.md#configure)).**

**项目启动会使用config.yml文件,必须存在**

Note: depending on your OS and firewall settings, you may have to accept a prompt asking if your application's binary (`blogd` in this case) can accept external connections.

根据您的操作系统和防火墙设置，您可能需要接受提示，询问您的申请的二进制（在此例中`blogd`）是否可以接受外部连接。

| Flag        | Default | Description                                              |
| ----------- | ------- | -------------------------------------------------------- |
| `--verbose` | `false` | Enable verbose output from processes **输出运行细节 -v** |
| `--path`    |         | Path to the project **路径**                             |

The first step of your own blockchain is already done. Using the default settings, a blockchain that has networking, consensus protocol with an own token is hereby established. From here on, you can implement logic that makes your own blockchain unique.

##### The Key-Value Store (KV)

###### ==**How to use types**==

In the SDK, data is stored in the multistore(多重存储). Key-Value pairs are saved in the KVStores. Multiple stores can be created and managed at the same time. We will use the store to save our data to the blockchain. Starport assists us in setting up the Key-Value Store with the command `type`. In order to use `type` we should give our type a fitting `typeName` with the intended fields that we want to use. If we wanted to store user with username and age, we would use the command

多样的存储可以同时的创建和管理, 使用type创建KV数据库

```shell
starport type [typeName] [field1] [field2:bool] ...More specific
```

以user结构体举例

```go
type user struct {
  username string,	// 默认是string
  age int 
}
```

```shell
starport type user username age:int
```

**This command generates messages, handlers, keepers, CLI and REST clients and type definition for `typeName` type**. A type can have any number of `field` arguments. **By default fields are strings, but `bool` and `int` are supported.**

==**默认创建的field是string类型,但是支持bool和int类型**==

Now a Key-Value Store for the user with fields username and age is created. We can create a new user with the command

**创建user实例**

```
myappcli tx myapp create-user "my-first-username" 35
```

Which creates the user with username `my-first-username` and age of `35`.

Another example,

```
starport type post title body
```

This command generates a type `Post` with two fields: `title` and `body`.

To add a post run `blogcli tx blog create-post "My title" "This is a blog" --from=user1`.

These are the basic commands for getting started with starport. From creating a first blockchain to adding your own data types and accessing the User Interface. In the next two chapters, we will be looking closer at th<u>e initial setup for starport and how to configure it</u>. Afterwards, we will be looking into more complex usecases, where each of the commands and more will be explained in detail.

###### ==Accounts on your blockchain==

An account on the blockchain is a **keypair(密钥对)** of private and public keys. **When you start your blockchain with starport, you can define the name of the keys and the amount of coins they start with.** The keys are created for you and displayed on startup. You can use these keys when interacting with your blockchain. **A list of user accounts is created during genesis of your application**. You can define them as follows in your `config.yml` file. See an example in chapter [configuration](https://github.com/tendermint/starport/blob/develop/docs/03_configuration/03_configuration.md).
**账户的体现是一对公私钥, 在开始启动区块链的时候可以定义你的账户keys以及初始coins, 初始化可以创建一系列的账户,你可以在config.yaml中进行定义.  **

| Key   | Required | Type            | Description                                       |
| ----- | -------- | --------------- | ------------------------------------------------- |
| name  | Y        | String          | Local name of the key pair                        |
| coins | Y        | List of Strings | Initial coins with denominations (e.g. "100coin") |

###### The initial validator

Blocks on a tendermint blockchain are created and **validated** by the so called `validators`. You can define the set of validators your blockchain starts with in your `config.yml`. The validator property describes your set of validators. Use a `name` that you have specified in the `accounts` array. The account should have enough tokens for staking purposes. See an example in chapter [configuration](https://github.com/tendermint/starport/blob/develop/docs/03_configuration/03_configuration.md).

**项目初始化也可以创建Validator, 同样的在config.yaml中定义,该帐户应有足够的tokens用于投注目的**

| Key    | Required | Type   | Description                                                  |
| ------ | -------- | ------ | ------------------------------------------------------------ |
| name   | Y        | String | Name of one the accounts                                     |
| staked | Y        | String | Amount of coins staked by your validator, should be >= 10^6 (e.g. "100000000stake") |

##### Summary

- With the command `starport app` a new blockchain can be initialised.
- A combination `starport app` and `starport serve` already let's you manage your blockchain out of the box.
- The default blockchain includes networking and a consensus protocol with your own token.
- Data is managed with the Key-Value Store and data types can be added with `starport type`.
- Accounts are created during genesis of the application. These can be configured in the `config.yml`.

#### 目录结构

```shell
x/{module}
├── client
│   ├── cli
│   │   ├── query.go
│   │   └── tx.go
│   └── rest
│       ├── query.go
│       └── tx.go
├── exported
│   └── exported.go
├── keeper
│   ├── invariants.go
│   ├── genesis.go
│   ├── keeper.go
│   ├── ...
│   └── querier.go
│   └── grpc_query.go
├── types
│   ├── codec.go
│   ├── errors.go
│   ├── events.go
│   ├── expected_keepers.go
│   ├── genesis.go
│   ├── keys.go
│   ├── msgs.go
│   ├── params.go
│   ├── types.proto
│   ├── ...
│   └── querier.go
│   └── {module_name}.pb.go
│   └── query.pb.go
│   └── genesis.pb.go
├── simulation
│   ├── decoder.go
│   ├── genesis.go
│   ├── operations.go
│   ├── params.go
│   └── proposals.go
├── abci.go
├── handler.go
├── ...
└── module.go
```

`abci.go`: The module's BeginBlocker and EndBlocker implementations (if any).**模块的开始块与结束块实现**

`client/`: The module's CLI and REST client functionality implementation and testing.**客户端**

`exported/`: The module's exported types -- typically type interfaces. **If a module relies on other module keepers, it is expected to receive them as interface contracts through the expected_keepers.go** (which are detailed below下文详述) design to **avoid having a direct dependency on the implementing module**. However, these contracts can define methods that operate on and/or return types that are specific to the contract's implementing module and this is where exported/ comes into play. Types defined here allow for expected_keepers.go in other modules to define contracts that use single canonical types. This pattern allows for code to remain DRY and also alleviates import cycle chaos.

**如果当前模块依赖其他模块的keeper,那么通过expected_keepers.go使之相关联,以此来避免直接依赖模块,然而，这些契约可以定义对特定于契约实现模块的和/或返回类型进行操作的方法，这就是export /comes发挥作用的地方。这里定义的类型允许使用expected_keepers。在其他模块中定义使用单一规范类型的契约。这种模式允许代码保持整洁，并且还减轻了导入周期的混乱。**

`handler.go`: The module's message handlers.

`keeper/`: The module's keeper implementation along with any auxiliary implementations such as the querier and invariants.

`types/`: The module's type definitions such as **messages, KVStore keys, parameter types, Protocol Buffer definitions, and expected_keepers.go contracts.  **

`module.go`: The module's implementation of the **AppModule and AppModuleBasic interfaces.**

#### 标准模块

##### What are modules

In the Cosmos SDK modules are the basis for the logic of your blockchain. Each module serves specific purposes and functions. The Cosmos SDK offers a variety of native modules to make a blockchain work. These modules handle authentication for users, token transfers, governance functions, staking of tokens, supply of tokens and many more.

If you want to change the default functionality of a module or just change certain hardcoded parameter, you can fork a module and change it, therefore owning your own logic for your blockchain. While forking and editing a module should be done carefully, this approach marks the Cosmos SDK as especially powerful, as you can experiment with different parameters as the standard implementation suggests.

Modules do not need to be created by a specific company or individual. They can be created by anyone and offered for general use to the public. Although there do exist standards that projects look into before integrating a module to their blockchain. It is recommended that a module has understandable specifications, handles one thing good and is well tested - optimally battle-tested on a live blockchain. When growing more complex, sometimes it makes more sense to have two modules instead of one module trying to "solve-it-all", this consideration can make it more attractive for other projects to use a module in their blockchain project.

在Cosmos中，SDK模块是区块链逻辑的基础。每个模块都有特定的用途和功能。Cosmos SDK提供了各种原生模块来让区块链工作。这些模块处理用户身份验证、令牌传输、治理功能、令牌权益、令牌供应等等。

如果你想改变一个模块的默认功能或者只是改变某个硬编码的参数，**你可以分叉一个模块并改变它，因此拥有你自己的区块链逻辑**。分叉和编辑模块应该小心，但这种方法标志着Cosmos SDK特别强大，因为你可以试验标准实现建议的不同参数。

模块不需要由特定的公司或个人创建。它们可以由任何人创建并提供给公众使用。尽管项目在将模块集成到他们的区块链之前确实存在一些标准。建议一个模块有可理解的规格，能很好地处理一件事，并且经过了很好的测试——最好是在实时的区块链上进行战斗测试。当变得更复杂时，有时使用两个模块比一个模块试图“解决所有问题”更有意义，这种考虑可以使其他项目在区块链项目中使用一个模块更有吸引力。

##### Standard modules

When creating a blockchain with starport or manually with the Cosmos SDK, there is a set of modules that you should be using in order to have a set of rules that define a blockchain in the first place. These modules are

**目录**

- What are modules
  - [Standard modules](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#standard-modules)
  - [Auth](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#auth)
  - [Bank](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#bank)
  - [Capability](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#capability)
  - [Staking](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#staking)
  - [Mint](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#mint)
  - [Distribution](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#distribution)
  - [Params](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#params)
  - [Governance](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#governance)
  - [Crisis](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#crisis)
  - [Slashing](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#slashing)
  - [IBC](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#ibc)
  - [Upgrade](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#upgrade)
  - [Evidence](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#evidence)
  - [Using modules](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#using-modules)
  - [Summary](https://github.com/tendermint/starport/blob/develop/docs/2 Architecture/3 Standard Modules.md#summary)

###### Auth

The `auth` module is responsible for accounts on the blockchain and basic transaction types that accounts can use.

**该模块负责处理区块链上的帐户和帐户可以使用的基本交易类型**

It introduces Gas and Fees as concepts in order to prevent the blockchain to bloat by not-identifyable accounts, as on public blockchains you do not have more information about accounts as the public key or balance of an account or the previous transaction history.

它引入Gas和费用作为概念，以防止区块链因无法识别的账户而膨胀，因为在公共区块链上，您没有更多关于账户的信息，如公钥、账户余额或以前的交易历史。

帐户的接口定义为

The interface of an Account is defined as

```go
// BaseAccount defines a base account type. It contains all the necessary fields
// for basic account functionality. Any custom account type should extend this
// 基本的账户函数, 没一个特定的账户类型都继承于此
// type for additional functionality (e.g. vesting).
message BaseAccount {
  option (gogoproto.goproto_getters)  = false;
  option (gogoproto.goproto_stringer) = false;
  option (gogoproto.equal)            = false;

  option (cosmos_proto.implements_interface) = "AccountI";

  string              address = 1;
  google.protobuf.Any pub_key = 2
      [(gogoproto.jsontag) = "public_key,omitempty", (gogoproto.moretags) = "yaml:\"public_key\""];
  uint64 account_number = 3 [(gogoproto.moretags) = "yaml:\"account_number\""];
  uint64 sequence       = 4;
}
```

The `auth` module exposes the account keeper where accounts can be stored or changed. Furthermore it exposes the types for Standard transactions, fees, signatures or replay-prevention. It also allows for vesting accounts, as that certain coins can be made accessible over a period of time to an entity. The vesting logic is mostly used for unbonding of staking but can also be used for other purposes.

auth模块公开帐户管理员，**帐户可以存储或更改**。此外，它还公开了标准事务、费用、签名或重放预防的类型。它还允许对账户进行归属，因为某个实体可以在一段时间内访问某些货币。归属逻辑主要用于权益的解绑定，但也可以用于其他目的。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/auth/spec/README.md)*

###### Bank

The `bank` module has its name because it acts as the module that allows for token transfers and checks for the validity of each transfer. Furthermore, it is responsible for checking the whole supply of the chain, the sum of all account balances.

bank模块之所以有自己的名称，是因为它充当了允许令牌转账和检查每个转账有效性的模块。此外，它还负责检查整个供应链，即所有帐户余额的总和。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/bank/spec/README.md)*

###### Capability

Full implementation of the IBC specification requires the ability to **create and authenticate object-capability keys at runtime** (i.e., during transaction execution), as described in ICS 5. In the IBC specification, capability keys are created for each newly initialised port & channel, and are used to authenticate future usage of the port or channel. Since channels and potentially ports can be initialised during transaction execution, the state machine must be able to create object-capability keys at this time.

IBC规范的完整实现需要在运行时(即交易执行期间)创建和验证对象功能键的能力，如ICS 5中所述。在IBC规范中，将为每个新初始化的端口和通道创建功能键，并用于验证端口port或通道channel未来的使用情况。由于通道和潜在的端口可以在事务执行期间初始化，因此状态机必须能够在此时创建对象功能键。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/capability/spec/README.md)*

###### Staking

The `staking` module allows for an advanced Proof of Stake system, where validators can be created and tokens delegated to validators.

**权益模块支持高级的权益证明POS系统，在该系统中可以创建验证器Validator，并将令牌委托给验证器。**

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/staking/spec/02_state_transitions.md#slashing)*

###### Mint

The minting mechanism is designed to allow for a flexible inflation rate determined by market demand targeting a particular bonded-stake ratio effect a balance between market liquidity and staked supply.

It can be broken down in the following way:

If the inflation rate is below the goal %-bonded the inflation rate will increase until a maximum value is reached If the goal % bonded (67% in Cosmos-Hub) is maintained, then the inflation rate will stay constant If the inflation rate is above the goal %-bonded the inflation rate will decrease until a minimum value is reached

铸币机制的目的是允许由市场需求决定的灵活通货膨胀率，以特定的债券比率为目标，以平衡市场流动性和债券供应。

它可以通过以下方式进行分解:

如果通货膨胀率低于%保税目标通货膨胀率会增加,直到达到最大值,如果目标%保税Cosmos-Hub(67%),那么通货膨胀率将保持不变,如果通货膨胀率高于目标%保税通货膨胀率会降低,直到达到最小值

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/mint/spec/README.md)*

###### Distribution

The `distribution` module is responsible to distribute the inflation of a Token. When new Tokens get created, they get distributed to the validators and their respective delegators, with a potential commission the validator takes. Each validator can choose a commission of those Token when creating a validator, this commission is editable.

分发模块负责分发令牌的膨胀。当新的令牌被创建时，它们被分发给验证器和它们各自的委托者，验证器接受潜在的委托。当创建一个验证器时，这个验证器是可编辑的。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/distribution/spec/README.md)*

###### Params

The `params` module allows for a global parameter store in your blockchain application. It is designed to hold the chain parameters so that they can be changed during runtime by governance. It allows to upgrade the blockchain parameters via the `government` module and take effect on an agreed upon time when the majority of the shareholders decide to make a change.

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/params/spec/README.md)*

Those modules are typically installed on default when using starport. There are a range of modules that are also part of the Cosmos SDK, additionally some other public modules have already reached a major level of usage and acceptance. We will look at more advanced modules in the next Chapter.

**params模块允许在区块链应用程序中存储全局参数**。**它被设计用来保存链参数**，以便在运行时通过治理更改它们。它允许通过政府模块升级区块链参数，并在多数股东决定更改时约定时间生效。

这些模块通常在使用starport时默认安装。还有一系列模块也是Cosmos SDK的一部分，另外一些公共模块已经达到了主要的使用和接受水平。我们将在下一章讨论更高级的模块。

###### Governance

The module enables Cosmos-SDK based blockchains to support an on-chain governance system. In this system, holders of the native staking token of the chain can vote on proposals on a 1 token 1 vote basis (from there it can be parameterized). Next is a list of features the module currently supports:

- Proposal submission: Users can submit proposals with a deposit. Once the minimum deposit is reached, proposal enters voting period
- Vote: Participants can vote on proposals that reached MinDeposit
- Inheritance and penalties: Delegators inherit their validator's vote if they don't vote themselves.
- Claiming deposit: Users that deposited on proposals can recover their deposits if the proposal was accepted OR if the proposal never entered voting period.

该模块使基于Cosmos-SDK的区块链能够支持链上治理系统。在这个系统中，链上本地下注令牌的持有者可以在1个令牌1个投票的基础上对提议投票(从那里可以参数化它)。下面是该模块当前支持的特性列表:

* 提案提交:用户可以提交提交提案的定金。一旦最低存款达到，提案进入表决期

* 投票:参与者可以对达到最低存款的提案进行投票

* 继承和惩罚:委托者如果自己不投票，就继承验证者的投票。

* 认领押金:如果提案被接受或者提案从未进入投票期，对提案进行押金认领的用户可以收回押金。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/gov/spec/README.md)*

###### Crisis

The crisis module halts the blockchain under the circumstance that a blockchain invariant is broken. Invariants can be registered with the application during the application initialization process.

危机模块在区块链不变式被打破的情况下**停止区块链**。在应用程序初始化过程中，可以向应用程序注册不变量。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/crisis/spec/README.md)*

###### Slashing

The slashing module enables Cosmos SDK-based blockchains to disincentivize any attributable action by a protocol-recognized actor with value at stake by penalizing them ("slashing").

Penalties may include, but are not limited to:

Burning some amount of their stake Removing their ability to vote on future blocks for a period of time. This module will be used by the Cosmos Hub, the first hub in the Cosmos ecosystem.

削减模块使Cosmos基于sdk的区块链能够通过**惩罚协议认可的参与者(“削减”)来阻止任何可归责的行为。**

罚款可包括但不限于:

烧了一些他们的股份，在一段时间内取消他们对未来区块的投票能力。这个模块将被宇宙中心Hub使用，宇宙生态系统中的第一个中心。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/slashing/spec/README.md)*

###### IBC

IBC allows to relay packets between chains and could be used with any compatible modules between two chains. The `IBC` module, as Inter-blockchain Communication, enables for example sending native tokens between blockchains. It is divided by a subset of specifications.

IBC允许**在链之间中继信息包**，可以与**两个链之间的任何兼容模块一起使用**。作为区块链间通信，IBC模块支持在区块链之间发送本地令牌。它被规范的子集所分割。

*Read the [specifications](https://github.com/cosmos/cosmos-sdk/blob/master/x/ibc/spec/README.md)*

###### Upgrade

`x/upgrade` is an implementation of a Cosmos SDK module that facilitates smoothly upgrading a live Cosmos chain to a new (breaking) software version. It accomplishes this by providing a BeginBlocker hook that prevents the blockchain state machine from proceeding once a pre-defined upgrade block time or height has been reached.

x/upgrade是Cosmos SDK模块的一个实现，它有助于将一个活生生的Cosmos链顺利**升级**到**一个新的(中断的)软件版本**。它通过提供一个BeginBlocker钩子来实现这一点，一旦达到预定义的升级块时间或高度，它就阻止区块链状态机继续运行。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/upgrade/spec/README.md)*

###### Evidence

`x/evidence` is an implementation of a Cosmos SDK module, per ADR 009, that allows for the submission and handling of arbitrary evidence of misbehavior such as equivocation and counterfactual signing.

The evidence module differs from standard evidence handling which typically expects the underlying consensus engine, e.g. Tendermint, to automatically submit evidence when it is discovered by allowing clients and foreign chains to submit more complex evidence directly.

x/evidence是Cosmos SDK模块的一个实现，根据ADR 009，**允许提交和处理不当行为的任意证据**，如含糊入词和反事实签署。

证据模块不同于标准的证据处理，后者通常期望底层的共识引擎(如Tendermint)在证据被发现时自动提交证据，允许客户和外国链直接提交更复杂的证据。

*Read the [specification](https://github.com/cosmos/cosmos-sdk/blob/master/x/evidence/spec/README.md)*

###### ==Using modules==

With starport you can add a module with the command `starport module create modulename`. When adding a module manually to a blockchain application, it requires to edit the `app/app.go` and the `myappcli/main.go` with the according entries. Starport manages the code edits and additions for you conveniently.

使用starport，**==您可以使用`starport module create modulename`命令添加模块。==**当手动添加一个模块到区块链应用程序时，它需要编辑`app/app.go`和`myappcli/main.go`,按照相应的条目去做。Starport为您方便地管理代码编辑和添加。

##### Summary

- Importing modules in a Cosmos SDK built blockchain exposes new functionalities for the blockchain.
- Any combination of modules is allowed.
- The modules define what can be done on the blockchain.
- Modules are editable, but the success of your blockchain will be dependend on choosing the correct modules for your blockchain, for functionality and security sake.
- ==**`starport module import <modulename>` lets you import modules into your blockchain application.**==

#### **编写自定义模块**

##### Writing custom modules

Starport allows you to jump directly into creating your own module. With the before described `type` function you can add new transaction types to your application. Under the hood, starport creates a handler, types and messages for you.

**Starport允许你直接创建你自己的模块。使用前面描述的`type`类型函数，您可以向应用程序添加新的交易类型。在底层，starport为您创建一个处理程序、类型和消息。**

Without using starport, you would need to manipulate these functions yourself. Here is what starport does when you add a `type`. Understanding what starport does might help in order to either add more complex structures or debug in case something does not work as it should.

**如果不使用starport，您将需要自己操作这些函数。以下是starport在添加类型时所做的。理解starport所做的可能有助于添加更复杂的结构或调试，以防某些东西不能正常工作。**

###### Proto

When using the type command a Protobuffer definition is created for you in the `proto` directory.

当使用type命令时，将在proto目录中为您创建一个Protobuffer定义。

It contains messages for full CRUD (Create, Read, Update, Delete) operations for your created transaction type.

它包含针对您创建的事务类型的完整CRUD(创建、读取、更新、删除)操作的消息。

###### Module

Once you have created your starport blockchain application, you will have your own module resident of `yourapp/x/yourmodule`, it comes predefined with a couple of files and folders which define types, functions and messages of your module.

一旦你创建了你的starport区块链应用程序，你就会在`yourapp/x/yourmodule`中拥有自己的模块，它预先定义了一些文件和文件夹，这些文件和文件夹定义了你的模块的类型、函数和消息。

###### Types

The `types` folder defines structures of your golang blockchain application. Here you can define your basic or more advanced types which will later be data and functions usable on your blockchain.

The message types are defined in the file `types/messages_type`, or other functions that you are planning to use.

`types`**文件夹定义了golang区块链应用程序的结构**。在这里，你可以定义你的基本类型或更高级的类型，这些类型以后会成为区块链上可用的数据和函数。

消息类型message types在文件` types/messages_type`中定义，或者在计划使用的其他函数中定义。

###### Client

There is a `rest` folder that takes care of the rest API that your module exposes.

The `cli` folder with the contents take care of the Command line interface commands that will be available to the user.

有一个rest文件夹负责模块**公开的rest API**

包含内容的命令行文件夹负责**用户可以使用的命令行接口命令。**

###### Frontend

Currently starport provides a basic Vue User-Interface that you can get inspired by or build ontop on. The source code is available in the `vue` folder. Written in JavaScript you can hop directly into writing the frontend for your application.

目前starport提供了一个基本的Vue用户界面，你可以从中得到启发，也可以在此基础上进行构建。源代码可以在vue文件夹中找到。用JavaScript编写，您可以直接跳到编写应用程序的前端。

[Learn more about Vue.](https://vuejs.org/)

##### Summary

- Starport bootstraps a module for you.
- You can change a module by modifying the files in `yourapp/x/` or the `proto` diretory.
- Starport has a Vue frontend where you can start to work immediately.

#### config.yaml配置

With Starport your blockchain can be configured with `config.yml`.

##### `accounts`

A list of user accounts created during genesis of your application.

| Key     | Required | Type            | Description                                       |
| ------- | -------- | --------------- | ------------------------------------------------- |
| name    | Y        | String          | Local name of a key pair                          |
| coins   | Y        | List of Strings | Initial coins with denominations (e.g. "100coin") |
| address | N        | String          | Address of the account in bech32                  |

Example

```yaml
accounts:
  - name: alice
    coins: ["1000token", "100000000stake"]
  - name: bob
    coins: ["500token"]
    address: cosmos1adn9gxjmrc3hrsdx5zpc9sj2ra7kgqkmphf8yw
```

##### `build`

| Key    | Required | Type   | Description                                                  |
| ------ | -------- | ------ | ------------------------------------------------------------ |
| binary | N        | String | Name of the node binary that will be built and used by Starport。 **Starport将构建和使用的节点二进制文件的名称** |

Example

```
build:
  binary: "mychaind"
```

##### `build.proto`

| Key               | Required | Type            | Description                                                  |
| ----------------- | -------- | --------------- | ------------------------------------------------------------ |
| path              | N        | String          | Path to protocol buffer files (default: `"proto"`)           |
| third_party_paths | N        | List of Strings | Path to thid-party protocol buffer files (default: `["third_party/proto", "proto_vendor"]`) |

##### `faucet`

A faucet service that **sends tokens to addresses**. Web UI is available by default on [http://localhost:4500](http://localhost:4500/).

**给用户拿初始货币的水龙头服务**

| Key       | Required | Type            | Description                                                  |
| --------- | -------- | --------------- | ------------------------------------------------------------ |
| name      | Y        | String          | Name of a key pair. `name` must be in `accounts`             |
| coins     | Y        | List of Strings | Coins with denominations sent per request  **每次给节点对应需要的coins值** |
| coins_max | N        | List of Strings | Maximum amount of tokens sent per address  **每个节点所要的token最大值** |
| port      | N        | Integer         | Port number (default: `4500`)。端口号                        |

Example

```
faucet:
  name: faucet
  coins: ["100token", "5foo"]
  coins_max: ["2000token", "1000foo"]
  port: 4500
```

##### `validator`

A blockchain has to **have at least one validator-node.** `validator` specifies the account that will be used to initialize the validator and parameters of the validator.

| Key    | Required | Type   | Description                                                  |
| ------ | -------- | ------ | ------------------------------------------------------------ |
| name   | Y        | String | Name of a key pair. `name` must be in `accounts`             |
| staked | Y        | String | Amount of coins to bond质押. Must be less or equal to the amount of coins account has |

Example

```
accounts:
  - name: alice
    coins: ["1000token", "100000000stake"]
validator:
  name: user1
  staked: "100000000stake"
```

##### `init.home`

A blockchain stores data and configuration in a data directory. This property specifies a path to the data directory.

**区块存储数据位置配置**

Example

```
init:
  home: "~/.myblockchain"
```

##### `init.config`

Overwrites properties in `config/config.toml` in the data directory.

**重写`config/config.toml`的参数**

##### `init.app`

Overwrites properties in `config/app.toml` in the data directory.

**重写`config/app.toml`的参数**

##### `init.keyring-backend`

Specifies a [keyring backend](https://docs.cosmos.network/master/run-node/keyring.html).

**初始化钥匙串**

钥匙圈具有用于与节点交互的私钥/公共密钥调配。例如，**在运行区块链节点之前需要设置验证器密钥，以便正确签名块。**私钥可以存储在不同的位置，称为"后端"，例如文件或操作系统自己的密钥存储。

从 v0.38.0 版本开始，Cosmos SDK 附带了一个新的钥匙圈实现，它提供了一组命令，以安全的方式管理加密密钥。新的钥匙圈支持多个存储后端，其中一些可能不适用于所有操作系统。

Example

```
init:
  keyring-backend: "os"
```

##### `genesis`

Overwrites properties in `config.genesis.json` in the data directory.

重写 `config.genesis.json` 的所有参数

Example

```
genesis:
  chain-id: "foobar"
```

#### starport

##### 项目架构

Scaffolding a project using Starport is done with the `starport app` command.

Currently, there are two versions of projects that can be scaffolded, which include a Launchpad version, or a Stargate version. The version can be specified by passing the `sdk-version` flag, followed by either `stargate` or `launchpad`.

目前，有两个版本的项目可以搭建，其中包括一个发射台版本，或一个星际之门版本。版本可以通过传递sdk-version标志来指定，然后是stargate或launchpad。

ie.

```
starport app github.com/user/app --sdk-version=stargate
```

==**SDK版本:**==

**Scaffolding a Stargate app currently uses version `^0.40` of the Cosmos SDK. ==星际之门使用0.40的cosmos版本==**

**Scaffolding a Launchpad app is currently the default that is being used by Starport, and uses version `0.39.x` of the Cosmos SDK.==发射台使用0.39.x版本==**

具体的架构是不同的, **开发注意版本**, 具体架构见:https://github.com/tendermint/starport/blob/develop/docs/3%20Starport/2%20Project%20Scaffolding.md

##### Type架构

You can scaffold types within Starport by running a command:

```
starport type [type-name] [field1:type1] [field2:type2] ...
```

具体文件架构见:https://github.com/tendermint/starport/blob/develop/docs/3%20Starport/3%20Type%20Scaffolding.md

