---
title: MySQL的学习-4-多表关系与范式
tags:
  - MySQL
categories:
  - technical
  - MySQL
toc: true
declare: true
date: 2020-09-01 19:16:00
---

# 数据库的设计

## 多表之间的关系

<!-- more -->

### 分类：
* 一对一(了解)：
  * 如：人和身份证
  * 分析：一个人只有一个身份证，一个身份证只能对应一个人
* 一对多(多对一)：
  * 如：部门和员工
  * 分析：一个部门有多个员工，一个员工只能对应一个部门
* 多对多：
  * 如：学生和课程
  * 分析：一个学生可以选择很多门课程，一个课程也可以被很多学生选择
### 实现关系：
* 一对多(多对一)：
  * 如：部门和员工
  * 实现方式：在多的一方建立外键，指向一的一方的主键。
  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901192827.png)
* 多对多：
  * 如：学生和课程
  * 实现方式：多对多关系实现需要借助第三张中间表。中间表至少包含两个字段，这两个字段作为第三张表的外键，分别指向两张表的主键
  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901193509.png)
* 一对一(了解)：
  * 如：人和身份证
  * 实现方式：一对一关系实现，可以在任意一方添加唯一外键指向另一方的主键。
  * 一对一的关系其实直接可以合成一张表
  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901193830.png)

### 案例

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901194758.png)

```sql
-- 创建旅游线路分类表 tab_category
-- cid 旅游线路分类主键，自动增长
-- cname 旅游线路分类名称非空，唯一，字符串 100
CREATE TABLE tab_category (
  cid INT PRIMARY KEY AUTO_INCREMENT,
  cname VARCHAR(100) NOT NULL UNIQUE
);

-- 创建旅游线路表 tab_route
/*
rid 旅游线路主键，自动增长
rname 旅游线路名称非空，唯一，字符串 100
price 价格
rdate 上架时间，日期类型
cid 外键，所属分类
*/
CREATE TABLE tab_route(
  rid INT PRIMARY KEY AUTO_INCREMENT,
  rname VARCHAR(100) NOT NULL UNIQUE,
  price DOUBLE,
  rdate DATE,
  cid INT,
  FOREIGN KEY (cid) REFERENCES tab_category(cid)
);

/*创建用户表 tab_user
uid 用户主键，自增长
username 用户名长度 100，唯一，非空
password 密码长度 30，非空
name 真实姓名长度 100
birthday 生日
sex 性别，定长字符串 1
telephone 手机号，字符串 11
email 邮箱，字符串长度 100
*/
CREATE TABLE tab_user (
  uid INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(100) UNIQUE NOT NULL,
  PASSWORD VARCHAR(30) NOT NULL,
  NAME VARCHAR(100),
  birthday DATE,
  sex CHAR(1) DEFAULT '男',
  telephone VARCHAR(11),
  email VARCHAR(100)
);

/*
创建收藏表 tab_favorite
rid 旅游线路 id，外键
date 收藏时间
uid 用户 id，外键
rid 和 uid 不能重复，设置复合主键，同一个用户不能收藏同一个线路两次
*/
CREATE TABLE tab_favorite (
  rid INT, -- 线路id
  DATE DATETIME,
  uid INT, -- 用户id
  -- 创建复合主键
  PRIMARY KEY(rid,uid), -- 联合主键
  FOREIGN KEY (rid) REFERENCES tab_route(rid),
  FOREIGN KEY(uid) REFERENCES tab_user(uid)
);
```







# 数据库设计的范式

## 概念：

设计数据库时，需要遵循的一些规范。要遵循后边的范式要求，必须先遵循前边的所有范式要求

设计关系数据库时，遵从不同的规范要求，设计出合理的关系型数据库，这些不同的规范要求被称为不同的范式，各种范式呈递次规范，越高的范式数据库冗余越小。

目前关系数据库有六种范式：第一范式（1NF）、第二范式（2NF）、第三范式（3NF）、巴斯-科德范式（BCNF）、第四范式(4NF）和第五范式（5NF，又称完美范式）。

## 分类：

### 第一范式（1NF）：

每一列都是不可分割的原子数据项

例子：

不满足第一范式：系列不为原子列

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901200011.png)

满足第一范式，如果不满足第一范式数据库表都建立不出来。但是其中有许多问题

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901195849.png)



### 第二范式（2NF）：

在1NF的基础上，非码属性必须完全依赖于码（在1NF基础上**消除非主属性对主码的部分函数依赖**）

* 几个概念：

  * **函数依赖**：A-->B,如果通过A属性(属性组)的值，可以确定唯一B属性的值。则称B依赖于A

    例如：学号-->姓名。  （学号，课程名称） --> 分数

  * **完全函数依赖**：A-->B， 如果A是一个属性组，则B属性值得确定需要依赖于A属性组中所有的属性值。

    例如：（学号，课程名称） --> 分数

  * **部分函数依赖**：A-->B， 如果A是一个属性组，则B属性值得确定只需要依赖于A属性组中某一些值即可。

    例如：（学号，课程名称） -- > 姓名

  * **传递函数依赖**：A-->B, B -- >C . 如果通过A属性(属性组)的值，可以确定唯一B属性的值，在通过B属性（属性组）的值可以确定唯一C属性的值，则称 C 传递函数依赖于A

    例如：学号-->系名，系名-->系主任

  * **码**：如果在一张表中，一个属性或属性组，被其他所有属性所完全依赖，则称这个属性(属性组)为该表的码

    例如：该表中码为：（学号，课程名称）

  * **主属性**：码属性组中的所有属性
  * **非主属性**：除过码属性组的属性

  表格设计的修改：

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901201530.png)

### 第三范式（3NF）：

在2NF基础上，任何非主属性不依赖于其它非主属性（在2NF基础上**消除传递依赖**）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901202005.png)

# 数据库的备份和还原

* 命令行：
  * 语法：
    * 备份： mysqldump -u用户名 -p密码 数据库名称 > 保存的路径
    * 还原：
      * 登录数据库
      * 创建数据库
      * 使用数据库
      * 执行文件。source 文件路径

* 图形化工具：

备份：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901204329.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901204430.png)

还原：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901204508.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200901204526.png)


