---
title: P2P网络编程-2-案例实践：P2P聊天应用
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-09-05 15:38:17
---

[TOC]

上一节学习了IPFS项目中P2P网络的构建工具libp2p的基本概念以及简单案例

接下来通过官方的聊天案例进一步了解该项目的使用

<!-- more -->

项目地址： https://github.com/libp2p/go-libp2p/tree/master/examples

我们从初代版本（手动发送接收）p2p聊天到具有节点发现功能的完全去中心化聊天学习libp2p的使用

# 一、初代版本

## 1.1 简介

项目使用Libp2p实现了一个简单的p2p聊天应用。该项目应用在两个节点中，需要至少满足下列条件一个：

* 都有一个私有IP地址（在同一个网络下）
* 至少有一个节点有一个公网地址

## 1.2 代码与解析

已将所有的注解写在代码中，可直接查看代码`p2pChat.go`

```go
package main

import (
	"bufio"
	"context"
	"crypto/rand"
	"flag"
	"fmt"
	"github.com/libp2p/go-libp2p"
	"github.com/libp2p/go-libp2p-core/crypto"
	"github.com/libp2p/go-libp2p-core/host"
	"github.com/libp2p/go-libp2p-core/network"
	"github.com/libp2p/go-libp2p-core/peer"
	"github.com/libp2p/go-libp2p-core/peerstore"
	mutiaddr "github.com/multiformats/go-multiaddr"
	"io"
	"log"
	mrand "math/rand"
	"os"
)

//
//  handleStream
//  @Description: 流处理函数
//  @param s	新的数据流
//
func handleStream(s network.Stream)  {
	log.Println("获得了新的流！")
	// 根据新的流创建读写流
	rw := bufio.NewReadWriter(bufio.NewReader(s), bufio.NewWriter(s))
	
	// 创建两个携程分别读写数据
	go readData(rw)
	go writeData(rw)

	// 流s始终开启，直到流两端的任何一方关闭他
}

//
//  readData
//  @Description: 按行读取字符串输出
//  @param rw
//
func readData(rw *bufio.ReadWriter)  {
	for {
		s, _ := rw.ReadString('\n')
		if s == "" {
			return
		}
		if s != "\n" {
			fmt.Printf("\x1b[32m%s\x1b[0m>", s)
		}
	}
}

//
//  writeData
//  @Description: 获取标准输入，按行写入数据
//  @param rw
//
func writeData(rw *bufio.ReadWriter)  {
	// 1. 创建标准输入流
	stdReader := bufio.NewReader(os.Stdin)
	for {
		fmt.Printf(">")
		// 2. 读取标准输入
		s, err := stdReader.ReadString('\n')
		if err != nil {
			log.Panic(err)
			return
		}
		// 3. 使用读写流进行写入
		rw.WriteString(fmt.Sprintf("%s\n", s))
		rw.Flush()
	}
}

//
//  makeHost
//  @Description: 生成一个节点主机
//  @param ctx	上下文环境
//  @param port	端口
//  @param randomness 随机源
//  @return host.Host
//  @return error
//
func makeHost(ctx context.Context, port int, randomness io.Reader) (host.Host, error) {
	// 1. 使用RSA和随机数流创建密钥对
	privKey, _, err := crypto.GenerateKeyPairWithReader(crypto.RSA, 2048, randomness)
	if err != nil {
		log.Panic(err)
		return nil, err
	}

	// 2. 根据端口构建多重地址
	newMultiaddr, err := mutiaddr.NewMultiaddr(fmt.Sprintf("/ip4/0.0.0.0/tcp/%d", port))

	// 3. 创建节点
	return libp2p.New(ctx,
		libp2p.ListenAddrs(newMultiaddr),		// 设置地址
		libp2p.Identity(privKey),				// 设置密钥
	)
}

//
//  startPeer
//  @Description: 节点被连接时开启流处理协议，仅被使用在流接受端
//  @param ctx
//  @param host
//  @param streamHandler
//
func startPeer(host host.Host, streamHandler network.StreamHandler)  {
	// 1. 设置流处理函数
	host.SetStreamHandler("/chat/1.0.0", streamHandler)

	// 2. 获取实际节点的TCP端口
	var port string
	for _, la := range host.Network().ListenAddresses() {
		// 获取当前节点Tcp协议监听的端口号
		// ValueForProtocol作用：获取mutiAddr中特殊协议的特殊值
		if tcpPort, err := la.ValueForProtocol(mutiaddr.P_TCP); err == nil {
			port = tcpPort
			break
		}
	}

	// 3. 输出
	if port == "" {
		log.Println("不能够找到真实本地端口")
		return
	}
	// host.ID().Pretty(): 返回节点ID的base58编码
	log.Printf("运行 './p2pChat -d /ip4/127.0.0.1/tcp/%v/p2p/%s' 在另一个控制台", port, host.ID().Pretty())
	log.Println("你可以用公网IP代替127.0.0.1")
	log.Println("等待连接中...")
	log.Println()
}

//
//  startPeerAndConnect
//  @Description: 链接目标节点并创建流
//  @param ctx	上下文
//  @param host 主机节点
//  @param destination	目标节点字符串
//  @return *bufio.ReadWriter	缓冲读写流
//  @return error
//
func startPeerAndConnect(host host.Host, destination string) (*bufio.ReadWriter, error) {

	// 输出一下主机地址信息
	for _, la := range host.Addrs() {
		log.Printf(" - %v\n", la)
	}
	log.Println()

	// 1. 将目标节点地址转换为mutiaddr格式
	desMultiaddr, err := mutiaddr.NewMultiaddr(destination)
	if err != nil {
		log.Panic(err)
		return nil, err
	}

	// 2. 提取目标节点的Peer ID信息
	info, err := peer.AddrInfoFromP2pAddr(desMultiaddr)
	if err != nil {
		log.Panic(err)
		return nil, err
	}

	// 3. 将目标节点的信息（id和地址）加入当前主机节点的节点池（后面的创建链接、流都需要使用）
	host.Peerstore().AddAddrs(info.ID, info.Addrs, peerstore.PermanentAddrTTL)

	// 4. 向目标节点开启流
	// context.Background(): 是一个空环境，一般用于初始化、测试
	newStream, err := host.NewStream(context.Background(), info.ID, "/chat/1.0.0")
	if err != nil {
		log.Panic(err)
		return nil, err
	}
	log.Println("已建立目标节点的连接")

	// 5. 根据新的流创建缓冲读写流返回
	rw := bufio.NewReadWriter(bufio.NewReader(newStream), bufio.NewWriter(newStream))
	return rw, nil
}

func main()  {
	// 1. 创建程序上下文环境
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 2. 命令行程序逻辑
	sourcePort := flag.Int("sp", 0, "源端口号")
	dest := flag.String("d", "", "目标地址字符串")
	help := flag.Bool("h", false, "帮助")
	debug := flag.Bool("debug", false, "Debug模式：每次执行都生成相同的node ID")
	flag.Parse()	// 解析输入命令行

	// 帮助
	if *help {
		fmt.Printf("这是一个使用libp2p编写的简单P2P聊天程序\n\n")
		fmt.Println("Usage: Run './p2pChat -sp <SOURCE_PORT>' where <SOURCE_PORT> can be any port number.")
		fmt.Println("Now run './p2pChat -d <MULTIADDR>' where <MULTIADDR> is multiaddress of previous listener host.")
		os.Exit(0)
	}

	// debug模式
	var r io.Reader
	if *debug {
		// 创建公私钥的时候使用相同的随机源(sourcePort)使得每次生成相同的Peer ID，不要在正式环境使用
		// 注意：mrand是math/rand，下面的rand是crypto/rand
		r = mrand.New(mrand.NewSource(int64(*sourcePort)))
	}else {
		r = rand.Reader
	}

	// 创建主机
	h, err := makeHost(ctx, *sourcePort, r)
	if err != nil {
		log.Panic(err)
		return
	}

	// 开启主机
	if *dest == "" {
		// 接收连接方
		startPeer(h, handleStream) 		// 将自己写的流处理函数导入
	}else {
		// 发送连接方
		rw, err := startPeerAndConnect(h, *dest)
		if err != nil {
			log.Panic(err)
			return
		}

		// 创建携程读写
		go readData(rw)
		go writeData(rw)
	}

	// 一直阻塞
	select {}
}
```

