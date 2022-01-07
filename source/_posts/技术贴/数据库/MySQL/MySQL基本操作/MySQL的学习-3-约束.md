---
title: MySQL的学习-3-约束
tags:
  - MySQL
categories:
  - technical
  - MySQL
toc: true
declare: true
date: 2020-08-31 19:28:40
---

# SQL基础

## 约束

* 概念： 对表中的数据进行限定，保证数据的正确性、有效性和完整性。	
	* 分类：
		1. 主键约束：primary key
		2. 非空约束：not null
		3. 唯一约束：unique
		4. 外键约束：foreign key

<!-- more -->

### 非空约束 not null，值不能为null
1. 创建表时添加约束
  ```sql
	CREATE TABLE stu(
		id INT,
		NAME VARCHAR(20) NOT NULL -- name为非空
	);
  ```
2. 创建表完后，添加非空约束
  ```sql
	ALTER TABLE stu MODIFY NAME VARCHAR(20) NOT NULL;
  ```
3. 删除name的非空约束
  ```sql
	ALTER TABLE stu MODIFY NAME VARCHAR(20);
  ```

```sql
CREATE TABLE people(
	id INT,
	NAME VARCHAR(20) NOT NULL # 添加非空约束
);

SELECT * FROM people;

INSERT INTO people(id, NAME) VALUES(1, '文杰');

INSERT INTO people(id, NAME) VALUES(2, NULL); #添加失败

ALTER TABLE people MODIFY NAME VARCHAR(20); #删除约束

INSERT INTO people(id, NAME) VALUES(2, NULL); #添加成功

ALTER TABLE people MODIFY NAME VARCHAR(20) NOT NULL; #添加约束

INSERT INTO people(id, NAME) VALUES(2, NULL); #添加失败
```


### 唯一约束 unique，值不能重复

* 创建表时，添加唯一约束
	```sql
	CREATE TABLE stu(
		id INT,
		phone_number VARCHAR(20) UNIQUE -- 添加了唯一约束		
	);
	```
	* 注意mysql中，唯一约束限定的列的值可以有多个null				
* 删除唯一约束
		
	```sql
	ALTER TABLE stu DROP INDEX phone_number;
	```		
* 在创建表后，添加唯一约束
	```sql
	ALTER TABLE stu MODIFY phone_number VARCHAR(20) UNIQUE;
	```

```sql
DROP TABLE people;


CREATE TABLE people(
	id INT,
	phone_num VARCHAR(20) UNIQUE #添加唯一约束
);

SELECT * FROM people;

#删除唯一约束

ALTER TABLE people DROP INDEX phone_num;


#添加唯一约束

ALTER TABLE people MODIFY phone_num VARCHAR(20) UNIQUE;

```


### 主键约束 primary key

* 注意：
	1. 含义：非空且唯一
	2. 一张表只能有一个字段为主键
	3. 主键就是表中记录的唯一标识

* 在创建表时，添加主键约束
```sql
create table stu(
	id int primary key,-- 给id添加主键约束
	name varchar(20)
);
```

* 删除主键
```sql
ALTER TABLE stu DROP PRIMARY KEY;
```
* 创建完表后，添加主键
```sql
ALTER TABLE stu MODIFY id INT PRIMARY KEY;
```

```sql
DROP TABLE peopele;

CREATE TABLE people(
	id INT PRIMARY KEY, #设置主键
	NAME VARCHAR(20)
);

SELECT * FROM people;


# 删除主键
ALTER TABLE people DROP PRIMARY KEY;

# 添加主键
ALTER TABLE people MODIFY id INT PRIMARY KEY;

```

* 自动增长： AUTO_INCREMENT

	*  概念：如果某一列是数值类型的，使用 auto_increment 可以来完成值得自动增长

	* 一般与主键id一起使用

	* 在创建表时，添加主键约束，并且完成主键自增长
	```sql
	create table stu(
		id int primary key auto_increment,-- 给id添加主键约束
		name varchar(20)
	);
	```
	* 删除自动增长
	ALTER TABLE stu MODIFY id INT;
	* 添加自动增长
	ALTER TABLE stu MODIFY id INT AUTO_INCREMENT;

```sql
DROP TABLE people;

CREATE TABLE people(
	id INT PRIMARY KEY AUTO_INCREMENT, #设置主键,并且自动增长
	NAME VARCHAR(20)
);

SELECT * FROM people;

INSERT INTO people VALUE(NULL, "文杰");

# 删除自动增长(注意这并不会删除掉主键约束，因为主键约束需要使用DROP PRIMARY KEY)
ALTER TABLE people MODIFY id INT;

# 添加自动增长(注意这也不会删除掉主键设置，只是附加)
ALTER TABLE people MODIFY id INT AUTO_INCREMENT;

```




### 外键约束：foreign key,让表于表产生关系，从而保证数据的正确性。

* 在创建表时，可以添加外键
	* 语法：
	```sql 
	create table 表名(
		....
		外键列
		constraint 外键名称 foreign key (外键列名称) references 主表名称(主表列名称)
	);
	```
* 删除外键
	ALTER TABLE 表名 DROP FOREIGN KEY 外键名称;
* 创建表之后，添加外键
	ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY (外键字段名称) REFERENCES 主表名称(主表列名称);

```sql
-- 解决方案：分成2张表
-- 创建部门表(id,dep_name,dep_location)
-- 一方，主表
CREATE TABLE department(
	id INT PRIMARY KEY AUTO_INCREMENT,
	dep_name VARCHAR(20),
	dep_location VARCHAR(20)
);
-- 创建员工表(id,name,age,dep_id)
-- 多方，从表
CREATE TABLE employee(
	id INT PRIMARY KEY AUTO_INCREMENT,
	NAME VARCHAR(20),
	age INT,
	dep_id INT, -- 外键对应主表的主键
	CONSTRAINT emp_dep_fk FOREIGN KEY (dep_id) REFERENCES department(id)
);
-- 添加2个部门
INSERT INTO department VALUES(NULL, '研发部','广州'),(NULL, '销售部', '深圳');
SELECT * FROM department;
-- 添加员工,dep_id表示员工所在的部门
INSERT INTO employee (NAME, age, dep_id) VALUES ('张三', 20, 1);
INSERT INTO employee (NAME, age, dep_id) VALUES ('李四', 21, 1);
INSERT INTO employee (NAME, age, dep_id) VALUES ('王五', 20, 1);
INSERT INTO employee (NAME, age, dep_id) VALUES ('老王', 20, 2);
INSERT INTO employee (NAME, age, dep_id) VALUES ('大王', 22, 2);
INSERT INTO employee (NAME, age, dep_id) VALUES ('小王', 18, 2);
SELECT * FROM employee;

# 删除外键
ALTER TABLE employee DROP FOREIGN KEY emp_dep_fk;

# 添加外键
ALTER TABLE employee ADD CONSTRAINT emp_dep_fk FOREIGN KEY (dep_id) REFERENCES department(id);
```

* 级联操作 
	* 添加级联操作
		语法：
		```sql
		ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY (外键字段名称) REFERENCES 主表名称(主表列名称) ON UPDATE CASCADE ON DELETE CASCADE ;
		```
	* 分类：
		1. 级联更新：ON UPDATE CASCADE 
		2. 级联删除：ON DELETE CASCADE 
	* 注意：级联操作的设置需要谨慎，特别是级联删除，因为会导致一系列的数据被直接删除。