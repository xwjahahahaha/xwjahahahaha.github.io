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
3. 选择 go build …..

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

