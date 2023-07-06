---
title: hexo博客的跨平台同步
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-07 14:05:54
---

# 需求

[TOC]

<!-- more -->

# 一、需求

* 原本在mac个人电脑上用hexo+github page搭建了个人博客网站
* 现在期望在办公环境的windows电脑上搭建同样的环境，并且同步自己的博客

# 二、基本方案

和网上的基本方案一样就是**创建github新的仓库分支在多台机器之间同步hexo的源文件**

分清hexo的工作空间：

* hexo源文件目录：使用`hexo g -d`等命令自动部署博客、创建md文件的目录
* githubPage目录(master分支)：博客网站的网页源文件

hexo的工作流程：

当我们在本地目录用hexo创建md文件、编写、最后`hexo g -d`上传的时候，hexo会帮我们的博客源文件转换为适合的html代码（当然也会根据你配置的博客网站主题）并通过git部署工具部署到githubPage的master分支，然后我们就可以通过提前配置好的域名去访问这个page从而实现个人博客网站的实现。

现在我们要做的就是在githubPage仓库中创建一个同步分支（sync）去同步hexo源文件目录

# 三、部署

1. 在新的主机上（windows）git clone你的博客仓库（指定`sync`分支）

   ```shell
   git clone -b sync git@github.com:xwjahahahaha/xwjahahahaha.github.io.git
   ```

2. 新的主机安装hexo环境

   windows电脑：

   ```shell
   sudo npm install cnpm -g --registry=https://registry.npm.taobao.org	// 使用cnpm镜像
   sudo cnpm install -g hexo-cli
   hexo -v
   ```

   mac：

   ```shell
   sudo brew install npm
   sudo npm install hexo-cli -g
   hexo -v
   ```

3. npm安装所有依赖，这样后面就可以直接使用`hexo new xxx`新建文章了

   ```shell
   # 安装主题插件，也就是node_modules
   sudo cnpm install --save
   # 或者
   sudo npm install
   ```

   > （可选）最好可以整个page仓库设置为private私有，这样就比较安全了（现在github私有仓库如果只有几个写作者貌似免费）
   >
   > setting往下拉：(我是已经改动过了)
   >
   > ![image-20220111160118977](http://xwjpics.gumptlu.work/image-20220111160118977.png)
   >
   > 注意：新旧主机都必须在sync分支下，master分支的操作交给hexo我们不用管

此时就完成了新主机这边的部署，接下来就是同步

# 四、同步

场景：新主机编写了一片博文，希望同步到原主机（其实就是git同步仓库分支）

1. 新主机：

   ```shell
   hexo new xxx			// 编写了新的博文
   git add .
   git commit -m "new blog"
   git push
   ```

2. 旧主机：

   ```shell
   git pull
   hexo g -d  // 推送到master
   ```

注意：

* 每次换一个主机编写的时候都先git pull一下避免分支冲突
* 可以选一个主机作为固定的hexo 部署

> 参考：
>
> * https://dora-cmon.github.io/posts/454ba26/
