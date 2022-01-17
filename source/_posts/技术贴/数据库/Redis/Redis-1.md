---
title: Redis-1
tags:
  - redis
categories:
  - technical
  - redis
toc: true
declare: true
date: 2020-12-18 15:24:24
---

# 一、基本介绍

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201218152511.png)

<!-- more -->

# 二、Redis的安装

Redis不推荐在Windows上使用，所以官网没有Windows版本

https://redis.com.cn/download.html

下载, 解压和编译 Redis 方法:

```
$ wget https://download.redis.io/releases/redis-6.0.8.tar.gz
$ tar xzf redis-6.0.8.tar.gz
$ cd redis-6.0.8
$ make
```

编译好的二进制文件在 `src` 目录里。启动 Redis:

```
$ src/redis-server
```

使用内置的客户端与Redis通讯:

```
$ src/redis-cli
redis> set foo bar
OK
redis> get foo	
"bar"
```

# 三、连接使用

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201218154057.png)

## **3.1 Redis的架构**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201218154533.png)

## 3.2 权限登录

redis默认是不需要账号密码的，但是可以设置：

修改redis.conf 中 的配置，加上认证。(把下面配置去掉注释，然后修改foobared为你指定的密码，重启redis-server即可生效。)

requirepass foobared

然后，客户端连接的时候，输入auth 密码 即可认证。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219111711.png)

## 3.3 redis服务操作

```shell
./src/redis-server							# 开启服务
./src/redis-cli -h 127.0.0.1 -p 6379 shutdown    	# 客户端控制重启
```

# 四、操作命令

[见官方文档](https://redis.com.cn/commands.html)

1. 添加key-val [set]

2. 查看当前redis 的所有key [keys *]
3. 获取key 对应的值. [get key]
4. 切换redis 数据库[select index]
5. 如何查看当前数据库的key-val 数量[dbsize]
6. 清空当前数据库的key-val 和清空所有数据库的key-val [flushdb flushall]

**redis有0~15号内存数据库，使用时要对应好数据库的位置**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219101335.png)

# 五、Redis 的CRUD 操作

## 5.1 Redis 的五大数据类型:

Redis 的五大数据类型是: String(字符串) 、Hash (哈希)、List(列表)、Set(集合)和zset(sorted set：有序集合)

## 5.2 String(字符串) -介绍

string 是redis 最基本的类型，一个key 对应一个value。

string 类型是二进制安全的。除普通的字符串外，也可以存放图片等数据。

redis 中字符串value 最大是**512M**

### 5.2.1 String(字符串) -CRUD

set [如果存在就相当于修改，不存在就是添加]

get  key	  -> 读取

del   key     -> 删除

### 5.2.2 字符串注意事项

* setex(set with expire)键秒值

  设置自动销毁时间

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219102431.png)

* mset[同时设置一个或多个key-value 对]

* mget[同时获取多个key-val]

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219102625.png)

## 5.3 Hash (哈希)-介绍

### 5.3.1 基本的介绍

**hash和map是类似的**

Redis hash 是一个键值对集合。`var user1 map[string]string`

<font color='green'>Redis hash 是一个string 类型的field 和value 的映射表，hash **特别适合用于存储对象**。</font>

**基本使用：** HSET 、 GSET

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219103214.png)

### 5.3.2 Hash-CRUD

hset	=> 存、改

hget	=> 取

hgetall =>取全部

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219103409.png)

hdel	 => 删除

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219103446.png)

### 5.3.3 细节

* hmset 和 hmget 一次性可以设置多个字段值和返回多个字段值

  在给user 设置name 和age 时，前面我们是一步一步设置,使用hmset 和hmget 可以一次性来设置多个filed 的值和返回多个field 的值。

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219103704.png)

* hlen 统计一个hash 有几个元素.

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219103720.png)

* hexists key field

  查看哈希表key 中，给定域field **是否存在**

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219103822.png)

## 5.4 List

列表是简单的字符串列表，按照插入顺序排序。**你可以添加一个元素到列表的头部（左边）或者尾部（右边）**。List 本质是个链表, List 的元素是有序的，元素的值**可以重复.**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219104038.png)

### 5.4.1 List-CRUD

**lpush/rpush/ 左/右插入**

**lpop/rpop/lrange	左右弹出/遍历读取**

**del  删除**

![image-20201219104516416](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20201219104516416.png)



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219104606.png)

### 5.4.2 注意事项

![image-20201219104633183](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20201219104633183.png)

## 5.5 Set(集合) - 介绍

Redis 的Set 是**string 类型**的无序集合。

底层是HashTable 数据结构, Set 也是存放很多字符串元素，字符串元素是**无序的**，而且元素的**值不能重复**

### 5.5.1 Set(集合)- CRUD

**sadd**
**smembers[取出所有值]**
**sismember[判断值是否是成员]**
**srem [删除指定值]**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219105127.png)

# 六、第三方操作redis

## 6.1 golang操作redis

### 6.1.1 安装依赖包

安装第三方开源Redis 库

1) 使用第三方开源的redis 库: `github.com/garyburd/redigo/redis`

2) 在使用Redis 前，先安装第三方Redis 库，在GOPATH 路径下执行安装指令:D:\goproject>go get github.com/garyburd/redigo/redis

3) 安装成功后,可以看到如下包

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219165356.png)

### 6.1.2 简单Set、Get

