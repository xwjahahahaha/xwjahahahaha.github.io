---
title: Parallels17新版本:让MacM1享受Windows11的配置全流程
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-08-17 09:32:29
---

[TOC]

Parallels Desktop是很多Mac用户使用虚拟机的常用软件，现在Parallels Desktop 17版本已经发布！

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/UdoYYw.png" alt="UdoYYw" style="zoom:40%;" />

17版本与微软合作一起提高了Windows11在Mac m1上的兼容性

已经有up做了测试，可以看看顺滑程度：[专为 M1 Mac 和 Win11 优化！体验 Parallels 17 带来的畅快新感受](https://www.bilibili.com/video/BV1Co4y1m783?p=1&share_medium=iphone&share_plat=ios&share_session_id=DFA74A3E-E8CE-49F9-9063-B2DE4601F4E1&share_source=WEIXIN&share_tag=s_i&timestamp=1629126544&unique_k=3SP924)

可以说日常办公的丝滑程度与一体机无异以及可上手的游戏体验（评论区说lol可以120帧）真的让人心动！

这里就记录一下配置的全流程

> <font color='#e54d42'>**所有需要的资源可以在文章底部获取！**</font>

<!-- more -->

# 一、下载Windows11镜像

<font color='#e54d42'>**此部分以下步骤适用于想自行diyWin11镜像的用户，嫌麻烦的可以直接跳过，百度云直接下载镜像（win11专业版中文）即可**</font>

> 链接: https://pan.baidu.com/s/1KMvF_UauWIhOxKEcHX6HiA 提取码: vzu9 
>
> **下载后可直接跳过到第二部分**

## 1.1 下载uup下载器

进入UUP dump 网站：https://uupdump.net

选择最新Dev通道版本，<font color='#e54d42'>记得选择Arm64架构！</font>

![image-20210817093921838](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817093921838.png)

![image-20210817094345287](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817094345287.png)

接下来选择要下载的三个设置，出了语言选择中文简体之外，其他没特殊要求一般就直接next

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817094533050.png" alt="image-20210817094533050" style="zoom:40%;" />

![image-20210817094517044](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817094517044.png)

最后download，下载。 <font color='#e54d42'>**注意这里下载的仅仅是uup下载器哦！**</font>

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817094700767.png" alt="image-20210817094700767" style="zoom:50%;" />

我们要运行的就是这个文件：

![7hj8he](http://xwjpics.gumptlu.work/qinniu_uPic/7hj8he.png)

## 1.2 下载镜像

打开终端，进入刚刚下载好的文件夹，然后输入：

`bash 22000/uup_download_macos.sh`

![wMx3FT](http://xwjpics.gumptlu.work/qinniu_uPic/wMx3FT.png)

会提示我们需要安装一些依赖才可以下载，如果你有`homebrew`那么就下载

(没有的话看教程安装[mac_m1编程环境搭建以及适配情况总结](https://blog.csdn.net/weixin_43988498/article/details/113815873?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522162925011416780357241558%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=162925011416780357241558&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v29-1-113815873.pc_v2_rank_blog_default&utm_term=m1&spm=1018.2226.3001.4450)，已安装忽略)

```shell
brew tap sidneys/homebrew
brew install aria2 cabextract wimlib cdrtools sidneys/homebrew/chntpw
```

安装完依赖后重新运行：

`bash 22000/uup_download_macos.sh`

开始下载

aria2工具下载完成后，已经将下载的源文件自动转换为了ISO类型的镜像文件。

> 编译**chntpw**时遇到错误：
>
> ![8JIsAU](http://xwjpics.gumptlu.work/qinniu_uPic/8JIsAU.png)
>
> 解决：https://github.com/sidneys/homebrew-homebrew/issues/2
>
> 1. 下载：https://github.com/sidneys/chntpw/archive/0.99.6.tar.gz
>
> 2. 解压，修改`Makefile`文件中`OSSLPATH`：
>
>    `OSSLPATH=/opt/homebrew/Cellar/openssl@1.0/1.0.2u` （要结合你自己下载的openssl版本）
>
>    ![7VTiWX](http://xwjpics.gumptlu.work/qinniu_uPic/7VTiWX.png)
>
>    我的homebrew采用的是`OSSLPATH=/opt/homebrew/Cellar/openssl@1.1/1.1.1k`
>
>    还是make报错：
>
>    ![image-20210817112219418](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817112219418.png)
>
>    我去找了homebrew的reb文件中openssl@1.0版本，但是也找不到，所以chntpw这个工具一直下载失败，如果有谁解决了可以交流一哈：）
>
> 3. 手动编译，拷贝可执行文件
>
>    ```shell
>    # 进入文件夹make
>    make
>    # 拷贝可执行文件到homebrew
>    cp chntpw /opt/hombrew/bin
>    ```
>
>    

# 二、PD17启动Win11

安装PD 17 https://www.parallels.cn/pd/general/?utm_source=baidu&utm_medium=ppc

点击继续：

![image-20210817123911263](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817123911263.png)

点击新建

![6r7CSC](http://xwjpics.gumptlu.work/qinniu_uPic/6r7CSC.png)

一般都会自动检测到下载的镜像，如果没有那么就手动选择之前下载的镜像文件：

![A2WBik](http://xwjpics.gumptlu.work/qinniu_uPic/A2WBik.png)

点击继续，不知道是不是显示错误，选择版本时选择下面的空白处，然后点完成

![tkklE5](http://xwjpics.gumptlu.work/qinniu_uPic/tkklE5.png)

下面就因人而异了

![image-20210817124418082](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817124418082.png)

![T9JDQt](http://xwjpics.gumptlu.work/qinniu_uPic/T9JDQt.png)

然后就等待几分钟安装成功：

![image-20210817124552767](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210817124552767.png)

![Z5wxJj](http://xwjpics.gumptlu.work/qinniu_uPic/Z5wxJj.png)

> 注意：一开始是无法联网的，重新启动虚拟机即可联网！

# 三、效果演示

## 3.1 融合模式

最喜欢的就是支持融合模式

右击虚拟机，按图示点击开启融合模式

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/NJmcy6.png" alt="NJmcy6" style="zoom:50%;" />



开启了融合模式后，**Windows的所有应用程序可以像Mac自带的app程序一样使用**

例如我们使用Alfred启动Windows的画图：

![tAnGlO](http://xwjpics.gumptlu.work/qinniu_uPic/tAnGlO.png)

可以直接搜索到安装在虚拟机C盘的Windows程序，就如同自己安装的应用程序一样，这样可以解决很多Mac不支持的好用的Windows工具，对于科研狗和学生党来说都是很好的办法

![JOgVg1](http://xwjpics.gumptlu.work/qinniu_uPic/JOgVg1.png)

## 3.2 游戏

虚拟机的浏览器打开[UFO test]([UFO Test: Multiple Framerates (testufo.com)](https://www.testufo.com/))， 在UFO Test网站上稳定的显示是60帧

![kcJa1T](http://xwjpics.gumptlu.work/qinniu_uPic/kcJa1T.png)

一般类的游戏运行应该不是问题

本人是8G内存的Mac m1 air 所以只能分给虚拟机4G的内存，实测LOL运行起来只有30~40帧，不能玩

但是b站有人测试16G内存貌似是可以120帧

# 四、购买

Parallels Desktop 17 官方购买地址：https://www.parallels.cn/products/desktop/buy/?pd&new

我们也可以使用启动器启动长期体验：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/V8deG4.png" alt="V8deG4" style="zoom:50%;" />

如果想使用启动器使用可关注公众号（已关注直接在公众号后台回复）回复`pd17`获取wangpan链接：

![2DrbmZ](http://xwjpics.gumptlu.work/qinniu_uPic/772419e3aa3d6d7d60c55a69c1e46f38.png)

觉得有用的话也请关注一波博主哦~~~
