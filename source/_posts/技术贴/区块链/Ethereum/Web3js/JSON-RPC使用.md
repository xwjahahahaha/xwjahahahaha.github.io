---
title: JSON-RPC使用
tags:
  - block_chain
categories:
  - technical
  - block_chain
toc: true
declare: true
date: 2020-07-26 18:52:24
---

# JSON-RPC介绍

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726185018.png)

说白了就是规定了外部访问私链的方式：JSON-RPC

<!-- more -->

# 实例

## 启动私链

不会创建私链和私链操作的可以看文章[用Geth构建一个以太坊私有链](https://myblog.gumptlu.work/2020/07/21/%E6%8A%80%E6%9C%AF%E8%B4%B4/%E5%8C%BA%E5%9D%97%E9%93%BE/%E7%94%A8Geth%E6%9E%84%E5%BB%BA%E4%B8%80%E4%B8%AA%E4%BB%A5%E5%A4%AA%E5%9D%8A%E7%A7%81%E6%9C%89%E9%93%BE/)

注意启动的时候参数需要加上 --rpc

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726190728.png)

## 使用curl调用私链

什么是curl：

curl是利用URL语法在命令行方式下工作的开源文件传输工具。它被广泛应用在Unix、多种Linux发行版中，并且有DOS和Win32、Win64下的移植版本。

windows系统先下载：https://curl.haxx.se/windows/ 并将bin文件夹目录添加到环境变量即可使用。

成功测试：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726191948.png)

接下来在打开一个cmd，准备开始外部访问私有链：

输入：

`curl -X POST -H "Content-Type: application/json" --data "{\"jsonrpc\": \"2.0\", \"method\": \"web3_clientVersion\", \"params\":[], \"id\":1}" http://localhost:8545`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726204624.png)

***注意！！！！windows此处有坑：data数据里外都用双引号，大括号内部双引号加上转义字符！！***

这行命令的作用是调取私有链的客户端版本。

**对命令各个部分的解析：**

前面是post请求，`-H "Content-Type: application/json"`指明http请求头文件格式，--data参数指明json内容，其中`"jsonrpc": "2.0"`固定写法指明jsonrpc版本，`method`指明要调用的函数，**这里的函数也就是我们在console里面使用的那些函数，只不过.换成了_(下划线)**。`params`是函数参数，`id`是对此调用“通道”的唯一标识，被调用的客户端返回数据时返回的id也是相同的id。最后面是url地址。

其他函数调用

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726210400.png)

这与console中的是一样的：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200726210426.png)