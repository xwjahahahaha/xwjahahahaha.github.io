---
title: Fabric框架的学习-3-手动组建Fabric网络
tags:
  - fabric
categories:
  - technical
  - fabric
toc: true
declare: true
date: 2020-10-08 10:38:28
---

# 四、 Fabric核心模块

在搭建自己的网络之前还需要了解下核心模块

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008111936.png)

<!-- more -->

对于模块的说明：

|   模块名称    |                             说明                             |
| :-----------: | :----------------------------------------------------------: |
|     peer      |                              /                               |
|    orderer    |                              /                               |
|   cryptogen   |              编写配置文件，根据配置文件生成证书              |
|  cryfigtxgen  |          生成创世块文件（不是创世快）、生成通道文件          |
| configtxlator | 对区块文件修改，可以将二进制的区块文件变为JSON的可读模式。以及动态的添加组织 |

操作这些模块，就主要使用这些模块给我们提供的shell命令，这些shell命令都在bin目录下

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008113235.png)

并且之前已经将这些命令加入到`/usr/local/bin`全局变量下了

**命令手册：**

http://cw.hubwiz.com/card/c/fabric-command-manual/1/1/1/

# 五、手动组建Fabric网络

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201016150737.png)

## 5.1 生成Fabric证书-yaml     命令：cryptogen

```shell
cryptogen --help
cryptogen showtemplate # 会输出配置文件的模板
```

Fabric V1.2.0中的版本cryptogen:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008113852.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201120203909.png)

对于每一子命令可以查看:

```shell
 cryptogen generate --help
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008124317.png)

获取配置文件模板

```shell
cryptogen showtemplate
# 会直接输出到终端
# 重定向到一个新yaml文件
cryptogen showtemplate > a.yaml
vim a.yaml  # 这样就可以直接使用了
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008124517.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008124647.png)

模板文档解释（一些注释做了删减）：

```shell
# ---------------------------------------------------------------------------
# "OrdererOrgs" - Definition of organizations managing orderer nodes
# ---------------------------------------------------------------------------
# 排序节点组织信息
OrdererOrgs:
  # ---------------------------------------------------------------------------
  # Orderer
  # ---------------------------------------------------------------------------
  - Name: Orderer  			# 排序节点组织名
    Domain: example.com		# 根域名，排序节点组织的根域名，在实际开发环境中需要使用真实已经备案的域名，测试环境下自己随便起就可以
    EnableNodeOUs: false	# 链码是否支持nodejs

    # ---------------------------------------------------------------------------
    # "Specs" - See PeerOrgs below for complete description
    # ---------------------------------------------------------------------------
    Specs:
      - Hostname: orderer  	# 子域名，因为一个域名只能绑定一个ip，而orderer是多个的，所以要多个子域名，这里就只有这一个orderer一个即可
      						# 访问这台orderer对应的域名就是orderer.example.com 

# ---------------------------------------------------------------------------
# "PeerOrgs" - Definition of organizations managing peer nodes
# ---------------------------------------------------------------------------
# peer节点组织
PeerOrgs:
  # ---------------------------------------------------------------------------
  # Org1
  # ---------------------------------------------------------------------------
  # 组织还可以进行细分，分为Org1、Org2等等，在下方添加即可
  - Name: Org1    	# 组织名字可以自己制定
    Domain: org1.example.com   		# 访问第一个组织用到的根域名
    EnableNodeOUs: false			# 链码是否支持nodejs
    Template:						# 模板， 根据默认的规则生成2个peer数据存储节点
      Count: 2						# 第一个节点域名： peer0.org1.example.com  第二个节点域名： peer1.org1.example.com
    Users:							# 创建普通用户的个数
      Count: 1
  # ---------------------------------------------------------------------------
  # Org2: See "Org1" for full specification
  # ---------------------------------------------------------------------------
  # 组织2同理
  - Name: Org2
    Domain: org2.example.com
    EnableNodeOUs: false
    Template:
      Count: 1
    Users:
      Count: 1
```

**注意事项：**

* Domain: example.com 根域名，排序节点组织的根域名，**在实际开发环境中需要使用真实已经备案的域名，测试环境下自己随便起就可以**

* **使用Specs或者Template指定节点的子域名两者都可以，区别就在于用Specs可以自己规划子域名，而Template则是例如pee0、peer1这样的默认设置**

  **两个还可以一起用，最终节点数是两者之和**

  例：

  ```shell
  Template:						
        Count: 2	 # 两个默认域名	
  Specs:
        - Hostname: orderer  # 一个指定域名
  # 这样是指定三个节点
  ```

----

小练习：

要求：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008133034.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008133055.png)

```shell
mkdir fabric-test01   #创建一个空目录作为环境
cd fabric-test01/
cryptogen showtemplate > crypto-config.yaml  # 创建空模板文件
vim crypto-config.yaml
# 修改模板
```

```shell
  2 # ---------------------------------------------------------------------------
  3 # "OrdererOrgs" - Definition of organizations managing orderer nodes
  4 # ---------------------------------------------------------------------------
  5 OrdererOrgs:
  6   # ---------------------------------------------------------------------------
  7   # Orderer
  8   # ---------------------------------------------------------------------------
  9   - Name: Orderer
 10     Domain: xwj.com
 11     EnableNodeOUs: false
 12 
 13     # ---------------------------------------------------------------------------
 14     # "Specs" - See PeerOrgs below for complete description
 15     # ---------------------------------------------------------------------------
 16     Specs:
 17       - Hostname: orderer
 18 
 19 # ---------------------------------------------------------------------------
 20 # "PeerOrgs" - Definition of organizations managing peer nodes
 21 # ---------------------------------------------------------------------------
 22 PeerOrgs:
 23   # ---------------------------------------------------------------------------
 24   # Org1
 25   # ---------------------------------------------------------------------------
 26   - Name: OrgGo
 27     Domain: orggo.xwj.com
 28     EnableNodeOUs: true
 29     Template:
 30       Count: 2
 31     Users:
 32       Count: 3
 33      
 34   # ---------------------------------------------------------------------------
 35   # Org2: See "Org1" for full specification
 36   # ---------------------------------------------------------------------------
 37   - Name: OrgCpp
 38     Domain: orgcpp.xwj.com
 39     EnableNodeOUs: true
 40     Template:
 41       Count: 2
 42     Users:
 43       Count: 3

```

