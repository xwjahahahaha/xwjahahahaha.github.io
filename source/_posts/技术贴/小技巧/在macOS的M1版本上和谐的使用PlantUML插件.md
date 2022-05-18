---
title: 在macOS的M1版本上和谐的使用PlantUML插件
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-11 18:51:25
---

# 一、PlantUML简介

[PlantUML Integration](https://plugins.jetbrains.com/plugin/7017-plantuml-integration)插件，用于将你的代码关系自动生成UML关系图，方便阅读源码

<!-- more -->

但是，安装使用时可能会出一些问题，下面以我的环境为例，记录正确安装的姿势

> * OS: masOS(m1)
> * Arch: arm
> * IDE: Goland

# 二、安装

安装依赖`graphviz`:

mac可以直接用`homebrew`: `brew install graphviz `

其他安装方式见官网：http://www.graphviz.org/download/

安装插件`PlantUML Integration`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511185603330.png" alt="image-20220511185603330" style="zoom: 33%;" />

# 三、解决问题

当安装好了后会发现出现如下这样的错误：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511185944631.png" alt="image-20220511185944631" style="zoom:50%;" />

具体原因在于源代码中的三个默认`dot`可执行文件路径：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511190128418.png" alt="image-20220511190128418" style="zoom:50%;" />

而我们使用`homebrew`安装在`/opt/homebrew/bin/dot`, 所以无法找到

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511190105069.png" alt="image-20220511190105069" style="zoom:33%;" />

解决方案就是修改这个插件配置（他这个配置页面找了半天….）

`Goland`右侧:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511190249631.png" alt="image-20220511190249631" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511190323558.png" alt="image-20220511190323558" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220511190342950.png" alt="image-20220511190342950" style="zoom:50%;" />

重新刷新一下就ok啦～
