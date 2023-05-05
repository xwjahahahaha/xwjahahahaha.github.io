---
title: mysql死锁故障排查案例
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2023-04-21 19:23:35
---

[TOC]


<!-- more -->

> 参考：
>
> * https://www.cnblogs.com/hunternet/p/11383360.html

# 1. 问题描述

并发的操作了两次update操作，导致了死锁，具体的log如下：

```shell
LATEST DETECTED DEADLOCK

------------------------

2023-04-20 18:40:42 0x7f8d889d8700

*** (1) TRANSACTION:

TRANSACTION 17303071376, ACTIVE 0 sec fetching rows

mysql tables in use 3, locked 3

LOCK WAIT 7 lock struct(s), heap size 1136, 3 row lock(s)

MySQL thread id 13507501, OS thread handle 140245884405504, query id 3006870350 10.20.59.200 eit7056897361_w updating

/* psm=it.saa.eita_consumer, ip=fdbd:dc02:ff:500:9cb0:e1ad:776d:349e, ip=- / UPDATE / psm=it.saa.eita_consumer, logid=c1988e1f-ec5d-4fc5-8c34-e52a018d3e10, ip=fdbd:dc02:ff:500:9cb0:e1ad:776d:349e, pid=217 */  email_group_department_employee_ref SET status=0,update_time='2023-04-20 18:40:42.808' WHERE email_group_id = 45019009923 AND employee_id = 545697 AND employee_role = 'MEMBER'

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 890 page no 2136 n bits 952 index idx_email_group_id of table eita.email_group_department_employee_ref trx id 17303071376 lock_mode X locks rec but not gap waiting

-----
*** (2) TRANSACTION:

TRANSACTION 17303071371, ACTIVE 0 sec fetching rows, thread declared inside InnoDB 1006

mysql tables in use 3, locked 3

8 lock struct(s), heap size 1136, 7 row lock(s), undo log entries 1

MySQL thread id 13507502, OS thread handle 140245859141376, query id 3006870341 10.20.59.152 eit7056897361_w updating

/* psm=it.saa.eita_consumer, ip=fdbd:dc02:1a:204::22, ip=- / UPDATE / psm=it.saa.eita_consumer, logid=c74a7158-692c-4eea-8f70-ae9e92994004, ip=fdbd:dc02:1a:204::22, pid=236 */  email_group_department_employee_ref SET status=0,update_time='2023-04-20 18:40:42.803' WHERE email_group_id = 45019009923 AND employee_id = 327271 AND employee_role = 'MEMBER'

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 890 page no 2136 n bits 952 index idx_email_group_id of table eita.email_group_department_employee_ref trx id 17303071371 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 890 page no 2516 n bits 240 index PRIMARY of table eita.email_group_department_employee_ref trx id 17303071371 lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (1)

------------
```

错误如下：

```shell
Error 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

其中：`email_group_id`和`employee_id`都构建了索引

# 2. 知识点

1. mysql innodb引擎支持事务，更新时采用的是行级锁。
2. 行级锁必须建立在索引的基础
3. 行级锁并不是直接锁记录，而是锁索引
4. <font color='#e54d42'>**如果一条SQL语句用到了主键索引，mysql会锁住主键索引，然后在其他索引上加锁；如果一条语句操作了非主键索引，mysql会先锁住非主键索引，再锁定主键索引。**</font>
5. 对于没有用索引的操作会采用表级锁

# 3. 分析与解决方案

## 1. 阻塞分析

首先明显可以从日志中看出是两个事务在同时执行：

* 事务1：17303071376 
* 事务2：17303071371

首先下面的log表示了事务1在等待`idx_email_group_id`索引的锁：

```shell
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 890 page no 2136 n bits 952 index idx_email_group_id of table eita.email_group_department_employee_ref trx id 17303071376 lock_mode X locks rec but not gap waiting
```

`lock_mode X locks rec but not gap waiting` => 这个是一个X排他锁并且是记录锁而不是一个临键锁（或者说只是在等待记录而不等待间隙）

而事务2的log页表示了他持有了上面事务1等待的同一个记录的锁：

```shell
*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 890 page no 2136 n bits 952 index idx_email_group_id of table eita.email_group_department_employee_ref trx id 17303071371 lock_mode X locks rec but not gap
```

这就表明事务1在等待事务2释放`space id 890 page no 2136 n bits 952 index idx_email_group_id `的锁

同时，事务2的log：

```shell
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 890 page no 2516 n bits 240 index PRIMARY of table eita.email_group_department_employee_ref trx id 17303071371 lock_mode X locks rec but not gap waiting
```

表明事务2在获取到非主键锁之后，需要获取主键锁，然而一直等待`space id 890 page no 2516 n bits 240 index PRIMARY`的主键锁而导致了阻塞

> 在这里不是事务1锁住了主键，因为业务场景下这两条SQL查询不会查到相同主键的行记录，所以可能是被另一个未知的事务并发操作时（用到了主键索引）获取到了事务2待更新的这个主键锁，导致其阻塞

> space id 890 page no 2516 n bits 240 index 的具体含义：
>
> space id 890 page no 2516是指MySQL存储引擎将表格存储在磁盘上，每个表格是一个空间（space id），每个空间中有多个页（page no），每个页面中有多个字段（n bits）

## 2. 解决方案

建立一个联合索引，对于此案例中将`email_group_id`和`employee_id`两个属性构建一个联合索引，这样事务1和事务2在遇到相同的`email_group_id`但是`employee_id`不同的情况的时候因为构建了联合索引索引不同所以不会导致等待锁住同一个索引的情况

# 4. 拓展

一个案例：

```sql
CREATE TABLE `user_item` (
  `id` BIGINT(20) NOT NULL,
  `user_id` BIGINT(20) NOT NULL,
  `item_id` BIGINT(20) NOT NULL,
  `status` TINYINT(4) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_1` (`user_id`,`item_id`,`status`)
) ENGINE=INNODB DEFAULT CHARSET=utf-8
```

```sql
update user_item set status=1 where user_id=? and item_id=?
```

上面操作的流程：

1. 由于用到了非主键索引，首先需要获取idx_1上的行级锁
2. 紧接着根据主键进行更新，所以需要获取主键上的行级锁
3. 更新完毕后，提交，并释放所有锁

如果在步骤1和2之间突然插入一条语句(或者并行运行了一个事务)：

```sql
update user_item .....where id=? and user_id=?
```

这条语句会先锁住主键索引，然后锁住idx_1

蛋疼的情况出现了，第一条语句获取了idx_1上的锁，等待主键索引上的锁；

第二条语句获取了主键上的锁，等待idx_1上的锁，这样就出现了死锁。

解决方案：

1. 先获取需要更新的记录的主键

   ```sql
   select id from user_item where user_id=? and item_id=?
   ```

2. 再通过这些主键id合并两个update

   ```sql
   update ... where id=? and user_id=? and item_id=?
   ```