使用--config命令去执行配置文件，如果不指定配置文件，则会按照模板文件来执行

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008134723.png)

```shell
cryptogen generate --config=crypto-config.yaml   # 出现自己写的组织名称则说明成功了
# 随后生成crypto-config文件
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008135019.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008135443.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008135656.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201008135903.png)

检查peer和user创建的个数，符合要求即成功

## 5.2 创世块文件和通道文件的生成命令：configtxgen

### 5.2.1 命令介绍：

参数如下：

```shell
configtxgen --help
# 没有子命令
Usage of configtxgen:
  -asOrg string  #所属组织，也就是为某个特定组织生成配置
  -channelID string  #channel名称，如果不指定默认是"testchainid"  必须是小写！!!!  
  -inspectBlock string #打印指定区块文件中配置内容   
  -inspectChannelCreateTx string #打印指定创建通道交易的配置文件   
  -outputAnchorPeersUpdate string #生成一个更新锚点的更新channel配置信息       
  -outputBlock string #输出区块文件路径     
  -outputCreateChannelTx string #指定一个路径，来生成channel配置文件  
  -profile string   #配置文件中的节点，用于生成相关配置文件，默认是 "SampleInsecureSolo")      
  -version #显示版本信息
```

执行这个命令，必须要指定**固定名字**的一个配置文件  configtx.yaml

```shell
cp /home/xwj/fabric-1.2/fabric-samples/first-network/configtx.yaml /home/xwj/fabric-1.2/test01/   
# 这个没有模板，所以在之前的first-network文件夹中复制一个到自己的目录中修改即可
```

```yaml
 6 ---
 15 Organizations:  # 固定的
 19     - &OrdererOrg  # 排序节点组织的名字，这里的组织名可以随便取  *OrdererOrg代表这整个块
 22         Name: OrdererOrg		# 组织名
 23 
 25         ID: OrdererMSP			# 排序节点组织的ID
 26 
 28         MSPDir: crypto-config/ordererOrganizations/example.com/msp   # 这里是创建证书时MSP的地址，这里是相对地址
 29 
 30     - &Org1						# 第一个组织，名字自己起
 33         Name: Org1MSP			# 第一个组织的名字
 34 
 36         ID: Org1MSP				# 组织ID
 37 
 38         MSPDir: crypto-config/peerOrganizations/org1.example.com/msp
 39 
 40         AnchorPeers:			# 锚节点
 44             - Host: peer0.org1.example.com   	# 指定该组织中任一个节点peer节点的域名，其就作为该组织的锚节点。一个组织只有一个锚节点
 45               Port: 7051						# 固定端口
 46 
 47     - &Org2
 50         Name: Org2MSP
 51 
 53         ID: Org2MSP
 54 
 55         MSPDir: crypto-config/peerOrganizations/org2.example.com/msp
 56 
 57         AnchorPeers:
 61             - Host: peer0.org2.example.com
 62               Port: 7051
 63 	### 如果有多个组织，基础添加即可
 83 Capabilities:  # 能力  V1.1之后出现的，设置时全部设置为True（为了向上兼容）
 86     Global: &ChannelCapabilities
 91         V1_1: true
 96     Orderer: &OrdererCapabilities
101         V1_1: true
102 
106     Application: &ApplicationCapabilities
111         V1_2: true
112 
121 Application: &ApplicationDefaults  #一般就用默认配置
122 
125     Organizations:
126 
135 Orderer: &OrdererDefaults   # 对orderer节点组织设置一些更加细的命令
136 
137     # Orderer Type: The orderer implementation to start
138     # Available types are "solo" and "kafka"
        # 共识机制 == 排序算法  ，Fabric V1.2  提供的共识机制有两种solo和kafka，测试环境一般用solo。真实环境用kafka（适用于高并发）
        # solo就是指orderer节点就只有一个不需要共识算法
139     OrdererType: solo   # 共识算法
140 
141     Addresses:			# orderer节点的地址
142         - orderer.example.com:7050   # 端口一般不修改
143 
145     BatchTimeout: 2s	# 多长时间产生一个区块
146 
		##### BatchTimeout、MaxMessageCount、AbsoluteMaxBytes这三者有一个超过了就产生区块
148     BatchSize:  
149 
150         # Max Message Count: The maximum number of messages to permit in a batch
151         MaxMessageCount: 10			# 交易的最大数量，，建议100
152 
153         # Absolute Max Bytes: The absolute maximum number of bytes allowed for
154         # the serialized messages in a batch.
155         AbsoluteMaxBytes: 99 MB		# 数据大小最大值，数据量达到99M也会产生区块，一般32M/64M
156 
160         PreferredMaxBytes: 512 KB 	# 建议的最大数量
161 
162     Kafka:
165         Brokers:    # 代理人
166             - 127.0.0.1:9092    #kafka的服务器
167 
170     Organizations:
171 
174 #   Profile    上面分散的设置的总结
180 Profiles:   # 固定的不能修改
181     # *是引用上面&的数据
182     TwoOrgsOrdererGenesis:   	# 区块名字 可以改
183         Capabilities:		 	# 能力，即上面设置的
184             <<: *ChannelCapabilities	# 通道的能力
185         Orderer:
186             <<: *OrdererDefaults   		# orderer组织的细节配置
187             Organizations:
188                 - *OrdererOrg			# orderer组织的配置
189             Capabilities:
190                 <<: *OrdererCapabilities	# orderer的能力
191         Consortiums:					# 联盟
192             SampleConsortium:			# 实例联盟 可以自定义
193                 Organizations:			# 两个peer节点组织
194                     - *Org1
195                     - *Org2
196     TwoOrgsChannel:						# 通道名字可以自定义
197         Consortium: SampleConsortium   	# 对应上面的联盟名字，要保持一致
198         Application:
199             <<: *ApplicationDefaults
200             Organizations:				# 事先声明组织在改通道中，后续其中的节点就可以选择加入该通道
201                 - *Org1
202                 - *Org2
203             Capabilities:
204                 <<: *ApplicationCapabilities
                                                                  
