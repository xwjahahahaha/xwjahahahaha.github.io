---
title: VBox工具虚拟CentOS-7系统-实现联网配置以及Xshell连接
tags:
  - linux
categories:
  - technical
  - linux
toc: true
declare: true
date: 2020-08-20 19:51:19
---

# Virtual Box配置CentOS7网络

看帖：https://www.cnblogs.com/wxw16/p/6256796.html

学习做个记录，以便以后使用。

Virtual Box可选的网络接入方式包括：

- NAT 网络地址转换模式(NAT,Network Address Translation)
- Bridged Adapter 桥接模式
- Internal 内部网络模式
- Host-only Adapter 主机模式

具体的区别网上的资料很多，就不再描述了，下面是一个最直接有效的配置，**配置CentOS7虚拟机里面能上外网，而主机与CentOS7虚拟机也能连通**。不论是学习还是使用，基本都能够满足。不废话，直接上图！

<!-- more -->

## 设置VBox

使用两块网卡，NAT(虚拟机访问互联网，使用10.0.2.x段)和host-only(虚拟机和主机互相通信，使用192.168.56.x段)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820195314.png)

1. 添加一个net网络

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820195429.png)

2. 设置host-only

  打开主机网络管理器

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820200912.png)

  默认会有一个网卡，网卡页面的设置不动，修改DHCP：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820201020.png)

  修改成如下：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820201141.png)


3. 找到虚拟机的设置，配置网卡

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820201302.png) 

  网卡一：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820201334.png)

  网卡二：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820201359.png)

  请注意两个网卡的MAC地址，后面会用到。

## 启动虚拟机

### CentOS-7配置NAT网络

开机以后，我们访问`ping www.baidu.com`，可以发现不能成功。通过`ip addr`命令查看网络配置。

（ps：别人的图，我已经配置好了）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820220004.png)

进入目录：`cd /etc/sysconfig/network-scripts/`找到文件ifcfg-enp0s3：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820220223.png)

修改/添加以下内容：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820220443.png)

保存退出。

重启网络：`service network restart`

此时在ping就成功了！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820220638.png)


### CentOS-7配置Host-only网络

还是network-scripts目录，复制ifcfg-enp0s3文件一份重命名为ifcfg-enp0s8

打开复制后的文件，修改：

- 修改HWADR为host-only网卡的MAC地址。

- 修改BOOTPROTO为static。

- 修改NAME为enp0s8。

- 修改UUID（可以随意改动一个值，只要不和原先的一样）。

- 添加IPADDR，可以自己制定，用于主机连接虚拟机使用。

- 添加NETMASK=255.255.255.0。


![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820221620.png)

同样得，重启网络。

使用`ip -addr`发现两个网卡都在启用：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820221808.png)

主机也可以ping通虚拟机了

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200820221954.png)


# Xshell连接

上面配置好了宿主机与虚拟机的链接后，配置Xshell连接CentOS就很简单了。

安装Xshell。网上资源很多。

新建会话：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821011640.png)

这里的IP地址就是上面配置Host-only网络的地址

在虚拟机上查看：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821011612.png)

然后点击身份验证：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821011657.png)

输入CentOS系统中的账号密码，可以是普通用户也可以是root：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821011752.png)

点击确定，连接

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821011904.png)

等待几秒：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821011925.png)



