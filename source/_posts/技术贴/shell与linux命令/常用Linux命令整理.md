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

[TOC]



# 1. Ubuntu 18.04 

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

# 7. 端口相关

**查找被占用的端口：**

```shell
netstat -tln | grep 8000
tcp        0      0 192.168.2.106:8000      0.0.0.0:*               LISTEN  
```

**查看被占用端口的PID：**

```shell
sudo lsof -i:8000

COMMAND PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME

nginx   850     root    6u  IPv4  15078      0t0  TCP 192.168.2.106:8000 (LISTEN)

nginx   851 www-data    6u  IPv4  15078      0t0  TCP 192.168.2.106:8000 (LISTEN)

nginx   852 www-data    6u  IPv4  15078      0t0  TCP 192.168.2.106:8000 (LISTEN)
```

**开启某一端口：**

`sudo /sbin/iptables -I INPUT -p tcp --dport 8080 -j ACCEPT`

# 8. unzip命令

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

# 9. 磁盘相关

```shell
du -sh [path]   # 显示文件大小
df -h   # 显示磁盘空间
```


挂载另一个磁盘：

https://blog.csdn.net/dongyuxu342719/article/details/82702357

Linux fdisk 是一个创建和维护分区表的程序，它兼容 DOS 类型的分区表、BSD 或者 SUN 类型的磁盘列表。

```shell
su -            // ROOT用户
df -h           // 查看已挂载磁盘的使用情况
fdisk -l        // 查看所有磁盘信息
fdisk /dev/sdb  // 进入某一个磁盘，此时就进入了fdisk控制台，注意这里不能具体到分区只能指定到磁盘(例如sda、sdb等)，进入后可以看到分区
// fdisk控制台中的基本命令：
n               // 在此磁盘新建分区
p               // 查看磁盘所有已有分区
l               // 显示分区类型
t               // 设置分区类型id
d               // 删除一个分区
w               // 保存退出（所有操作生效最后都需要此操作）


// 有时候重写了分区表但是可能会看不到分区：
partprobe /dev/sdb    // 该命令可以不重启查看分区，如果提示正忙那么可能需要重启才可


// 新建分区后需要格式化文件类型
mkfs.ext4 /dev/sdb1 


// 新挂载的分区让其每次重启都挂载则需要修改配置文件
vim /etc/fstab          //按格式填入
```

# 10. linux工具

## 1. ReadLink

* 作用：确定一个链接类型文件的真正执行文件

* 常用：

  * `-f` 递归的不断通过链接确定最终的真正的执行文件

    `readlink -f $path` 如果`$path`没有链接/不是链接文件，就显示自己本身的绝对路径，**可用来确定当前文件的绝对路径**
    
    shell脚本语言中：
    
    ```shell
    BASEDIR="$(dirname $(readlink -f "$0"))"  # 获取当前执行文件的绝对父目录路径
    ```


* 例子：

    ```shell
    #系统中的awk命令到底是执行哪个可以执行文件呢？
    $ readlink /usr/bin/awk  
    /etc/alternatives/awk  ----> 其实这个还是一个符号连接  
    $ readlink /etc/alternatives/awk  
    /usr/bin/gawk  ----> 这个才是真正的可执行文件  
    ```

## 2. Diff

https://www.cnblogs.com/wangqiguo/p/5793448.html

* 作用：比较两个文本文件的不同，并输出不同的行，不会改变原文件的内容

* 输出：输出的内容代表**第一个文件应该怎样操作才可以变得和第二个文件一样**

  * 默认的normal模式：

    输出中第一行数字表示修改行号区间，字母表示操作(a=add,c=change,d=delete)

    输出下面的行<表示左边的文件要修改的具体行，>右边文件具体的行

* 其他模式：Context、Unified模式

## 3. Patch

https://www.runoob.com/linux/linux-comm-patch.html

* 作用：补丁工具，让用户利用设置修补文件的方式，修改，更新原始文件