```

**锚节点**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201009133022.png)



### 5.2.2 修改配置文件

事先需要做的事：

* 找出orderer的msp的位置：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201009134412.png)

  `/home/xwj/fabric_1.2/fabric-test01/crypto-config/ordererOrganizations/xwj.com/msp`

---

* 找出两个peer组织的msp的位置

  go组织：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201009134721.png)

  `/home/xwj/fabric_1.2/fabric-test01/crypto-config/peerOrganizations/orggo.xwj.com/msp`

  cpp组织同样的方法。。。不再赘述

  `/home/xwj/fabric_1.2/fabric-test01/crypto-config/peerOrganizations/orgcpp.xwj.com/msp`

configtx.yaml 配置文件如下

```yaml
---
Organizations:

    - &OrdererOrg

        Name: OrdererOrg

        ID: OrdererMSP
        
        MSPDir: crypto-config/ordererOrganizations/xwj.com/msp

    - &org_go

        Name: OrgGoMSP

        ID: OrgGoMSP

        MSPDir: crypto-config/peerOrganizations/orggo.xwj.com/msp

        AnchorPeers:

            - Host: peer0.orggo.xwj.com
              Port: 7051

    - &org_cpp

        Name: OrgCppMSP


        ID: OrgCppMSP

        MSPDir: crypto-config/peerOrganizations/orgcpp.xwj.com/msp

        AnchorPeers:
            - Host: peer0.orgcpp.xwj.com
              Port: 7051

Capabilities:

    Global: &ChannelCapabilities
   
        V1_1: true

  
    Orderer: &OrdererCapabilities

        V1_1: true

    Application: &ApplicationCapabilities

        V1_2: true


Application: &ApplicationDefaults

    Organizations:

Orderer: &OrdererDefaults

    OrdererType: solo

    Addresses:
        - orderer.xwj.com:7050

    BatchTimeout: 2s

    BatchSize:

        MaxMessageCount: 100

        AbsoluteMaxBytes: 32 MB

        PreferredMaxBytes: 512 KB

    Kafka:

        Brokers:
            - 127.0.0.1:9092


    Organizations:

Profiles:

    XwjOrdererGenesis:
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *org_go
                    - *org_cpp
    XwjOrgsChannel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *org_go
                - *org_cpp
            Capabilities:
                <<: *ApplicationCapabilities
                                                                                                                                
