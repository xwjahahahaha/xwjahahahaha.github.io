---
title: swarm-XDai主网免bzz质押和rpc全套搭建教程-windows
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-06-22 10:58:10
---

> 本教程的最终效果是： 搭建一个Swarm主网节点，参与Swarm项目
>
> 文章不构成任何购买建议，请自行负责

# 一、清楚一些事情

* Swarm主网上线是在以太坊的POA侧链XDAI链上运行的，为了避免主网拥堵以及高额的gas

  <font color='#e54d42'>**所以之前空头一些相关Goerli测试网络的配置都不在适用**</font>

* XDAI链的主链币是XDAI，Bzz是其中的合约代币，类比理解：

  | 区块链网络/链 | 以太坊主链 | Goerli测试链（空头） | 侧链XDAI（正式） |
  | ------------- | ---------- | -------------------- | ---------------- |
  | 主链币        | ETH        | gETH                 | XDAI             |
  | 合约代币      | BZZ        | gBZZ                 | xBZZ             |

  *  主链币是一条区块链的核心币，由交易转移、共识一致，数量依托整个区块链系统结构设计。交易的Gas费就是主链币
  * 合约代币是区块链运行的合约逻辑中存储的一串数字，对，就是数字（虽然主链币也是数字），数量变化依托合约逻辑结构设计。

* Bee客户端是Swarm项目的客户端，在本地运行，**很多配置可以通过配置文件修改运行**。

