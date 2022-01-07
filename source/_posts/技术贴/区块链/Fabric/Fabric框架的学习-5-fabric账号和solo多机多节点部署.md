---

title: Fabric框架的学习-5-fabric账号和solo多机多节点部署
tags:
  - fabric
categories:
  - technical
  - fabric
toc: true
declare: true
date: 2020-10-14 16:53:32
---

# 八、Fabric账号

## 8.1 Fabric账号

* 账号：<font color='orange'>Fabric中的账号实际上是根据PKI规范生成的一组证书和密钥文件</font>

* 账号的作用：
  * 保证记录在区块链中的数据不可篡改、不可逆
  * Fabric中每条交易都会加上发起者的标签（签名证书），同时用发起人的私钥进行加密
  * 如果交易需要其他组织的节点提供背书，那么背书节点也会在交易中加入自己的签名

<!-- more -->

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014170752.png)

需要找谁/哪个组织的证书，就去改目录下找其MSP文件夹即可！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014171029.png)

## 8.2 账号的使用场景

* <font color='cornflowerblue'>启动orderer</font>

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014171418.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014171543.png)

* <font color='cornflowerblue'>启动peer</font>

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014171714.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014172157.png)

  直接就把msp文件挂载在容器中了

* <font color='cornflowerblue'>创建channel</font>

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014172253.png)

  **创建通道是在客户端完成的，并且必须是Admin创建**

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014172507.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014172908.png)



## 8.3 Fabric-ca

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014203833.png)

当需要动态的添加用户的时候，使用cryptogen就过于繁琐了，Fabric提供了fabric-ca机制

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014204223.png)



可以通过fabric-ca-client连接fabric-ca注册账号，但是弊端是使用命令行的方式，这对用户来说是难以接受的。

官方提供了已经写好的二进制可执行文件供我们访问：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014204355.png)

一般我们会通过一些sdk调用

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014204450.png)

通过sdk编写客户端实现： 

* <font color='green'>连接Fabric—ca服务，创建账户</font>
* <font color='green'>访问peer节点查询数据</font>

<font color='red'>以此来代替之前配置的cli容器</font>

---

在一个fabric网络中有多个组织，那么fabric-ca该如何部署？

* 在每个组织中部署一个<font color='red'>Fabric-ca</font>，这样创建出来的用户可以访问整个组织中的所有peer节点

---

从图中还可以看出，fabric-ca服务器还可以设置代理，使用一些关系型的数据库。

## 8.4 将fabric-ca加入到网络

进入`fabric-samples/basic-network`文件夹中：

```shell
vim docker-compose.yml
# 官方实例中配置了有关fabric-ca的一些相关配置
```

```yaml
 12	ca.example.com: 						# fabric-ca的服务名，自定义
 13     image: hyperledger/fabric-ca  		# 依赖的镜像
 14     environment:
 15       - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server	# 不需要改，容器中的home目录
 16       - FABRIC_CA_SERVER_CA_NAME=ca.example.com				# 服务器名字，自定义
 		  # fabric-ca服务器证书文件目录中的证书文件
 		  # 要明确当前的fabric-ca属于哪个组织！！
 17       - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.example.com-cert.pem
 		  # 私钥文件
 		  # 这两个文件的路径只需要写文件名即可，因为下方的数据卷映射
 18       - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/4239aa0dcd76daeeb8ba0cda701851d14504d31aad1b2ddddbac6a57365e497c_sk
 19     ports:
 20       - "7054:7054"		# fabric-ca绑定的端口，不改
 		#启动命令， admin:adminpw前面是用户名后面是密码
 21     command: sh -c 'fabric-ca-server start -b admin:adminpw'
 22     volumes:
 		# 修改自己的路径，实现数据卷
 23       - ./crypto-config/peerOrganizations/org1.example.com/ca/:/etc/hyperledger/fabric-ca-server-config
 24     container_name: ca.example.com	# 容器名，自己指定
 25     networks:
 26       - basic			# 工作的网络
 27
```

修改：

分别给两个组织加上fabric-ca服务器

