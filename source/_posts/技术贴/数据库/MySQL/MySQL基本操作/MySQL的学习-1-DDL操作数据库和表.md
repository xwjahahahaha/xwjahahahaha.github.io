---
title: MySQL的学习-1-DDL操作数据库和表
tags:
  - MySQL
categories:
  - technical
  - MySQL
toc: true
declare: true
date: 2020-08-22 17:44:44
---

# MySQL基础

## 安装

我安装的是MySQL5.5版本。

安装前记得卸载以前的版本，防止最后一步无响应。（PS:以前安装过PHPstudy记得在里面卸载MySQL组件）

具体安装、配置不再赘述，可看网上文章。

<!-- more -->

## MySQL服务启动与停止

两种方法

1. **管理员**启动cmd输入：`services.msc`进入windows服务管理，然后在其中找到MySQL启动或关闭

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822193511.png)

2. **管理员**启动cmd，输入`net stop mysql`停止服务,`net start mysql`启动服务

我安装配置的时候名字起的是mysql5.5:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822193312.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822193416.png)

## MySQL的登录与退出

- 登录：

  登录本地：

  `mysql -uroot -proot`，-u接账号，-p接密码。

  或者匿名显示密码`mysql -uroot -p` 然后会提示输入密码。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822194117.png)

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822194144.png)

  登录其他电脑的MySQL服务

  `mysql -h 127.0.0.1 -uroot -proot`或者`mysql --host=127.0.0.1 --user=用户名 --password=密码`,-h后面是ip地址，后面的是账户信息。这里的密码是连接目标主机的密码。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822194543.png)

退出：`exit`或者`quit`


## MySQL目录结构

### MySQL安装目录：

* 位置： "basedir="D:/develop/MySQL/" （安装的位置）
* 配置文件 my.ini

### MySQL数据目录：

* 位置："datadir="C:/ProgramData/MySQL/MySQL Server 5.5/Data/"
* 几个概念
	* 数据库：文件夹
	* 表：文件
 	* 数据：数据

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822195618.png)


# SQL基础

## 什么是SQL？

Structured Query Language：结构化查询语言

其实就是定义了操作所有关系型数据库的规则。每一种数据库操作的方式存在不一样的地方，称为“方言”。

## SQL通用语法

1. SQL 语句可以单行或多行书写，以分号结尾。
2. 可使用空格和缩进来增强语句的可读性。
3. MySQL 数据库的 SQL 语句不区分大小写，关键字建议使用大写。
4. 3 种注释
	* 单行注释: -- 注释内容 或 # 注释内容(mysql 特有)  （PS:注意--后面有个空格）
	* 多行注释: /* 注释 */

## SQL的分类

* DDL(Data Definition Language)数据定义语言
	用来定义数据库对象：数据库，表，列等。关键字：create, drop,alter 等
* DML(Data Manipulation Language)数据操作语言
	用来对数据库中表的数据进行增删改。关键字：insert, delete, update 等
* DQL(Data Query Language)数据查询语言
  用来查询数据库中表的记录(数据)。关键字：select, where 等
* DCL(Data Control Language)数据控制语言(了解)
  用来定义数据库的访问权限和安全级别，及创建用户。关键字：GRANT， REVOKE 等

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822203951.png)

## DDL:操作数据库

### C(Create):创建

* 创建数据库：
	* create database 数据库名称;
* 创建数据库，判断不存在，再创建：
	* create database if not exists 数据库名称;
* 创建数据库，并指定字符集
	* create database 数据库名称 character set 字符集名;
* 练习： 创建db4数据库，判断是否存在，并制定字符集为gbk
	* create database if not exists dbch4 character set gbk;

![](http://xwjpics.gumptlu.work/qinniu_uPic/20200822205242.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822205418.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822205538.png)

### R(Retrieve)：查询
* 查询所有数据库的名称:
	* show databases;
* 查询某个数据库的字符集:查询某个数据库的创建语句
	* show create database 数据库名称;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822205104.png)

### U(Update):修改

* 修改数据库的字符集
	* alter database 数据库名称 character set 字符集名称;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822210619.png)

### D(Delete):删除

* 删除数据库
	* drop database 数据库名称;
	* 判断数据库存在，存在再删除
	* drop database if exists 数据库名称;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822210809.png)

### 使用数据库

* 查询当前正在使用的数据库名称
	* select database();
* 使用数据库
	* use 数据库名称;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200822210916.png)

## DDL:操作数据表

### C(Create):创建

* 语法：

```sql
create table 表名(
	列名1 数据类型1,
	列名2 数据类型2,
	....
	列名n 数据类型n
);		
```

* 注意：最后一列，不需要加逗号（,）

* 数据库类型：
	* int：整数类型
		* age int,
	* double:小数类型
		* score double(5,2)
	* date:日期，只包含年月日，yyyy-MM-dd
	* datetime:日期，包含年月日时分秒	 yyyy-MM-dd HH:mm:ss
	* timestamp:时间戳类型	包含年月日时分秒	 yyyy-MM-dd HH:mm:ss	
		* 如果将来不给这个字段赋值，或赋值为null，则默认使用当前的系统时间，来自动赋值
	* varchar：字符串
		* name varchar(20):姓名最大20个字符
		* zhangsan 8个字符  张三 2个字符
	
* 创建表实例
	create table student(
		id int,
		name varchar(32),
		age int ,
		score double(4,1),
		birthday date,
		insert_time timestamp
	);

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831151108.png)

* 复制表：
	* create table 表名 like 被复制的表名;	 

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831151350.png)

### R(Retrieve)：查询
* 查询某个数据库中所有的表名称
	* show tables;
* 查询表结构
	* desc 表名;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831145531.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831145553.png)


### U(Update):修改

* 修改表名
	alter table 表名 rename to 新的表名;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831152131.png)

* 修改表的字符集
	alter table 表名 character set 字符集名称;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831152204.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831152250.png)

* 添加一列
	alter table 表名 add 列名 数据类型;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831152354.png)

* 修改列名称 类型
	alter table 表名 change 列名 新列别 新数据类型;
	alter table 表名 modify 列名 新数据类型;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831152518.png)

* 删除列
	alter table 表名 drop 列名;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831152608.png)

### D(Delete):删除

* drop table 表名;
* drop table  if exists 表名 ;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831151503.png)