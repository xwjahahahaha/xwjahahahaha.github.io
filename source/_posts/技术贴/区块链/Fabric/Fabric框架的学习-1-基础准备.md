---
title: Fabric框架的学习-1-基础准备
tags:
  - Fabric
categories:
  - technical
  - Fabric
toc: true
declare: true
date: 2020-10-05 10:16:55
---

# 一、Shell脚本基础

## 1.1 概念

一系列的shell命令的集合，还可以加入一些逻辑操作 （if else for）将这些命令放入一个脚本文件中

## 1.2 基本格式

命名格式：xxx.sh  (一般后缀为sh，也可不写)

书写格式：

```shell
# test.sh  #是shell脚本的注释
#！/bin/bash  # 指定解析shell脚本使用的命令解释器  /bin/sh也可以这是不写这一行的默认解释器
# 一系列的shell命令
ls 
pwd
echo "hello world"
```

## 1.3 shell脚本的执行

```shell
# shell脚本的编写完之后，必须添加执行权限
chmod u+x xxx.sh
# 执行脚本
./xxx.sh
sh xxx./sh
```

<!-- more -->

shell脚本从上至下依次执行，中途有错误则中断

## 1.4 shell脚本中的变量

* 变量的定义

  1. 普通变量(本地变量)

     ```shell
     # 定义变量，定义变量必须要赋值， =前后不允许有空格，有空格是比较（类似于==）
     temp=999
     # 普通变量只能在当前进程中使用
     ```

  2. 环境变量（一般大写）

     ```shell
     # 可以理解为全局变量，在当前操作系统中可全局访问
     # 分类
     	- 系统自带
     		例如： -PWD  -PATH
     	- 用户自定义
     		- 将普通变量提升为系统变量  前面加个set或export
     		GOPATH=/home/zoro/go/src ->普通环境变量
     		set GOPATH=/home/zoro/go/src ->系统环境变量
     		export GOPATH=/home/zore/go/src ->系统环境变量
     ```

* 位置变量

  > 执行脚本的时候，可以给脚本传递参数，脚本内部使用这些参数就需要使用这些位置变量

  ```shell
  # 执行脚本
  ./test.sh aa bb cc dd ...
  ```

  * $0: 执行的文件名vi吗
  * $1: 第一个参数，aa
  * $2: 第二个参数，bb
  * ....

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005150929.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005151017.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005150951.png)

  **注意多余的参数也传进去了，只是没有使用**

* 特殊变量

  * $#: 获取传递的参数的个数

  * $@: 给脚本传递的所有参数

  * $?: 脚本执行完成之后的状态，成功==0 or 失败！=0

  * $$: 脚本进制执行之后对应的进程ID

    ```shell
    echo "Hello World"
    echo "第一个参数: $0"
    echo "第二个参数: $1"
    echo "第三个参数: $2"
    echo "第四个参数: $3"
    echo "第五个参数: $4"
    echo "传递的所有参数的个数: $#"
    echo "传递的所有参数: $@"
    echo "脚本执行完的状态: $?"
    echo "脚本执行的进程编号: $$"
    ```

    

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005151703.png)

  直接输入`echo $?`查看的是上一个进程的状态

* 普通变量取值

  ```shell
  # 变量定义
  value=123  #默认以字符串处理
  echo $value #打印变量前面要加$
  echo ${value} #这样也可以，但是相对麻烦
  ```

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005152427.png)

* 取命令执行结果的两种方式

  把命令执行的结果用变量保存：

  ```shell
  var=$(shell命令)
  var=`shell命令`
  ```

  

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005152702.png)

* 引号的使用

  ```shell
  # 双引号
  echo "xxxx $var"  #会将其中变量的值打印输出
  # 单引号
  echo 'xxxx $var'  #会原样输出，不会取变量值
  ```


## 1.5 条件判断和循环



