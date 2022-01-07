---
title: mqtt协议与go语言实现
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2021-07-12 09:11:57
---

> 学习资料：
>
> https://studygolang.com/articles/14452
>
> https://blog.csdn.net/jacky128256/article/details/105610456

# 一、什么是MQTT

**MQTT（Message Queuing Telemetry Transport，消息队列遥测传输协议）**，是一种**基于发布/订阅（publish/subscribe）模式**的“轻量级”通讯协议，该协议构建于TCP/IP协议上，由IBM在1999年发布。

MQTT最大优点在于，可以以极少的代码和有限的带宽，为连接远程设备提供实时可靠的消息服务。作为一种低开销、低带宽占用的即时通讯协议，使其在物联网、小型设备、移动应用等方面有较广泛的应用

MQTT是一个基于客户端-服务器的消息发布/订阅传输协议。MQTT协议是轻量、简单、开放和易于实现的，这些特点使它适用范围非常广泛。在很多情况下，包括受限的环境中，如：机器与机器（M2M）通信和物联网（IoT）。其在，通过卫星链路通信传感器、偶尔拨号的医疗设备、智能家居、及一些小型化设备中已广泛使用

> <font color='#39b54a'>MQTT还有一个特点就是客户端之间不用相互通信, MQTT通信更像是邮箱服务，发布者发布消息到服务器，接收者只要订阅了其服务在线后即可收到</font>

实现MQTT协议需要客户端和服务器端通讯完成，在通讯过程中，MQTT协议中有三种身份：**发布者（Publish）、代理（Broker）（服务器）、订阅者（Subscribe）**。其中，消息的发布者和订阅者都是客户端，消息代理是服务器，消息发布者可以同时是订阅者。

MQTT传输的消息分为：主题（Topic）和负载（payload）两部分：

（1）Topic，可以理解为消息的类型，订阅者订阅（Subscribe）后，就会收到该主题的消息内容（payload）；

（2）payload，可以理解为消息的内容，是指订阅者具体要使用的内容。

> <font color='#39b54a'>Topic就是消息名，payload就是消息体</font>

<!-- more -->

MQTT会构建底层网络传输：它将建立客户端到服务器的连接，提供两者之间的一个**有序的、无损的、基于字节流的双向传输。**

当应用数据通过MQTT网络发送时，MQTT会把与之相关的**服务质量（QoS）**和**主题名（Topic）**相关连。

# 二、Go语言MQTT服务器Broker的搭建

服务端用erlang编写的一个开源项目：emqqtd

```shell
# 下载安装包
wget https://github.com/emqx/emqx/releases/download/v4.0.4/emqx-ubuntu18.04-v4.0.4.zip
cd mqttd/emqx
.
├── bin
├── data
├── erts-10.5.2
├── etc
├── lib
├── log
└── releases
# 开启服务
./bin/emqx start
# 查看状态
./bin/emqx_ctl status
# 停止服务
./bin/emqx stop
```

找到自己的IP，访问`http://[你的IP]:18083/#/clients`

* 用户名：admin
* 密码：public

即可进入服务器的控制台

