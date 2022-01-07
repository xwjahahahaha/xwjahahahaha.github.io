---
title: Linux操作系统-安装Centos7环境
tags:
  - linux
categories:
  - technical
  - linux
toc: true
declare: true
date: 2020-08-07 18:36:33
---

# Centos7虚拟机的安装

## virtualBox工具的安装

首先需要下载VirtualBox来安装虚拟机。

这里直接贴链接，不行的话可以去官网下载：[VirtualBox-6.1.12点击下载](https://download.virtualbox.org/virtualbox/6.1.12/VirtualBox-6.1.12-139181-Win.exe)

浏览器慢的话可以交给迅雷，不过文件不大。

安装的话可以一直下一步即可。

此教程安装的版本：
  - VirtualBox：VirtualBox-6.1.12
  - CentOS：CentOS-7-x86_64

<!-- more -->

## 配置虚拟机环境

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807004303.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807004651.png)

其中，“名称”为新建的虚拟电脑的别名，可以随意取但最好有标识意义。“类型”我们安装的是linux系统，选择linux。“版本”这个比较特殊，我们安装的是Centos，但下拉列表中没有这一项，**由于Centos的内核和Redhat的一样，他们彼此兼容性很高，我们可以选择Redhat。**

**注意这里的坑！**

版本选择中可能没有64bit的版本，这时需要检查你电脑的cpu和b主板是否支持虚拟化。目前主流的基本都支持，很多都是未开启虚拟化。对应自己的电脑去百度搜索对应在Bios中如何设置即可！！

我是联想intel主板，贴一下修改过程：

开机时按F2进去bios界面；

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807005155.png)

很多网上教程说在security里面，找了半天没找到！坑！

贴一下各种平台bios修改虚拟化设置的文章，不同的可以去里面找找：

https://iknow.lenovo.com.cn/detail/dc_125894.html

ok，上面弄完就选好了：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807005453.png)

这里有第二个坑，由于我们安装的是centos6.9，通过进入镜像文件，通过“install to hard drive”方式安装，这里如果设置成1G内存，会导致后续安装时加载安装文件失败的情况，所以这里建议最小设置2G。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807010036.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807010131.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807010254.png)

这边按自己的情况设置虚拟硬盘大小，我是在f盘空间大，所以就大一些

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807010438.png)

最后就o了

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807010540.png)

# 虚拟机上安装CentOS-7

打开设置

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807011319.png)

右边选择存储：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807011334.png)

找到自己准备好的linux系统镜像文件(后缀iso)，没有自动检测到可以手动添加文件

没有文件的可以[点击下载CentOS-7-x86_64-Everything-2003](http://centos.ustc.edu.cn/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2003.iso)或者去找你想要的版本（中科大镜像站）：http://centos.ustc.edu.cn/centos/7/isos

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807011458.png)

选注册找到自己的镜像文件

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807011530.png)


调整系统启动顺序，在设置界面中，选择“系统”将光驱调整为第一启动项。至此，系统镜像配置完毕。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807011725.png)

注意：这里的硬盘不能取消，并且最好把硬盘设置到前面，安装成功后会通过硬盘启动。软盘可以取消。

全部安装完成后的设置情况：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807185605.png)

双击启动虚拟机，开始跑啦！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807011852.png)


# 安装CentOS7

1. 正常安装CentOS 7
2. 测试光盘再进入安装
3. 进入除错模式

一般直接选择第一个安装，如果有问题的话可以先测试光盘。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807081257.png)

因为磁盘容量小于2TB，系统会默认使用MBR分区表来安装

所以，想使用GPT的话，在安装前需要进行操作：

1. 光标移动到Install CentOS 7 按下【tab】
2. 在出现界面中输入参数：
  `inst.gpt` (注意和前面空一格，最后的是光标不是下划线)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200807090420.png)

设置好了以后就会进入安装程序的检测系统过程

完毕后就可以开始选择语言、一些基本的设置（**重点是自己分配磁盘空间**）。这里就不一一说明了，具体可看《鸟叔的Linux私房菜》。

ps:其实是Linux虚拟机启动的时候使用picgo截图上传容易蓝屏，导致picgo错误无法启动，要重新配置。