```yaml
cppca.xwj.com:
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=cppca.xwj.com
      - FABRIC_CA_SERVER_CA_CERTFILE=ca.orgcpp.xwj.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=c2568c1f148e548dc09eadf76e351a466df3ae8ab18dadba132cf6f1809a2dbc_sk
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start -b admin:adminpw'
    volumes:
      - ./crypto-config/peerOrganizations/orgcpp.xwj.com/ca/:/etc/hyperledger/fabric-ca-server-config
    container_name: cppca.xwj.com
    networks:
      - byfn

  goca.xwj.com:
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=goca.xwj.com
      - FABRIC_CA_SERVER_CA_CERTFILE=ca.orggo.xwj.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=b44962fdc3416c9342bba9a13ca91d40876398740f68031a0be77191c0d7a0b1_sk
    ports:
      - "8054:7054"		# 端口映射前面的是宿主机端口，不能重复
    command: sh -c 'fabric-ca-server start -b admin:adminpw'
    volumes:
      - ./crypto-config/peerOrganizations/orggo.xwj.com/ca/:/etc/hyperledger/fabric-ca-server-config
    container_name: goca.xwj.com
    networks:
      - byfn
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014213005.png)

这样显示就是启动成功

启动成功后重新创建网络，进入客户端.....等一系列操作

## 8.5 官方客户端fabric-ca-client的使用

在bin目录下有一个命令fabric-ca-client时官方给用户注册账户提供的工具

```shell
Hyperledger Fabric Certificate Authority Client

Usage:
  fabric-ca-client [command]

Available Commands:
  affiliation Manage affiliations
  enroll      Enroll an identity
  gencrl      Generate a CRL
  gencsr      Generate a CSR
  getcacert   Get CA certificate chain
  identity    Manage identities
  reenroll    Reenroll an identity
  register    Register an identity
  revoke      Revoke an identity
  version     Prints Fabric CA Client version

