---
title: gomod模式
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 19:11:31
---

> * http://c.biancheng.net/view/5712.html

# 一、 go mod包管理

## 1.1 Go Module综述

go module 是Go语言从 1.11 版本之后官方推出的版本管理工具，并且从 Go1.13 版本开始，go module 成为了Go语言默认的依赖管理工具。

<!-- more -->

Modules 官方定义为：

Modules 是相关 Go 包的集合，是源代码交换和版本控制的单元。Go语言命令直接支持使用 Modules，包括记录和解析对其他模块的依赖性，Modules 替换旧的基于 GOPATH 的方法，来指定使用哪些源文件。

==如何使用 Modules？==

1) 首先需要把 golang 升级到 1.11 版本以上（现在 1.13 已经发布了，建议使用 1.13）。

2) 设置 GO111MODULE。

==GO111MODULE==

在Go语言 1.12 版本之前，要启用 go module 工具首先要设置环境变量 GO111MODULE，不过在Go语言 1.13 及以后的版本则不再需要设置环境变量。通过 GO111MODULE 可以开启或关闭 go module 工具。

- GO111MODULE=off 禁用 go module，编译时会从 GOPATH 和 vendor 文件夹中查找包；
- GO111MODULE=on 启用 go module，编译时会忽略 GOPATH 和 vendor 文件夹，只根据 go.mod下载依赖；
- GO111MODULE=auto（默认值），当项目在 GOPATH/src 目录之外，并且项目根目录有 go.mod 文件时，开启 go module。


Windows 下开启 GO111MODULE 的命令为：

`set GO111MODULE=on 或者 set GO111MODULE=auto`

MacOS 或者 Linux 下开启 GO111MODULE 的命令为：

`export GO111MODULE=on 或者 export GO111MODULE=auto`

在开启 GO111MODULE 之后就可以使用 go module 工具了，也就是说在以后的开发中就没有必要在 GOPATH 中创建项目了，并且还能够很好的管理项目依赖的第三方包信息。

go mod 的常用命令：

| 命令            | 作用                                           |
| --------------- | ---------------------------------------------- |
| go mod download | 下载依赖包到本地（默认为 GOPATH/pkg/mod 目录） |
| go mod edit     | 编辑 go.mod 文件                               |
| go mod graph    | 打印模块依赖图                                 |
| go mod init     | 初始化当前文件夹，创建 go.mod 文件             |
| go mod tidy     | 增加缺少的包，删除无用的包                     |
| go mod vendor   | 将依赖复制到 vendor 目录下                     |
| go mod verify   | 校验依赖                                       |
| go mod why      | 解释为什么需要依赖                             |

<!-- more -->

## 1.2 Go PROXY

proxy 顾名思义就是代理服务器的意思。大家都知道，国内的网络有防火墙的存在，这导致有些Go语言的第三方包我们无法直接通过`go get`命令获取。GOPROXY 是Go语言官方提供的一种通过中间代理商来为用户提供包下载服务的方式。要使用 GOPROXY 只需要设置环境变量 GOPROXY 即可。

目前公开的代理服务器的地址有：

- goproxy.io；
- goproxy.cn：（推荐）由国内的七牛云提供。


Windows 下设置 GOPROXY 的命令为：

go env -w GOPROXY=https://goproxy.cn,direct

MacOS 或 Linux 下设置 GOPROXY 的命令为：

export GOPROXY=https://goproxy.cn,direct

Go语言在 1.13 版本之后 GOPROXY 默认值为 https://proxy.golang.org，在国内可能会存在下载慢或者无法访问的情况，所以十分建议大家将 GOPROXY 设置为国内的 goproxy.cn。

## 1.3 Go Get

使用go get命令下载指定版本的依赖包并自动完成编译和安装, 实际上是两步操作：

1. 下载源码包`git clone`(下载位置在`GOPATH/pkg`目录下，使用`go module`模式下载位置在`GOPATH/src/mod`下)
2. 执行`go install`安装(可执行二进制文件会保存在`GOPATH/bin`下)

执行`go get `命令，在下载依赖包的同时还可以指定依赖包的版本。

- 运行`go get -u`命令会将项目中的包升级到最新的次要版本或者修订版本；
- 运行`go get -u=patch`命令会将项目中的包升级到最新的修订版本；
- 运行`go get [包名]@[版本号]`命令会下载对应包的指定版本或者将对应包升级到指定的版本。

提示：`go get [包名]@[版本号]`命令中版本号可以是 x.y.z 的形式，例如 `go get foo@v1.2.3`，也可以是 git 上的分支或 tag，例如` go get foo@master`，还可以是 git 提交时的哈希值，例如` go get foo@e3702bed2`

