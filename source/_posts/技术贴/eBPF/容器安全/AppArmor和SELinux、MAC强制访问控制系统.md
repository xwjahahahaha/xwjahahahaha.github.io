---
title: AppArmor和SELinux、MAC强制访问控制系统
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-11 20:55:39
---

[TOC]


<!-- more -->

> 参考：
>
> * https://www.cnblogs.com/-Lei/archive/2013/02/24/2923947.html
> * https://www.cnblogs.com/zhangrui153169/p/15349167.html

# 一、基本概念

AppArmor (Application Armor)最初由 Immunix 开发，随后由 Novell 维护，其是Linux内核的一个安全模块，AppArmor允许系统管理员将每个程序与一个安全配置文件关联，从而限制程序的功能。简单的说，**AppArmor是与SELinux类似的一个访问控制系统，通过它你可以指定程序可以读、写或运行哪些文件，是否可以打开网络端口等**。作为对传统Unix的自主访问控制模块的补充，AppArmor提供了强制访问控制机制MAC，它已经被整合到2.6版本的Linux内核中，您可以在 SUSE、OpenSUSE，以及 Ubuntu 中找到 AppArmor。 同样，在redhat、centos、fedora上，也会找到SElinux 。

## AppArmor与SELinux的对比

AppArmor是 SELinux 的替代方法，也使用了 Linux 安全模块（LSM）框架。由于 SELinux 和 AppArmor 使用了同样的框架，所以它们可以互换。

