---
title: 常用Linux命令整理
tags:
  - linux
categories:
  - technical
  - linux
toc: true
declare: true
date: 2020-10-18 21:13:50
---

# 1.Ubuntu 18.04 

## 开启SSH

```shell
# 检查ssh服务是否开启
sudo ps -e | grep ssh
# 如果出现“xxxx? 00:00:00 sshd”,代表服务开启
# 如果没反应则开始配置
# 尝试开启服务，如果显示找不到命令，则安装openssh-server
sudo /etc/init.d/ssh start
sudo apt-get update
sudo apt-get install openssh-server
# 再次检查
sudo ps -e | grep ssh
# 启动服务
sudo /etc/init.d/ssh start
# 检查服务
sudo service ssh status 
# 防火墙添加规则
sudo ufw allow 22
```

<!-- more -->

## SSH 无法连接

可能原因与解决方法:

> 检查IP和账户密码是否有错
>
> 1.被防火墙挡了
>
> ​	关闭防火墙`sudo ufw disable`
>
> 2.端口没开放
>
> ​	检查:   `lsof -i:22`
>
> 3.ssh服务没开   
>
> ​	见上方

## 操作本机MAC地址

* 查看本机MAC地址

  以下四种均可：

  ```shell
  ifconfig | awk '/eth/{print $1,$5}'
  
  arp -a | awk '{print $4}
  
  sudo lshw -C network
  
  sudo lshw -c network | grep serial
  ```

## 修改主机名

`hostnamectl set-hostname 新主机名`

## 域名无法访问

报错**Name or service not known**

```shell
$ vim /etc/resolv.conf 
```

加入

```shell
$ nameserver 8.8.8.8
$ nameserver 8.8.4.4
```

# 2. Tmux分屏的使用

**Tmux 就是会话与窗口的"解绑"工具，将它们彻底分离。**

> （1）它允许在单个窗口中，同时访问多个会话。这对于同时运行多个命令行程序很有用。
>
> （2） 它可以让新窗口"接入"已经存在的会话。
>
> （3）它允许每个会话有多个连接窗口，因此可以多人实时共享会话。
>
> （4）它还支持窗口任意的垂直和水平拆分。

安装

```shell
# Ubuntu 或 Debian
$ sudo apt-get install tmux

# CentOS 或 Fedora
$ sudo yum install tmux

# Mac
$ brew install tmux
```

使用

```shell
$ tmux   #启动
$ exit  #退出或者ctrl+d 
$ tmux new -s <session-name> # 新建会话
$ tmux detach  	# 分离回话 Ctrl+b d  上面命令执行后，就会退出当前 Tmux 窗口，但是会话和里面的进程仍然在后台运行。
$ tmux ls		# 显示所有会话
$ tmux kill-session -t <session-name>   # 命令用于杀死某个会话。
$ tmux switch -t <session-name>			# 切换会话
$ tmux attach -t <session-name> 	 #进入某个会话


# 划分上下两个窗格
$ tmux split-window

# 划分左右两个窗格
$ tmux split-window -h

# 光标切换到上方窗格
$ tmux select-pane -U

# 光标切换到下方窗格
$ tmux select-pane -D

# 光标切换到左边窗格
$ tmux select-pane -L

# 光标切换到右边窗格
$ tmux select-pane -R

# 当前窗格上移
$ tmux swap-pane -U

# 当前窗格下移
$ tmux swap-pane -D
```

快捷键

- `Ctrl+b d`：分离当前会话。
- `Ctrl+b s`：列出所有会话。
- `Ctrl+b $`：重命名当前会话。
- `Ctrl+b %`：划分左右两个窗格。
- `Ctrl+b "`：划分上下两个窗格。
- `Ctrl+b <arrow key>`：光标切换到其他窗格。`<arrow key>`是指向要切换到的窗格的方向键，比如切换到下方窗格，就按方向键`↓`。
- `Ctrl+b ;`：光标切换到上一个窗格。
- `Ctrl+b o`：光标切换到下一个窗格。
- `Ctrl+b {`：当前窗格与上一个窗格交换位置。
- `Ctrl+b }`：当前窗格与下一个窗格交换位置。
- `Ctrl+b Ctrl+o`：所有窗格向前移动一个位置，第一个窗格变成最后一个窗格。
- `Ctrl+b Alt+o`：所有窗格向后移动一个位置，最后一个窗格变成第一个窗格。
- `Ctrl+b x`：关闭当前窗格。
- `Ctrl+b !`：将当前窗格拆分为一个独立窗口。
- `Ctrl+b z`：当前窗格全屏显示，再使用一次会变回原来大小。
- `Ctrl+b Ctrl+<arrow key>`：按箭头方向调整窗格大小。
- `Ctrl+b q`：显示窗格编号。

