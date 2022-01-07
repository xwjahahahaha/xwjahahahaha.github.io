---
title: 在mac与windows之间共享文件
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-02-15 01:57:43
---

# mac访问windows的文件

前提：双方在同一个wifi下

首先在windows中对需要共享的文件夹右键选择共享到家庭组（此时可能会提示需要开启家庭网络，开启即可）

然后在mac中打开访达，按住command + k 然后输入`smb://对应windows在该局域网下的ip地址`即可看到之前共享的文件夹。


<!-- more -->

https://blog.csdn.net/weixin_39522312/article/details/111228666