```

### 5.2.3 命令执行配置文件

* 创建创世块

    ```shell
    configtxgen -profile XwjOrdererGenesis -outputBlock ./genesis.block
    ```

    **注意参数不是configtx.yaml的文件名，而是其中Profiles创世块的这个名字（因为是创建创世块的命令需要这些配置信息）：**

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201009141629.png)

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201011101633.png)

    没有报错信息，并且看到genesis.block文件生成则说明成功了。

    为了适应后面的配置文件的默认目录，一般会将genesis.block文件放到channel-artifacts文件夹下:

    ```shell
    mkdir channel-artifacts
    mv genesis.block channel-artifacts/
    ```

* 生成通道文件

  ```shell
  configtxgen -profile XwjOrgsChannel -outputCreateChannelTx channel-artifacts/channel.tx -channelID xwjchannel
  # outputCreateChannelTx 参数后接 保存的文件名，一般tx后缀
  # channelID 是自己指定的通道的id，这个与configtx.yaml中的配置名字无关
  ```

  注意后接通道配置名：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201120203730.png)
  
  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201120203741.png)
  
* 生成锚节点更新文件

  > 这个操作是可选的，可以做可以不做

  这个锚节点更新文件是为了当你选定的锚节点失效的时候，更新整个组织的锚节点（重新指定一个锚节点）的作用

  ```shell
  configtxgen -profile XwjOrgsChannel -outputAnchorPeersUpdate channel-artifacts/GoMSPanchors.tx -channelID xwjChannel -asOrg OrgGoMSP
  # profile 还是指定通道配置名
  # outputAnchorPeersUpdate 指定输出文件名
  # channelID 之前设置的通道id
  # asOrg 指明锚节点属于哪个组织
  configtxgen -profile XwjOrgsChannel -outputAnchorPeersUpdate channel-artifacts/CPPMSPanchors.tx -channelID xwjChannel -asOrg OrgCppMSP
  ```

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201011104202.png)

  对应生成两个更新文件

  **每个组织都生成一个对应的锚节点更新文件**

## 5.3 docker-compose文件的编写

### 5.3.1 文件关系介绍

一个网络中有非常多的各类节点，每个节点都是在docker容器中运行的，需要编写docker-compose文件去管理这些docker

* docker-compose-cli.yaml

  官方实例first-network中的docker-compose-cli.yaml解析

     ```shell
      version: '2'   # docker-compose的版本号

      volumes:		# 官方实例有五个节点，所以分别对应五个数据卷的映射  映射本地主机地址一般在/var/lib/docker/volumes
        # 这里的映射只写了一半，在docker-compose-base中写了完整的映射关系
        orderer.example.com:
        peer0.org1.example.com:
        peer1.org1.example.com:
        peer0.org2.example.com:
        peer1.org2.example.com:

      networks:		# 节点所属的网络
        byfn:

      services:		# 服务

        orderer.example.com:	# 服务名 可以自己指定
          extends: 				# 继承下方文件中的服务
            file:   base/docker-compose-base.yaml
            service: orderer.example.com   # 具体继承文件中哪项服务
          container_name: orderer.example.com 	# 指定容器的名字
          networks:								# 节点所在网络
            - byfn

        peer0.org1.example.com:
          container_name: peer0.org1.example.com
          extends:
            file:  base/docker-compose-base.yaml
            service: peer0.org1.example.com
          networks:
            - byfn

        peer1.org1.example.com:
          container_name: peer1.org1.example.com
          extends:
            file:  base/docker-compose-base.yaml
            service: peer1.org1.example.com
          networks:
            - byfn

        peer0.org2.example.com:
          container_name: peer0.org2.example.com
          extends:
            file:  base/docker-compose-base.yaml
            service: peer0.org2.example.com
          networks:
            - byfn

        peer1.org2.example.com:
          container_name: peer1.org2.example.com
          extends:
            file:  base/docker-compose-base.yaml
            service: peer1.org2.example.com
      cli:  # 客户端的终端  可以用linux也可以用nodejs、java编写  这里是linux
          container_name: cli   # 容器名字
          image: hyperledger/fabric-tools:$IMAGE_TAG  # 客户端所对应的镜像
          tty: true # 终端打开
          stdin_open: true # 标准输入打开
          environment:   # 环境变量
          - GOPATH=/opt/gopath
          - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
          #- CORE_LOGGING_LEVEL=DEBUG
          - CORE_LOGGING_LEVEL=INFO
          - CORE_PEER_ID=cli
          - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
          - CORE_PEER_LOCALMSPID=Org1MSP
          - CORE_PEER_TLS_ENABLED=true
          -CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
                - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
                - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
                - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
              working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer # 进入镜像时其工作目录
              command: /bin/bash    #  命令
              volumes:   # 数据卷的挂载
                  - /var/run/:/host/var/run/
                  - ./../chaincode/:/opt/gopath/src/github.com/chaincode
                  - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
                  - ./scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/
                  - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts  # 上面生成的那些文件
              depends_on:		# 启动顺序
                - orderer.example.com
                - peer0.org1.example.com
                - peer1.org1.example.com
                - peer0.org2.example.com
                - peer1.org2.example.com
              networks:   		# 客户端所在网络
                - byfn

     ```

  以上配置文件会启动6个容器（包含客户端cli）

* base目录下的docker-compose-base.yaml

  ```yaml
  version: '2'
  
  services:
  
    orderer.example.com:  							# 被上面那个文件所继承的服务
      container_name: orderer.example.com  			# 容器名
      image: hyperledger/fabric-orderer:$IMAGE_TAG	# $IMAGE_TAG环境变量指定运行镜像的版本tag	
      environment:									# orderer镜像运行需要使用的环境变量
        - ORDERER_GENERAL_LOGLEVEL=INFO
        - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
        - ORDERER_GENERAL_GENESISMETHOD=file
        - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
        - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
        - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
        # enabled TLS
        - ORDERER_GENERAL_TLS_ENABLED=true
        - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
        - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
        - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
      working_dir: /opt/gopath/src/github.com/hyperledger/fabric	# 启动后的工作目录
      command: orderer  	# 启动order服务
      volumes:    		# 数据卷映射
      - ../channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block   # 创世块映射
      # order节点的身份证书msp
      - ../crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp
      # tls证书
      - ../crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls
      # 完整的映射关系
      # 把/var/lib/docker/volumes/order.xwj.com 映射到/var/hyperledger/production/orderer
      - orderer.example.com:/var/hyperledger/production/orderer
      ports:
        - 7050:7050  # 端口映射  orderer一个端口用于通信，而peer有两个端口
  
    peer0.org1.example.com:
      container_name: peer0.org1.example.com
      extends:    	# 又继承了一个文件peer-base.yaml中的peer-base服务，peer-base是peer节点的一个通用配置，所以单独提出来了
        file: peer-base.yaml
        service: peer-base
      environment: 	
        - CORE_PEER_ID=peer0.org1.example.com
        - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
        - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:7051
        - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
        - CORE_PEER_LOCALMSPID=Org1MSP
      volumes:
          - /var/run/:/host/var/run/
          - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp
          - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls
          - peer0.org1.example.com:/var/hyperledger/production
      ports:
        - 7051:7051	# 一般情况 ~51端口用于通信
        - 7053:7053	# ~53端口用于事件传播
    peer1.org1.example.com:
      container_name: peer1.org1.example.com
      extends:
        file: peer-base.yaml
        service: peer-base
      environment:
        - CORE_PEER_ID=peer1.org1.example.com
        - CORE_PEER_ADDRESS=peer1.org1.example.com:7051
        - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org1.example.com:7051
        - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
        - CORE_PEER_LOCALMSPID=Org1MSP
      volumes:
          - /var/run/:/host/var/run/
          - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/etc/hyperledger/fabric/msp
          - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/etc/hyperledger/fabric/tls
          - peer1.org1.example.com:/var/hyperledger/production
  
      ports:
        - 8051:7051
        - 8053:7053
  
    peer0.org2.example.com:
      container_name: peer0.org2.example.com
      extends:
        file: peer-base.yaml
        service: peer-base
      environment:
        - CORE_PEER_ID=peer0.org2.example.com
        - CORE_PEER_ADDRESS=peer0.org2.example.com:7051
        - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:7051
        - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org2.example.com:7051
        - CORE_PEER_LOCALMSPID=Org2MSP
      volumes:
          - /var/run/:/host/var/run/
          - ../crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp:/etc/hyperledger/fabric/msp
          - ../crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls:/etc/hyperledger/fabric/tls
          - peer0.org2.example.com:/var/hyperledger/production
      ports:
        - 9051:7051
        - 9053:7053
  
    peer1.org2.example.com:
      container_name: peer1.org2.example.com
      extends:
        file: peer-base.yaml
        service: peer-base
      environment:
        - CORE_PEER_ID=peer1.org2.example.com
        - CORE_PEER_ADDRESS=peer1.org2.example.com:7051
        - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org2.example.com:7051
        - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:7051
        - CORE_PEER_LOCALMSPID=Org2MSP
      volumes:
          - /var/run/:/host/var/run/
          - ../crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/msp:/etc/hyperledger/fabric/msp
          - ../crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls:/etc/hyperledger/fabric/tls
          - peer1.org2.example.com:/var/hyperledger/production
      ports:
        - 10051:7051
        - 10053:7053
  
  ```

  peer-base.yaml文件：

  ```yaml
  version: '2'
  
  services:
    peer-base:
      image: hyperledger/fabric-peer:$IMAGE_TAG
      environment:
        - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
        # the following setting starts chaincode containers on the same
        # bridge network as the peers
        # https://docs.docker.com/compose/networking/
        - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_byfn
        - CORE_LOGGING_LEVEL=INFO
        #- CORE_LOGGING_LEVEL=DEBUG
        - CORE_PEER_TLS_ENABLED=true
        - CORE_PEER_GOSSIP_USELEADERELECTION=true
        - CORE_PEER_GOSSIP_ORGLEADER=false
        - CORE_PEER_PROFILE_ENABLED=true
        - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
        - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
        - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
      working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer  # 工作目录
      command: peer node start		# 启动peer
  ```

### 5.3.2 客户端角色需要使用的环境变量

```yaml
 # 客户端docker容器启动之后，go的工作目录
 - GOPATH=/opt/gopath		# 一般不变
	  # docker启动之后的本地套接字，一般不变
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # 日志级别： critical 严重错误 | error | warning | notice | info | debug
      - CORE_LOGGING_LEVEL=INFO
      # 当前客户端节点的id，自己指定
      - CORE_PEER_ID=cli
      # 客户端需要连接的peer节点地址（同一个组织下），客户端必须要连接一个peer节点才能通信，端口一般不该
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      # 客户端、peer节点所属的组织
      - CORE_PEER_LOCALMSPID=Org1MSP
      # 通信是否使用tls加密，true对应下方的一系列设置
      - CORE_PEER_TLS_ENABLED=true
      # 证书文件 这些文件对应的是客户端要连接peer节点的证书目录
      -CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
      # 私钥文件
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
      # 根证书文件
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
      # 指定当前客户端的身份，用户的证书
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
```

修改客户端的配置：

```yaml
 cli:
    container_name: cli
    image: hyperledger/fabric-tools:latest
    tty: true
    stdin_open: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=INFO
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.orggo.xwj.com:7051
      - CORE_PEER_LOCALMSPID=OrgGoMSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/users/Admin@orggo.xwj.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./chaincode/:/opt/gopath/src/github.com/chaincode
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    depends_on:
      - orderer.xwj.com
      - peer0.orggo.xwj.com
      - peer1.orggo.xwj.com
      - peer0.orgcpp.xwj.com
      - peer1.orgcpp.xwj.com
    networks:
      - byfn
