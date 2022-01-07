---
title: mac_m1编程环境搭建以及适配情况总结
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-02-13 17:02:14
---

# mac_m1编程环境搭建以及适配情况总结

## 一、基础

mac m1的环境变量配置文件在`/etc/zshrc`

big sur版本11.2.1环境变量配置文件在`~/.zshrc`

一般网络上下载的安装包后缀为`dmg`是适配的

搜索目前是否有支持arm架构的软件环境网站：https://doesitarm.com

mac专门包管理工具HomeBrew（安装方式见下方）：https://formulae.brew.sh

Homebrew是一款Mac OS平台下的软件包管理工具，拥有安装、卸载、更新、查看、搜索等很多实用的功能。简单的一条指令，就可以实现包管理，而不用你关心各种依赖和文件路径的情况，十分方便快捷。

<!-- more -->

## 二、安装HomeBrew

原文：https://blog.csdn.net/qq_29101773/article/details/112425894

HomeBrew的所有问题这篇文章基本都有：https://mintimate.cn/2020/04/05/Homebrew/ 

打开终端 (文件夹位置得在/opt/homebrew)

```
cd /opt

mkdir homebrew

curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
```

下载完就ok赖

之后使用homebrew装软件时可能会报如下错

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210110112920144.png#pic_center)

按照提示 获取当对应目录权限即可

`sudo chown -R $(whoami) /opt/homebrew`

安装完成后 brew -v 查看安装是否成功
若提示找不到brew
需要在zshrc里手动加上(macOS Big Sur默认是使用zsh,使用bash的话需要修改/etc/bashrc)
先备份一下原文件

```
sudo cp /etc/zshrc /etc/zshrz_bak
sudo vi /etc/zshrc
```

在最下面增加

```
export HOMEBREW_HOME=/opt/homebrew
export PATH=$PATH:$HOMEBREW_HOME/bin
```

然后保存文件
`source /etc/zshrc`
重新打开一个新终端 执行`brew -v `查看是否成功

----

出现的问题：

1. 输出版本brew -v 出现`Homebrew/homebrew-core N/A`

   原因：安装未完全，检查三个库brew.git、Homebrew-core、Homebrew-cask是否都有：

   没有的话可以根据链接对应下载：`https://mintimate.cn/2020/04/05/Homebrew/#Arm版本`

2. 错误如下：

   ```shell
   Error: Could not 'git stash' in /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core!Please stash/commit manually if you need to keep your changes or, if not, run: 
   cd /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core
   git reset --hard origin/master
   ```

   按提示操作即可

---

HomeBrew的默认软件安装位置（m1）：

一般都是安装在此目录`opt/homebrew/Cellar`

可以使用**brew list 软件名**确定安装位置

## 二、Go环境

目前go 16.1 beta 原生支持，其他版本都需要转译 

不知道为什么doesitarm网站上给的地址进不去，所以先下载goland，然后使用goland下载安装包

goland下载地址：https://www.jetbrains.com/go/download/#section=mac

