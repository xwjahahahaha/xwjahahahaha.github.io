---
title: grpc的使用
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-13 16:19:45
---

[TOC]


<!-- more -->

> 参考：
>
> * https://grpc.io/docs/languages/go/quickstart/

# 一、基本使用

详细的使用见：https://grpc.io/docs/languages/go/basics/

基本步骤如下：

- Define a service in a `.proto` file.
- Generate server and client code using the protocol buffer compiler.
- Use the Go gRPC API to write a simple client and server for your service.

## 为什么要使用grpc？

也就是grpc的优点，在于：

* grpc能够处理不同语言和环境之间的所有复杂通信
* 高效的序列化、简单的IDL和非常容易的接口更新

# 二、配置相关

## 2.1 加密配置

来自于：https://icebergu.com/archives/grpc-transport

### 不使用任何加密

客户端创建连接的时候默认必须使用加密传输，否则会直接报错

```bash
15:59:53 did not connect: grpc: no transport security set (use grpc.WithInsecure() explicitly or set credentials)
exit status 1
```

**禁用安全传输**

```go
// client
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
```

### 开启 TLS/SSL 加密

TLS 是一种常见的端对端安全传输协议 grpc 可以使用 [credentials.TransportCredentials](https://godoc.org/google.golang.org/grpc/credentials#TransportCredentials) 结构来方便的开启安全传输

#### 服务端

使用 `credentials.NewServerTLSFromFile` 生成 `credentials.TransportCredentials` 调用 `grpc.NewServer` 时传递 [grpc.Creds](https://godoc.org/google.golang.org/grpc#Creds)

```go
creds, err := credentials.NewServerTLSFromFile(certFile, keyFile)
if err != nil {
  log.Fatalf("Failed to generate credentials %v", err)
}
 
lis, err := net.Listen("tcp", ":0")
server := grpc.NewServer(grpc.Creds(creds))
 
// ignore Serve error
server.Serve(lis)
```

#### 客户端

生成 `credentials.TransportCredentials`

```go
// serverNameOverride 用于测试， 通常为 "" 空字符串
func NewClientTLSFromCert(cp *x509.CertPool, serverNameOverride string) TransportCredentials
 
func NewClientTLSFromFile(certFile, serverNameOverride string) (TransportCredentials, error)
```

调用 `grpc.Dial` 时传递 `grpc.WithTransportCredentials`

```go
creds, _ := credentials.NewClientTLSFromFile(certFile, "")
conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
```