```

### 5.3.3 orderer节点需要使用的环境变量

环境配置解析：

```yaml
 environment:									# orderer镜像运行需要使用的环境变量
      - ORDERER_GENERAL_LOGLEVEL=INFO			# 日志级别
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0	# orderer监听的ip地址
      - ORDERER_GENERAL_GENESISMETHOD=file		# 创世块的来源，指定file来源就是文件
      # 创世快对应的文件，这个目录被挂载，不需要改
      #  - ../channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block 
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP	# orderer的所属组id
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp	# 当前节点的msp账号
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true  		# 是否使用tls加密	
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key	# 私钥
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt	# 证书
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]		# 根证书
```

修改环境配置：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012124344.png)

### 5.3.4 peer节点需要使用的环境变量

peer节点需要配置的环境变量解析：

```yaml
extends:    	# 又继承了一个文件peer-base.yaml中的peer-base服务，peer-base是peer节点的一个通用配置，所以单独提出来了
      file: peer-base.yaml
      service: peer-base
    environment: 	
      - CORE_PEER_ID=peer0.org1.example.com 			 			# 当前peer节点的名字
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051				# peer节点的ip地址信息
      # 启动后向哪些节点发起gossip连接，以加入网络
      # gossip流言协议，每个节点只向没有收到流言的节点传播
      # 一般配置就写自己的ip+端口就可以了（不知道写谁的话）
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:7051	
      # 为了被其他节点感知到。设置自己的地址和端口，不设置别的节点就不知道
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
      # peer节点所属组的组id
      - CORE_PEER_LOCALMSPID=Org1MSP

# peer-base.yaml 中环境配置

environment:
	  #	本地套接字地址 一般不需要改 
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      # 当前节点属于哪个网络
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_byfn
      - CORE_LOGGING_LEVEL=INFO 				# 日志级别
      #- CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true 				# 是否加密
      - CORE_PEER_GOSSIP_USELEADERELECTION=true	# 下方单独解释
      - CORE_PEER_GOSSIP_ORGLEADER=false		# 下方单独解释
      - CORE_PEER_PROFILE_ENABLED=true 			# 在peer节点中有一个profile服务，在此指定是否开启
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
```

- CORE_PEER_GOSSIP_USELEADERELECTION=true   是否使用自动选举
- CORE_PEER_GOSSIP_ORGLEADER=false  当前节点是否为leader节点

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012130100.png)

一个组织中的leader节点可以手动设置，也可以让网络自动指定。但是手动设置的节点一旦故障那么网络不会自动替代，所以一般采用网络自动选举的机制CORE_PEER_GOSSIP_USELEADERELECTION环境变量就是设置是否由网络指定

CORE_PEER_GOSSIP_ORGLEADER=false 表明当前节点是否为leader节点，如果设置了网络自动选举，那么这一项就设置为false



修改docker-compose-cli.yaml   docker-compose-base.yaml   peer-base.yaml

```yaml
# docker-compose-cli文件中的peer部分
peer0.orggo.xwj.com:
    container_name: peer0.orggo.xwj.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer0.orggo.xwj.com
    networks:
      - byfn

  peer1.orggo.xwj.com:
    container_name: peer1.orggo.xwj.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer1.orggo.xwj.com
    networks:
      - byfn

  peer0.orgcpp.xwj.com:
    container_name: peer0.orgcpp.xwj.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer0.orgcpp.xwj.com
    networks:
      - byfn

  peer1.orgcpp.xwj.com:
    container_name: peer1.orgcpp.xwj.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer1.orgcpp.xwj.com
    networks:
      - byfn

# docker-compose-base.yaml文件中的peer部分
  peer0.orggo.xwj.com:
    container_name: peer0.orggo.xwj.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.orggo.xwj.com
      - CORE_PEER_ADDRESS=peer0.orggo.xwj.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.orggo.xwj.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.orggo.xwj.com:7051
      - CORE_PEER_LOCALMSPID=OrgGoMSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls:/etc/hyperledger/fabric/tls
        - peer0.orggo.xwj.com:/var/hyperledger/production
    ports:
      - 7051:7051
      - 7053:7053

  peer1.orggo.xwj.com:
    container_name: peer1.orggo.xwj.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.orggo.xwj.com
      - CORE_PEER_ADDRESS=peer1.orggo.xwj.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.orggo.xwj.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.orggo.xwj.com:7051
      - CORE_PEER_LOCALMSPID=OrgGoMSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/orggo.xwj.com/peers/peer1.orggo.xwj.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/orggo.xwj.com/peers/peer1.orggo.xwj.com/tls:/etc/hyperledger/fabric/tls
        - peer1.orggo.xwj.com:/var/hyperledger/production

    ports:
      - 8051:7051
      - 8053:7053

  peer0.orgcpp.xwj.com:
    container_name: peer0.orgcpp.xwj.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.orgcpp.xwj.com
      - CORE_PEER_ADDRESS=peer0.orgcpp.xwj.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.orgcpp.xwj.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.orgcpp.xwj.com:7051
      - CORE_PEER_LOCALMSPID=OrgCppMSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/orgcpp.xwj.com/peers/peer0.orgcpp.xwj.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/orgcpp.xwj.com/peers/peer0.orgcpp.xwj.com/tls:/etc/hyperledger/fabric/tls
        - peer0.orgcpp.xwj.com:/var/hyperledger/production
    ports:
      - 9051:7051
      - 9053:7053

  peer1.orgcpp.xwj.com:
    container_name: peer1.orgcpp.xwj.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.orgcpp.xwj.com
      - CORE_PEER_ADDRESS=peer1.orgcpp.xwj.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.orgcpp.xwj.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.orgcpp.xwj.com:7051
      - CORE_PEER_LOCALMSPID=OrgCppMSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/orgcpp.xwj.com/peers/peer1.orgcpp.xwj.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/orgcpp.xwj.com/peers/peer1.orgcpp.xwj.com/tls:/etc/hyperledger/fabric/tls
        - peer1.orgcpp.xwj.com:/var/hyperledger/production
    ports:
      - 10051:7051
      - 10053:7053
                                
