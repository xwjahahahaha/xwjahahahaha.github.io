---
title: m1芯片Mac无法调试Goland的解决方案
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-07-12 11:20:31
---

查询了很多资料文章，下面这篇给了启发，记录一下我的解决方案

https://blog.csdn.net/zsi386/article/details/116357850?spm=1001.2014.3001.5501

# 问题描述

我的环境：

* go version :  go1.16.5 darwin/arm64.  (下载时记得下载arm架构的)

* goland version: 2020.3

* mac version:Big Sur 11.4 MacBook Air m1芯片

  ![MvQldJ](http://xwjpics.gumptlu.work/qinniu_uPic/MvQldJ.png) 

<!-- more -->

调试出现的问题：

<font color='#e54d42'>可以断点停下来，但是无法**下一步和查看变量**，无报错</font>

> 断点都不可以停下的需要检查一下go的版本是否下载的arm架构
>
> https://studygolang.com/dl
>
> ![SjFTRk](http://xwjpics.gumptlu.work/qinniu_uPic/SjFTRk.png)

# 解决方法

下载go-delve/delve

`git clone https://github.com/go-delve/delve.git	`

下载慢的可以用这个：`git clone https://github.com.cnpmjs.org/go-delve/delve.git  `



可以把它放在了GOPATH下的src/github.com//go-delve下，进入clone下来的delve中（`cd delve`），切换分支：

`git checkout -b darwin-arm64-lldb`

然后进入工程目录`cd ./cmd/dlv/`，重新编译：

```shell
go build
go install
```

(不需要修改代码，目前1205 bug估计已修复)

会在你的`GOPATH/bin`下重新生成二进制文件`dlv`

我的版本信息：

```shell
Delve Debugger
Version: 1.6.1
Build: $Id: 114218c22f3791287c4bc2f4ff35a846a1416ee9 $
```

设置你的goland (`Help>Edit custom properties`)指向它就可以：

`dlv.path=/path/to/dlv` (<font color='#e54d42'>路径要改</font>)，然后可以debug了：

![DHncf0](http://xwjpics.gumptlu.work/qinniu_uPic/DHncf0.png)

![VWLHrO](http://xwjpics.gumptlu.work/qinniu_uPic/VWLHrO.png)

# 出现原因

dlv老版本bug：

Big Sur11.3，lldb成了1205，dlv处理了1200，没处理1205

更新版本即可
