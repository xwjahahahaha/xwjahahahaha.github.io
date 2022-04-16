---
title: Docker的学习-2
tags:
  - docker
categories:
  - technical
  - docker
toc: true
declare: true
date: 2020-10-05 22:13:56
---

# 二、Docker的基本操作

## 2.1 安装Docker

```shell
# 1. 安装Docker的依赖环境
yum install yum-utils device-mapper-persistent-data lvm2
```

```shell
# 2. 设置一下下载Docker的镜像源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

```shell
# 3. 安装Docker
yum makacache fast   # 安装makacache fast的缓存
yum -y install docker-ce 	#安装Docker服务
```

<!-- more -->

```shell
# 4. 启动，并设置为开机自启动，测试
# 启动Docker服务
systemctl start docker
# 设置开机自启动
systemctl enable docker
# 测试
docker run hello-world
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005210546.png)

## 2.2 Docker的中央仓库

1. Docker官方镜像中央仓库，特点：镜像全下载慢（服务器在国外）    https://hub.docker.com/
2. 国内的镜像网站：网易蜂巢、daoCloud。。。     http://hub.daocloud.io/
3. 公司内部会采用私服的方式拉取镜像（添加配置）

```json
# 在/etc/docker/daemon.json
{
	"registry-mirrors": ["https://registry.docker-cn.com"],
    "insecure-registries": ["ip:port"]  #这里写内部的ip和端口
}
# 重启两个服务
systemctl daemon-reload
systemctl restart docker
```

## 2.3 镜像的操作

```shell
# 1. 拉取镜像到本地
docker pull 镜像名称[:tag]
# 例
docker pull daocloud.io/library/tomcat:8.5.15-jre8
# 2. 查看全部本地镜像
docker images 
# 3. 删除本地镜像
docker rmi 镜像的唯一标识
# 4. 镜像的导入与导出（不规范）
# 将本地的镜像导出
docker save -o 导出的路径 镜像id
# 加载本地镜像文件
docker load -i 镜像文件
# 修改名称
docker tag 标识id 新名称:Taget（版本）
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005212206.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005212256.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005214154.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005214225.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005214347.png)

## 2.4 容器的操作

### 2.4.1 运行容器

```shell
# 简单操作
docker run 镜像的标识|镜像名称[:tag]
# 常用操作
docker run -d -p 宿主机端口:容器端口 --name 容器名称 镜像的标识|镜像名称[:tag]
# -d 代表后台运行容器
# -p 宿主机端口：容器端口为了映射当前Linux的端口和容器的端口  例如tomcat默认是8080，linux的给的端口是8081 则需要映射： 8081：8080# 
# --name 指定容器的名称
# -i: 交互式操作。
# -t: 终端。
# /bin/bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 /bin/bash。
# example
docker run -itd --name xwj_ubuntu01 9140108b62dc /bin/bash # 运行ubuntu容器
```

### 2.4.2 查看正在运行的容器

```shell
docker ps [-qa]
# -a 查看全部的容器，包括没有运行的容器
# -q 只查看容器的标识其他的不查看
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005215422.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005215447.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005215537.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005215344.png)

### 2.4.3 查看容器的日志

```shell
docker logs -f 容器id
# -f 可以滚动查看日志的最后几行
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005220056.png)

### 2.4.4 进入到容器内部

```shell
docker exec -it 容器id /bin/bash
exit #返回
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005220413.png)

### 2.4.5 删除容器 （删除容器前需要先停止容器）

```shell
docker stop 容器id
docker stop $(docker ps -qa) #停止全部容器
docker rm 容器id
docker rm $(docker ps -qa)  #删除全部容器
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201005220912.png)

### 2.4.6 重新启动容器

```shell
docker start 容器id
```

### 2.4.7 将宿宿主机的内容复制到容器内部

```shell
docker cp 文件名称 容器id:容器内部路径
# 例
docker cp ssm.war fe:/usr/local/tomcat/webapps/
```

### 2.4.8 保存容器中的修改

```shell
docker commit 容器id 新的镜像名:tag版本号
```

## 2.5 数据卷

将宿主机的文件复制到容器内部去使用，这对于数据的维护十分的困难，可以采用数据卷来解决这个问题。

数据卷： 将宿主机的一个目录**映射**到容器的一个目录中

可以在宿主机中操作目录中的内容，那么容器内部映射的文件也会随之改变

```shell
# 1. 创建数据卷
docker volume create 数据卷名称
# 创建数据卷之后，默认会存放在一个目录下   /var/lib/docker/volumes/数据卷名称/_data
# 2. 查看数据卷的详细信息
docker volume inspect 数据卷名称
# 3. 查看全部数据卷
docker volume ls
# 4. 删除数据卷
docker volume rm 数据卷名称
# 5. 应用数据卷
# 当映射时你的数据卷不存在，docker会帮你创建， 会将容器内部自带的文件存储在默认的存放路径中
docker run -v 数据卷名称：容器内部的路径 镜像id
# 直接指定一个路径为数据卷的存放位置，这个路径下是空的
docker run -v 路径：容器内部路径 镜像id
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006104835.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006105026.png)

![image-20201006110128788](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20201006110128788.png)

## 2.6 docker清理

docker 清理命令（最全的命令都在这）,centos docker 清理

\1. 杀死所有运行容器

```
# docker kill $(docker ps -a -q)  
```

\2. 删除所有容器

```
# docker rm $(docker ps -a -q)  
```

\3. 删除所有镜像

```
# docker rmi $(docker images -q)  
```

\4. 停止 docker 服务

```
# systemctl stop docker  
```

\5. 删除存储目录

```
# rm -rf /etc/docker  

# rm -rf /run/docker  

# rm -rf /var/lib/dockershim  

# rm -rf /var/lib/docker  
```

  如果发现删除不掉，需要先 umount，如

```
# umount /var/lib/docker/devicemapper  
```

\6. 卸载 docker

  查看已安装的 docker 包

```
# yum list installed | grep docker  
```

  卸载相关包

```shell
# yum remove docker-engine docker-engine-selinux.noarch  
```

# Tips

删除镜像报错：

```shell
Error response from daemon: conflict: unable to delete 6a8900cc249d (must be forced) - image is referenced in multiple repositories
```

原因：镜像在多个存储库中被引用

解决：加`-f`强制删除即可 => `docker rmi -f image id`j即可：