![RYTlik](http://xwjpics.gumptlu.work/qinniu_uPic/RYTlik.png)

# 三、Go客户端访问简单API

客户端用golang客户端的库：“github.com/eclipse/paho.mqtt.golang”

```shell
# 下载依赖包
go get -u github.com/eclipse/paho.mqtt.golang
```

实例如下：

编写了两个函数一个发布一个订阅，传入参数即可服务

修改`EMQServerAddress`为你服务器的IP

```go
package main

// 与后端mqtt服务交互

import (
	"fmt"
	mqtt "github.com/eclipse/paho.mqtt.golang"
	"log"
	"os"
	"strconv"
	"time"
)

const EMQServerAddress = "你的IP"

// 创建全局mqtt publish消息处理 handler
var messagePubHandler mqtt.MessageHandler = func(client mqtt.Client, msg mqtt.Message) {
	fmt.Println("Push Message:")
	fmt.Printf("TOPIC: %s\n", msg.Topic())
	fmt.Printf("MSG: %s\n", msg.Payload())
}

// 创建全局mqtt sub消息处理 handler
var messageSubHandler mqtt.MessageHandler = func(client mqtt.Client, msg mqtt.Message) {
	fmt.Println("收到订阅消息:")
	fmt.Printf("Sub Client Topic : %s \n", msg.Topic())
	fmt.Printf("Sub Client msg : %s \n", msg.Payload())
}

// 连接的回掉函数
var connectHandler mqtt.OnConnectHandler =func(client mqtt.Client) {
	fmt.Println("新的连接!" + " Connected")
}

// 丢失连接的回掉函数
var connectLostHandler mqtt.ConnectionLostHandler = func(client mqtt.Client, err error) {
	fmt.Printf("Connect loss: %v\n", err)
}

func init() {
	// 配置错误提示
	mqtt.DEBUG = log.New(os.Stdout, "		[mqttDEBUG]", 0)
	mqtt.ERROR = log.New(os.Stdout, " 	[mqttERROR]", 0)
}

/**
 * @Description: 发布订阅
 * @param clientID
 * @param addr
 * @param topic
 * @param payload
 */
func Push(topic string, qos byte, retain bool, payload string) {
	// opts ClientOptions 用于设置 broker，端口，客户端 id ，用户名密码等选项
	opts := mqtt.NewClientOptions().AddBroker("tcp://" + EMQServerAddress + ":1883").SetClientID("test_push")
	opts.SetKeepAlive(60 * time.Second)
	// Message callback handler，在没有任何订阅时，发布端调用此函数
	opts.SetDefaultPublishHandler(messagePubHandler)
	opts.SetPingTimeout(1 * time.Second)
	opts.OnConnect = connectHandler
	opts.OnConnectionLost = connectLostHandler
	client := mqtt.NewClient(opts)
	if token := client.Connect(); token.Wait() && token.Error() != nil {
		panic(token.Error())
	}
	//发布消息
	// qos是服务质量: ==1: 一次, >=1: 至少一次, <=1:最多一次
	// retained: 表示mqtt服务器要保留这次推送的信息，如果有新的订阅者出现，就会把这消息推送给它（持久化推送）
	token := client.Publish(topic, qos, retain, payload)
	token.Wait()
	fmt.Println("Push Data : "+topic, "Data Size is "+strconv.Itoa(len(payload)))
	fmt.Println("Disconnect with broker")
	client.Disconnect(250)
}

/**
 * @Description: 订阅或取消订阅
 * @param clientID
 * @param addr
 * @param topic
 * @param isSub
 */
func Subscription(topic string, qos byte, isSub bool, handleFun func([]byte)) {
	opts := mqtt.NewClientOptions().AddBroker("tcp://" + EMQServerAddress + ":1883").SetClientID("sub_test")
	opts.SetKeepAlive(60 * time.Second)
	opts.SetPingTimeout(1 * time.Second)
	opts.OnConnect = func(client mqtt.Client) {
		fmt.Println("New Subscription! Connected" + " => " + topic)
	}
	opts.OnConnectionLost = connectLostHandler
	client := mqtt.NewClient(opts)
	if token := client.Connect(); token.Wait() && token.Error() != nil {
		panic(token.Error())
	}

	if isSub {
		// 订阅消息
		if token := client.Subscribe(topic, qos, func(client mqtt.Client, msg mqtt.Message) {
			fmt.Printf("Receive Subscribe Message :")
			fmt.Printf("Sub Client Topic : %s, Data size is  %d \n", msg.Topic(), len(msg.Payload()))
			if len(msg.Payload()) > 0 {
				handleFun(msg.Payload())
			}
		}); token.Wait() && token.Error() != nil {
			fmt.Println(token.Error())
			os.Exit(1)
		}
	} else {
		// 取消订阅
		if token := client.Unsubscribe(topic); token.Wait() && token.Error() != nil {
			fmt.Println(token.Error())
			os.Exit(1)
		}
	}
}
```

