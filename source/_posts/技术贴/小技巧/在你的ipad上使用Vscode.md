---
title: 在你的ipad上使用Vscode
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2020-11-29 09:06:34
---

单独在ipad上运行肯定是不行的，毕竟不是pc端。所以code-server的功能就是将其部署在服务器上，ipad使用网页或app访问其服务，使用服务器的资源来跑代码，而ipad前端上只负责撸代码与运行就可以啦

# 一、预备条件

* 一个云服务器（推荐学生买个阿里云学生机，一年一百来块钱，便宜又好用）
* 一个ipad

<!-- more -->

# 二、配置服务器

远程连接到云服务器，我的是阿里云的学生机

* 系统：Ubuntu18.04

执行以下步骤:

```shell
# 创建文件夹下载code server安装包
wget https://github.com/cdr/code-server/releases/download/3.2.0/code-server-3.2.0-linux-x86_64.tar.gz


# 解压到一个你想放置的地方
tar -xvzf code-server-3.2.0-linux-x86_64.tar.gz

# 进入解压文件夹，改一下名字吧
cd ....
mv code-server-3.2.0-linux-x86_64 code-server

# 进去，写两个脚本文件，一个启动，一个关闭   见下方
cd code-server 
vim ./start.sh
vim ./shut.sh
```

start.sh :

```shell
export PASSWORD="xxxx"			# 写你的code-server登录密码
nohup ./code-server --port 9999 --host 0.0.0.0 --auth password > run.log 2>&1 & 		# 端口可以自己指定，其他不改，后台运行
echo $! > save_pid.txt		
```

shut.sh : 

```shell
kill -9 'cat save_pid.txt'  # 关闭这个进程，关闭code-server服务
```

对于code-server的命令参数详细可见:

```shell
Usage: code-server [options] [path]

Options
     --auth                The type of authentication to use. [password, none]
     --cert                Path to certificate. Generated if no path is provided.
     --cert-key            Path to certificate key when using non-generated cert.
     --disable-updates     Disable automatic updates.
     --disable-telemetry   Disable telemetry.
  -h --help                Show this output.
     --open                Open in browser on startup. Does not work remotely.
     --bind-addr           Address to bind to in host:port.
     --socket              Path to a socket (bind-addr will be ignored).
  -v --version             Display version information.
     --user-data-dir       Path to the user data directory.
     --extensions-dir      Path to the extensions directory.
     --list-extensions     List installed VS Code extensions.
     --force               Avoid prompts when installing VS Code extensions.
     --install-extension   Install or update a VS Code extension by id or vsix.
     --uninstall-extension Uninstall a VS Code extension by id.
     --show-versions       Show VS Code extension versions.
     --proxy-domain        Domain used for proxying ports.
-vvv --verbose             Enable verbose logging.
```

下一步：<font color='red'>启动服务与打开服务器端口</font>

```shell
# 给上面的两个文件加权限
chmod u+x ./start.sh
chmod u+x ./shut.sh

#在服务器上启动服务
./start.sh
```

打开阿里云服务器控制台：（我的是轻量应用服务器）这一步很重要！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129092131.png)

服务器配置完毕！

打开浏览器访问    你的服务器IP + 端口   看到如下界面就表示ok

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129092255.png)



# 三、配置ipad

到了Ipad这就很简单，可以通过网页访问，也可以选择配套的App使用（当然选这个啦）

在App store中搜索：**Serverditer**软件  

进入后选择 **Self Hosted Server** （自己都配置好啦）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129092812.png)



没啥问题就可以开撸啦！

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129092903.png)