Flags:
      --caname string                  Name of CA
      --csr.cn string                  The common name field of the certificate signing request
      --csr.hosts stringSlice          A list of space-separated host names in a certificate signing request
      --csr.names stringSlice          A list of comma-separated CSR names of the form <name>=<value> (e.g. C=CA,O=Org1)
      --csr.serialnumber string        The serial number in a certificate signing request
  -d, --debug                          Enable debug level logging
      --enrollment.attrs stringSlice   A list of comma-separated attribute requests of the form <name>[:opt] (e.g. foo,bar:opt)
      --enrollment.label string        Label to use in HSM operations
      --enrollment.profile string      Name of the signing profile to use in issuing the certificate
  -H, --home string                    Client's home directory (default "$HOME/.fabric-ca-client")
      --id.affiliation string          The identity's affiliation
      --id.attrs stringSlice           A list of comma-separated attributes of the form <name>=<value> (e.g. foo=foo1,bar=bar1)
      --id.maxenrollments int          The maximum number of times the secret can be reused to enroll (default CA's Max Enrollment)
      --id.name string                 Unique name of the identity
      --id.secret string               The enrollment secret for the identity being registered
      --id.type string                 Type of identity being registered (e.g. 'peer, app, user') (default "client")
  -M, --mspdir string                  Membership Service Provider directory (default "msp")
  -m, --myhost string                  Hostname to include in the certificate signing request during enrollment (default "$HOSTNAME")
  -a, --revoke.aki string              AKI (Authority Key Identifier) of the certificate to be revoked
  -e, --revoke.name string             Identity whose certificates should be revoked
  -r, --revoke.reason string           Reason for revocation
  -s, --revoke.serial string           Serial number of the certificate to be revoked
      --tls.certfiles stringSlice      A list of comma-separated PEM-encoded trusted certificate files (e.g. root1.pem,root2.pem)
      --tls.client.certfile string     PEM-encoded certificate file when mutual authenticate is enabled
      --tls.client.keyfile string      PEM-encoded key file when mutual authentication is enabled
  -u, --url string                     URL of fabric-ca-server (default "http://localhost:7054")

Use "fabric-ca-client [command] --help" for more information about a command.
```

注册一个管理员账号：

`fabric-ca-client enroll -u http://admin:adminpw@192.168.1.90:8054`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201015122409.png)

会在用户目录下生成.fabric-ca-client的文件夹其中就包含了msp

# 九、Solo共识下多机多节点部署

---

所有的节点分离部署，每台主机上有一个节点

|  名称   |    ip     |         HostName          | 虚拟网端口映射 |  组织机构   |
| :-----: | :-------: | :-----------------------: | :------------: | :---------: |
| orderer | 10.0.2.5  |    orderer.example.com    | 10.0.2.5:2222  |   orderer   |
|  peer0  | 10.0.2.9  | peer0.orgbmw.example.com  | 10.0.2.9:3333  | OrgBmw宝马  |
|  peer1  |           | peer1.orgbmw.example.com  |                |    宝马     |
|  peer0  | 10.0.2.10 | peer0.orgbenz.example.com | 10.0.2.10:4444 | OrgBenz奔驰 |
|  peer1  |           | peer0.orgbenz.example.com |                |    奔驰     |

没有多台实体主机，那么可以采用虚拟集群来实现。具体可以看我写的这一篇：

https://blog.csdn.net/weixin_43988498/article/details/109159785

我使用虚拟集群来体验测试多机多节点部署

## 9.1 准备工作

n台主机需要创建一个<u>名字相同</u>的工作目录，<font color='orange'>为的是能够连接上同一个网络</font>

```shell
# 10.0.2.5
mkdir ~/carfabric
# 10.0.2.6
mkdir ~/carfabric 
# 10.0.2.7
mkdir ~/carfabric
```

```shell
# 生成证书模板
cryptogen showtemplate > crypto-config.yaml
# 修改配置
vim crypto-config.yaml

# 生成证书
cryptogen generate --config=crypto-config.yaml

# 生成通道文件、创世块
cp ~/fabric-1.2/fabric-samples/first-network/configtx.yaml .  # 复制一份
# 修改配置文件
vim configtx.yaml
# 生成创世快
configtxgen -profile CarOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
# 生成通道文件
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID carchannel
```

## 9.2 不同节点不同的配置文件

### 9.2.1 部署orderer排序节点  主机： 10.0.2.5

编写docker-compose文件：

```yaml
version: '2'

services:

  orderer.example.com:
    container_name: orderer.example.com
    image: hyperledger/fabric-orderer:latest
    environment:
      - ORDERER_GENERAL_LOGLEVEL=INFO
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls
    ports:
      - 7050:7050
	networks:
	  default:
	    aliases:
	      - carFabric
```

**不在继承，在继承的文件中摘出来，写到一个文件**

注意networks的写法，表示使用默认的网络，所有节点加入到这个默认的网络中，aliases起别名，这里的网络名字不要瞎写，<font color='red'>要写工作文件的目录名</font>

网络同名，这样多个节点才能互相访问！

启动节点：`docker-compose -f docker-composre-orderer.yaml up -d`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201118173847.png)

### 9.2.2 部署peer0.orgbmw.com节点  10.0.2.9  10.0.2.10

分别进入这两个主机，做以下两步:

1. 拷贝 channel-artifacts 、 crypto-config两个文件到这两个主机
2. 编写yaml文件

示例：peer0.orgbenz.com的配置，包含peer节点、cli以及ca节点