## 1.3 测试运行

我的环境：

* 一个公网IP服务器：47.103.203.133
* 一个本地电脑

首先启动连接接受者（公网服务器）端口1234：

```shell
go build p2pChat.go 
./p2pChat -sp 1234
```

![WHeG9e](http://xwjpics.gumptlu.work/qinniu_uPic/WHeG9e.png)

然后启动连接发起者本地电脑，注意：将127.0.0.1换成服务器IP地址：

```sehll
./p2pChat -d /ip4/47.103.203.133/tcp/1234/p2p/QmSz6C8oMoNEAUJyQkBFbPXLZTGgRpmT85npqtYGAr9Vyb
```

![ggkAZZ](http://xwjpics.gumptlu.work/qinniu_uPic/ggkAZZ.png)

如此，两者就可以通话了～！

![4l6pgZ](http://xwjpics.gumptlu.work/qinniu_uPic/4l6pgZ.png)

# 二、节点发现

## 2.1 简介

上面的方法需要提前了解节点的地址才能够连接上，真正的p2p需要实现节点的自动发现

以下两个步骤在上面的初代版本中已经实现：

1. 创建上下文环境
2. 配置一个p2p host节点
3. 设置本地流默认处理函数

现在节点发现需要实现以下步骤：

4. **初始化一个新的$DHT$客户端与host主机对等**
5. **连接IPFS的引导节点（使用DHT发现附近的引导节点）**
6. **使用相同的键申明同一p2p节点网络（在聚会前约好地点）**
7. **寻找附近的对等点**
8. **向附近对等点开启stream**

## 2.2 代码与解析

`p2pChat.go`文件和详解如下：

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	"github.com/ipfs/go-log/v2"
	"github.com/libp2p/go-libp2p"
	"github.com/libp2p/go-libp2p-core/network"
	"github.com/libp2p/go-libp2p-core/peer"
	"github.com/libp2p/go-libp2p-core/protocol"
	discovery "github.com/libp2p/go-libp2p-discovery"
	dht "github.com/libp2p/go-libp2p-kad-dht"
	"github.com/multiformats/go-multiaddr"
	"os"
	"sync"
)

// 重制一个事件名称
var logger = log.Logger("rendezvous")

//
//  handleStream
//  @Description: 流处理函数
//  @param s	新的数据流
//
func handleStream(s network.Stream)  {
	logger.Info("获得了新的流！")
	// 根据新的流创建读写流
	rw := bufio.NewReadWriter(bufio.NewReader(s), bufio.NewWriter(s))

	// 创建两个携程分别读写数据
	go readData(rw)
	go writeData(rw)

	// 流s始终开启，直到流两端的任何一方关闭他
}

//
//  readData
//  @Description: 按行读取字符串输出
//  @param rw
//
func readData(rw *bufio.ReadWriter)  {
	for {
		s, _ := rw.ReadString('\n')
		if s == "" {
			return
		}
		if s != "\n" {
			fmt.Printf("\x1b[32m%s\x1b[0m>", s)
		}
	}
}

//
//  writeData
//  @Description: 获取标准输入，按行写入数据
//  @param rw
//
func writeData(rw *bufio.ReadWriter)  {
	// 1. 创建标准输入流
	stdReader := bufio.NewReader(os.Stdin)
	for {
		fmt.Printf(">")
		// 2. 读取标准输入
		s, err := stdReader.ReadString('\n')
		if err != nil {
			panic(err)
		}
		// 3. 使用读写流进行写入
		rw.WriteString(fmt.Sprintf("%s\n", s))
		rw.Flush()
	}
}

func main()  {
	// 1.设置log日志输出等级
	log.SetAllLoggers(log.LevelWarn)
	log.SetLogLevel("rendezvous", "info")

	// 2. 命令行
	help := flag.Bool("h", false, "帮助")
	// 解析
	config, err := ParseFlags()
	if err != nil {
		panic(err)
	}
	// 帮助
	if *help {
		fmt.Println("This program demonstrates a simple p2p chat application using libp2p")
		fmt.Println()
		fmt.Println("Usage: Run './p2pChat in two different terminals. Let them connect to the bootstrap nodes, announce themselves and connect to the peers")
		flag.PrintDefaults()
		return
	}

	// 3. 创建上下文
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 4. 创建本地节点
	host, err := libp2p.New(ctx,
		libp2p.ListenAddrs([]multiaddr.Multiaddr(config.ListenAddresses)...), // 如果配置了监听节点就初始化加入
	)
	if err != nil {
		panic(err)
	}
	logger.Info("创建本地节点: ", host.ID())
	logger.Info(host.Addrs())

	// 5. 开启本地节点流处理函数
	host.SetStreamHandler(protocol.ID(config.ProtocolID), handleStream)


	// 6. 初始化DHT客户端
	kademliaDHT, err := dht.New(ctx, host)
	if err != nil {
		panic(err)
	}

	// 7. 引导DHT客户端，让本地节点维护自己的DHT，这样就可以在未来没有引导节点时加入新的节点
	logger.Debug("引导DHT中...")
	if err := kademliaDHT.Bootstrap(ctx); err != nil {
		panic(err)
	}

	// 8. 连接引导节点
	var wg sync.WaitGroup // 创建多携程等待池
	for _, peerAddr := range config.BootstrapPeers {
		// 转换地址格式： mutiaddr -> AddrInfo
		peerInfo, _ := peer.AddrInfoFromP2pAddr(peerAddr)
		wg.Add(1)		// 加入携程
		go func() {
			defer wg.Done()	// 预先关闭携程
			// 连接任务
			if err := host.Connect(ctx, *peerInfo); err != nil {
				logger.Warn(err)
			}else {
				logger.Info("成功连接引导节点: ", peerInfo.ID)
			}
		}()
	}
	wg.Wait()			// 阻塞等待所有携程结束

	// 9. 使用相同的键申明同一p2p节点网络（在聚会前约好地点）
	logger.Info("正在声明当前网络...")
	// RoutingDiscovery是内容路由的一个实例
	routingDiscovery := discovery.NewRoutingDiscovery(kademliaDHT)
	// 持续的申明一个服务并用一个键唯一标记
	discovery.Advertise(ctx, routingDiscovery, config.RendezvousString)
	logger.Info("成功申明网络!唯一标识符:", config.RendezvousString)

	// 10. 在申明的网络中寻找其他节点
	logger.Info("正在寻找其他节点")
	peerChan, err := routingDiscovery.FindPeers(ctx, config.RendezvousString)	// 返回的是一个AddrInfo Channel
	if err != nil {
		panic(err)
	}
	// 遍历
	for peer := range peerChan {
		if peer.ID == host.ID() {
			continue
		}
		logger.Debug("找到新节点: ", peer.ID)
		logger.Debug("开始连接新节点: ", peer.ID)
		stream, err := host.NewStream(ctx, peer.ID, protocol.ID(config.ProtocolID))
		if err != nil {
			logger.Warn("连接节点 ", peer.ID, " 失败!")
			logger.Warn(err)
			continue
		}else {
			// 成功连接，创建读写流
			rw := bufio.NewReadWriter(bufio.NewReader(stream), bufio.NewWriter(stream))
			go writeData(rw)
			go readData(rw)
		}
		logger.Info("成功连接节点: ", peer.ID)
	}

	select {}
}
```

`flag.go`文件如下:

```go
package main

import (
	"flag"
	dht "github.com/libp2p/go-libp2p-kad-dht"
	maddr "github.com/multiformats/go-multiaddr"
	"strings"
)

// 重命名多地址数组类型
type addrList []maddr.Multiaddr

// 将flag命令输入数据存储在自定义结构体中需要实现Value接口，需要实现两个抽象函数String和Set
func (al *addrList) String() string {
	strs := make([]string, len(*al))
	for i, addr := range *al {
		strs[i] = addr.String()
	}
	return strings.Join(strs, ",")
}

func (al *addrList) Set(value string) error {
	addr, err := maddr.NewMultiaddr(value)
	if err != nil {
		return err
	}
	*al = append(*al, addr)
	return nil
}

func StringsToAddrs(addrStrings []string) (maddrs []maddr.Multiaddr, err error) {
	for _, addrString := range addrStrings {
		addr, err := maddr.NewMultiaddr(addrString)
		if err != nil {
			return maddrs, err
		}
		maddrs = append(maddrs, addr)
	}
	return
}

// 本地节点配置环境（代替数据库）
type Config struct {
	RendezvousString string
	BootstrapPeers   addrList		// 引导节点集合
	ListenAddresses  addrList		// 监听节点集合
	ProtocolID       string
}

func ParseFlags() (Config, error){
	config := Config{}
	flag.StringVar(&config.RendezvousString, "rendezvous", "19021902",
		"标识一组节点的唯一字符串。与你的朋友分享这些，让他们与你联系")
	flag.Var(&config.BootstrapPeers, "peer", "向当前节点添加一组引导节点数组（mutiaddress格式）")
	flag.Var(&config.ListenAddresses, "listen", "向当前节点添加一组监听节点数组（mutiaddress格式）")
	flag.StringVar(&config.ProtocolID, "pid", "/p2pChat/1.1.0", "给stram Hander设置一个协议号")
	flag.Parse()

	if len(config.BootstrapPeers) == 0 {
		// 如果没有设置引导节点，那么就使用dht默认的引导节点集合
		config.BootstrapPeers = dht.DefaultBootstrapPeers
	}
	return config, nil
}
```

## 2.3 测试运行

```shell
go build -o p2pChat ./
./p2pChat -listen /ip4/0.0.0.0/tcp/8888
# 另一台机器
./p2pChat -listen /ip4/0.0.0.0/tcp/6666
```



![T2JS2T](http://xwjpics.gumptlu.work/qinniu_uPic/T2JS2T.png)

![uRcN3n](http://xwjpics.gumptlu.work/qinniu_uPic/uRcN3n.png)

# 三、总结

## 3.1 libp2p节点发现构建流程





![libp2p_host](http://xwjpics.gumptlu.work/qinniu_uPic/libp2p_host.png)



## 3.2 libp2p中地址的转换关系

![libp2p_addr](http://xwjpics.gumptlu.work/qinniu_uPic/libp2p_addr.png)



> 学习资料来源：
>
> https://github.com/libp2p/go-libp2p/tree/master/examples/chat
>
> https://github.com/libp2p/go-libp2p/tree/master/examples/chat-with-rendezvous