# peer-base.yaml


version: '2'

services:
  peer-base:
    image: hyperledger/fabric-peer:latest
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_byfn
      - CORE_LOGGING_LEVEL=INFO
      #- CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
```

### 5.3.5 配置文件结构目录

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012162439.png)

### 5.3.6 docker-compose 启动配置文件中的镜像

现在first-network中测试：

```shell
cd first-network
sudo ./byfn.sh down  		# 先关闭以防止之前开启过
sudo ./byfn.sh generate  	# 创建证书，因为配置文件中需要使用
docker-compose -f docker-compose-cli.yaml up -d    	# 使用配置文件运行各个镜像
docker-compose -f docker-compose-cli.yaml ps		# 查看，全部为绿色up状态才表示镜像开启成功
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012135222.png)

注意： 这里的done并不代表着就启动成功了，必须还要去查看

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012135328.png)

 查看，全部为**绿色up**状态才表示镜像开启成功

---

**启动自己的构建的网络配置文件：**

将docker-compose-cli文件更名为docker-compose，为了方便使用docker-compose命令，不用在敲-f  文件名

创建文件夹，把 docker-compose-base.yaml   peer-base.yaml两个文件放到base文件夹下（适应配置文件中的数据卷映射）

```shell
# 先关闭之前所有已运行的镜像
docker stop $(docker ps -qa)
# 或者
docker-compose down -v
# 检查是否还有相关镜像运行
docker ps
# 没有后开始运行配置文件 
# 在包含docker-compose.yaml文件的目录下运行，会自动找docker-compose.yaml文件执行
docker-compose up -d
# 检查运行状态
docker-compose ps
```

运行成功则出现：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012164042.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012165059.png)

COMPOSE_PROJECT_NAME是在peer-base.yaml中设置的

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012165312.png)

test01_byfn   => network网络名字是  [当前目录_配置文件中的名字]

所以，我们可以设置环境变量：

```shell
vim ~/.bashrc
# 加入这一行
export COMPOSE_PROJECT_NAME=xwj_fabric_12  # 自己取名   不要用特殊的符号，小写的字母加数字其实就可以了
# 保存
source ~/.bashrc
# 重新运行
docker-compose up -d
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201012165958.png)

大功告成！

## 5.4 客户端操作peer节点  命令：peer

### 5.4.0 客户端对peer节点的总体操作流程

* 创建通道，通过客户端节点来完成

  * 进入到客户端节点来完成

* 将每个组织中的每个节点都加入到通道中，也是通过客户端完成

  * 以客户端同时只能连接一个peer节点

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013124520.png)

    所以需要<u>改变客户端的配置属性</u>连接到不同的客户端，实现一个一个的加入到通道中

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013124731.png)

* 给<font color='red'>每个</font>peer节点安装链码(智能合约) -> 链代码程序：go nodejs java

* 对智能合约进行初始化，对应智能合约中的init函数

  * <font color="red">智能合约只需要初始化一次，在任意节点均可，数据会自动同步到各个组织的各个节点中</font>

* 对数据进行查询（读），对数据进行交易（写）

### 5.4.1 创建channel通道

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013175611.png)

**这个动作需要在cli容器中运行，也就是上面配置好的文件，docker-compose启动后的运行容器cli中**

```shell
# 进入cli容器
docker exec -it cli bash
# 创建通道
peer channel create -o ubuntu.itcast.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA
# 参数说明
	-o 后面接连接的orederer的地址，hostname：port
	-$CHANNEL_NAME: 创建通道文件的时候写的IDchannel的id，没指定默认为mychannel，通道创建成功后会在磁盘生成一个文件：$CHANNEL_NAME.block
	-$CORE_PEER_TLS_ENABLED： 和docker通信时是否启用tls
	-$ORDERER_CA： 使用tls时，所使用的orderer节点的pem格式证书文件
peer channel create -o orderer.xwj.com:7050 -c xwjchannel -f ./channel-artifacts/channel.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/xwj.com/orderers/orderer.xwj.com/msp/tlscacerts/tlsca.xwj.com-cert.pem
```

orderer证书的<font color='orange'>绝对路径目录</font>（$ORDERER_CA），可按同样的方式去以下目录查找：（此目录是在镜像中的目录，与本机目录无关）

`/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/xwj.com/orderers/orderer.xwj.com/msp/tlscacerts/tlsca.xwj.com-cert.pem`

**注意：必须先进入cli容器中再执行创建通道的命令**

**注意：如果提示路径错误请检查！存放证书的文件夹我们之前命名为crypto-config，映射到cli镜像中则变成了crypto，所以命令中需要注意。**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013115850.png)

**注意：$CHANNEL_NAME不能为出现大写字母，不然会报错！（只能是小写或者数字或者破折号）**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013120836.png)

重新创建通道文件（命令可见上方），命名规范要注意，然后再次执行即可：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013121252.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013121508.png)

显示0 block（创世区块）则说明创建成功！

### 5.4.2 加入通道

```shell
$ peer channel join[flags],常用参数为：
	-b, --blockpath: 通过peer channel create命令生成的通道文件
# example
$ peer channel join -b 生成的通道block文件
peer channel join -b xwjchannel.block
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013180545.png)

<font color='orange'>注意，加入到channel只有当前一个peer节点成功了，并不是所有的peer，还需要加入其他节点</font>

