---
title: go-zero项目开发重点记录
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-18 14:29:46
---

> * https://go-zero.dev/cn

[TOC]

记录学习官方文档中项目开发的重点知识

<!-- more -->

# 一、整体框架

![image-20220410174912353](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410174912353.png)

# 二、一般的开发流程

![image-20220410175528816](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410175528816.png)

https://go-zero.dev/cn/dev-flow.html

官方给出了整体的开发流程：

- goctl环境准备
- 数据库设计
- 业务开发
- 新建工程
- 创建服务目录
- 创建服务类型（api/rpc/rmq/job/script）
- 编写api、proto文件
- 代码生成
- 生成数据库访问层代码model
- 配置config，yaml变更
- 资源依赖填充（ServiceContext）
- 添加中间件
- 业务代码填充
- 错误处理

# 三、业务分解重点

* **微服务的拆分**：对于一个业务纵向的拆分成多个子系统，每个子系统有独立的缓存和数据库机制，每一个子系统都是一个服务，对外提供了两种访问该系统的方式：**api和rpc**

  > * <font color='#39b54a'>**API的理解：**API可以直接理解为网关，但是拆解为每一个微服务后，每一个微服务都有一个单独的API的原因在，如果使用一个整体的API连接后面多个RPC，那么一旦轻微的改动都会影响API的改动和重启，所以对大的API进行了拆分，当然拆分后就需要在前方设置一个总网关，这个网关才是真正意义上的网关(`nginx、kong、apisix`)</font>
  > * <font color='#39b54a'>RPC：</font>
  >
  > 

* **rpc调用链**：服务之间的调用是单向的，不会出现循环调用；因为互相依赖调用会互相影响认为对方服务不可用，进入死循环；如果有大量服务之间互相调用，此时应该考虑是否是业务拆分是否合理

* **常见服务类型的目录结构**：

  一个服务不单单只有rpc和api，还可能有如rmq（消息处理系统），cron（定时任务系统），script（脚本）等，所以一个服务的结构可能如下所示：

  ```shell
  user
      ├── api //  http访问服务，业务需求实现
      ├── cronjob // 定时任务，定时数据更新业务
      ├── rmq // 消息处理系统：mq和dq，处理一些高并发和延时消息业务
      ├── rpc // rpc服务，给其他子系统提供基础数据访问
      └── script // 脚本，处理一些临时运营需求，临时数据修复
  ```

* 一个常见的工程目录： 

  ```shell
  mall // 工程名称
  ├── common // 通用库
  │   ├── randx
  │   └── stringx
  ├── go.mod
  ├── go.sum
  └── service // 服务存放目录
      ├── afterSale
      │   ├── api
      │   └── model
      │   └── rpc
      ├── cart
      │   ├── api
      │   └── model
      │   └── rpc
      ├── order
      │   ├── api
      │   └── model
      │   └── rpc
      ├── pay
      │   ├── api
      │   └── model
      │   └── rpc
      ├── product
      │   ├── api
      │   └── model
      │   └── rpc
      └── user
          ├── api
          ├── cronjob
          ├── model
          ├── rmq
          ├── rpc
          └── script
  ```

# 四、实现jwt鉴权功能

https://go-zero.dev/cn/jwt.html

## 1. Jwt Token的组成

* jwt组成

![UY7YbE](D:\Users\sangfor\Desktop\UY7YbE.png)

* Header：同步由以下json结构生成，生成的方式是将整个json字符串经过Base64Url编码

  ```json
  {
    "typ": "JWT",			// 表示是一个JWT字符串
    "alg": "HS256"		// 表示Hash摘要算法
  }
  ```

* Payload: 用来承载需要传递的数据，其一个属性对成为claim标准，这样为claim标准，同样Base64Url编码

  ```json
  {
    "sub": "123456",
    "name": "Jone",
    "admin": true
  }
  ```

* Signature: 将前两个字符串加一个`.`拼接然后通过head头中的hash摘要算法处理

## 2. 项目使用重点：

* api描述文件中，如果路由需要鉴权，那么在其上添加鉴权的`@server`就表示开启

  ```go
  @server(
      jwt: Auth
  )
  service search-api {
      @handler search
      get /search/do (SearchReq) returns (SearchReply)
  }
  
  
  service search-api {
      @handler ping
      get /search/ping
  }
  ```

  `/search/do`有鉴权功能，而`/search/ping`路由不需要jwt鉴权

* `jwt token`的鉴权是go-zero内部已经封装了，你只需在api文件中定义服务时简单的声明一下即可。

* 项目添加Jwt Token验证的整体流程如下：

![Snipaste_2022-01-19_11-31-52](D:\Users\sangfor\Desktop\Snipaste_2022-01-19_11-31-52.png)

* **token中的负载claims（k/v对），在验证方可以通过req拿到，gozero会将用户生成token时传入的kv原封不动的放在http.Request的Context中，会传递到逻辑层的l.ctx中**

