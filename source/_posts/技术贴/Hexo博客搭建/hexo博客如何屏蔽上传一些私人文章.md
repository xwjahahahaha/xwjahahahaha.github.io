---
title: hexo博客如何屏蔽上传一些私人文章
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-11 15:31:2
---

日常使用hexo编写博客的时候，有一些个人日记或者私密的文章不希望上传到自己的博客网站被别人看到，但是还是希望能够通过hexo这一套来编写，网上已经有很多的方法：

* 添加一个Front-matter然后改主题的配置ejs文件屏蔽
* 使用插件index2
* ......（等等等）

这些都太麻烦了并且还有很多缺点。。。

<!-- more -->

其实一个方式就可以实现：

在你不希望渲染的md文章文件前加上一个下划线"_"即可

官网给出了两种方案：

https://hexo.io/zh-cn/docs/configuration.html

```
# 不要用 exclude 来忽略 'source/_posts/' 中的文件。你应该使用 'skip_render'，或者在要忽略的文件的文件名之前加一个下划线 '_'
```

![image-20220111153659976](http://xwjpics.gumptlu.work/image-20220111153659976.png)

没错就是如此简单，**遇到问题首先看官方文档！遇到问题首先看官方文档！遇到问题首先看官方文档！**
