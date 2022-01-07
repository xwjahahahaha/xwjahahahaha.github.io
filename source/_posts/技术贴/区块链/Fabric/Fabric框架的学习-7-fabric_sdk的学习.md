---
title: Fabric框架的学习-7-fabric_sdk的学习
tags:
  - fabric
categories:
  - technical
  - fabric
toc: true
declare: true
date: 2020-11-19 22:06:19
---



fabric提供了许多sdk直接与fabric网络进行交互。

# 十二、fabric sdk

## 12.1 fabric sdk go

下载sdk：

https://github.com/hyperledger/fabric-sdk-go

内容学习自：https://my.oschina.net/u/3843525/blog/3197958

fabric go sdk调用grpc可以与指定的peer和orderer进行通信：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119221529.png)

<!-- more -->

### 12.1.1 sdk目录介绍

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119221712.png)

有2个目录需要注意一下，**internal**和**third_party**，它们两个包含了fabric go sdk依赖的一些代码，来自于fabric、fabric-ca，当使用到fabric的一些类型时，应当使用以下的方式，而不是直接导入fabric或者fabric-ca：

```
import "github.com/hyperledger/fabric go sdk/third_party/github.com/hyperledger/fabric/xxx"
```

pkg目录是fabric go sdk的主要实现，doc文档介绍了不同目录所提供的功能，以及给出了接口调用样例：

- pkg/fabsdk：主package，主要用来**生成fabsdk**以及fabric go sdk中其他pkg使用的option context。
- pkg/client/channel：主要用来**调用、查询Fabric链码**，或者注册链码事件。
- pkg/client/resmgmt：主要用来Hyperledger fabric**网络的管理**，比如创建通道、加入通道，安装、实例化和升级链码。
- pkg/client/event:配合channel模块来进行Fabric**链码事件的注册和过滤**。
- pkg/client/ledger：主要用来实现Fabric**账本的查询，查询区块、交易、配置**等。
- pkg/client/msp：主要用来管理**fabric网络中的成员关系。**

### 12.1.2 使用步骤

- 为client编写配置文件config.yaml
- 为client创建fabric sdk实例fabsdk
- 为client创建resource manage client，简称RC，RC用来进行管理操作的client， 比如通道的创建，链码的安装、实例化和升级等
- 为client创建channel client，简称CC，CC用来链码的调用、查询以及链码事件 的注册和取消注册

### 12.1.3 配置文件config.yaml

client使用sdk与fabric网络交互，需要告诉sdk两类信息：

- **我是谁：**即当前client的信息，包含**所属组织、密钥和证书文件的路径等**， 这是每个client<u>专用</u>的信息。
- **对方是谁：**即fabric网络结构的信息，**channel、org、orderer和peer等 的怎么组合起当前fabric网络的**，**这些结构信息应当与configytx.yaml中是一致的**。这是通用配置，每个客户端都可以拿来使用。另外，<font color='green'>这部分信息并不需要是完整fabric网络信息，如果当前client只和部分节点交互，那配置文件中只需要包含所使用到的网络信息。</font>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201119222649.png)



### 12.1.4 使用go mod管理fabric go sdk项目的依赖

fabric go sdk目前本身使用go modules管理依赖，从go.mod可知，依赖的一些包指定了具体的版本， 如果你的项目依赖的版本和fabric go sdk依赖的版本不同，会产生编译问题。

因此建议项目也使用go moudles管理依赖，然后相同的软件包可以使用相同的版本，可以这样操作：

- go mod init初始化好项目的go.mod文件。
- 编写代码，完成后运行go mod run，会自动下载依赖的项目，但版本可能与 fabric go sdk中的依赖版本不同，编译存在问题。
- 把go.mod中的内容复制到项目的go.mod中，然后保存，go mod会自动合并相同的依赖， 运行go mod tidy，会自动添加新的依赖或删除不需要的依赖。

### 12.1.5 创建fabric go sdk的入口实例

通过config.FromFile解析配置文件，然后通过fabsdk.New创建fabric go sdk的入口实例。

```go
import "github.com/hyperledger/fabric go sdk/pkg/core/config"
import "github.com/hyperledger/fabric go sdk/pkg/fabsdk"

sdk, err := fabsdk.New(config.FromFile(c.ConfigPath))
if err != nil {
  log.Panicf("failed to create fabric sdk: %s", err)
}
```

### 12.1.6 创建fabric go sdk的资源管理客户端

管理员账号才能进行Hyperledger fabric网络的管理操作，所以创建资源管理客户端一定要使用管理员账号。