* shell脚本的if条件判断

  ```shell
  # if语句
  if [ 条件判断 ];then
  	逻辑处理 -> shell命令
  	xxxx
  fi
  # =====================或
  if [ 条件判断 ]
  then
  	逻辑处理 -> shell命令
  	xxxx
  fi
  # if ... elif ... fi
  if [ 条件判断 ];then
  	逻辑处理 -> shell命令
  	xxxx
  elif [ 条件判断 ];then
  	shell命令
  	xxx
  elif [ 条件判断 ];then
      shell命令
      xxx
  else
  	shell命令
  fi
  ```

  注意：if和中括号中间有空格，中括号中的条件判断与中括号两边都有空格

  ```shell
  #!/bin/bash
  #对传递到脚本内部的文件名做判断
  if [ -d $1 ];then
          echo "$1 是一个目录"
  elif [ -s $1 ];then
          echo "$1 是一个文件,且文件不为空"
  else
          echo "$1 不是一个文件也不是一个目录"
  fi
  ```

  常用判断条件：

  * 关于文件属性

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005154552.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005155043.png)

  * 关于字符串的判定

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005155131.png)

  * 常见数值判定

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005155245.png)

  * 逻辑运算操作符

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005155338.png)

  

* shell脚本for循环

  ```shell
  # shell中有for、where
  # 语法： for 变量 in 集合; do; done
  #!/bin/bash
  # 对当前目录下的文件进行遍历
  list=$(ls)
  for var in $list;do
          echo "当前文件: $var"
  done
  ~     
  ```

## 1.6 shell脚本中的函数

```shell
# 没有函数的修饰，也没有参数也没有返回值
# 格式
funcName(){
	# 得到第一个参数
	arg1=$1
	# 得到第二个参数
	arg2=$2
	# 得到第三个参数
	arg3=$3
	函数体 ->一系列的shell命令  逻辑循环和判断
}
# 没有参数列表，但是可以传参
# 函数调用
funcName aa bb cc dd
# 函数调用之后的状态
0：成功
非0：失败

```

```shell
#!/bin/bash
# 判断传递进来的文件名是不是目录，如果不是就创建
# 定义函数
funcIsDir(){
        if [ -d $1 ];then
                echo "$1是一个目录"
        else
                # 创建这个目录
                mkdir $1
                if [ 0 -ne $? ];then
                        echo "目录创建失败"
                        exit
                fi
                echo "目录创建成功"
                ls -a
        fi
}

# 函数的调用
funcIsDir $1
```

# 二、Fabric基本概念

**hyperledger 超级账本的生态图：**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201010220540.png)

Fabric

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201010220831.png)

* 准备工作

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005162800.png)

* 作用

  docker  => 类似于虚拟机，为了区块链中的多节点模拟

  docker-compose => docker的管理工具

  go =>  开发区块链

  nodejs => 写客户端

  python => 部分部件需要使用
  
* **所有安装配置可见下方，或者见https://blog.csdn.net/qq_40632760/article/details/94594177**

## 2.1 安装docker

环境是Ubuntu18，具体安装文档可见文章https://blog.csdn.net/qq_40632760/article/details/94594177



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007090856.png)

```shell
#安装基本软件
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates curl git software-properties-common lrzsz -y
# 添加阿里的docker镜像仓库
$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 更新软件源
$ sudo apt-get update
# 安装docker
$ sudo apt-get install docker-ce -y
```

**安装docker的permission denied问题：**

没有执行权限

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006152452.png)

解决方案：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007090909.png)

```shell
# 将用户加入该 group 内。然后退出并重新登录就生效啦。
$ sudo gpasswd -a ${USER} docker
# 重启docker服务
$ systemctl restart docker
# 当前用户切换到docker群组
$ newgrp - docker
$ docker version
	Client:
         Version:           18.06.1-ce
         API version:       1.38
         Go version:        go1.10.3
         Git commit:        e68fc7a
         Built:             Tue Aug 21 17:24:51 2018
         OS/Arch:           linux/amd64
         Experimental:      false

    Server:
         Engine:
         Version:          18.06.1-ce
         API version:      1.38 (minimum version 1.12)
         Go version:       go1.10.3
         Git commit:       e68fc7a
         Built:            Tue Aug 21 17:23:15 2018
         OS/Arch:          linux/amd64
         Experimental:     false

```

