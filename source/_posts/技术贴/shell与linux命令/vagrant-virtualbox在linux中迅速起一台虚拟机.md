---
title: vagrant+virtualbox在linux中迅速起一台虚拟机
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-19 14:32:02
---

[TOC]


<!-- more -->

> 参考：
>
> * https://www.vagrantup.com/downloads

默认是Ubuntu 

# 一、安装

Vagrant:

```shell
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

virtualbox：

```shell
sudo apt-get update
sudo apt-get install virtualbox
```

# 二、选择你想要的Linux版本

https://app.vagrantup.com/boxes/search

想要的所有版本都可以在上方的链接中寻找，因为我们虚拟机后端（或者叫Provider）使用的是virtualbox，所以记得选择对应后端版本的使用

# 三、开始使用

选择好之后，如下三步就可以使用了

```shell
$ vagrant init hashicorp/bionic64 
$ vagrant up  
$ vagrant ssh  
# vagrant@bionic64:~$ _
```
