---
title: P2P网络编程-3-案例实践：PubSub
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-09-26 13:59:04

---

libp2p网络通信中还有一种方式就是PubSub模式，也称订阅发布的模式，官方给出了订阅发布模式的一个案例=> 聊天室

在此学习记录一下

官方代码地址：https://github.com/libp2p/go-libp2p/tree/master/examples/pubsub

<!-- more -->

# 一、效果演示

![chat-example](http://xwjpics.gumptlu.work/qinniu_uPic/chat-example.gif)

# 二、代码理解

## 2.1 总体框架

总的来说代码构成由这五个步骤：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210926142032549.png" alt="image-20210926142032549" style="zoom:50%;" />

1~2两步较为简单不再赘述, 下面几点分点描述

## 2.2 创建mDNS节点发现服务

注意，使用mDNS作为节点发现需要保证所有的节点在同一个局域网即节点发现的范围在同一个局域网下

```go
// discoveryNotifee gets notified when we find a new peer via mDNS discovery
// 节点发现的通告结构体，继承Notifee
type discoveryNotifee struct {
	h host.Host
}

// HandlePeerFound connects to peers discovered via mDNS. Once they're connected,
// the PubSub system will automatically start interacting with them if they also
// support PubSub.
// 继承函数，节点发现后的处理函数：自动链接节点
func (n *discoveryNotifee) HandlePeerFound(pi peer.AddrInfo) {
	fmt.Printf("discovered new peer %s\n", pi.ID.Pretty())
	err := n.h.Connect(context.Background(), pi)
	if err != nil {
		fmt.Printf("error connecting to peer %s: %s\n", pi.ID.Pretty(), err)
	}
}

// setupDiscovery creates an mDNS discovery service and attaches it to the libp2p Host.
// This lets us automatically discover peers on the same LAN and connect to them.
func setupDiscovery(ctx context.Context, h host.Host) error {
	// setup mDNS discovery to find local peers
	disc := mdns.NewMdnsService(h, DiscoveryServiceTag)

	n := discoveryNotifee{h: h}
	disc.RegisterNotifee(&n)
	return nil
}
```

## 2.3 加入聊天房

每个节点通过订阅房间的统一topic实现PubSub

### 1.聊天房数据结构

与一个topic一一对应，可以通过ChatRoom.Publish在topic中发布消息，并且接收所有的消息到Messages的channel中。

```go
type ChatRoom struct {
	// Messages is a channel of messages received from other peers in the chat room
	Messages chan *ChatMessage

	ctx   context.Context
	ps    *pubsub.PubSub
	topic *pubsub.Topic
	sub   *pubsub.Subscription

	roomName string
	self     peer.ID
	nick     string
}

// ChatMessage gets converted to/from JSON and sent in the body of pubsub messages.
type ChatMessage struct {
	Message    string
	SenderID   string
	SenderNick string
}

func (cr *ChatRoom) Publish(message string) error {
	m := ChatMessage{
		Message:    message,
		SenderID:   cr.self.Pretty(),
		SenderNick: cr.nick,
	}
	msgBytes, err := json.Marshal(m)
	if err != nil {
		return err
	}
	return cr.topic.Publish(cr.ctx, msgBytes)
}
```

### 2.加入聊天房逻辑

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210926143906101.png" alt="image-20210926143906101" style="zoom:50%;" />

分为三步，成功后返回一个新的ChatRoom实例

```go
func JoinChatRoom(ctx context.Context, ps *pubsub.PubSub, selfID peer.ID, nickname string, roomName string) (*ChatRoom, error) {
	// join the pubsub topic
	topic, err := ps.Join(topicName(roomName))
	if err != nil {
		return nil, err
	}

	// and subscribe to it
	sub, err := topic.Subscribe()
	if err != nil {
		return nil, err
	}

	cr := &ChatRoom{
		ctx:      ctx,
		ps:       ps,
		topic:    topic,
		sub:      sub,
		self:     selfID,
		nick:     nickname,
		roomName: roomName,
		Messages: make(chan *ChatMessage, ChatRoomBufSize),
	}

	// start reading messages from the subscription in a loop
	go cr.readLoop()
	return cr, nil
}
```

发布和订阅较为直观，下面是循环读取：

### 3.循环读取消息内容

循环读取内容，并将被容加入到消息channel中

```go
// readLoop pulls messages from the pubsub topic and pushes them onto the Messages channel.
func (cr *ChatRoom) readLoop() {
	for {
		// Next returns the next message in our subscription
		// 找到下一个消息
		msg, err := cr.sub.Next(cr.ctx)
		if err != nil {
			close(cr.Messages)
			return
		}
		// only forward messages delivered by others
		// 只接收别人的消息
		if msg.ReceivedFrom == cr.self {
			continue
		}
		// 反序列化
		cm := new(ChatMessage)
		err = json.Unmarshal(msg.Data, cm)
		if err != nil {
			continue
		}
		// send valid messages onto the Messages channel
		// 把消息加入 Messages channel
		cr.Messages <- cm
	}
}
```

### 4.获取当前topic所有连接者

```go
func (cr *ChatRoom) ListPeers() []peer.ID {
  // ListPeers returns a list of peers we are connected to in the given topic.
	return cr.ps.ListPeers(topicName(cr.roomName))
}
```

对于UI部分不是重点，会使用即可

总体来说案例使用较为简单，可以快速上手！