## 2.2 安装docker-compose

方法一：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007090919.png)

方法二：

```shell
# 下载1.25.0 docker compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 添加可执行权限ls
sudo chmod +x /usr/local/bin/docker-compose

# 测试安装
sudo docker-compose --version

```

## 2.3 go环境的安装

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006154317.png)

```shell
# 安装go
# 1. 使用wget工具下载安装包
wget https://dl.google.com/go/go1.11.linux-amd64.tar.gz
# 2. 解压tar包到/usr/local
tar zxvf go1.11.linux-amd64.tar.gz -C /usr/local
# 3. 创建Go目录
mkdir $HOME/go
# 4. 用vi打开~./bashrc，配置环境变量
vim ~/.bashrc
# 5. 增加下面的环境变量，保存退出
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
# 6. 使环境变量立即生效, 一些命令二选一
source ~/.bashrc  
. ~/.bashrc
# 7. 检测go是否安装好
go version
```

GOROOT就是go的安装路径

GOPATH是作为编译后二进制的存放目的地和import包时的搜索路径 (其实也是你的工作目录, 你可以在src下创建你自己的go源文件, 然后开始工作)。

1. GOPATH之下主要包含三个目录: bin、pkg、src
2. bin目录主要存放可执行文件; pkg目录存放编译好的库文件, 主要是*.a文件; src目录下主要存放go的源文件

## 2.4 nodejs的安装

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201007090934.png)

环境变量配置方法二：

```shell
# wget https://nodejs.org/dist/v8.11.4/node-v8.11.4-linux-x64.tar.xz  #比较慢
# 使用淘宝镜像
sudo wget https://npm.taobao.org/mirrors/node/latest-v8.x/node-v8.11.4-linux-x64.tar.xz
sudo tar xvf node-v8.11.4-linux-x64.tar.xz -C /opt
cd /opt
sudo mv node-v8.11.4-linux-x64/ node-v8.11.4  # 更名
vim ~/.bashrc  				# .bashrc文件的配置仅当前用户生效
sudo vim /etc/profile		#/etc/profile文件的配置对所有用户生效
# 在文件下面加两行
export NODEJS_HOME=/opt/node-v8.11.4
export PATH=$PATH:$NODEJS_HOME/bin
# 重新加载
source ~/.bashrc	
. /etc/profile 
# 验证
node -v
```

**.bashrc文件的配置仅当前用户生效**

**/etc/profile文件的配置对所有用户生效**

## 2.5 部署hyperledger Fabric

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006164116.png)

```sh
# 1. 创建放置的目录，进入该目录，下载脚本
cd ~
mkdir hyperledger-fabric  	
cd hyperledger-fabric
# 下载并执行脚本
# 后接参数第一个 1.2.0=> Fabric的版本 第二个1.2.0=>Fabric的认证机构Fabric的版本  第三个是第三方库的版本   前两个参数一般保持一致
# 老版本 （视频中的老版本）
curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/master/scripts/bootstrap.sh | bash -s 1.2.0 1.2.0 0.4.10
# 新版本
curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/master/scripts/bootstrap.sh | bash -s -- 2.0.0 1.4.4 0.4.18 
```

**注意：这一步很极可能会提示拒绝访问**

### 拒绝访问的解决方法

解决方法：https://blog.csdn.net/laoxuan2011/article/details/

#### step-1

在 https://site.ip138.com/raw.Githubusercontent.com/

输入raw.githubusercontent.com查询IP地址

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006165328.png)

#### step-2

修改hosts Ubuntu，CentOS及macOS直接在终端输入

```shell
sudo vi /etc/hosts
```

添加以下内容保存即可 （IP地址查询后相应修改，可以ping不同IP的延时 选择最佳IP地址）

```shell
151.101.108.133 raw.githubusercontent.com
151.101.76.133 raw.githubusercontent.com
151.101.228.133 raw.githubusercontent.com
```