* 整体结构大约如下所示：

  （当然，现实不可能以太坊节点与Swarm节点完全分离，很可能有错综复杂的重叠）

  ![nERnYe](http://xwjpics.gumptlu.work/qinniu_uPic/nERnYe.png)

<!-- more -->

# 二、RPC访问配置

## 2.1 使用GetBlock（推荐）

类似于之前测试网的https://infura.io，[Getblock](https://getblock.io/)也是一个提供在线PRC服务的网站，不用自建RPC

使用方法：

![305O4A](http://xwjpics.gumptlu.work/qinniu_uPic/305O4A.png)

输入邮箱，名称注册

打开邮箱会看到注册的详细信息网站

注册成功后，进入控制台如图：

![z6hCnz](http://xwjpics.gumptlu.work/qinniu_uPic/z6hCnz.png)

就可以拿到你的API密钥了

最终你的的Swap-endpoint配置就是:

```yaml
swap-endpoint: https://stake.getblock.io/mainnet/?api_key=这里换成你的API密钥
```

## 2.2 搭建自己的RPC基站

玩过测试网的都知道，之前Swarm测试网(Goerli测试网络)用的RPC基站即Swap-endpoint是在https://infura.io上注册的，每天限制免费10万次请求，现在不用这个了。

为了让我们的Bee客户端能够通过rpc请求获取到XDAI的链上数据并且没有请求次数的限制，我们可以运行一个自己的XDAI网络RPC基站

### 2.2.1 下载工具Nethermind

Nethermind是以太坊客户端快速同步节点搭建工具，支持很多种测试网络，网址如下：

https://docs.nethermind.io/nethermind/

进入Download：

https://docs.nethermind.io/nethermind/ethereum-client/download-sources

下载链接：

https://downloads.nethermind.io/

选择你的电脑操作系统版本

![UvbZCE](http://xwjpics.gumptlu.work/qinniu_uPic/UvbZCE.png)

下载解压后如图：

![zQNNQG](http://xwjpics.gumptlu.work/qinniu_uPic/zQNNQG.png)

### 2.2.2 设置WebSocket为启动

打开配置文件夹configs找到xdai.cfg, 用记事本打开

![TwbDti](http://xwjpics.gumptlu.work/qinniu_uPic/TwbDti.png)

修改webSocket为启用：

![74OoV0](http://xwjpics.gumptlu.work/qinniu_uPic/74OoV0.png)

保存退出

### 2.2.3 启动节点，开始同步

双击Nethermind.Launcher.exe启动，上下左右移动选择，回车确定

![g5NZt6](http://xwjpics.gumptlu.work/qinniu_uPic/g5NZt6.png)

选择以太坊节点

![YBPIkb](http://xwjpics.gumptlu.work/qinniu_uPic/YBPIkb.png)

选择XDAI测试链

![JbOVik](http://xwjpics.gumptlu.work/qinniu_uPic/JbOVik.png)

选择快速同步模式

![x1syFS](http://xwjpics.gumptlu.work/qinniu_uPic/x1syFS.png)

接下来的一连串配置：

一般本地使用就是127.0.0.1, 服务器的话就用服务器的IP

![IQsPU0](http://xwjpics.gumptlu.work/qinniu_uPic/IQsPU0.png)

启动后等待同步，同步的时间**很长。。。。耐心等待**

默认就是8546端口，符合bee客户端swap-endpoint的默认配置，所以不需要改配置

打开浏览器访问`localhost:8546`出现`Nethermind JSON RPC`即可

最后你的Swap-endpoint应该是：（也就是默认的配置）

```yaml
swap-endpoint: ws://localhost:8546
```

# 三、获取XDAI

部署支票簿合约就需要发起交易，发起交易就需要交易费，在XDAI链上，交易费就是XDAI币

XDAI是稳定币，目前交易所的价格等同与USDT**大约一美元**，所以**如果找别人买最好看清楚给的价格**

购买之前，先配置一下MetaMask小狐狸钱包

## 3.1 MetaMask小狐狸钱包的配置

打开设置：

![U9vUzt](http://xwjpics.gumptlu.work/qinniu_uPic/U9vUzt.png)

下拉找到网络 => 添加网络

XDAI侧链网络配置如下：

![P0pxtb](http://xwjpics.gumptlu.work/qinniu_uPic/P0pxtb.png)

网络名词：xDAI Chain

新增RPC URL：https://rpc.xdaichain.com(注意标点符号的半角和全角，需要半角)

链ID：100

符号：xDAI

区块浏览器：https://blockscout.com/xdai/mainnet/

---

## 3.2 添加XDAI链的xBzz

添加代币 => 代币合约地址

地址：0xdBF3Ea6F5beE45c02255B2c26a16F300502F68da

## 3.3 交易所购买

https://m.ascendex.com/register?inviteCode=UHCVPQPWA

点市场——右上角搜索——xDAI，就可以看到

如果从其他交易所提U进来，推荐走TRC20链。

由于erc20的手续费较高，提币可以选择TRC20，并且提币和充币，都要选择TRC20，切记一一对应，别搞错了。

购买后将你的XDAI转移到你的Bee账户中

# 四、配置1.0版本Bee

找一个空文件夹，在官网下载：

全部下载地址： https://github.com/ethersphere/bee/releases/tag/v1.0.0

windows地址：https://github.com/ethersphere/bee/releases/download/v1.0.0/bee-windows-amd64.exe

下载exe放入空文件夹

进入空文件夹创建一个文件, 文件全称：`bee.yaml` 注意拓展名是Yaml，如果没有设置显示拓展名，具体的windows显示文件拓展名见：

https://jingyan.baidu.com/article/a3a3f811154df38da3eb8a51.html

## 4.1 零质押与Gas费的配置

### 1. 零质押配置

如果追求最小成本，可以将质押的初始Bzz设置为0

在`bee.yaml`文件中编辑

`swap-initial-deposit: "0"`

### 2. 提高Gas费配置

为了加快支票薄合约的部署速度，也就是运行Bee的等待交易上链时间，可以提高Gas费也就是XDAI

配置如下：

`swap-deployment-gas-price: "修改你想要的gas费，也不要太大"`

### 3. 最终yaml整体配置

```yaml
api-addr: :1633																																			# 端口可自行修改
cache-capacity: "1000000"
config: .\.bee.yaml
data-dir: .\.bee																																		# 所有数据文件都在当前文件夹，如果有问题整体删除即可
debug-api-addr: :1635
debug-api-enable: true
full-node: true
mainnet: true
network-id: "1"
network-id: "100"
p2p-addr: :1634
password: "xxxxxxx"
swap-deployment-gas-price: "999999"																									# 自行修改交易费
swap-enable: true
swap-endpoint: https://stake.getblock.io/mainnet/?api_key=xxxxxxxxxxxxxxxxxxx				# 修改成你的api keys
swap-initial-deposit: "0"																														# 自行修改质押
```

### 4. 单机多节点

这里的多节点是**多端口**，**多个控制台启动**

端口避免重复例如：

```yaml
# 第一台节点
api-addr: :1633		
debug-api-addr: :1635
debug-api-enable: true
p2p-addr: :1634

# 第二台节点
api-addr: :1643		
debug-api-addr: :1645
debug-api-enable: true
p2p-addr: :1644
```

多节点就是创建多个文件夹，多个cmd，多次启动

# 五、启动Bee_1.0开始工作

在文件夹下启动cmd，输入如下命令启动：

`bee-windows-amd64.exe start --config bee.yaml`

![phoBCw](http://xwjpics.gumptlu.work/qinniu_uPic/phoBCw.png)

# 六、导出私钥

## 6.1 不使用clef（推荐）

教程不使用bee-clef，使用Bee默认会启动创建的一对公私钥，所以要将私钥导出来，加载到例如MetaMask钱包中

https://github.com/jmozah/exportSwarmKey （要自行用go环境编译）

windows版本的编译完成main文件：

链接: https://pan.baidu.com/s/1EEEwxJ70ZrcUSbTS3AMHOQ  密码: tb09  （博主不负任何责任，自行考良）

在文件夹下运行：

`exportKeys_windows.exe .bee/keys/ 你的密码(配置文件中配置)`

**显示的第三个`.bee\keys\swarm_key`中的私钥就是目标私钥**

## 6.2 使用bee-clef

使用clef在上面的配置文件加上配置：（具体配置方法见其他文章）

```yaml
clef-signer-enable: true
clef-signer-endpoint: ""									
clef-signer-ethereum-address: ""
```

将clef账户导入到MetaMask中

`cd /var/lib/bee-clef`

文件夹下:

![82lRk0](http://xwjpics.gumptlu.work/qinniu_uPic/82lRk0.png)

或者终端运行`bee-clef-keys`

![eydkFy](http://xwjpics.gumptlu.work/qinniu_uPic/eydkFy.png)

会自动导入到主目录下,txt中就是密码

打开MetaMask中导入:

![719qCf](http://xwjpics.gumptlu.work/qinniu_uPic/719qCf.png)

![fPc6RB](http://xwjpics.gumptlu.work/qinniu_uPic/fPc6RB.png)

# 七、其他

## XDAI浏览器地址

https://blockscout.com/xdai/mainnet/