加入其它的节点，通道无需重新创建，只需要修改当前客户端的环境变量，使其连接其他的peer节点就可以了

具体改变只有一下内容：

例：go组织的二个节点，在cli容器中输入以下命令：

```shell
export CORE_PEER_ADDRESS=peer0.orggo.xwj.com:7051
export CORE_PEER_LOCALMSPID=OrgGoMSP
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/users/Admin@orggo.xwj.com/msp
```

```shell
export CORE_PEER_ADDRESS=peer1.orggo.xwj.com:7051
export CORE_PEER_LOCALMSPID=OrgGoMSP
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer1.orggo.xwj.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer1.orggo.xwj.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer1.orggo.xwj.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/users/Admin@orggo.xwj.com/msp
```

cpp组织的两个节点：

```shell
export CORE_PEER_ADDRESS=peer0.orgcpp.xwj.com:7051
export CORE_PEER_LOCALMSPID=OrgCppMSP
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer0.orgcpp.xwj.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer0.orgcpp.xwj.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer0.orgcpp.xwj.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/users/Admin@orgcpp.xwj.com/msp
```

```shell
export CORE_PEER_ADDRESS=peer1.orgcpp.xwj.com:7051
export CORE_PEER_LOCALMSPID=OrgCppMSP
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer1.orgcpp.xwj.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer1.orgcpp.xwj.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer1.orgcpp.xwj.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/users/Admin@orgcpp.xwj.com/msp
```



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013182055.png)

显示成功，则此节点完成。

一个一个节点这样操作即可

### 5.4.3 更新锚节点

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013210511.png)



锚节点的指定在docker-compose.yaml中就指定好了，要更换才需要操作，非必需操作

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013210541.png)

锚节点的更新文件路径：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013210719.png)

### 5.4.4 安装链码

准备工作：

```shell
# 复制first-network中的一个链码到项目目录
cp ~/fabric-1.2/fabric-samples/chaincode/chaincode_example02/go/chaincode_example02.go chaincode/
# 重命名 
cd chaincode/
mv chaincode_example02.go test.go
# 进入客户端cli
docker exec -it cli bash
# 查看映射文件是否已经存在
ls /opt/gopath/src/github.com/chaincode/
# 发现重命名的文件存在则说明已经成功
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013191347.png)

---

```shell
# 安装链码
peer chaincode install [flags], 常用参数为:
	-c, --ctor: Json格式的构造函数，默认是“{}”
	-l, --lang: 编写链码的编程语言，默认是golang
	-n, --name: 链码的名字
	-p, --path: 链码源代码的目录，从$GOPATH/src 路径后开始写
	-v, --version: 当前操作的链码的版本，适用于这些命令：install/instantiate/upgrade
# example
peer chaincode install -n 链码的名字 -v 链码的版本 -l 链码的语言 -p 链码的位置
	- 链码名字自定义
	- 链码的版本，自己根据实际情况去指定
	- 这里的路径不是写完全的绝对路径，而是从$GOPATH/src 路径后开始写 
	
peer chaincode install -n testcc -v 1.0 -l golang -p github.com/chaincode/
# 安装完之后的检查
peer chaincode list --installed
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013192356.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013203716.png)

返回响应为<font color='green'>200, OK</font>则代表成功

<font color='red'>注意，每个peer节点都需要安装链码！初始化只需要初始化一个就可以,和上面加入通道同样的操作（修改配置，一个一个安装）</font>

### 5.4.5 链码初始化

<font color='orange'>注意链码的初始化只需要在一个节点上即可，其后会自动进行同步</font>

```shell
peer chaincode instantiate [flags], 常用参数为:
	-C, --channelID: 当前命令运行的通道，默认值是“testchainid”
	-c, --ctor: JSON格式的构造参数，默认值是“{}”
	-l, --lang: 编写链码的编程语言，默认是golang
	-n, --name: 链码的名字
	-P, --policy: 当前Chaincode的背书策略
	-v, --version: 当前操作的链码的版本，适用于这些命令：install/instantiate/upgrade
	--tls: 通信时是否使用tls加密
	--cafile: 当前orderer节点pem格式的tls证书文件，要使用绝对路径
# example
# -c '{"Args":["init","a","100","b","200"]}'			# 给构造函数传参数
# -P "AND ('OrgGoMASP.member', 'OrgCppMSP.member')"    	
# AND表示这两个组织中的成员都需要参与，member表示组织中任一节点均可。如果是“OR”则表示任何一个组织成员都可以
peer chaincode instantiate -o orderer节点地址：端口 --tls true --cafile orderer节点pem格式的证书文件 -C 通道名称 -n 链码名称 -l 链码语言 -v 链码语言 -v 链码版本 -c 链码函数调用 -P 背书策略
```

背书策略：交易的规则，哪些人参与这笔交易，确定后进行模拟交易（测试）

<font color='orange'>注意这里的背书策略目前是初始化的指定作用，真正使用还需要到交易产生的时候</font>

```shell
peer chaincode instantiate -o orderer.xwj.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/xwj.com/orderers/orderer.xwj.com/msp/tlscacerts/tlsca.xwj.com-cert.pem -C xwjchannel -n testcc -l golang -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "AND ('OrgGoMSP.member', 'OrgCppMSP.member')"
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013204651.png)

<font color='red'>此处错误集合：</font>

1. <font color='red'>err Proposal response was not successful, error code 500, msg cannot get package for chaincode (testcc:1.0)</font>

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013205252.png)

   解决办法：

   检查链码是否布置安装存在

   `peer chaincode list --installed`

   不存在的话先安装。<font color='orange'>退出cli容器要从创建channel通道开始重头再来</font>

2. <font color='red'>API error (404): network xwj_fabric_1.2_byfn not found</font>

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013205659.png)

   peer-base.yaml文件中的网络设置环境变量与启动网络的配置不一致，我之前`export COMPOSE_PROJECT_NAME=xwj_fabric_1.2`  这个“.”导致了问题，所以自己取名不要用“点”符号

   重新启动网络，重新操作

### 5.4.6 查询

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201014152848.png)

```shell
# 查询账户A余额
peer chaincode query -C xwjchannel -c '{"Args":["query","a"]}' -n testcc
# 查询账户B的余额
peer chaincode query -C xwjchannel -c '{"Args":["query","b"]}' -n testcc
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201015180917.png)

### 5.4.7 交易

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201015181218.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201015181257.png)