![6stgBW](http://xwjpics.gumptlu.work/qinniu_uPic/6stgBW.png)

安装后在选择GOROOT时选择Download，然后选择1.16beta版本（选好下载的地址）

然后将此目录加入环境变量`/etc/zshrc`

```
export GOROOT=/Users/xwj/sdk/go1.16beta1
export PATH=$PATH:$GOROOT/bin
```

测试`go version`

如果要使用GOPATH的话在自行设置。

## 三、node、npm、git、hexo、docker、docker-compose均已支持

下载好Homebrew就很简单了，直接使用`brew install 环境名`即可

* node更换淘宝源以及更改下载包根目录

  ```shell
  # 更换源
  npm config set registry https://registry.npm.taobao.org --global
  
  npm config set disturl https://npm.taobao.org/dist --global
  
  确认成功：
  
  npm config get registry
  
  # 更改根目录
  
  npm config ls              查看默认安装路径
  
  npm config set prefix "your setting path"     设置路径，如："D:\local software manager\download\node\node local respo	sity"
  ```

* hexo

  新换的机器迁移Hexo博客：

  https://blog.csdn.net/eternity1118_/article/details/71194395?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control

* docker

  docker已经原生支持m1了，直接在homebrew中安装即可

## 四、mac上的工具软件

### 1.BetterTouchTool

原生支持了，可以在doesitarm中搜索,触控板快捷操作的神。支持各种自定义手势快捷。

可以在https://xclient.info上搜索下载（破解版）。

https://folivora.ai/downloads

### 2.Silicon info

检测当前app是不是m1原生支持的小插件 

https://mac.softpedia.com/get/Utilities/Silicon-Info.shtml

![goOqip](http://xwjpics.gumptlu.work/qinniu_uPic/goOqip.png)

## 五、屏幕截图与图床

截图工具下载：

snipaste 和 uPic

这两款软件均可以在官网下载，可以在：https://xclient.info上搜索下载。

## 六、java、idea、mvn

* jdk：

  https://www.azul.com/downloads/zulu-community/?package=jdk

  zulu的jdk 顶！

  官网下载过慢的话使用百度云：

  下载地址：
  链接: https://pan.baidu.com/s/11kUi3mA5X8L_4Jiy6TfLSw 提取码: pmia

*  maven

  https://maven.apache.org/download.cgi
  maven.apache.org
  

安装后

修改/etc/zshrc

添加环境变量

export MAVEN_HOME=/Users/…/apache-maven-3.6.3

export PATH=$PATH:$MAVEN_HOME/bin

重新加载

## 七、navicate、redis

navicate：

https://www.macwk.com/soft/navicat-premium

redis客户端：
https://gitee.com/qishibo/AnotherRedisDesktopManager/releases

## 八、知云翻译

https://www.yuque.com/xtranslator/zy/wv60oc

## 九、VScode

目前只有rosetta转译

https://code.visualstudio.com/Download

官网下载过慢的解决：https://zhuanlan.zhihu.com/p/112215618

## 十、HBuilderX和微信小程序

微信小程序：https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html

都可在官网下载最新版macos版本，实测可以使用

只是我在HBuilderX中直接运行到小程序是无法打开小程序开发工具的

![8sqbP4](http://xwjpics.gumptlu.work/qinniu_uPic/8sqbP4.png)

但是可以通过手动打开微信小程序开发工具选择文件夹打开（路径在项目根目录下的unpackage/dist/dev/mp-weixin），双方可以保持同步更新编译

![SAF50r](http://xwjpics.gumptlu.work/qinniu_uPic/SAF50r.png)

## 十一、Parallels Desktop虚拟机

没想到m1这么快就有了windows、Linux虚拟机的适配，本人没这个需求，有需求可见下方链接视频配置：

简介：https://www.iplaysoft.com/pd-windows10-arm.html

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/hx7FTh.png" alt="hx7FTh" style="zoom:50%;" />

预览版下载：https://www.parallels.com/blogs/parallels-desktop-apple-silicon-mac/
在公告里选择尝鲜技术预览版本

注册、登录进去之后象征性的看一看!!

然后记一下step 3 中的 Activation key 之后安装时就用这个码来通过验证

![QtUqgg](http://xwjpics.gumptlu.work/qinniu_uPic/QtUqgg.png)

然后点击step 3里的DOWNLOAD下载pd的安装包

* Windows10

  链接: https://pan.baidu.com/s/1xn6y3SsE7nMSc55EUh9Llg 密码: iqv4

## 十二、游戏

https://www.bilibili.com/video/BV1By4y117A4

下载Cross Over （m1支持转译版本）

它支持m1芯片运行steam游戏（部分）自行探索

# Tips

**==以下均是手贱更新Big sur到11.2.1的后果，之前的环境变量没了!!!==**

## 1.git出现错误

```
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
```

解决方法
终端输入

> xcode-select --install

按提示安装即可