​		可以用于进一步的验证用户身份（例如IP等）

# 五、实现中间件

## 1. 中间件原理

中间件就是**包装原始应用并并添加一些额外的功能**，也就是在请求的过程中新增一层，最大的特点就是封装好之后可以实现简单的拔插。

例如在一个http请求的处理过程中加入例如log记录、权限管理等中间操作，一个语句就可以使用

一般会有两个函数：

* `Next()`：让请求进入下一层。（在`gin`中做了优化也可以不写）
* `Abort()`：中断在当前层

<img src="D:\Users\sangfor\Desktop\uFzPJb.png" alt="uFzPJb" style="zoom: 50%;" />

## 2. 项目中使用

* go-zero有两种中间件：

  * 路由中间件：只适用于用于某一个路由，规则取决于`api`中`@server`是否放在该路由服务上方

    ```go
    @server(
    	// 开启jwt鉴权
    	jwt: Auth
    	middleware: Example // 路由中间件申明
    )
    service search-api {
    	@handler search
    	get /search/do (SearchReq) returns (SearchReply)
    }
    ```

  * 全局中间件：所有路由都会使用的中间件，服务范围是整个服务, 可以直接在服务`main`函数添加`rest.Server.Use()`即可

    ```go
    func main() {
    	flag.Parse()
    
    	var c config.Config
    	conf.MustLoad(*configFile, &c)
    
    	ctx := svc.NewServiceContext(c)
    	server := rest.MustNewServer(c.RestConf)
    	defer server.Stop()
    	
    	// 使用全局中间件
    	server.Use(func(next http.HandlerFunc) http.HandlerFunc {
    		return func(writer http.ResponseWriter, request *http.Request) {
    			logx.Infof("全局中间件触发")
    			next(writer, request)
    		}
    	})
    	
    	handler.RegisterHandlers(server, ctx)
    
    	fmt.Printf("Starting server at %s:%d...\n", c.Host, c.Port)
    	server.Start()
    }
    ```

* 添加了路由中间件配置会自动生成一个`middleware`文件夹，在其中编写逻辑

* 把其他服务传递给中间件：用闭包的方式调用

  ```go
  // 模拟的其它服务实体
  type AnotherService struct{}
  
  func (s *AnotherService) GetToken() string {
      return stringx.Rand()
  }
  
  // 常规中间件
  func middleware(next http.HandlerFunc) http.HandlerFunc {
      return func(w http.ResponseWriter, r *http.Request) {
          w.Header().Add("X-Middleware", "static-middleware")
          next(w, r)
      }
  }
  
  // 调用其它服务的中间件
  func middlewareWithAnotherService(s *AnotherService) rest.Middleware {
      return func(next http.HandlerFunc) http.HandlerFunc {
          return func(w http.ResponseWriter, r *http.Request) {
              w.Header().Add("X-Middleware", s.GetToken())
              next(w, r)
          }
      }
  }
  ```

# 六、RPC编写和调用

* go-zero使用zrpc实现在多个子服务之间调用，zrpc基于grpc实现
* 大多数RPC服务都是该子服务给其他子服务提供的数据调用接口
* 一个服务的api与rpc可以独立启动、服务，但是api需要在rpc服务启动之后启动，这样当请求到达api的时候rpc服务才已经准备好

# 七、错误处理

错误信息都是以plain text形式返回的。业务中还会定义一些业务性错误，常用做法都是通过 `code`、`msg` 两个字段来进行业务处理结果描述，并且希望能够以json响应体来进行响应。

- 业务处理正常

  ```json
    {
      "code": 0,
      "msg": "successful",
      "data": {
        ....
      }
    }
  ```

- 业务处理异常

  ```json
    {
      "code": 10001,
      "msg": "参数错误"
    }
  ```

# 错误使用记录

## 1.生成的model层代码的数据库model实例没有缓存选项

错误描述：

编写向model层的服务依赖的时候，`UserModel: model.NewUserModel(sqlx.NewMysql(c.Mysql.DataSource), c.CacheRedis),`没有第二个选项即cache

原因：

检查发现生成的代码model对象的方法就是没有缓存选项，所以检查生成的语句是否正确，发现最后忘记加上参数`-c`

解决：

`goctl model mysql ddl -src user.sql -dir . -c`

## 2. empty etcd hosts

错误描述：启动service调用rpc的etcd服务发现配置时启动报如上错误

原因：yaml配置文件编写错误

解决：Etcd配置上面要添加具体的RPC名与config.go对应

```yaml
UserRpc:
  Etcd:
    Hosts:
      - 127.0.0.1:2379
    Key: user.rpc
```

```go
type Config struct {
	rest.RestConf
	Auth struct {
		AccessSecret string
		AccessExpire int64
	}
	UserRpc zrpc.RpcClientConf
}
```

