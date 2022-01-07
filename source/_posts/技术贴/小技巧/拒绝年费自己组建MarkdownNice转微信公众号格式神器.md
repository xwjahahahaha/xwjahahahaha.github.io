---
title: 拒绝年费自己组建MarkdownNice转微信公众号格式神器
tags:
  - markdown
categories:
  - technical
  - markdown
toc: true
declare: true
date: 2021-07-12 14:12:29
---

> 资料来源：
>
> https://cloud.tencent.com/developer/article/1811081

# 简单介绍

墨滴的公众号排版格式转换服务非常的好用，对于一些带有公式的Markdown也能够完美的转换成为公众号的格式

它的简单功能介绍可见连接：https://zhuanlan.zhihu.com/p/104209040

总之，如果你喜欢用Markdown写公众号，那么这个是你的不二之选

但是，官方的app软件只有试用期7天，年费的开销也挺大, 在线转网站必须要登陆等等限制（且自动会把内容发到社区）。。。

好在其代码开源，所以我们可以自己搭建一个mdnice服务

<!-- more -->

![8pVlof](http://xwjpics.gumptlu.work/qinniu_uPic/8pVlof.png)

# 准备工作

## 硬件

一台云服务器，或者有公网IP带域名解析的服务器主机

建议阿里云、腾讯等学生机，便宜够用。

## 软件

有一个注册的域名

nodejs npm 环境需要提前安装. (如果不会也可跳过)

# 搭建流程

## 1. 下载官方包

下载官方的压缩包， 链接： [https://github.com/mdnice/markdown-nice/archive/refs/heads/master.zip](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fmdnice%2Fmarkdown-nice%2Farchive%2Frefs%2Fheads%2Fmaster.zip)

进入文件夹，下载依赖

```shell
# 安装依赖包
npm i
# 编译软件, 获得可直接部署的项目文件夹
npm run build
```

没有环境的可以直接下载编译好的文件链接：  [https://zhaoolee.lanzoui.com/iZqoQnqrt9e](https://links.jianshu.com/go?to=https%3A%2F%2Fzhaoolee.lanzoui.com%2FiZqoQnqrt9e)

正确完整的文件目录如下：

![mEouy8](http://xwjpics.gumptlu.work/qinniu_uPic/mEouy8.png)

## 2. 部署

发送到你的云服务器中，放置到如下目录

`/usr/share/nginx/mdnice`    (文件夹名称可自行命名, 我就叫mdnice)

## 3. 添加域名解析

在你的云服务器服务商网站中找到域名解析，以阿里云为例：

添加一个你喜欢子域名前缀，这里我就是`mdnice.gumptlu.work`

![uo4FFB](http://xwjpics.gumptlu.work/qinniu_uPic/uo4FFB.png)

## 4. nginx配置

在云服务器的`/etc/nginx/conf.d`目录下添加一个conf后缀文件, 这里命名就以`mdnice.gumptlu.work.conf`为例

![nalT1T](http://xwjpics.gumptlu.work/qinniu_uPic/nalT1T.png)

文件中写入解析内容：

```shell
server {
  listen 80; 
  server_name mdnice.gumptlu.work;
  charset  utf-8;
 
  location / { 
    root /usr/share/nginx/mdnice;			# 要与部署的路径对应
    index index.html index.htm;
  }
}
```

重启nginx：

```shell
# 测试配置文件
nginx -t
# 重启nginx
systemctl restart nginx
```

## 5. 访问地址

`http://mdnice.gumptlu.work`

ok!

![s6284p](http://xwjpics.gumptlu.work/qinniu_uPic/s6284p.png)

# Tips

如果之前开启过ssl用于https，那么需要关闭ssl，否则80端口会被自动转发到443, The plain HTTP request was sent to HTTPS port

解决：

开配置文件，查看HTTPS server段的配置：

修改前：

```shell
server {
        listen       443 ssl;
        server_name  localhost;
        ...
}
```

修改方式，将监听端口后的“ssl”删除，即：

```shell
server {
        listen       443;
        server_name  localhost;
        ...
}
```