通过fabsdk.WithOrg("Org1")和fabsdk.WithUser("Admin")指定Org1的Admin账户，使用sdk.Context创建clientProvider，然后通过resmgmt.New创建fabric go sdk资源管理客户端。

```go
import 	"github.com/hyperledger/fabric go sdk/pkg/client/resmgmt"

rcp := sdk.Context(fabsdk.WithUser("Admin"), fabsdk.WithOrg("Org1"))
rc, err := resmgmt.New(rcp)
if err != nil {
  log.Panicf("failed to create resource client: %s", err)
}
```

### 12.1.7 使用fabric go sdk的资源管理客户端安装链码

安装Fabric链码使用资源管理客户端的InstallCC接口，需要指定resmgmt.InstallCCRequest以及在哪些peers上面安装。resmgmt.InstallCCRequest指明了链码ID、链码路径、链码版本以及打包后的链码。

打包链码需要使用到链码路径CCPath和GoPath，GoPath即本机的$GOPATH，CCPath是相对于GoPath的相对路径，如果路径设置不对，会造成sdk找不到链码。

```go
// pack the chaincode
ccPkg, err := gopackager.NewCCPackage("github.com/hyperledger/fabric-samples/chaincode/chaincode_example02/go/", "/Users/shitaibin/go")
if err != nil {
  return errors.WithMessage(err, "pack chaincode error")
}

// new request of installing chaincode
req := resmgmt.InstallCCRequest{
  Name:    c.CCID,
  Path:    c.CCPath,
  Version: v,
  Package: ccPkg,
}

reqPeers := resmgmt.WithTargetEndpoints("peer0.org1.example.com")
resps, err := rc.InstallCC(req, reqPeers)
if err != nil {
  return errors.WithMessage(err, "installCC error")
}
```

### 12.1.8 使用fabric go sdk的资源管理客户端实例化链码

实例化链码需要使用fabric go sdk的资源管理客户端的InstantiateCC接口，需要通过ChannelID、 resmgmt.InstantiateCCRequest和peers，指明在哪个channel上实例化链码，请求包含了链码的ID、路径、版本，以及初始化参数和背书策略，背书策略可以通过cauthdsl.FromString生成。

```go
// endorser policy
org1OrOrg2 := "OR('Org1MSP.member','Org2MSP.member')"
ccPolicy, err := cauthdsl.FromString(org1OrOrg2)
if err != nil {
  return errors.WithMessage(err, "gen policy from string error")
}

// new request
args := packArgs([]string{"init", "a", "100", "b", "200"})
req := resmgmt.InstantiateCCRequest{
  Name:    c.CCID,
  Path:    c.CCPath,
  Version: v,
  Args:    args,
  Policy:  ccPolicy,
}

// send request and handle response
reqPeers := resmgmt.WithTargetEndpoints("peer0.org1.example.com")
resp, err := rc.InstantiateCC(ChannelID, req, reqPeers)
if err != nil {
  return errors.WithMessage(err, "instantiate chaincode error")
}
```

### 12.1.9 使用fabric go sdk的资源管理客户端升级链码

再fabric go sdk中，升级链码和实例化链码是非常相似的，不同点只在请求是resmgmt.UpgradeCCRequest，调用的接口是rc.UpgradeCC：

```go
// endorser policy
org1AndOrg2 := "AND('Org1MSP.member','Org2MSP.member')"
ccPolicy, err := c.genPolicy(org1AndOrg2)
if err != nil {
  return errors.WithMessage(err, "gen policy from string error")
}

// new request
args := packArgs([]string{"init", "a", "100", "b", "200"})
req := resmgmt.UpgradeCCRequest{
  Name:    c.CCID,
  Path:    c.CCPath,
  Version: v,
  Args:    args,
  Policy:  ccPolicy,
}

// send request and handle response
reqPeers := resmgmt.WithTargetEndpoints("peer0.org1.example.com")
resp, err := rc.UpgradeCC(ChannelID, req, reqPeers)
if err != nil {
  return errors.WithMessage(err, "instantiate chaincode error")
}
```

### 12.1.10 使用fabric go sdk的通道客户端调用链码

在fabric go sdk中，使用通道客户端的Execute接口调用链码，使用入参channel.Request和peers指明要让哪些peer上执行链码，进行背书。channel.Request指明了要调用的链码，以及链码内要Invoke的函数args，args是序列化的结果，序列化是自定义的，只要链码能够按相同的规则进行反序列化即可。