**开启tmux窗口滚动:** 

1. ctrl + b 然后输入冒号 :
2. `set-window-option -g mode-mouse on`

# 5. SCP命令

```shell
scp [-1246BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
[-l limit] [-o ssh_option] [-P port] [-S program]
[[user@]host1:]file1 [...] [[user@]host2:]file2
```

简易写法:

```shell
scp [可选参数] file_source file_target 
```

**参数说明：**

- -1： 强制scp命令使用协议ssh1
- -2： 强制scp命令使用协议ssh2
- -4： 强制scp命令只使用IPv4寻址
- -6： 强制scp命令只使用IPv6寻址
- -B： 使用批处理模式（传输过程中不询问传输口令或短语）
- -C： 允许压缩。（将-C标志传递给ssh，从而打开压缩功能）
- -p：保留原文件的修改时间，访问时间和访问权限。
- -q： 不显示传输进度条。
- -r： 递归复制整个目录。
- -v：详细方式显示输出。scp和ssh(1)会显示出整个过程的调试信息。这些信息用于调试连接，验证和配置问题。
- -c cipher： 以cipher将数据传输进行加密，这个选项将直接传递给ssh。
- -F ssh_config： 指定一个替代的ssh配置文件，此参数直接传递给ssh。
- -i identity_file： 从指定文件中读取传输时使用的密钥文件，此参数直接传递给ssh。
- -l limit： 限定用户所能使用的带宽，以Kbit/s为单位。
- -o ssh_option： 如果习惯于使用ssh_config(5)中的参数传递方式，
- -P port：注意是大写的P, port是指定数据传输用到的端口号
- -S program： 指定加密传输时所使用的程序。此程序必须能够理解ssh(1)的选项。

# 6. nginx

```shell
/usr/sbin/nginx   #启动
/usr/sbin/nginx -s stop # 关闭
/usr/sbin/nginx -s reload # 重启
```

# 7.端口相关

**1）查找被占用的端口：**

```shell
netstat -tln | grep 8000
tcp        0      0 192.168.2.106:8000      0.0.0.0:*               LISTEN  
```

**2）查看被占用端口的PID：**

```shell
sudo lsof -i:8000

COMMAND PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME

nginx   850     root    6u  IPv4  15078      0t0  TCP 192.168.2.106:8000 (LISTEN)

nginx   851 www-data    6u  IPv4  15078      0t0  TCP 192.168.2.106:8000 (LISTEN)

nginx   852 www-data    6u  IPv4  15078      0t0  TCP 192.168.2.106:8000 (LISTEN)
```

**3）kill掉该进程**

```shell
sudo kill -9 850
```

# 8.unzip命令

```shell
unzip [-cflptuvz][-agCjLMnoqsVX][-P <密码>][.zip文件][文件][-d <目录>][-x <文件>] 或 unzip [-Z]
```

-x 文件列表 解压缩文件，但不包括指定的[file](http://www.linuxso.com/command/file.html)文件。

-v 查看压缩文件目录，但不解压。

-t 测试文件有无损坏，但不解压。

**-d 目录 把压缩文件解到指定目录下。**

-z 只显示压缩文件的注解。

-n 不覆盖已经存在的文件。

-o 覆盖已存在的文件且不要求用户确认。

-j 不重建文档的目录结构，把所有文件解压到同一目录下。

# 9. 磁盘存储

```shell
du -sh [path]   # 显示文件大小
df -h   # 显示磁盘空间
```

