---
title: Nodejs找不到模块的解决
tags:
  - nodejs
categories:
  - technical
  - nodejs
toc: true
declare: true
date: 2020-08-10 23:42:10
---

# nodejs require模块找不到

1. 首先检查安装的模块是不是全局安装。
2. 找到自己电脑全局安转的目录，输入命令：`npm prefix -g`，出现的是你的安装目录，安装目录中就有你的node_modules文件夹，把这个加到安装目录的后面。

<!-- more -->

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811005822.png)

3. 检查nodejs查找模块的路径是否由这个路径，输入命令：`node`进入node控制台，再输入`module.paths`

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811010013.png)

4. 没有上面的路径，就把上面的加入到全局变量的NODE_PATH中。这样node就会自动把其添加到查找路径中。或者手动添加：`module.paths.push('添加的路径')`

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811010144.png)

5. 重启shell然后node或重新运行。即可


# 如果路径都对了，还是找不到模块

那么，想一想是否移动了node_modules文件夹，而导致其模块中的快捷方式出错：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200811194508.png)


解决办法，删掉node_modules，重新安装。


