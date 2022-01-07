---
title: hexo出现的问题
tags:
  - hexo
categories:
  - technical
  - hexo
toc: true
declare: true
date: 2021-05-25 23:11:19
---

# 1. Nunjucks Error: [Line 319, Column 36] unexpected token: .

<!-- more -->

> **错误描述:**
>
> Version 9 of Highlight.js has reached EOL and is no longer supported.
>
> Please upgrade or ask whatever dependency you are using to upgrade.
>
> https://github.com/highlightjs/highlight.js/issues/2877
>
> FATAL Something's wrong. Maybe you can find the solution here: https://hexo.io/docs/troubleshooting.html
>
> Nunjucks Error: [Line 319, Column 36] unexpected token: .
>
> ![770zhs](http://xwjpics.gumptlu.work/qinniu_uPic/770zhs.png)
>
> **错误原因:**
>
> 在提示给出的解决文档中说明了原因:
>
> Hexo使用Nunjucks来渲染帖子(旧版本中使用了Swig，它共享类似的语法)。使用
>
> ```{{}}
> {{}}或{%%}
> ```
>
> 封装的内容将被解析，并可能导致问题。您可以通过使用原始标记插件包装它来跳过解析，例如单反勾或三反勾。 或者，Nunjucks标签可以通过渲染器的选项(如果支持)，API或前端问题禁用。
>
> * 意思就是**你写的文章内容含有两个大括号这样的字符,会导致Nunjucks解析渲染的错误/产生冲突(因为它解析也是两个大括号)**
>
> <font color='#e54d42'>解决方法:</font>
>
> 根据蓝字的提示,找到对应的文章位置,将导致混乱的地方修改(使用三个大括号):
>
> 例如:
>
> ![YEiPGd](http://xwjpics.gumptlu.work/qinniu_uPic/YEiPGd.png)