```yaml
version: '2'

services:
  peer0.orgbenz.example.com:
    container_name: peer0.orgbenz.example.com
    image: hyperledger/fabric-peer:latest
    environment:
      - CORE_PEER_ID=peer0.orgbenz.example.com
      - CORE_PEER_ADDRESS=peer0.orgbenz.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.orgbenz.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.orgbenz.example.com:7051
      - CORE_PEER_LOCALMSPID=OrgBmwMSP
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=carfabric_default
      - CORE_LOGGING_LEVEL=INFO
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    volumes:
      - /var/run/:/host/var/run/
      - ./crypto-config/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/msp:/etc/hyperledger/fabric/msp
      - ./crypto-config/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls:/etc/hyperledger/fabric/tls
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 7051:7051
      - 7053:7053
    networks:
      default:
        aliases:
          - carFabric
    extra_hosts:
      - "orderer.example.com:10.0.2.5"
      - "peer0.orgbmw.example.com:10.0.2.9"

# 配置一个客户端
  cli:
    container_name: car-cli
    image: hyperledger/fabric-tools:latest
    tty: true
    stdin_open: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=INFO
      - CORE_PEER_ID=car-cli
      - CORE_PEER_ADDRESS=peer0.orgbenz.example.com:7051
      - CORE_PEER_LOCALMSPID=OrgBmwMSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls/ca.crt
     -CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/users/Admin@orgbenz.example.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
      - /var/run/:/host/var/run/
      - ./chaincode/:/opt/gopath/src/github.com/chaincode
      - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
      - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
      - ./deploy.sh:/opt/gopath/src/github.com/hyperledger/fabric/peer/deploy.sh
    depends_on:
      - peer0.orgbenz.example.com
    extra_hosts:
      - "orderer.example.com:10.0.2.5"
      - "peer0.orgbmw.example.com:10.0.2.9"
      - "peer0.orgbenz.example.com:10.0.2.10"
    networks:
      default:
        aliases:
          - carFabric
#配置ca节点
  benzca.example.com:
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=benzca.example.com
      - FABRIC_CA_SERVER_CA_CERTFILE=ca.orgbenz.example.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=2b78578a95a083fb4a820dd60a300b62bb2944b3b404e8ce727d6e2a880300c8_sk
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start -b admin:adminpw'
    volumes:
      - ./crypto-config/peerOrganizations/orgbenz.example.com/ca/:/etc/hyperledger/fabric-ca-server-config
    container_name: benzca.example.com
    networks:
      default:
        aliases:
          - carfabric

```

配置完了之后，使用docker-compose启动

### 9.2.3 后续操作

我们使用官方的fabcar链码部署整个链码

#### 1.将准备好的链码放到chaincode文件夹中

```shell
//使用示例的链码
sudo mv ../fabric-1.2/fabric-samples/chaincode/fabcar/go/fabcar.go ./chaincode/
sudo chmod u+x chaincode/fabcar.go
```

#### 2.启动peer节点的容器后，进入客户端节点

#### 3.创建通道

```shell
peer channel create -o orderer.example.com:7050 -c testchannel -f ./channel-artifacts/channel.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

创建完通道文件后生成对应的channelID.block文件，首先通过把其放到channel-artifacts文件夹下（数据卷映射，宿主机就可以直接拿到了），再通过scp命令拷贝到其他要加入到通道中的peer节点

```shell
mv testchannel.block channel-artifacts/
scp channel-artifacts/testchannel.block vagrant@10.0.2.10:/home/vagrant/carfabric/channel-artifacts
```

#### 4.将当前节点分别都加入到通道中

```shell
peer channel join -b channel-artifacts/testchannel.block
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201118203049.png)

#### 5.安装链码

```shell
peer chaincode install -n carsys -v 1.0 -l golang -p github.com/chaincode/
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201118204311.png)

#### 6.初始化链码

```shell
peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C testchannel -n carsys -l golang -v 1.0 -c '{"Args":["init"]}' -P "AND ('OrgBmwMSP.member', 'OrgBenzMSP.member')" --connTimeout 30s
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119090558.png)

<font color='green'>没有任何返回值，就代表成功！</font>

#### 7.使用合约

##### 初始化汽车调取invoke函数

调用链码上的此函数，初始化10辆车