```shell
peer chaincode invoke -o orderer.xwj.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/xwj.com/orderers/orderer.xwj.com/msp/tlscacerts/tlsca.xwj.com-cert.pem -C xwjchannel -n testcc --peerAddresses peer0.orggo.xwj.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orggo.xwj.com/peers/peer0.orggo.xwj.com/tls/ca.crt --peerAddresses peer1.orgcpp.xwj.com:7051 --tlsRootCertFiles  /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/orgcpp.xwj.com/peers/peer1.orgcpp.xwj.com/tls/ca.crt -c '{"Args": ["invoke", "a", "b", "10"]}'
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201016195608.png)

### 5.4.8 自动化shell脚本

先进入客户端

deploy.sh 

```shell
# 将各个组织的所有节点加入到一个通道中，对每个节点部署链码，并对其中一个初始化链码
# 变量
CHANNEL_ID=xwjchannel 								# 通道id
CHANNEL_FIEL=./channel-artifacts/channel.tx			# 通道文件
ORDER_CAFILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/xwj.com/orderers/orderer.xwj.com/msp/tlscacerts/tlsca.xwj.com-cert.pem					# order的tls证书

# 组织
ORG01_NAME=orggo.xwj.com
ORG01_MSPID=OrgGoMSP

ORG02_NAME=orgcpp.xwj.com
ORG02_MSPID=OrgCppMSP

# 节点配置
ORDERER=orderer.xwj.com
ORDERER_PORT=7050
ORG01_PEERS=(peer0.$ORG01_NAME peer1.$ORG01_NAME)
ORG02_PEERS=(peer0.$ORG02_NAME peer1.$ORG02_NAME)
PEERS_PORT=7051


# 链码配置
CHAINCODE_NAME=testcc
CHAINCODE_VERSION=1.0
CHAINCODE_LANG=golang
CHAINCODE_PATH=github.com/chaincode/

# 背书策略
ENDORSE_POLICY="AND('$ORG01_MSPID.member','$ORG02_MSPID.member')"

# 初始化链码函数调用
INIT_FUNC='{"Args":["init","a","100","b","200"]}'


# 创建通道
peer channel create -o $ORDERER:$ORDERER_PORT -c $CHANNEL_ID -f $CHANNEL_FIEL --tls true --cafile $ORDER_CAFILE
if [ -f $CHANNEL_ID.block ];then
	echo -e "\033[32m create channel: $CHANNEL_ID path: $CHANNEL_FIEL ==> OK! \033[0m"
else
	echo -e "\033[31m create channel: $CHANNEL_ID ==> ERR! \033[0m"
fi	


# 遍历各个组织 给每个节点加入通道和安装链码
# 对于第一个组织Org1
for PEER in ${ORG01_PEERS[@]};do
    # 切换配置（换节点）
    export CORE_PEER_ADDRESS=$PEER:$PEERS_PORT
    export CORE_PEER_LOCALMSPID=$ORG01_MSPID
    export CORE_PEER_TLS_ENABLED=true
    export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG01_NAME/peers/$PEER/tls/server.crt
    export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG01_NAME/peers/$PEER/tls/server.key
    export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG01_NAME/peers/$PEER/tls/ca.crt
    export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG01_NAME/users/Admin@$ORG01_NAME/msp
    
	# 加入通道并且安装链码
    echo -e "\033[32m $CORE_PEER_ADDRESS Configuring...... \033[0m"
    peer channel join -b $CHANNEL_ID.block
    peer chaincode install -n $CHAINCODE_NAME -v $CHAINCODE_VERSION -l $CHAINCODE_LANG -p $CHAINCODE_PATH
    echo -e "\033[32m $PEER Add channel: $CHANNEL_ID ==> OK! \033[0m"
    echo -e "\033[32m $PEER Install chaincode: $CHAINCODE_NAME ==> OK! \033[0m"
done

# 对于第二个组织
for PEER in ${ORG02_PEERS[@]};do
    # 切换配置（换节点）
    export CORE_PEER_ADDRESS=$PEER:$PEERS_PORT
    export CORE_PEER_LOCALMSPID=$ORG02_MSPID
    export CORE_PEER_TLS_ENABLED=true
    export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG02_NAME/peers/$PEER/tls/server.crt
    export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG02_NAME/peers/$PEER/tls/server.key
    export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG02_NAME/peers/$PEER/tls/ca.crt
    export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/$ORG02_NAME/users/Admin@$ORG02_NAME/msp

	# 加入通道并且安装链码
    echo -e "\033[32m $CORE_PEER_ADDRESS Configuring...... \033[0m"
    peer channel join -b $CHANNEL_ID.block
    peer chaincode install -n $CHAINCODE_NAME -v $CHAINCODE_VERSION -l $CHAINCODE_LANG -p $CHAINCODE_PATH
    echo -e "\033[32m $PEER Add channel: $CHANNEL_ID ==> OK! \033[0m"
    echo -e "\033[32m $PEER Install chaincode: $CHAINCODE_NAME ==> OK! \033[0m"
done


# 对其中一个节点初始化链码
peer chaincode instantiate -o $ORDERER:$ORDERER_PORT --tls true --cafile $ORDER_CAFILE -C $CHANNEL_ID -n $CHAINCODE_NAME -l $CHAINCODE_LANG -v $CHAINCODE_VERSION -c $INIT_FUNC -P $ENDORSE_POLICY
echo -e "\033[32m $CORE_PEER_ADDRESS Instantiate chaincode: $CHAINCODE_NAME ==> OK! \033[0m"
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201016221237.png)

爽的一p

# Tips

## 1. 从Linux的vim中复制内容到系统的剪贴板

```shell
# 下载第三方插件
sudo apt-get install vim-gnome
# 再使用yy命令即可
```

注意，通过远程访问的无法复制到远程访问主机的剪贴板

## 2. 重复创建通道出错

<font color='red'>error validating ReadSet: readset expected key [Group]  /Channel/Application at version 0, but got version 1</font>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201013185923.png)

原因：旧的通道还没删除，新的通道重复创建

解决办法，重新部署：

```shell
$ docker-compose down -v
$ docker volume prun
$ docker-compose up -d
```

## 3.注意docker数据卷不是实时共享的

进入客户端内部后，在外部修改内部数据卷关联文件，docker容器内部是不会发生改变的，需要重新启动。