```go
// new channel request for invoke
args := packArgs([]string{"a", "b", "10"})
req := channel.Request{
  ChaincodeID: c.CCID,
  Fcn:         "invoke",
  Args:        args,
}

// send request and handle response
// peers is needed
reqPeers := channel.WithTargetEndpoints("peer0.org1.example.com")
resp, err := cc.Execute(req, reqPeers)
if err != nil {
  return errors.WithMessage(err, "invoke chaincode error")
}
log.Printf("invoke chaincode tx: %s", resp.TransactionID)
```

### 12.1.11 使用fabric go sdk的通道客户端查询链码

在fabric go sdk中，查询和调用链码是非常相似的，使用相同的channel.Request，指明了Invoke链码中的query函数，然后调用cc.Query进行查询操作，这样节点不会对请求进行背书：

```go
// new channel request for query
req := channel.Request{
  ChaincodeID: c.CCID,
  Fcn:         "query",
  Args:        packArgs([]string{keys}),
}

// send request and handle response
reqPeers := channel.WithTargetEndpoints(peer)
resp, err := cc.Query(req, reqPeers)
if err != nil {
  return errors.WithMessage(err, "query chaincode error")
}

log.Printf("query chaincode tx: %s", resp.TransactionID)
log.Printf("result: %v", string(resp.Payload))
```

## 12.2 fabric-sdk-go web环境搭建

https://github.com/kevin-hf/kongyixueyuan

基础环境node/go等等不在赘述

基础fabric网络构建参考上一节多机多节点部署

基础网络构建好了之后，创建通道。安装链码等等就基于sdk-go来解决

### 12.2.1 配置Fabric-sdk

配置文件config.yaml:

```yaml
name: "carfabric-network"

version: 1.0.0

client:
  # 客户端属于的组织，必须是organizations定义的组织
  organization: OrgBmwMSP

  # 日志等级
  logging:
    level: debug

  # MSP根目录
  cryptoconfig:
    path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config

  # 某些SDK支持插件化的kv数据库，通过指定的credentialStore实现
  credentialStore:
    # 可选，用于用户证书材料存储，如果所有的证书材料被嵌入到配置文件，则不需要
    path: .\car-store
    # 可选，指定Go SDK实现的CryptoSuite实现
    cryptoStore:
      # 指定用于加密密钥存储的底层KV数据库
      path: .\car-msp

  # 客户端的BCCSP模块配置
  BCCSP:
    security:
      enabled: true
      default:
        provider: "SW"
      hashAlgorithm: "SHA2"
      softVerify: true
      level: 256

  # tls通信安全
  tlsCerts:
    # 可选，当连接到peers，orderes时使用系统证书池，默认为false
    systemCertPool: false

    # 可选，客户端和peers与orderes进行TLS握手的密钥和证书
    client:
      key:
        path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbmw.example.com\users\Admin@orgbmw.example.com\tls\client.key
      cert:
        path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbmw.example.com\users\Admin@orgbmw.example.com\tls\client.crt

# Fabric区块链网络中参与的组织列表
organizations:
  OrgBmw:
    mspid: OrgBmwMSP
    # 组织的MSP存储位置，绝对路径
    # admin用户
    cryptoPath: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbmw.example.com\users\Admin@orgbmw.example.com\msp
    peers:
      - peer0.orgbmw.example.com
    # 可选，证书颁发机构签发×××明，Fabric-CA是一个特殊的证书管理机构，提供REST API支持动态证书管理，如登记、撤销、重新登记
    # 下列部分只为Fabric-CA服务器设置
    certificateAuthorities:
      - bmwca.example.com

  OrgBenz:
    mspid: OrgBenzMSP
    # admin用户
    cryptoPath: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbenz.example.com\users\Admin@orgbenz.example.com\msp
    peers:
      - peer0.orgbenz.example.com
    certificateAuthorities:
      - benzca.example.com

# 发送交易请求或通道创建、更新请求到的orderers列表
# 如果定义了超过一个orderer，SDK使用哪一个orderer由代码实现时指定
orderers:
  # orderer节点，可以定义多个
  orderer.example.com:
    url: localhost:7050
    # 以下属性由gRPC库定义，会被传递给gRPC客户端构造函数
    grpcOptions:
      ssl-target-name-override: orderer.example.com
      # 下列参数用于设置服务器上的keepalive策略，不兼容的设置会导致连接关闭
      # 当keep-alive-time被设置为0或小于激活客户端的参数，下列参数失效
      keep-alive-time: 0s
      keep-alive-timeout: 20s
      keep-alive-permit: false
      fail-fast: false
      # allow-insecure will be taken into consideration if address has no protocol defined, if true then grpc or else grpcs
      allow-insecure: false

    tlsCACerts:
      # Certificate location absolute path
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\ordererOrganizations\example.com\tlsca\tlsca.example.com-cert.pem

# peers节点列表
peers:
  # peer节点定义，可以定义多个
  peer0.orgbmw.example.com:
    # URL用于发送背书和查询请求
    url: localhost:7051
    # eventUrl is only needed when using eventhub (default is delivery service)
    eventUrl: localhost:7053

    grpcOptions:
      ssl-target-name-override: peer0.orgbmw.example.com
      keep-alive-time: 0s
      keep-alive-timeout: 20s
      keep-alive-permit: false
      fail-fast: false
      # allow-insecure will be taken into consideration if address has no protocol defined, if true then grpc or else grpcs
      allow-insecure: false

    tlsCACerts:
      # Certificate location absolute path
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbmw.example.com\tlsca\tlsca.orgbmw.example.com-cert.pem

  peer0.orgbenz.example.com:
    # this URL is used to send endorsement and query requests
    url: localhost:8051
    # eventUrl is only needed when using eventhub (default is delivery service)
    eventUrl: localhost:8053

    grpcOptions:
      ssl-target-name-override: peer0.orgbenz.example.com
      # These parameters should be set in coordination with the keepalive policy on the server,
      # as incompatible settings can result in closing of connection.
      # When duration of the 'keep-alive-time' is set to 0 or less the keep alive client parameters are disabled
      keep-alive-time: 0s
      keep-alive-timeout: 20s
      keep-alive-permit: false
      fail-fast: false
      # allow-insecure will be taken into consideration if address has no protocol defined, if true then grpc or else grpcs
      allow-insecure: false

    tlsCACerts:
      # Certificate location absolute path
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbenz.example.com\tlsca\tlsca.orgbenz.example.com-cert.pem

#
# Fabric-CA is a special kind of Certificate Authority provided by Hyperledger Fabric which allows
# certificate management to be done via REST APIs. Application may choose to use a standard
# Certificate Authority instead of Fabric-CA, in which case this section would not be specified.
# 两个CA设置
certificateAuthorities:
  bmwca.example.com:
    url: localhost:7054
    tlsCACerts:
      # Certificate location absolute path
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbmw.example.com\ca\ca.orgbmw.example.com-cert.pem

    # Fabric-CA supports dynamic user enrollment via REST APIs. A "root" user, a.k.a registrar, is
    # needed to enroll and invoke new users.
    registrar:
      enrollId: admin
      enrollSecret: adminpw
    # [Optional] The optional name of the CA.
    caName: bmwca.example.com

  benzca.example.com:
    url: localhost:8054
    tlsCACerts:
      # Certificate location absolute path
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbenz.example.com\ca\ca.orgbenz.example.com-cert.pem

    # Fabric-CA supports dynamic user enrollment via REST APIs. A "root" user, a.k.a registrar, is
    # needed to enroll and invoke new users.
    registrar:
      enrollId: admin
      enrollSecret: adminpw
    # [Optional] The optional name of the CA.
    caName: benzca.example.com

```

