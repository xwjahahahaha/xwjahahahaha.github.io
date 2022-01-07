---
title: mac使用Homebrew下载旧版本/历史版本项目
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-03-04 09:36:38
---

mac中使用brew下载软件都是默认下载的最新版本,下载旧版本时需要一定的方法

以我下载starport软件为例,默认下载0.14.0 ,但是我需要0.13.1版本

# 一、找到From的.rb文件

查看brew安装的软件源:

`brew info 你的软件名`

![3KapYY](http://xwjpics.gumptlu.work/qinniu_uPic/3KapYY.png)

<!-- more -->

可以看到默认为最新版本,==找到From的地址==

## 二、打开From地址查看.rb文件

![9Dp1cs](http://xwjpics.gumptlu.work/qinniu_uPic/9Dp1cs.png)

其实brew就是通过此地址进行下载的,打开该地址

# 三、找到对应的版本

根据上面的地址,找到你需要的版本下载对应的旧版本压缩包即可!

![wL1SNe](http://xwjpics.gumptlu.work/qinniu_uPic/wL1SNe.png)

