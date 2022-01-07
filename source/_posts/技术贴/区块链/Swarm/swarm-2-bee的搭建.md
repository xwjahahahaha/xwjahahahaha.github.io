---
title: swarm-2-bee的搭建
tags:
  - swarm
categories:
  - technical
  - swarm
toc: true
date: 2021-05-15 13:20:29
---

> 资料:
>
> https://docs.ethswarm.org/docs/installation/quick-start
>
> https://www.yuque.com/docs/share/712630dc-fa67-4318-a36e-502f871a0136?#
>
> 环境:
>
> * Ubuntu:  18.04.2 LTS

# 一、安装Bee

https://geth.ethereum.org/docs/clef/tutorial

## 1.1 Bee Clef

Go Ethereum’s Clef是以太坊的上的签名和密钥管理工具, 对于Swarm, 对应的工具就是Bee Clef, 能够方便的帮助我们签名大量交易与管理密钥.

### 1. 安装:

Ubuntu:

```shell
wget https://github.com/ethersphere/bee-clef/releases/download/v0.4.9/bee-clef_0.4.9_amd64.deb
sudo dpkg -i bee-clef_0.4.9_amd64.deb
```

centOS

```shell
wget https://github.com/ethersphere/bee-clef/releases/download/v0.4.9/bee-clef_0.4.9_amd64.rpm
sudo rpm -i bee-clef_0.4.9_amd64.rpm
```