* 使用： 一般都会结合diff使用：https://www.cnblogs.com/cute/archive/2011/04/29/2033011.html
  * 命令`diff A B > C` ,一般A是原始文件，B是修改后的文件，C称为A的补丁文件。
  * `patch A C `  就能得到B(还是A文件，内容变成了B), 这一步叫做对A打上了B的名字为C的补丁。
  * `patch -R B C `  (B还是A文件，只不过内容是B)就可以将A文件内容重新还原到A了。
* `rej`文件：打补丁的时候如果原本的文件对应补丁的内容已经有了并且还使用了强制模式，那么就会冲突，会自动生成`.rej`文件需要手动处理（修改补丁包/源文件）

> 其他常用操作文章：https://www.cnblogs.com/hellokitty2/p/7674237.html

## 4. xargs

https://www.runoob.com/linux/linux-comm-xargs.html

* 作用：一般配合管道使用，可以将管道或标准输入（stdin）数据转换成命令行参数，也能够从文件的输出中读取数据。捕获一个命令的输出，然后传递给另外一个命令。
* 使用： `somecommand | xargs -item  command`

## 5. dirname

* 作用：dirname命令去除文件名中的非目录部分，删除最后一个“\”后面的路径，显示父目录。 

* 语法：`dirname [选项] 参数`

* 例子：

  ```shell
  [root@liang ~]# dirname /etc/httpd/
  /etc
  ```

### 6. BASH_SOURCE

* 作用：`BASH_SOURCE`表示的是用户所在的目录到脚本的路径

* 使用：

  ```shell
  #!/bin/bash
  echo ${BASH_SOURCE}  # 或者用BASH_SOURCE[0]一样
  ```

  ```shell
  [root@hadoop01 sbin]# ./test 
  ./test
  [root@hadoop01 sbin]# cd ..
  [root@hadoop01 hadoop-2.7.7]# sbin/test 
  sbin/test
  ```

### 7. Trap

* 作用：捕捉信号并作出相应的处理
* 使用：`trap 'COMMAND' <信号名或信号值>      `
  * 可以捕捉的信号有：HUP INT等
  * 不适用捕捉的信号：KILL  TERM

# 11.tar

* `tar -xf file_name.tar -C /target/directory`  解压到制定目录 

# 12. 查找文件

* `find / -name <文件名>`

# 13. 替换国内镜像源

> 相关原文章：https://www.jianshu.com/p/cf542fcc09fa

创建额外的一个list文件：`/etc/apt/sources.list.d/domestic_mirrors.list`，添加如下的阿里源：

```shell
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```

更新：

```shell
sudo apt-get update
```

# 14. 开机自启脚本设置

> https://zhuanlan.zhihu.com/p/405184491

1. 创建我们需要开机自启动的脚本，例如test.sh，其内容如下：

```bash
#!/bin/bash

cd ~/
touch 11111111111.txt
```

**注意，开头一定要加上：**

```text
#!/bin/bash
```

2. 在/etc/systemd/user目录下创建一个systemd服务文件, 命名为user-defined.service, 内容如下：

```bash
[Unit]
After=network.service

[Service]
ExecStart=/home/hqc/test.sh

[Install]
WantedBy=default.target
```

其中，After表示服务何时启动，After=network.service 表示网络连接完成后，启动我们的服务；ExecStart表示我们的脚本（步骤1中的test.sh)的路径；WantedBy默认填default.target。

注意，ExecStart=/home/hqc/test.sh 这里一定不能用 ~/ 来代替/home/$USER

3. 将systemd服务文件和我们的脚本更改权限，使其可执行。

```bash
sudo chmod 744 ~/test.sh
sudo chmod 664 /etc/systemd/user/user-defined.service
```

4. 重新加载系统的systemd服务文件，并启用我们自己写的user-defined.service文件。

```bash
sudo systemctl daemon-reload
systemctl --user enable user-defined.service
```