```go
func (s *SmartContract) initLedger(APIstub shim.ChaincodeStubInterface) sc.Response {
	cars := []Car{
		Car{Make: "Toyota", Model: "Prius", Colour: "blue", Owner: "Tomoko"},
		Car{Make: "Ford", Model: "Mustang", Colour: "red", Owner: "Brad"},
		Car{Make: "Hyundai", Model: "Tucson", Colour: "green", Owner: "Jin Soo"},
		Car{Make: "Volkswagen", Model: "Passat", Colour: "yellow", Owner: "Max"},
		Car{Make: "Tesla", Model: "S", Colour: "black", Owner: "Adriana"},
		Car{Make: "Peugeot", Model: "205", Colour: "purple", Owner: "Michel"},
		Car{Make: "Chery", Model: "S22L", Colour: "white", Owner: "Aarav"},
		Car{Make: "Fiat", Model: "Punto", Colour: "violet", Owner: "Pari"},
		Car{Make: "Tata", Model: "Nano", Colour: "indigo", Owner: "Valeria"},
		Car{Make: "Holden", Model: "Barina", Colour: "brown", Owner: "Shotaro"},
	}

	i := 0
	for i < len(cars) {
		fmt.Println("i is ", i)
		carAsBytes, _ := json.Marshal(cars[i])
		APIstub.PutState("CAR"+strconv.Itoa(i), carAsBytes)
		fmt.Println("Added", cars[i])
		i = i + 1
	}

	return shim.Success(nil)
}
```

调用汽车初始化函数：

```shell
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C testchannel -n carsys --peerAddresses peer0.orgbmw.example.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbmw.example.com/peers/peer0.orgbmw.example.com/tls/ca.crt --peerAddresses peer0.orgbenz.example.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls/ca.crt -c '{"Args": ["initLedger"]}'
```

##### 查询初始化的车辆：