再次输入该命令应该就OK了

### 下载过慢的解决方法

给docker添加镜像

添加阿里云镜像：

```shell
# 有就修改，没有就创建
vi /etc/docker/daemon.json
# 文件内编辑配置  注意：这里yourID需要自行注册替换
{
          "registry-mirrors": ["https://{yourId}.mirror.aliyuncs.com"]
}
# 写完保存，重启docker
sudo systemctl daemon-reload
sudo systemctl restart docker
# 注意阿里的镜像需要你先注册阿里云账户，详细地址https://www.aliyun.com/product/acr?spm=5176.10695662.1362911.1.3ae2262cdaoPFB
# 检查是否配置好
docker info
# 在底部看到你的加速地址则说明ok
```

### 下载内容

这个命令下载的内容分为三个部分：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006170745.png)

前两个下载可能较慢，需要耐心等待

下载的依赖镜像一览：

 ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006170450.png)

**相关解释：**

* peer：区块的存储
* orderer：Fabric中是没有矿工的，这里的order就是负责数据打包的
* ccenv: GO语言的一个运行环境
* fabric-tools: 会用到的许多工具
* fabric-ca: 生成证书。联盟链、私有链加入必须要身份认证，通过了的才能加入。 =>生成账号
* fabric-couchdb: 数据库
* fabric-kafka: 排序，需要使用zookeeper
* fabric-zookeeper:  与kafka相辅相成

最终完成了之后会有此输出显示：

```shell
===> List out hyperledger docker images
hyperledger/fabric-ca        1.2                 66cc132bd09c        2 years ago         252MB
hyperledger/fabric-ca        1.2.0               66cc132bd09c        2 years ago         252MB
hyperledger/fabric-ca        latest              66cc132bd09c        2 years ago         252MB
hyperledger/fabric-tools     1.2                 379602873003        2 years ago         1.51GB
hyperledger/fabric-tools     1.2.0               379602873003        2 years ago         1.51GB
hyperledger/fabric-tools     latest              379602873003        2 years ago         1.51GB
hyperledger/fabric-ccenv     1.2                 6acf31e2d9a4        2 years ago         1.43GB
hyperledger/fabric-ccenv     1.2.0               6acf31e2d9a4        2 years ago         1.43GB
hyperledger/fabric-ccenv     latest              6acf31e2d9a4        2 years ago         1.43GB
hyperledger/fabric-orderer   1.2                 4baf7789a8ec        2 years ago         152MB
hyperledger/fabric-orderer   1.2.0               4baf7789a8ec        2 years ago         152MB
hyperledger/fabric-orderer   latest              4baf7789a8ec        2 years ago         152MB
hyperledger/fabric-peer      1.2                 82c262e65984        2 years ago         159MB
hyperledger/fabric-peer      1.2.0               82c262e65984        2 years ago         159MB
hyperledger/fabric-peer      latest              82c262e65984        2 years ago         159MB
```

**注意：这些镜像没有放在fabric-simples目录下，都放在`/var/lib/docker/image`目录中**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006185347.png)

如果需要删除环境，记得在这里删除镜像

## 2.6 配置环境变量

`hyperledger-fabric/fabric-samples/bin`

其中放置了一些可执行的二进制文件

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006190515.png)



把这些命令都复制到`/usr/local/bin`中

```shell
cd hyperledger-fabric/fabric-samples/bin
sudo cp * /usr/local/bin
# 如果/usr/local/bin已经在PATH中了就已经完成了，没有的话加入到PATH中
# 查看PATH
echo $PATH
```

## 2.7 ubuntu安装命令合集

