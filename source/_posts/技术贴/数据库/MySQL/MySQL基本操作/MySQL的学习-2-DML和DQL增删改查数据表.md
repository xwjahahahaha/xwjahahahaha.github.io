---
title: MySQL的学习-2-DML和DQL增删改查数据表
tags:
  - MySQL
categories:
  - technical
  - MySQL
toc: true
declare: true
date: 2020-08-31 15:32:05
---

# SQL基础

## DML：增删改表中数据

### 添加数据：

* 语法：

	* insert into 表名(列名1,列名2,...列名n) values(值1,值2,...值n);

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831154304.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200831154430.png)

* 注意：
	* 列名和值要一一对应。
	* 如果表名后，不定义列名，则默认给所有列添加值

		insert into 表名 values(值1,值2,...值n);

	* 除了数字类型，其他类型需要使用引号(单双都可以)引起来
  

<!-- more -->

### 删除数据

* 语法：
	* delete from 表名 [where 条件]
* 注意：
	* 如果不加条件，则删除表中所有记录。
	* 如果要删除所有记录
		* delete from 表名; -- 不推荐使用。有多少条记录就会执行多少次删除操作
		* TRUNCATE TABLE 表名; -- 推荐使用，效率更高 先删除表，然后再创建一张一样的表。

### 修改数据

* 语法：
	* update 表名 set 列名1 = 值1, 列名2 = 值2,... [where 条件];

* 注意：
	* 如果不加任何条件，则会将表中所有记录全部修改。


## DQL：查询表中的记录

* select * from 表名;
	
### 语法

select
	字段列表

from
	表名列表

where
	条件列表

group by
	分组字段

having
	分组之后的条件

order by
	排序

limit
	分页限定

### 基础查询

* 多个字段的查询

	select 字段名1，字段名2... from 表名；
	* 注意：
		* 如果查询所有字段，则可以使用*来替代字段列表。
* 去除重复：
	
	* distinct
* 计算列
	* 一般可以使用四则运算计算一些列的值。（一般只会进行数值型的计算）
	* ifnull(表达式1,表达式2)：null参与的运算，计算结果都为null
    * 表达式1：哪个字段需要判断是否为null
    * 如果该字段为null后的替换值(表达式2)。
* 起别名：
	
	* as：as也可以省略

```sql
CREATE TABLE student ( 
	id INT, -- 编号 
	NAME VARCHAR(20), -- 姓名 
	age INT, -- 年龄 
	sex VARCHAR(5), -- 性别 
	address VARCHAR(100), -- 地址 
	math INT, -- 数学 
	english INT -- 英语 
); 
INSERT INTO student(id,NAME,age,sex,address,math,english) VALUES (1,'马云',55,'男','杭州',66,78),(2,'马化腾',45,'女','深圳',98,87),(3,'马景涛',55,'男','香港',56,77),(4,'柳岩',20,'女','湖南',76,65),(5,'柳青',20,'男','湖南',86,NULL),(6,'刘德华',57,'男','香港',99,99),(7,'马德',22,'女','香港',99,99),(8,'德玛西亚',18,'男','南京',56,65);

SELECT * FROM student;

SELECT NAME, age FROM student;

SELECT DISTINCT address FROM student;

SELECT address FROM student;

SELECT math, english, math+ IFNULL(english, 0) AS SUM FROM student;

```
### 条件查询

1. where子句后跟条件
2. 运算符
	* \> ，< 、<= 、>= 、= 、<>
	* BETWEEN...AND  (包含边界)
	* IN( 集合) 
	* LIKE：模糊查询
			* 占位符：
			* _:单个任意字符
			* %：多个任意字符
	* IS NULL  
	* and  或 &&
	* or  或 || 
	* not  或 ! 或 <>

```sql

SELECT * FROM student WHERE age > 12 AND age < 55;
SELECT * FROM student WHERE age BETWEEN 12 AND 55; --包含边界
SELECT * FROM student WHERE age = 22 OR age = 45;
SELECT * FROM student WHERE age IN (22, 45, 55);
SELECT * FROM student WHERE english IS NULL;
SELECT * FROM student WHERE sex != '男';
SELECT * FROM student WHERE sex <> '女';



SELECT * FROM student WHERE NAME LIKE '马%';
SELECT * FROM student WHERE NAME LIKE "%德%";
SELECT * FROM student WHERE NAME LIKE '___';
```

### 排序查询

* 语法：order by 子句
	* order by 排序字段1 排序方式1 ，  排序字段2 排序方式2...

* 排序方式：
	* ASC：升序，默认的。
	* DESC：降序。

* 注意：
	* 如果有多个排序条件，则当前边的条件值一样时，才会判断第二条件。


### 聚合函数

* count：计算个数
	* 一般选择非空的列：主键
	* count(*)
* max：计算最大值
* min：计算最小值
* sum：计算和
* avg：计算平均值	

* 注意：聚合函数的计算，排除null值。
	解决方案：
		1. 选择不包含非空的列进行计算
		2. IFNULL函数

```sql

SELECT COUNT(NAME) FROM student;

SELECT COUNT(english) FROM student;

SELECT COUNT(*) FROM student;

SELECT COUNT(IFNULL(english, 0))  AS SUM FROM student;

SELECT MAX(math) FROM student;

SELECT MIN(math) FROM student;

SELECT SUM(math) FROM student;

SELECT AVG(math) FROM student;
```

### 分组查询

* 语法：group by 分组字段；
* 注意：
	* 分组之后查询的字段：分组字段、聚合函数
	* where 和 having 的区别？
		* **where 在分组之前进行限定，如果不满足条件，则不参与分组。having在分组之后进行限定，如果不满足结果，则不会被查询出来**
		* **where 后不可以跟聚合函数，having可以进行聚合函数的判断。**

```sql
SELECT sex, COUNT(id) FROM student GROUP BY sex;

-- 按照性别分组。分别查询男、女同学的平均分
SELECT sex, AVG(math) FROM student GROUP BY sex;

-- 按照性别分组。分别查询男、女同学的平均分,人数
SELECT sex, AVG(math), COUNT(id) FROM student GROUP BY sex;

-- 按照性别分组。分别查询男、女同学的平均分,人数 要求：分数低于70分的人，不参与分组
SELECT sex, AVG(math), COUNT(id) FROM student WHERE math > 70 GROUP BY sex;

--  按照性别分组。分别查询男、女同学的平均分,人数 要求：分数低于70分的人，不参与分组,分组之后。人数要大于2个人
SELECT sex, AVG(math), COUNT(id) 人数 FROM student WHERE math > 70 GROUP BY sex HAVING 人数 > 2;
```

### 分页查询

1. 语法：limit 开始的索引,每页查询的条数;
2. 公式：**开始的索引 = （当前的页码 - 1） * 每页显示的条数**
3. limit 是一个MySQL"方言"

```sql
-- 每页显示3条记录 
SELECT * FROM student LIMIT 0,3; --第一页
SELECT * FROM student LIMIT 3,3; --第二页（2-1）*3
SELECT * FROM student LIMIT 6,3; --第二页（3-1）*3
```