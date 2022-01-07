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

现在我们要做的就是在githubPage仓库中创建一个同步分支去同步hexo源文件目录

# 三、部署

1. 在githubPage仓库页面上创建新的分支`sync`（可以随便取），并设置其为默认分支

<img src="http://xwjpics.gumptlu.work/image-20220107142921887.png" alt="image-20220107142921887" style="zoom: 80%;" />

![image-20220107143114823](http://xwjpics.gumptlu.work/image-20220107143114823.png)

2. 在新的主机上（windows）git clone这个仓库，并进入目录，此时应该就在`sync`分支（因为设置了其为默认，如果不在则`git checkout sync`）

3. 删除该目录下所有其他文件，只剩下`.git`文件

4. 复制原主机（mac）中的hexo源文件目录中除去`.deploy_git, node_modules/, public/ `这几个目录的其他所有文件到新的主机刚刚git clone的目录下（因为这几个目录后面都可以生成）

5. 修改` .gitignore`，如果没有就创建一个,如果有就检查一下是否一样，输入以下内容，在 `git push `时自动忽略：

   > 注意，因为 git 不能嵌套上传，如果在 theme 中有克隆过的主题文件，需要将主题文件夹中的 .git 文件夹删掉。

   ```text
   .DS_Store
   Thumbs.db
   db.json
   *.log
   node_modules/
   public/
   .deploy*/
   ```

6. 新的主机（windows电脑）安装hexo环境

   下载Node.js并安装

   ```shell
   sudo npm install cnpm -g --registry=https://registry.npm.taobao.org	// 使用cnpm镜像
   sudo cnpm install -g hexo-cli
   hexo -v
   sudo cnpm install --save   // 安装主题插件，也就是node_modules
   hexo clean && hexo g -d    // 尝试通过hexo工具推送博客
   ```

7. 返回github Page仓库页面，将master分支改回默认，然后注意看一下自己的Page主页分支有没有改变,一定要是master（可能改动默认分支会让sync变为主页，导致Page网站无法访问）

   ![image-20220107144859767](http://xwjpics.gumptlu.work/image-20220107144859767.png)

8. 旧主机同理同步这个sync分支，先git init再添加remote，最后git pull，应该是一样的内容

注意：新旧主机都必须在sync分支下，master分支的操作交给hexo我们不用管

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
