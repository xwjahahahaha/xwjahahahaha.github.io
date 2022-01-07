---
title: Docker的学习-3-Docker的应用
tags:
  - docker
categories:
  - technical
  - docker
toc: true
declare: true
date: 2020-10-06 10:19:11
---

# 三、Docker自定义镜像

中央仓库的镜像也是Docker的用户自己上传上去的。

```shell
# 1. 创建一个Dockerfile文件（无后缀名），并且指定自定义镜像信息
# Dockerfile文件中常用的内容
from: 指定当前自定义镜像依赖的环境
copy: 将相对路径（相对于dockerfile文件）下的内容复制到自定义镜像中
workdir: 申明镜像的默认工作目录
cmd： 需要执行的命令（在workdir下执行，cmd可以写多个，但是只以最后一个为准）
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006111102.png)

<!-- more -->

```shell
# 2. 将准备好的Dockfile和相应的文件拖拽到Linux系统中，通过docker命令制作镜像
docker build -t 镜像名称:[tag] .     # 注意最后一个.表示当前目录
```

#  四、Docker-Compose

运行一个镜像，需要添加大量的参数，可以通过Docker-Compose编写这些参数

Docker-Compose可以帮我们批量的管理容器，只需要使用docker-compose.yml文件去维护即可

## 4.1 下载Docker-Compose

```shell
# 1. 在github上下载
https://github.com/docker/compose/releases/download/1.24.1/docker-compose-Linux-x86_64  # 1.24.1版本链接
https://github.com/docker/compose/tags   # 各版本目录
# 2. 将下载好的文件拖拽进linux系统中
# 3. 修改文件文件名称，并给予可执行的权限
mv docker-compose-Linux-x86_64 docker-compose
chmod u+x ./docker_compose 
# 4. 方便后期操作，配置环境变量
# 将docker-compose移动到/usr/local/bin目录下 
vim /etc/profile
# 添加目录
export PATH=/usr/local/bin
source /etc/profile
# 5. 测试
# 在任意目录下输入
docker-compose
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006120520.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201006134422.png)

## 4.2 yml文件格式

yml文件以key: value方式来指定配置信息，多个配置信息以换行+缩进的方式区分

**注意：yml文件key:空格value，之间有个空格**， **yml的缩进是两个空格构成的**，**在yml文件中不要使用制表符（不识别）！**

```yml
version: "3.1"
services: 
  mysql:              # 服务的名称
    restart: always   # 代表只要docker启动，这个容器就跟着启动
    image: daocloud.io/library/mysql:5.7.4   # 指定镜像的路径
    container_name: test_mysql  # 指定容器的名称，docker ps可以看到的名字，也就是--name参数
    ports:
      - 3306:3306     # 端口号映射，多个映射照样在下方同格式添加,也就是-p参数
    environment:
      MYSQL_ROOT_PASSWORD: root   # 指定MySql的Root用户登录密码
      TZ: Asia/Shanghai           # 指定时区
    volumes:
      - /opt/docker_mysql_tomcat/mysql_data:/var/lib/mysql       # 映射数据卷
  tomcat:
    restart: always
    image: daocloud.io/library/tomcat:8.5.15-jre8
    container_name: test_tomcat
    ports:
      - 8080:8080
    environment:
      TZ: Asia/Shanghai
    volumes:
      - /home/gump/myProjects/docker/docker_mysql_tomcat/tomcat_webapps:/usr/local/tomcat/webapps
      - /home/gump/myProjects/docker/docker_mysql_tomcat/tomcat_logs:/usr/local/tomcat/logs
```

## 4.3 使用docker-compose命令管理容器

使用docker-compose命令时，默认会在当前目录下找docker-compose.yml文件

```shell
# 1. 使用yml文件启动管理的容器
docker-compose up -d
# 2. 关闭并删除容器
docker-compose down
# 3. 开启|关闭|重启由yml维护的容器
docker-compose start|stop|restart
# 4. 查看yml管理的容器
docker-compose ps
# 5. 查看日志
docker-compose logs -f
```

## 4.4 使用docker-compose配置Dockerfile

使用yml配置使用自定义镜像

**使用yml文件以及Dockerfile文件在生成自定义镜像的同时启动当前镜像，并且由docker-compose去管理容器**

以自定义镜像ssm项目为例：

### docker_compose.yml文件

```yml
version: "3.1"
services: 
  ssm:              # 服务的名称
    restart: always   # 代表只要docker启动，这个容器就跟着启动
    build:           # 构建自定义镜像
      context: ../   #指定dockerfile文件的路径
      dockerfile: Dockerfile   # 指定dockerfile的文件名字
    image: sm:1.0.1       # 镜像就是上方build构建的，指定名称即可
    container_name: ssm
    ports:
      - 8081:8080
    environment:
      TZ: Asia/Shanghai
```

### Dockerfile文件

```dockerfile
from daocloud.io/library/tomcat:8.5.15-jre8		#依赖的环境
copy ssm.var /usr/local/tomcat/webapps     # 环境自添加的数据
```

```shell
# 可以直接基于这两个文件构建自定义镜像并管理
docker-compose up -d
# 如果自定义镜像不存在，会帮助我们创建，如果已经存在会直接运行这个镜像
# 如果想重新构建
docker-compose build
# 运行时重新构建
docker-compose up -d --build
```