* `-d`:让命令程序只执行下载动作，而不执行安装动作。
* `-f`:仅在使用`-u`标记时才有效。该标记会让命令程序忽略掉对已下载代码包的导入路径的检查(import)。如果下载并安装的代码包所属的项目是你从别人那里Fork过来的，那么这样做就尤为重要了。
* `-fix`:让命令程序在下载代码包后先执行修正动作，而后再进行编译和安装。
* `-insecure`:允许命令程序使用非安全的scheme（如HTTP）去下载指定的代码包。如果你用的代码仓库（如公司内部的Gitlab）没有HTTPS支持，可以添加此标记,请在确定安全的情况下使用它。
* `-t`:让命令程序同时下载并安装指定的代码包中的测试源码文件中依赖的代码包。
* `-u`:让命令利用网络来更新已有代码包及其依赖包。默认情况下，该命令只会从网络上下载本地不存在的代码包，而不会更新已有的代码包。
* `-v`:打印出被构建的代码包的名字
* `-x`:打印出用到的命令

## 1.4 go.mod与go.sum文件

go.mod 提供了 module、require、replace 和 exclude 四个命令：

- module 语句指定包的名字（路径）；
- require 语句指定的依赖项模块；
- replace 语句可以替换依赖项模块；
- exclude 语句可以忽略依赖项模块。

示例:

```go
module hello

go 1.13

require (
    github.com/labstack/echo v3.3.10+incompatible // indirect
    github.com/labstack/gommon v0.3.0 // indirect
    golang.org/x/crypto v0.0.0-20191206172530-e9b2fee46413 // indirect
)
```

go module 安装 package 的原则是先拉取最新的 release tag，若无 tag 则拉取最新的 commit，详见[ Modules 官方](https://github.com/golang/go/wiki/Modules)介绍。

go 会自动生成一个 go.sum 文件来**记录** dependency tree：

```go
github.com/davecgh/go-spew v1.1.0/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/labstack/echo v3.3.10+incompatible h1:pGRcYk231ExFAyoAjAfD85kQzRJCRI8bbnE7CX5OEgg=
github.com/labstack/echo v3.3.10+incompatible/go.mod h1:0INS7j/VjnFxD4E2wkz67b8cVwCLbBmJyDaka6Cmk1s=
github.com/labstack/gommon v0.3.0 h1:JEeO0bvc78PKdyHxloTKiF8BD5iGrH8T6MSeGvSgob0=
github.com/labstack/gommon v0.3.0/go.mod h1:MULnywXg0yavhxWKc+lOruYdAhDwPK9wf0OL7NoOu+k=
github.com/mattn/go-colorable v0.1.2 h1:/bC9yWikZXAL9uJdulbSfyVNIR3n3trXl+v8+1sx8mU=
... 省略很多行
```

可以使用命令`go list -m -u all`来检查可以升级的 package，使用`go get -u need-upgrade-package`升级后会将新的依赖版本更新到 go.mod * 也可以使用`go get -u`升级所有依赖。

**使用 replace 替换无法直接获取的 package**

由于某些已知的原因，并不是所有的 package 都能成功下载，比如：golang.org 下的包。

modules 可以通过在 go.mod 文件中使用 replace 指令替换成 github 上对应的库，比如：

```go
replace (
  golang.org/x/crypto v0.0.0-20190313024323-a1f597ede03a => github.com/golang/crypto v0.0.0-20190313024323-a1f597ede03a
)
```

或者

```go
replace golang.org/x/crypto v0.0.0-20190313024323-a1f597ede03a => github.com/golang/crypto v0.0.0-20190313024323-a1f597ede03a
```

## 1.5 Go Build多环境编译

https://www.cnblogs.com/Tiago/p/6409533.html

```shell
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ./
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build ./
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build ./
```

## 1.6 Go init

* 在GoPath下使用`go mod init`,默认生成的module 名称是从GoPath/src目录下开始算的
* 不在GoPath下, 需要使用`go mod init [example.com/m/v1]`自定义本地包名称, 包内import默认就是从此自定义module名下开始算 

## 1.7 GoMod模式本地包导入问题

* 出现情况: **多重go mod**

  使用本地私有包中含有mod, 项目总体还有一个go mod

  ![SEdKFD](http://xwjpics.gumptlu.work/qinniu_uPic/SEdKFD.png)

* 运行/编译错误:`Package xxxx is not in GOROOT`

  原因: 未识别本地包

* 解决方法:

  直接使用`go mod tidy` 会去网络上寻找自定义的本地包, 等很久报错(根本就没有), 所以要使用replace的方法

  1. 在<font color='#e54d42'>**项目最外层**</font>的go mod中将内层所有的含有go mod的私有包都做类似于如下repalce:

     `replace test/hello => ./hello`

  2. `go mod tidy`

     => `go: found test/hello in test/hello v0.0.0-00010101000000-000000000000`

  3. 再次运行,成功

* 建议: 项目最好在最外层使用一个go mod文件