```shell
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C testchannel -n carsys --peerAddresses peer0.orgbmw.example.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbmw.example.com/peers/peer0.orgbmw.example.com/tls/ca.crt --peerAddresses peer0.orgbenz.example.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls/ca.crt -c '{"Args": ["queryAllCars"]}'
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119100826.png)

返回了所有的车辆,解析后的数据

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119105655.png)

##### 新建一辆车

链码中汽车的结构体：

```go
type Car struct {
	Make   string `json:"make"`
	Model  string `json:"model"`
	Colour string `json:"colour"`
	Owner  string `json:"owner"`
}
```

```shell
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C testchannel -n carsys --peerAddresses peer0.orgbmw.example.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbmw.example.com/peers/peer0.orgbmw.example.com/tls/ca.crt --peerAddresses peer0.orgbenz.example.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgbenz.example.com/peers/peer0.orgbenz.example.com/tls/ca.crt -c '{"Args": ["Invoke","createCar","BMW","THE2","yellow","xuwenjie"]}'
```















#### 8.后续升级链码

和安装链码一样，但是版本号要修改

```shell
peer chaincode install -n carsys -v 1.1 -l golang -p github.com/chaincode/
```

**升级链码：** 使用upgrade，<font color='red'>版本号要一致，链码名称不变，背书策略要与之前的一致</font>

```shell
peer chaincode upgrade -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C testchannel -n carsys -l golang -v 1.1 -c '{"Args":["init"]}' -P "AND ('OrgBmwMSP.member', 'OrgBenzMSP.member')" --connTimeout 30s
```

同样的没有提示就说明完成了

<font color='red'>fabric的链码都是在容器中运行的，退出客户端就可以查看到两个版本：</font>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119111205.png)





#### <font color='red'>9.不使用cli，外部调用此合约</font>

学自https://www.tuoluocaijing.cn/article/detail-45754.html

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119112746.png)

物理网络组成是：

1. **Fabric CA（ca.example.com）：**整个基本网络的证书颁发机构（CA）。我们稍后将使用此Fabric CA生成客户端认证。
2. **Orderer（orderer.example.com）：**基本网络中只有一个订购者。
3. **Peer（peer0.org1.example.com）：**基本网络中只有一个组织（Org1），Org1中只有一个对等体。

这几乎是规模最小的Fabric网络，足以测试业务区块链应用程序。
如果我们看一下docker compose yaml文件（docker composer.yml），它用于为网络提供节点（容器），我们会看到除上述三个容器之外定义的另外两个容器

- **CouchDB：**它被Peer用作世界状态数据库。
- **CLI：**此命令行界面（CLI）容器用于演示期间的链代码交互。它适用于链代码的演示和故障排除，**并且在生产中不需要，因为交互主要来自客户端应用程序。**























































## 9.3 注意事项

### 1. 通道重复问题

<font color='red'>Error: got unexpected status: BAD_REQUEST -- error applying config update to existing channel 'testchannel': error authorizing update: error validating ReadSet: proposed update requires that key [Group]  /Channel/Application be at version 0, but it is currently at version 1</font>

原因：旧的通道还没删除，新的通道重复创建

如果**提示通道重复**，可使用下面的命令重新开启容器：

docReload.sh 

```shell
docker-compose -f docker-composre-peer.yaml down -v
docker volume prune -f
docker-compose -f docker-composre-peer.yaml up -d
```

<font color='green'>注：</font> docker-composre-peer.yaml是你的docker-compose文件名，如果文件名是docker-compose，那么不需要-f参数，即`docker-compose up -d`

### 2. 初始化链码提示无法连接

<font color='red'>Error: error getting endorser client for instantiate: endorser client failed to connect to 0.0.0.0:7051: failed to create new connection: connection error: desc = "transport: error while dialing: dial tcp 0.0.0.0:7051: connect: connection refused"</font>

解决方法：链码初始化命令之前去除掉sudo

### 3. 初始化链码提示找不到网络

<font color='red'>Error: could not assemble transaction, err proposal response was not successful, error code 500, msg error starting container: error starting container: Failed to generate platform-specific docker build: Error executing build: API error (404): **network carFabric_default not found** </font>

我的网络名字是carfabric明显这里是错的。

解决办法：

1. 检查启动容器的时候是否有创建网络的提示,有提示才表明创建了网络

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119090142.png)

   如果没有的话尝试和通道重复一样的删除之前的数据卷，重新启动

   ```shell
   docker-compose -f docker-composre-peer.yaml down -v
   docker volume prune
   docker-compose -f docker-composre-peer.yaml up -d
   ```

2. 检查**各个主机的工作文件名与设置的网络名是否一致**，因为第一点生成的网络是根据当前文件名来的。

3. 检查orderer、peer配置文件

   orderer：

   ```yaml
   networks:
   	  default:
   	    aliases:
   	      - carFabric
   ```

   peer:    **CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE**参数

   ```yaml
   - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=carfabric_default
   
   networks:
   	  default:
   	    aliases:
   	      - carFabric
   ```

   























# Tips

## 1.System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down

问题原因:

我启动centos容器的命令是：

```
docker run -d --name centos_1 -it  centos:latest /bin/bash
```

需要修改为

```
docker run -tid --name centos_1 --privileged=true centos:latest /sbin/init
```

也就是<font color='orange'>加--privileged=true</font>，修改/binbash 为/sbin/init

修改后，就可以正常启动服务了

## 2. Docker容器centos或unbuntu无法使用 systemctl 命令解决方案

systemctl 命令时出现（System has not been booted with systemd as init system (PID 1). Can't operat....信息。

解决方案：<font color='red'>/sbin/init</font>

例如：Centos8

```shell
docker run -itd --name centos --privileged=true centos /sbin/init  # 使用这个命令
docker exec -it centos /bin/bash
```

注意:<font color='red'>--privileged=true</font>一定要加上的。

## 3. docker启动时出现Job for docker.service failed because the control process exited with error code错误

docker的engine基于Device Mapper提供的一种存储驱动,而它又依赖与 devicemapper，存储的数据目录在/var/lib/docker下。
首先进入docker存储数据目录。

```
cd /var/lib/docker
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190318152240846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NjUxMjQz,size_16,color_FFFFFF,t_70)
删除该目录下的所有文件夹/文件

```
rm -rf *
```

删除的时候，可能会出现设备或者资源忙的情况。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190318152523540.png)
解决办法
先用fuser显示出当前哪个程序在使用磁盘上的aufs
然后用umount卸载正在使用aufs的应用程序

```
fuser -m aufs/
fuser -k aufs/
umount aufs/
rm -rf aufs/
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019031815281273.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NjUxMjQz,size_16,color_FFFFFF,t_70)
删除以后就可以成功重启docker。

```
sudo systemctl restart docker
```