[其他系统安装](https://docs.ethswarm.org/docs/installation/bee-clef/)

文件已经下载到`/etc/bee-clef`文件夹下,默认情况不需要修改配置

![uXe1eR](http://xwjpics.gumptlu.work/qinniu_uPic/uXe1eR.png)

### 2. 运行服务

```shell
systemctl status bee-clef
```

![wLaNsJ](http://xwjpics.gumptlu.work/qinniu_uPic/wLaNsJ.png)

<!-- more -->

持续输出log:

```shell
journalctl -f -u bee-clef.service
```

后面当连接到Bee后就会显示:

![Zi5Bk7](http://xwjpics.gumptlu.work/qinniu_uPic/Zi5Bk7.png)

### 3. 文件位置

Configuration files are stored in `/etc/bee-clef/`

Key material and other data is stored in `/var/lib/bee-clef/`

### 4. 账户列表

由于Bee需要Clef来自动签署许多事务，所以我们必须将Clef作为一种服务来运行，并且具有宽松的权限和规则集。

为了确保Clef只与Bee签署交易，我们必须保护`Clef.ipc`文件。通过创建一个Bee用户并设置权限，使得只有这个用户才能使用ipc套接字。

通过外部API来获取Clef中管理的账户列表:

```shell
echo '{"id": 1, "jsonrpc": "2.0", "method": "account_list"}' | nc -U /var/lib/bee-clef/clef.ipc
```

> 注意:  后面是ipc文件地址,默认在`/var/lib/bee-clef/`下

## 1.2 Bee

### 1.安装

Ubuntu

```shell
wget https://github.com/ethersphere/bee/releases/download/v0.5.3/bee_0.5.3_amd64.deb
sudo dpkg -i bee_0.5.3_amd64.deb
bee version 		# 检查版本
```

CentOS:

```shell
wget https://github.com/ethersphere/bee/releases/download/v0.5.3/bee_0.5.3_amd64.rpm
sudo rpm -i bee_0.5.3_amd64.rpm
```

### 2.默认启动

加权限再启动否则会报错:

```shell
sudo chown -R bee:bee /var/lib/bee
```

```shell
# 按/etc/bee/bee.yaml的配置启动
systemctl start bee 			# 启动
systemctl stop bee				# 停止
systemctl status bee			# 查看状态
```

```shell
# 手动启动
bee start
```

> <font color='#e54d42'>注意: 两种启动方式选择一种即可, 在使用按配置启动时如果不成功会反复自动尝试启动,可能会造成`bee start`的无法启动,所以使用第二种启动时需要先`systemctl stop bee `</font>

![ZOR0Wq](http://xwjpics.gumptlu.work/qinniu_uPic/ZOR0Wq.png)

输入密码, 这个密码用于保护你私钥以及可以代表你的Swarm地址

> Error: get chain id: Post "http://localhost:8545": dial tcp 127.0.0.1:8545: connect: connection refused
>
> 原因: 在默认配置下连接的是本地的8545端口测试网络, 如果本地没有区块链网络的话就会报错
>
> 解决: **后面会使用以太坊的Goerli测试网络,会在启动时加上参数或者自行修改配置文件**

**还需要一些配置和要求见下方**

### 3.Goerli测试网启动配置

**当第一运行Bee节点到测试网络上会在测试网络中借助支票工厂合约部署你的“支票”合约(支票簿), 支票用于链下核算(类似于微支付通道), 减轻链上压力提高交易效率.**

一旦部署了支票簿，Bee将在支票簿合同中存入一定数量的gBZZ (Goerli测试网上的BZZ代币)，这样它就可以为其他节点的服务支付报酬。

所以首先我们需要**能够连接上测试网络Goerli**

* 首先注册测试网依赖节点服务

  注册一个swap-endpoint地址：https://infura.io

  **第7分钟开始看,**注册视频教程：https://www.bilibili.com/video/BV1EV411Y7yM   

  拿到Goerli的的swap-endpoint地址：[https://goerli.infura.io/v3/](https://goerli.infura.io/v3/fe00e6f4a50b4a2fb2dc25ecb532a5ad)*********

  ![owFB2v](http://xwjpics.gumptlu.work/qinniu_uPic/owFB2v.png)

重新加参数启动:

```shell
bee start \
  --verbosity 5 \
  --swap-endpoint https://mainnet.infura.io/v3/xxxxxxxx \		# 依赖的地址
  --debug-api-enable
```

部署支票簿合约没有足够的资金(需要10gbzz)而warning:

![ZevDvK](http://xwjpics.gumptlu.work/qinniu_uPic/ZevDvK.png)

### 4.连接Clef配置

在获取代币之前还需要一步,就是连接自己创建的Bee_Clef账户管理工具, 使用Clef管理的账户而不使用自动创建的账户:

重新配置启动:

```shell
rm -rf ~/.bee								# 因为之前没有启用clef, 先删除掉之前的文件,需要重新设置密码
bee start --verbosity 5 --swap-endpoint https://goerli.infura.io/v3/xxxxxxxxxxxxxx --debug-api-enable --clef-signer-enable --clef-signer-endpoint /var/lib/bee-clef/clef.ipc 
```

![TM775e](http://xwjpics.gumptlu.work/qinniu_uPic/TM775e.png)

### 5.获取fund代币

部署支票簿合约需要一个基本的资金, 所以我们需要在测试网Goerli上通过其**水龙头合约**获取基本的gBzz代币

水龙头地址: [Swarm Goerli Faucet](https://faucet.ethswarm.org/). (现在很难拿到)

gETH水龙头地址: https://goerli-faucet.slock.it/

或者根据这篇文章中的方法通过发推特获取gEth(推荐): https://www.yundongfang.com/Yun41916.html

![TVsF6k](http://xwjpics.gumptlu.work/qinniu_uPic/TVsF6k.png)

### 6.clef钱包导入MetaMask

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

### 7. gETH转换gBZZ

通过点击在启动warning中的提示地址<font color='#e54d42'>**转换**一些gETH为gBZZ:</font>

![9IJcpA](http://xwjpics.gumptlu.work/qinniu_uPic/9IJcpA.png)

`https://bzz.ethswarm.org/?transaction=buy&amount=10&slippage=30&receiver=你的地址`

![vKC9a5](http://xwjpics.gumptlu.work/qinniu_uPic/vKC9a5.png)

成功后上测试网浏览器检查是否到账:

[https://goerli.etherscan.io/address/**你的地址**](https://goerli.etherscan.io/address/0xBEd09CeAe4517236A778c9D5D43DA3e80794012C)

把**加重的**那一段改成你的，然后进入看看。有没有币。

> <font color='#e54d42'>**注意: 发布发票合约最少需要10个gBZZ以及少量(0.01?)的gETH, 如果不够的话就多弄几次**</font>
>
> 在Metamsk中可以添加代币显示你的gBzz余额, 点击添加代币, <font color='#39b54a'>**gBZZ的地址为: 0x2ac3c1d3e24b45c6c310534bc2dd84b5ed576335**</font>

### 8. 修改配置文件启动

启动加很多参数过于麻烦, 这里将所有的配置修改到配置文件中,然后默认启动即可, 需要修改的几项配置文件如下(主要修改的就是`swap-endpoint`):

位置: `/etc/bee/bee.yaml`

```yaml
clef-signer-enable: true
clef-signer-endpoint: /var/lib/bee-clef/clef.ipc
config: /etc/bee/bee.yaml
data-dir: /var/lib/bee
debug-api-addr: 127.0.0.1:1635
debug-api-enable: true
password-file: /var/lib/bee/password
swap-enable: true
swap-endpoint: https://goerli.infura.io/v3/*************
```

按配置启动:

`systemctl start bee `

或者手动配置启动:

```shell
# 在当前项目目录下, 运行
vim mypsw.txt
# 输出密码保存
# 启动(替换你的依赖)
bee start --verbosity 5 --swap-endpoint https://goerli.infura.io/v3/xxxxxxxxx --debug-api-enable --clef-signer-enable --clef-signer-endpoint /var/lib/bee-clef/clef.ipc --password-file ./mypsw.txt --db-capacity 5000000
```

启动后就会自动消费gETH和gBZZ调用Goerli测试网上的工厂合约,部署<font color='#e54d42'>我们的支票簿合约</font>

![o8lVMA](http://xwjpics.gumptlu.work/qinniu_uPic/o8lVMA.png)

```shell
INFO[2021-05-16T13:39:55+08:00] no chequebook found, deploying new one.      
TRAC[2021-05-16T13:39:56+08:00] sending transaction 50491f8e17d91a8cd0dcecf39223f9162d8d89a0f5bca761ae037cd4d03d4181 with nonce 6 
INFO[2021-05-16T13:39:57+08:00] deploying new chequebook in transaction 50491f8e17d91a8cd0dcecf39223f9162d8d89a0f5bca761ae037cd4d03d4181 
```

> 支票簿工厂合约地址: 0xf0277caffea72734853b834afc9892461ea18474
>
> 可自行在https://goerli.etherscan.io/上搜索查看

### 9. 启动后测试

1. 查看bee日志

   `journalctl -u bee -f`
   查看bee-clef日志:

   `journalctl -u bee-clef -f`

2. 查看bee链接状态

   `curl http://localhost:1633`

   ![u3weRf](http://xwjpics.gumptlu.work/qinniu_uPic/u3weRf.png)

3. 查看链接对等节点数:

   `curl -s http://localhost:1635/peers | jq '.peers | length'`

   ![pEXxS2](http://xwjpics.gumptlu.work/qinniu_uPic/pEXxS2.png)

4. 查看自己钱包地址

   `curl -s localhost:1635/addresses | jq .ethereum`

   ![CWHDF5](http://xwjpics.gumptlu.work/qinniu_uPic/CWHDF5.png)

5. 查看支票合约账本地址

   `curl -s http://localhost:1635/chequebook/address | jq .chequebookaddress`

   ![KmK9N2](http://xwjpics.gumptlu.work/qinniu_uPic/KmK9N2.png)

### 10. 支票查看与提取

0. 查看区块欢迎度:

   `curl -X GET http://localhost:1645/topology | jq .population`

1. 下载cashout.sh脚本并赋予执行权限:

   ```shell
   wget -O cashout.sh https://gist.githubusercontent.com/ralph-pichler/3b5ccd7a5c5cd0500e6428752b37e975/raw/b40510f1172b96c21d6d20558ca1e70d26d625c4/cashout.sh && chmod +x cashout.sh
   ```

2. 修改提取阈值

   默认阈值太大,难以积攒,我们改小一点 `vim cashout.sh`

   修改第三行:

   ![Hrps0w](http://xwjpics.gumptlu.work/qinniu_uPic/Hrps0w.png)

   保存退出

3. 查看是否有支票

   * 自动

     `./cashout.sh`

     有支票就会有返回结果,没有的话就什么都不显示

   * 手动

     `curl localhost:1635/settlements | jq`

     余额：`curl localhost:1635/chequebook/balance | jq`

     支票：`curl localhost:1635/chequebook/cheque | jq`

     ![z1R0K6](http://xwjpics.gumptlu.work/qinniu_uPic/z1R0K6.png)

4. 提取支票

   * 自动

     `./cashout.sh cashout-all`

   * 手动

     ```shell
     curl -XPOST http://你的ip:1635/chequebook/cashout/peer地址
     ```

     会返回交易地址

     ![bCpkE0](http://xwjpics.gumptlu.work/qinniu_uPic/bCpkE0.png)

     成功后上测试网查看如下:

     ![4k3Ax3](http://xwjpics.gumptlu.work/qinniu_uPic/4k3Ax3.png)

5. 创建定时提取支票任务

   `crontab -e`

   输入3回车, 在文件中写入:

   `00 02 * * * [你的cashout.sh脚本目录] cashout-all`

   例如: `00 02 * * * /root/cashout.sh cashout-all`

   前面的02...是指每天凌晨两点执行cashout

6. 自动查看状态脚本:

   五秒查看一次:

   ```shell
   while true ;do
     echo "当前运行状态:"
     curl http://localhost:1633
     echo "连接数:"
     curl -s http://localhost:1635/peers | jq '.peers | length'
     echo "存储状态:"
     df -h | awk 'NR==4{print $3, $4, $5}NR==1{print $3, $4, $5}'
     echo "支票:"
     bash /root/projects/swarm_bee/cashout.sh
     echo "==================="
    12 sleep 10; done;
   ```

   ![o4vRt8](http://xwjpics.gumptlu.work/qinniu_uPic/o4vRt8.png)

### 11.文件位置

Configuration files are stored in `/etc/bee/`

State, chunks and other data is stored in `/var/lib/bee/`

## 1.3 windows运行Bee

https://www.yuque.com/docs/share/77e34e79-24ac-4cdf-adfa-3b0399d5242c?#

winodws导出私钥工具地址 :

https://github.com/jmozah/exportSwarmKey

## 1.4 k8s搭建

```shell
# 导入仓库
helm repo add ethersphere https://ethersphere.github.io/helm 
helm repo update

# 安装chart
helm install bee-test ethersphere/bee
```