（配置对应之前的solo模式下的多机多节点carfabric网络）

### 12.2.2 定义一个结构体

配置文件完成指定的配置信息之后，我们开始编写代码。

在项目的根目录下添加一个名为 `sdkInit` 的新目录，我们将在这个文件夹中创建 SDK，并根据配置信息创建应用通道

```shell
$ mkdir sdkInit
```

为了方便管理 Hyperledger Fabric 网络环境，我们将在 `sdkInit` 目录中创建一个 `fabricInitInfo.go` 的源代码文件，用于定义一个结构体，包括 Fabric SDK 所需的各项相关信息

```shell
$ vim sdkInit/fabricInitInfo.go 
```

`fabricInitInfo.go `源代码如下：

```go
package sdkInit

import (
	"github.com/hyperledger/fabric-sdk-go/pkg/client/resmgmt"
)

type InitInfo struct {
	ChannelID     string
	ChannelConfig string
	OrgAdmin      string
	OrgName       string
	OrdererOrgName	string
	OrgResMgmt *resmgmt.Client
}
```

### 12.2.3 创建SDK实例

在sdkInit文件夹下创建start.go文件

```go
package main

import (
	"fabric_go/fabcar/sdkInit"
	"fmt"
	"github.com/hyperledger/fabric-sdk-go/pkg/core/config"
	"github.com/hyperledger/fabric-sdk-go/pkg/fabsdk"
)

const ChaincodeVersion  = "1.0"


//实例化一个sdk入口实例
func SetupSDK(ConfigFile string, initialized bool) (*fabsdk.FabricSDK, error) {
	if initialized {
		return nil, fmt.Errorf("Fabric 已被实例化！")
	}
	//使用config格式化初始配置文件yaml，并创建实例
	sdk, err := fabsdk.New(config.FromFile(ConfigFile))
	if err != nil {
		return nil, fmt.Errorf("实例化Fabric sdk失败！%v", err)
	}
	fmt.Println("Fabric sdk 初始化成功！")
	return sdk, nil
}

func main() {
	fabricSDK, err := SetupSDK("../../config.yaml", false)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(fabricSDK)
}
```

