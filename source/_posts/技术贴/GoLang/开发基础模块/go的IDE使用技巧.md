---
title: goland的使用
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2021-03-06 16:43:53
---

# 一、Goland IDE

## 1. goland无法识别go mod的依赖包

解决方法: 在编辑器设置GOPROXY, 这样goland就会识别索引你项目使用go mod在`GOPATH/pkg/mod`文件夹下的依赖包

<!-- more -->

![PtyEVd](http://xwjpics.gumptlu.work/qinniu_uPic/PtyEVd.png)

## 2. 使用go mod模式导致本地包无法导入问题

因为使用go module进行包管理, 所以引用包路径不再是从`GOPATH/src`开始的相对位置

1. 查看你项目下的go.mod文件

2. 第一行的modul名就是你本地原始包的前缀名

   ![wfe1hF](http://xwjpics.gumptlu.work/qinniu_uPic/wfe1hF.png)

3. 修改项目中的引用即可

   `import "github.com/user/nameservice/[本地包名/路径]"`

## 3. goland调试

### 简单调试

1. 在断点代码处打上红色断点
2. 选择项目包右键Debugger
3. 选择 `go build …..`

3. F7下一步

### 详细配置

![JfUTLr](http://xwjpics.gumptlu.work/qinniu_uPic/JfUTLr.png)

![9Xjlvv](http://xwjpics.gumptlu.work/qinniu_uPic/9Xjlvv.png)

补充说明:

Run Kind 

* 如果是debugger整个项目包的话, 就选择Package
* 如果是单个go程序的话,就用file
* 不管哪一种,下面path路径都要正确

Program argument:

* 这个参数是编译好了之后的二进制文件执行的参数, 例如`./main.exe a b c` 这里就填写` a b c`

![LI32j8](http://xwjpics.gumptlu.work/qinniu_uPic/LI32j8.jpg)

### 远程调试

* 在远程主机上安装`dlv`

  `go get -u github.com/go-delve/delve/cmd/dlv`

* 在goland上创建`deployment`自动同步

* 在goland上创建`go remote`，填入远程IP和端口

  ![image-20220509162343261](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220509162343261.png)

* 在远程主机项目中运行：

  `dlv debug --headless --listen=:2345 --api-version=2 --accept-multiclient `

  此命令会编译并运行go项目，如果还附带参数的话，可以在后面加上参数，例如:

  `dlv debug --headless --listen=:2345 --api-version=2 --accept-multiclient -- [参数]`

  然后远程服务器就会监听2345端口（如果是云服务器记得打开防火墙规则）

* 在goland的代码中打断点，开始调试

## 4. 设置import顺序

设置自动编排import的顺序：

1. 标准包
2. 本地项目包
3. 外部包(`GitHub`)

按图示勾选上即可：

![image-20220510091446619](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220510091446619.png)

# 二、vscode

## 2.1 vscode实现远程调试golang程序

https://blog.csdn.net/turbock/article/details/102665187

### 1.开启dlv服务器

安装dlv：

```shell
go get -u github.com/derekparker/delve/cmd/dlv
#验证dlv安装
dlv version
```

在项目目录下：

```shell
dlv debug --headless --listen=:2345 --api-version=2 -- <执行文件的额外参数> 
```

如果需要`root`权限运行的话，可以通过下面的配置实现:

```shell
sudo go build -buildvcs=false ./ && sudo dlv exec ./<可执行文件> --headless --listen=:2345 --api-version=2 -- <执行文件的额外参数> 
```

* `--headless`指定只使用调试服务模式
* `--listen`指定了调试服务的端口，要与`vscode`的配置一致
* `--log`启用了调试日志输出，可关闭
* `--api-version 2`选择调试api的版本，这里必须指定为2，否则不能调试
* `-- `双减号加空格后填写程序启动的参数

也可以添加`--accept-multiclient`参数，让多个客户端同时连接此server进行调试或者单客户端的多次调试，但是这样会让服务端无法通过`ctrl+c`关闭，所以见仁见智

### 2. 配置vscode

点击调试栏中的齿轮按钮，配置调试连接参数。此时`vscode`中打开了 `launch.json `文件的编辑界面，这个文件存储在当前文件夹路径下的 `.vscode `目录中，里面可以存储多组调试配置

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [

        {
            "name": "xxxxx",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "remotePath": "${workspaceFolder}",
            "port": 2345,     // 对上server的端口
            "host": "10.219.204.206",
            "dlvLoadConfig": {
                "followPointers": true,
                "maxVariableRecurse": 1,
                "maxStringLen": 99999, //调试监视变量能显示的字符串最大长度
                "maxArrayValues": 64,
                "maxStructFields": -1
              }
        }
    ]
}
```

### 3. 调试快捷键

* F5 开始调试
* shift + F5 停止调试
* F9 设置/删除断点
* F10 单步跳过(不进入程序内部)
* F11 单步调试(进入内部一步步调试)
* shift + F11 跳出