* AppArmor比SELinux**更加简单**：AppArmor 的开发初衷是因为人们认为 SELinux 太过复杂，不适合普通用户管理。AppArmor 包含一个完全可配置的 MAC 模型和一个学习模式。
* AppArmor的**适配性更加好**：SELinux 的一个问题在于，它需要一个支持扩展属性的文件系统；而 AppArmor 对文件系统没有任何要求。
* AppArmor更加**易用**：Novell 出的[apparmor-overview文档](http://pan.baidu.com/s/1sjlVT3B)的第19页中做了一个对比，同样对一个ftp程序做相同的限制，使用apparmor的规则代码只是SELinux的1/4 ，而且从代码的可读性上来看，apparmor的代码也更易理解和易读。
* SELinux更加**安全**：安全性上，毋庸置疑，SELinux更安全，看下[SELinux](http://baike.baidu.com/view/487687.htm)的出身，其是美国国家安全局「NSA」和SCC（Secure Computing Corporation）开发的 Linux的一个扩张强制访问控制安全模块。再从理论上也可以了解到SELinux与Apparmor最大的区别在于：**Apparmor使用文件名（路径名）最为安全标签，而SELinux使用文件的inode作为安全标签**，这就意味着，Apparmor机制可以通过修改文件名而被绕过，另外，在文件系统中，只有inode才具有唯一性。

# 二、Apparmor基本使用

## 1、查询当前Apparmor的状态

```shell
sudo apparmor_status
```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711211353696.png" alt="image-20220711211353696" style="zoom:50%;" />

Apparmor包含了42个profile文件，而且有39个都处于enforce状态。

Apparmor的profile文件分为两类：`enforce`与`complain mode`，存在于`/etc/apparmor.d/`目录下，对此官方也对于两种状态给予了相应的解释：

**Enforcing**

1. Enforcing: This means the profile is **actively** protecting the application. By default, Ubuntu already locks down the CUPS daemon for you, but you will see several other profiles listed that you can set to enforce mode at any time.

表示当前profile配置已经激活给某个应用使用了，如果某个程序不符合其profile文件的限制，程序行为将会失败。

**Complain** 

1. Complain: This means a profile exists but is **not yet actively** protecting the application. Instead, it is sort of in "debug" mode and will put "complain" messages into `/var/log/messages`. What this means is that if the application wants to read, write, or execute something that isn't listed in the profile, it will complain. This is how you generally create a profile.

表示当前的profile配置存在，但是没有完全激活/起作用给应用进程，因为如果某个程序不符合其profile文件的限制，改程序就会被apparmor“打小报告”，即将该程序的行为记录在系统日志中，但是程序访问行为会成功，比如本来没有让某个程序访问某个文件，但就是访问，仅仅报告一下，文件访问会成功，如果在enforce模式下，文件访问就会失败。

## 2、更改profile文件的状态

如果想把某个profile置为enforce状态，执行如下命令： 

`sudo enforce <application_name>`

如果想把某个profile置为complain状态，执行如下命令：

 `sudo complain <application_name>`

 在修改了某个profile的状态后，执行如下命令使之生效：

 `sudo /etc/init.d/apparmor restart` 

## 3、profile文件的构建与管理

ubuntu发行版预定义了一些profile，可以通过如下命令安装ubuntu下的些预定义的profile文件：

`sudo apt-get install apparmor-profiles`

通过工具来管理profile，比较著名是：`apparmor-utils`，通过如下命令进行安装：

`sudo apt-get install apparmor-utils`

此工具最常用的两个命令为：`aa-genprof`和`aa-logprof`，前者用来生成profile文件，后者用来查询处于apparmor的日志记录，如：

`sudo aa-genprof slapd `   //生成openldap程序的profile文件

该包中也还有其他工具，具体可以参看[ubuntu server guide apparmor ](https://wiki.ubuntu.com/AppArmor)页面，这里列出了最近版本的用法。

## 4、自定义编写profile文件

一个示例如下： 

```yaml
#include <tunables/global>
/usr/bin/kopete { //需要限制的应用程序的名称
  #include <abstractions/X>
  #include <abstractions/audio>
  #include <abstractions/base>
  #include <abstractions/kde>
  #include <abstractions/nameservice>
  #include <abstractions/user-tmp>
  //限制其在对家目录下几个文件的读写权限
  deny @{HOME}/.bash* rw,
  deny @{HOME}/.cshrc rw,
  deny @{HOME}/.profile rw,
  deny @{HOME}/.ssh/* rw,
  deny @{HOME}/.zshrc rw,
  //对以下文件具有读、写、或可执行的权限
  /etc/X11/cursors/oxy-white.theme r,
  /etc/default/apport r,
  /etc/kde4/* r,
  /etc/kde4rc r,
  /etc/kderc r,
  /etc/security/* r,
  /etc/ssl/certs/* r,
  owner /home/*/ r,
  /opt/firefox/firefox.sh Px,
  /usr/bin/convert rix,
  /usr/bin/kde4 rix,
  /usr/bin/kopete r,
  /usr/bin/kopete_latexconvert.sh rix,
  /usr/bin/launchpad-integration ix,
  /usr/bin/xdg-open mrix,
  /usr/lib/firefox*/firefox.sh Px,
  /usr/lib/kde4/**.so mr,
  /usr/lib/kde4/libexec/drkonqi ix,
  /usr/share/emoticons/ r,
  /usr/share/emoticons/** r,
  /usr/share/enchant/** r,
  /usr/share/kde4/** r,
  /usr/share/kubuntu-default-settings/** r,
  /usr/share/locale-langpack/** r,
  /usr/share/myspell/** r,
  owner @{HOME}/.config/** rwk,
  owner @{HOME}/.kde/** rwlk,
  owner @{HOME}/.local/share/mime/** r,
  owner @{HOME}/.thumbnails/** rw,
  owner @{HOME}/Downloads/ rw,
  owner @{HOME}/Downloads/** rw,
}
```

想了解更多，可以查看[ubuntu论坛](http://ubuntuforums.org/showthread.php?t=1008906)里的相关内容。这里只对部他内容作下简要说明：

> r = read
>
> w = write
>
> l = link
>
> k = lock
>
> a = append
>
> ix = inherit = Inherit the parent's profile.
>
> px = requires a separate profile exists for the application, with environment scrubbing.
>
> Px = requires a separate profile exists for the application, without environment scrubbing.
>
> ux and Ux = Allow execution of an application unconfined, with and without environmental scrubbing. (use with caution if at all).
>
> m = allow executable mapping.

注：apparmor的相关信息也可以参看[debian wiki页面](https://wiki.debian.org/AppArmor/HowTo) 。

# 三、SELinux相关

## 1、SELinux模式

SELinux 拥有三个基本的**操作模式**，当中 Enforcing 是缺省的模式。此外，它还有一个 targeted 或 mls 的修饰语。这管制 SELinux 规则的应用有多广泛，当中 targeted 是较宽松的级别。 

1. Enforcing： 这个缺省模式会在系统上启用并实施 SELinux 的安全性政策，拒绝访问及记录行动
2. Permissive： 在 Permissive 模式下，SELinux 会被启用但不会实施安全性政策，而只会发出警告及记录行动。Permissive 模式在排除 SELinux 的问题时很有用
3. Disabled： SELinux 已被停用

从上面不难看出，两者在强制模式上作用一样，而SELinux的宽松模式（Permissive）对应着apparmor的complain模式。

## 2、SELinux的模式查看与修改

可以通过sestatus命令查看SELinux的状况：

 `sestatus` 

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220711212656629.png" alt="image-20220711212656629" style="zoom: 67%;" />

```shell
# 如果是enable
SELinux status: enabled
SELinuxfs mount: /selinux
Current mode: enforcing
Mode from config file: enforcing
Policy version: 21
Policy from config file: targeted
```

可以运行如下命令，修改SELinux的运行状态：

`setenforce [ Enforcing | Permissive | 1 | 0 ]`

## 3、查看程序或文件的SELinux信息

常见的属于 [coreutils 的工具](http://www.gnu.org/software/coreutils/manual/html_node/index.html)如 ps、ls 等等，可以通过增加 `Z `选项的方式获知 SELinux 方面的信息。如：

`# ps auxZ | grep lldpad`

```shell
system_u:system_r:initrc_t:s0 root 1000 8.9 0.0 3040 668 ? Ss 21:016:08 /usr/sbin/lldpad -d
```

`# ls -Z /usr/lib/xulrunner-2/libmozjs.so`

```shell
-rwxr-xr-x. root root system_u:object_r:lib_t:s0 /usr/lib/xulrunner-2/libmozjs.so
```

## 4、配置示例

需求：让 Apache 可以访问位于非默认目录下的网站文件/data1/361way.com

先用` semanage fcontext -l | grep '/var/www' `获知默认 /var/www 目录的 SELinux 上下文：

1. `/var/www(/.*)? all files system_u:object_r:httpd_sys_content_t:s0`

从中可以看到 Apache 只能访问包含 `httpd_sys_content_t `标签的文件。假设希望 Apache 使用 `/data1/361way.com `作为网站文件目录，那么就需要给这个目录下的文件增加` httpd_sys_content_t `标签，分两步实现：

**a、** /data1/361way.com 这个目录下的文件添加默认标签类型

`semanage fcontext -a -t httpd_sys_content_t '/data1/361way.com(/.*)?'`

**b、**用新的标签类型标注已有文件

 `restorecon -Rv /data1/361way.com`

之后 Apache 就可以使用该目录下的文件构建网站了。

其中 restorecon 在 SELinux 管理中很常见，起到恢复文件默认标签的作用。比如当从用户主目录下将某个文件复制到 Apache 网站目录下时，Apache 默认是无法访问，因为用户主目录的下的文件标签是 user_home_t。此时就需要 restorecon 将其恢复为可被 Apache 访问的 httpd_sys_content_t 类型：

```shell
restorecon -v /data1/361way.com/foo.com/html/file.html
restorecon reset /data1/361way.com/foo.com/html/file.html context unconfined_u:object_r:user_home_t:s0->system_u:object_r:httpd_sys_content_t:s0 
```

SELinux更多的介绍和用法，可以参看centos wiki页在上的相关介绍：

http://wiki.centos.org/zh/HowTos/SELinux

http://wiki.centos.org/zh/TipsAndTricks/SelinuxBooleans