<font color='red'>注意：测试之前需要满足依赖，否则会提示working directory is not part of a module</font>

### 12.2.3 满足依赖

在运行应用程序之前，需要将 Go 源代码时行编译，但在开始编译之前，我们需要使用一个 `vendor` 目录来包含应用中所需的所有的依赖关系。 在我们的GOPATH中，我们有Fabric SDK Go和其他项目。 在尝试编译应用程序时，Golang 会在 GOPATH 中搜索依赖项，但首先会检查项目中是否存在`vendor` 文件夹。 如果依赖性得到满足，那么 Golang 就不会去检查 GOPATH 或 GOROOT。 这在使用几个不同版本的依赖关系时非常有用（可能会发生一些冲突，比如在例子中有多个BCCSP定义，通过使用像[`dep`](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&rurl=translate.google.com&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/golang/dep&xid=25657,15700002,15700019,15700124,15700149,15700168,15700186,15700201&usg=ALkJrhgelyRl7D3pIJRpuA8cynagkWYHXg)这样的工具在`vendor`目录中来处理这些依赖关系。

```shell
export GO111MODULE=on
# windows下
go env -w GO111MODULE=on
go mod init
go mod vendor
```

随后会在项目目录下生成vendor文件

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201120203612.png)



12.2.4 实例结果运行结果:













# Tips

## 1. working directory is not part of a module</font>

没有满足依赖

https://blog.csdn.net/cwcmcw/article/details/107347245

解决方法：

```shell
export GO111MODULE=on
go mod init
go mod vendor
```

## 2. The system cannot find the file specified.

运行sdk代码时提示找不到文件，所以是config.yaml配置文件除了问题，请看具体的报错位置，并检查config.yaml中注意的地方：

<font color='red'>**所有的文件路径都写成绝对路径！！！**</font>

特别的在写orderer、peer、ca节点的配置时，msp证书文件要写绝对路径！！

 ```yaml
orderers:
  orderer.example.com:
    url: http://10.0.2.5:7050

    grpcOptions:
      ssl-target-name-override: orderer.example.com
      keep-alive-time: 0s
      keep-alive-timeout: 20s
      keep-alive-permit: false
      fail-fast: false
  
      allow-insecure: false
    tlsCACerts:
      # Certificate location absolute path
       # 注意这里要求绝对路径！！！！！！
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\ordererOrganizations\example.com\tlsca\tlsca.example.com-cert.pem

peers:
  peer0.orgbmw.example.com:
    # this URL is used to send endorsement and query requests
    url: http://10.0.2.9:7051
    # eventUrl is only needed when using eventhub (default is delivery service)
    eventUrl: http://10.0.2.9:7053

    grpcOptions:
      ssl-target-name-override: peer0.orgbmw.example.com
      keep-alive-time: 0s
      keep-alive-timeout: 20s
      keep-alive-permit: false
      fail-fast: false
      # allow-insecure will be taken into consideration if address has no protocol defined, if true then grpc or else grpcs
      allow-insecure: false

    tlsCACerts:
      # Certificate location absolute path
       # 注意这里要求绝对路径！！！！！！
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbmw.example.com\tlsca\tlsca.orgbmw.example.com-cert.pem



benzca.example.com:
    url: http://10.0.2.10:7054
    tlsCACerts:
      # Certificate location absolute path
       # 注意这里要求绝对路径！！！！！！
      path: D:\MyProgramPackage\block_chain\go_projects\src\fabric_go\fabcar\crypto-config\peerOrganizations\orgbenz.example.com\ca\ca.orgbenz.example.com-cert.pem
 ```

## 3. 配置文件的坑

创建通道提示：<font color='red'>create channel failed: SendEnvelope failed: calling orderer 'http://localhost:7050' failed: Orderer Client Status Code: (2) CON
NECTION_FAILED. Description: dialing connection on target [http://localhost:7050]: connection is in TRANSIENT_FAILURE</font>

原因：直接使用localhost而不是用http://localhost。。。。

修改配置文件中的所有http://localhost



