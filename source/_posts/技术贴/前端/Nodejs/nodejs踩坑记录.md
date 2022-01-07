---
title: nodejs踩坑记录
tags:
  - nodejs
categories:
  - technical
  - nodejs
toc: true
declare: true
date: 2021-03-01 12:41:22
---

# npm运行出错Missing required argument #1

原因:目前的nodejs过时了,与npm不匹配

更新nodejs到最新版本

```shell
1) sudo npm install -g n
2) sudo n latest
3) sudo npm install -g npm
4) hash -d npm
5) npm i
```

<!-- more -->

原文:[How to fix npm ERR! typeerror Error: Missing required argument #1 | Web Hosting Forum (cyberhour.com)](https://www.cyberhour.com/community/threads/how-to-fix-npm-err-typeerror-error-missing-required-argument-1.262/)