```shell
# 安装镜像并启动镜像
docker pull ubuntu
docker run -itd --name xwj_ubuntu01 --privileged=true 157844909278 /sbin/init
# 进入镜像
docker exec -it xwj_ubuntu01 bash

# 安装基本软件
apt-get install apt-transport-https ca-certificates curl git software-properties-common lrzsz -y net-tools inetutils-ping wget build-essential tree

# 安装docker
apt-get update
apt-get install docker-ce -y
# 阿里镜像加速
vi /etc/docker/daemon.json
# 文件内编辑配置  注意：这里yourID需要自行注册替换
{
	"registry-mirrors": ["https://{yourId}.mirror.aliyuncs.com"]  # 自己阿里云上登录即可
}
# 写完保存，重启docker
systemctl daemon-reload
systemctl restart docker
docker info

# 安装docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version

# 安装go
# 1. 使用wget工具下载安装包
wget https://dl.google.com/go/go1.11.linux-amd64.tar.gz
# 2. 解压tar包到/usr/local
tar zxvf go1.11.linux-amd64.tar.gz -C /usr/local
# 3. 创建Go目录
mkdir $HOME/go
# 4. 用vi打开~./bashrc，配置环境变量
vim ~/.bashrc
# 5. 增加下面的环境变量，保存退出
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
# 6. 使环境变量立即生效, 一些命令二选一
source ~/.bashrc  
. ~/.bashrc
# 7. 检测go是否安装好
go version


# 安装node环境
sudo wget https://npm.taobao.org/mirrors/node/latest-v8.x/node-v8.11.4-linux-x64.tar.xz
tar xvf node-v8.11.4-linux-x64.tar.xz -C /opt
cd /opt
mv node-v8.11.4-linux-x64/ node-v8.11.4  # 更名
vim ~/.bashrc  				# .bashrc文件的配置仅当前用户生效
# 在文件下面加两行
export NODEJS_HOME=/opt/node-v8.11.4
export PATH=$PATH:$NODEJS_HOME/bin
# 重新加载
source ~/.bashrc	
# 验证
node -v

# 安装fabric-1.2
# 1. 创建放置的目录，进入该目录，下载脚本
cd ~
mkdir hyperledger-fabric  	
cd hyperledger-fabric
# 下载并执行脚本
# 后接参数第一个 1.2.0=> Fabric的版本 第二个1.2.0=>Fabric的认证机构Fabric的版本  第三个是第三方库的版本   前两个参数一般保持一致
# 老版本 （视频中的老版本）
curl -sSL https://raw.githubusercontent.com/hyperledger/fabric/master/scripts/bootstrap.sh | bash -s 1.2.0 1.2.0 0.4.10

# bin添加到环境变量
vim ~/.bashrc
export PATH=$PATH:/root/fabric-samples/bin
source ~/.bashrc
```



## 2.8 脚本安装fabric基本环境的脚本

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201120132836.png)

```SHELL
curl -O https:/hyperledger.github.io/composer/latest/prereqs-ubuntu.sh
chmod u+x prereqs-ubuntu.sh

sudo ./prereqs-ubuntu.sh
```

一键部署fabric的基本环境

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201120132757.png)



node、npm、docker、docker-compose、python

**如果遇到add-apt-repository: command not found 的解决方法:**

https://blog.csdn.net/wolfqong/article/details/79420667

```shell
sudo apt-get install python-software-properties
sudo apt-get install apt-file
apt-file update
apt-file search add-apt-repository
sudo apt-get install software-properties-common
```

# Tips

## 1. 重新登录

重新登录了以后可能会导致没有权限，这时再把自己添加到docker用户组中即可=>  `newgrp - docker`

## 2. 安装Fabric出现错误

如果这个bug一直无法解决，最好的办法就是重新装系统，重新安装环境，多来几次就可以了（微笑.jpg）。

（ps: 说多了都是泪）

## 3. 安装docker出错

具体命令:`apt-get install docker-ce -y`

> vagrant@ubuntu-bionic:~$ sudo apt-get install docker-ce -y
> Reading package lists... Done
> Building dependency tree
> Reading state information... Done
> <font color='red'>Package docker-ce is not available, but is referred to by another package.</font>
> This may mean that the package is missing, has been obsoleted, or
> is only available from another source

解决方法：

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
vim  /etc/apt/sources.list.d/docker.list 
# 写入
deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
# 更新
sudo apt-get update
# 重新安装
apt-get install docker-ce -y
```