```go
package main

import (
	"fmt"
	"github.com/garyburd/redigo/redis"
	"log"
)

func main() {
	//连接到redis服务器
	conn, err := redis.Dial(
		"tcp",
		"47.103.203.133:6379",
	)
	defer conn.Close()
	if err != nil {
		log.Panic("连接失败！", err)
	}

	//写入数据
	_, err = conn.Do("SET", "xwj", "hahaha")
	if err != nil {
		log.Panic("执行失败！", err)
	}

	//读取数据
	//将读取的数据断言，因为返回的是一个接口
	s, err := redis.String(conn.Do("GET", "xwj"))
	if err != nil {
		log.Panic("读取失败!", err)
	}
	fmt.Println("内容为:" , s)
}
```

### 6.1.3 读取Hash

```go
func main() {
	c, err := redis.Dial("tcp", "47.103.203.133:6379")
	if err != nil {
		log.Panic("连接错误!", err)
	}
	defer c.Close()
	_, err = c.Do("Hmset", "people01", "name", "xwj", "age", 18, "job", "it")
	if err != nil {
		log.Panic("写入错误！", err)
	}
	name, err := redis.String(c.Do("HGet", "people01", "name"))
	if err != nil {
		log.Panic("读取失败！",err)
	}
	age, err := redis.Int(c.Do("HGet", "people01", "age"))
	if err != nil {
		log.Panic("读取失败！",err)
	}
	job, err := redis.String(c.Do("HGet", "people01", "job"))
	if err != nil {
		log.Panic("读取失败！",err)
	}
	fmt.Println(name, strconv.Itoa(age), job)
}
```

### 6.1.4 批量读取

```go
func main() {
	conn, err := redis.Dial("tcp", "47.103.203.133:6379")
	if err != nil {
		log.Panic("连接失败!", err)
	}
	defer conn.Close()
	_, err = conn.Do("Hmset", "people02", "name", "zz", "age", 28, "job", "it")
	if err != nil {
		log.Panic("写入错误!", err)
	}
	//strings可以将多个不同类型的接口值都断言为string数组
	strings, err := redis.Strings(conn.Do("hmget", "people02", "name", "age", "job"))
	if err != nil {
		log.Panic("读取失败!", err)
	}
	fmt.Println(strings)
}
```

### 6.1.5 给数据设置有效时间

核心代码:

`_, err = c.Do("expire", "name", 10)`

### 6.1.6 操作List

核心代码

```go
_, err = c.Do("lpush", "heroList", "no1:宋江", 30, "no2:卢俊义", 28)
r, err := redis.String(c.Do("rpop", "heroList"))
```

# 七、Redis连接池

通过Golang 对Redis 操作， 还可以通过Redis 链接池, 流程如下：

1) 事先初始化一定数量的链接，放入到链接池

2) **当Go 需要操作Redis 时，直接从Redis 链接池取出链接即可。**

3) 这样可以节省临时获取Redis 链接的时间，从而提高效率.

4) 示意图

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219172237.png)



```go
//定义一个全局的pool
var pool *redis.Pool
//当启动程序时，就初始化连接池
func init() {
	pool = &redis.Pool{
		MaxIdle: 8, 		//最大空闲链接数
		MaxActive: 0, 		// 表示和数据库的最大链接数， 0 表示没有限制
		IdleTimeout: 100, 	// 最大空闲时间
		Dial: func() (redis.Conn, error) { // 初始化链接的代码， 链接哪个ip 的redis
			return redis.Dial("tcp", "47.103.203.133:6379")
		},
	}
}

func main() {
	//先从pool 取出一个链接
	conn := pool.Get()
	defer conn.Close()

	//同样的使用
	_, err := conn.Do("Set", "name", "汤姆猫~~")
	if err != nil {
		fmt.Println("conn.Do err=", err)
		return
	}
	//取出
	r, err := redis.String(conn.Do("Get", "name"))
	if err != nil {
		fmt.Println("conn.Do err=", err)
		return
	}
	fmt.Println("r=", r)
	//如果我们要从pool 取出链接，一定保证链接池是没有关闭
	//defer pool.Close()

	//同样的使用
	conn2 := pool.Get()
	_, err = conn2.Do("Set", "name2", "汤姆猫~~2")
	if err != nil {
		fmt.Println("conn.Do err~~~~=", err)
		return
	}
	//取出
	r2, err := redis.String(conn2.Do("Get", "name2"))
	if err != nil {
		fmt.Println("conn.Do err=", err)
		return
	}
	fmt.Println("r=", r2)
	//fmt.Println("conn2=", conn2)
}
```

# Tips

## 1.DENIED Redis is running in protected mode because protected mode is enabled, no bind address was specified, no authenticatio n password is requested to clients.

需要修改配置文件redis.conf

因为我是放在服务器上的，而默认的配置时本地并且开启了保护模式，所以修改下

把`bind 127.0.0.1`注释掉

把protected-mode设置为no

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201219165808.png)

## 2. NOAUTH Authentication required.

如果设置了密码，那么就需要你修改Dial连接的参数

进入Dial函数，设置dialOptions字段，password为密码。

```
func Dial(network, address string, options ...DialOption) (Conn, error) {
	do := dialOptions{
		dialer: &net.Dialer{
			KeepAlive: time.Minute * 5,
		},
		password: "xxxxxx",   #你的密码
	}
	for _, option := range options {
		option.f(&do)
	}
	if do.dial == nil {
		do.dial = do.dialer.Dial
	}

	netConn, err := do.dial(network, address)
	if err != nil {
		return nil, err
	}
	.......
}
```

dialOptions结构体全部字段：

```
type dialOptions struct {
	readTimeout  time.Duration
	writeTimeout time.Duration
	dialer       *net.Dialer
	dial         func(network, addr string) (net.Conn, error)
	db           int
	password     string
	useTLS       bool
	skipVerify   bool
	tlsConfig    *tls.Config
}
```

这样修改了源代码之后，就可以正常使用了。