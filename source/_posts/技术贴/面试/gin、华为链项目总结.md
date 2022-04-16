---
title: gin、项目总结
tags:
  - null
categories:
  - null
toc: true
date: 2021-11-22 14:17:28
---

[TOC]

<!-- more -->

# 一、gin

## 1. gin的特点

* 快速：基于Radix树的路由，小内存占用，没有反射

* 支持中间件：传入的http请求可以通过一系列的中间件和最终操作处理（例如：logger、Authorization等）

  ```go
  // 不使用默认中间件
  r := gin.New()
  // Default 使用 Logger 和 Recovery 中间件
  r := gin.Default()
  ```

* Crash处理：可以catch一系列在http请求中的panic然后recover，保证服务始终可用

  ```go
  // Recovery 中间件会恢复(recovers) 任何恐慌(panics) 如果存在恐慌,中间件将会写入500返回给客户端。
  func Recovery() HandlerFunc {
  	return RecoveryWithWriter(DefaultErrorWriter)
  }
  ```

* Json验证：可以解析并验证请求的json，例如检查所需要的值是否存在

* 路由组：路由可以按照是否需要授权分组

  通过分组和中间件，可以让不同的分组请求适配于不同的中间件。每个路由都可以添加任意数量的中间件

  ```go
  auth := r.Group("api/v1")
  auth.Use(middleware.JwtToken())
  {
    ...				// 需要授权才可以使用
  }
  public := r.Group("api/v1")
  {
    ...				// 无需授权
  }
  
  // 你可以为每个路由添加任意数量的中间件。
  r.GET("/benchmark", MyBenchLogger(), benchEndpoint)
  ```

* 错误管理：提供了一系列方法收集HTTP请求期间的错误方便log、数据库记录

* 内置渲染：gin为json、xml、html的渲染提供了易于使用的API

  ```go
  //  gin.H 是 map[string]interface{} 的一种快捷方式
  c.JSON(200, gin.H{
    "message": "pong",
  })
  
  c.XML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
  c.YAML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
  c.HTML(http.StatusOK, "index.tmpl", gin.H{
    "title": "Main website",
  })
  ```

* 可拓展性：方便创建一个中间件

## 2. 什么是中间件

中间件就是**包装原始应用并添加一些额外的功能**，也就是在请求的过程中新增一层，最大的特点就是封装好之后可以实现简单的拔插。例如在一个http请求的处理过程中加入例如log记录、权限管理等中间操作，一个语句就可以使用

## 3. 怎样自己写一个中间件

```go
func Logger() gin.HandlerFunc {
   return func(c *gin.Context) {
      t := time.Now()
      // Set example variable
      c.Set("example", "12345")
      // before request
      c.Next()
      // after request
      latency := time.Since(t)
      log.Print(latency) //时间  0s
      // access the status we are sending
      status := c.Writer.Status()
      log.Println(status) //状态 200
   }
}
func main() {
   r := gin.New()
   r.Use(Logger())
 
   r.GET("/test", func(c *gin.Context) {
      example := c.MustGet("example").(string)
      // it would print: "12345"
      log.Println(example)
   })
   // Listen and serve on 0.0.0.0:8080
   r.Run(":8080")
}
```

返回一个`gin.HandlerFunc`类型的函数即可

## 4. gin绑定获取请求参数

### 模型绑定：

请求体绑定到结构体，需要注意结构体的`json tag`要与客户端发送的请求数据体一致 

* `ShouldBindQuery` 函数只绑定 url 查询参数而忽略 post 数据

* `ShouldBindJSON`函数可以绑定Post请求中的json实例，赋值给对应的结构体

  ```go
  var data model.User            //创建实例
  err := c.ShouldBindJSON(&data) //绑定JSON赋值给User，就是接收前端的数据并转化为模型
  ```

* Gin提供了两类绑定方法：
  - Must bind
    * **Methods** - `Bind`, `BindJSON`, `BindXML`, `BindQuery`, `BindYAML`
    * **Behavior** - 这些方法属于 `MustBindWith` 的具体调用。 如果发生绑定错误，则**请求终止**，并触发 `c.AbortWithError(400, err).SetType(ErrorTypeBind)`。响应状态码被设置为 400 并且 `Content-Type` 被设置为 `text/plain; charset=utf-8`。 如果您在此之后尝试设置响应状态码，Gin会输出日志 `[GIN-debug] [WARNING] Headers were already written. Wanted to override status code 400 with 422`。 如果您希望更好地控制绑定，考虑使用 `ShouldBind` 等效方法。
  - Should bind
    * **Methods** - `ShouldBind`, `ShouldBindJSON`, `ShouldBindXML`, `ShouldBindQuery`, `ShouldBindYAML`
    * **Behavior** - 这些方法属于 `ShouldBindWith` 的具体调用。 **如果发生绑定错误，Gin 会返回错误并由开发者处理错误和请求。**

### 参数获取：

查询字符串参数：

```go
// 示例 URL： /welcome?firstname=Jane&lastname=Doe
router.GET("/welcome", func(c *gin.Context) {
  firstname := c.DefaultQuery("firstname", "Guest")
  lastname := c.Query("lastname") // c.Request.URL.Query().Get("lastname") 的一种快捷方式

  c.String(http.StatusOK, "Hello %s %s", firstname, lastname)
})
```

### 路由参数获取：

```go
// 此 handler 将匹配 /user/john 但不会匹配 /user/ 或者 /user
router.GET("/user/:name", func(c *gin.Context) {
  name := c.Param("name")
  c.String(http.StatusOK, "Hello %s", name)
})

// 此 handler 将匹配 /user/john/ 和 /user/john/send
// 如果没有其他路由匹配 /user/john，它将重定向到 /user/john/
router.GET("/user/:name/*action", func(c *gin.Context) {
  name := c.Param("name")
  action := c.Param("action")
  message := name + " is " + action
  c.String(http.StatusOK, message)
})
```

## 5. gin使用goroutine时的注意点

当在中间件或 handler 中启动新的 Goroutine 时，**不能**使用原始的上下文，必须使用只读副本

```go
r.GET("/long_async", func(c *gin.Context) {
  // 创建在 goroutine 中使用的副本
  cCp := c.Copy()
  go func() {
    // 用 time.Sleep() 模拟一个长任务。
    time.Sleep(5 * time.Second)

    // 请注意您使用的是复制的上下文 "cCp"，这一点很重要
    log.Println("Done! in path " + cCp.Request.URL.Path)
  }()
})

r.GET("/long_sync", func(c *gin.Context) {
  // 用 time.Sleep() 模拟一个长任务。
  time.Sleep(5 * time.Second)

  // 因为没有使用 goroutine，不需要拷贝上下文
  log.Println("Done! in path " + c.Request.URL.Path)
})
```

## 6. gin自定义http的配置

```go
func main() {
	router := gin.Default()

	s := &http.Server{
		Addr:           ":8080",
		Handler:        router,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}
	s.ListenAndServe()
}
```

## 7. gin的重定向

HTTP 重定向很容易。 内部、外部重定向均支持。

```go
r.GET("/test", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")
})
```

通过 POST 方法进行 HTTP 重定向。请参考 issue：[#444](https://github.com/gin-gonic/gin/issues/444)

```go
r.POST("/test", func(c *gin.Context) {
	c.Redirect(http.StatusFound, "/foo")
})
```

路由重定向，使用 `HandleContext`：

```go
r.GET("/test", func(c *gin.Context) {
    c.Request.URL.Path = "/test2"
    r.HandleContext(c)
})
r.GET("/test2", func(c *gin.Context) {
    c.JSON(200, gin.H{"hello": "world"})
})
```

## 8. gin的静态文件处理

```go
func main() {
	router := gin.Default()
	router.Static("/assets", "./assets")
	router.StaticFS("/more_static", http.Dir("my_file_system"))
	router.StaticFile("/favicon.ico", "./resources/favicon.ico")

	// 监听并在 0.0.0.0:8080 上启动服务
	router.Run(":8080")
}
```

## 9. 中间件的洋葱模型

中间件的模型如下，在gin中每个中间件会通过:

* `c.Next()`：让请求进入下一层。（在`gin`中做了优化也可以不写）
* `c.Abort()`：中断在当前层

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/uFzPJb.png" alt="uFzPJb" style="zoom: 50%;" />

本质上是一个`handler`函数的数组搭配一个索引指针`index`, 然后不断的向下**递归**执行

```go
const AbortIndex = math.MaxInt8 / 2

type Context struct {
	Handlers []func(c *Context)
	Index    int8
}

func (c *Context) GET(path string, f func(c *Context)) {
	c.Handlers = append(c.Handlers, f)
}

func (c *Context) Use(f func(c *Context)) {
	c.Handlers = append(c.Handlers, f)
}

func (c *Context) Run() {
	c.Handlers[0](c) // 洋葱模式的开始，执行第一个函数
}

func (c *Context) Next() {
	if c.Index < int8(len(c.Handlers)) {
		c.Index++
		c.Handlers[c.Index](c) // 执行Next会向下递归执行
	}
}

func (c *Context) Abort() {
	c.Index = AbortIndex // 指定index为一个需要终止的值
}

func main() {
	c := &Context{}
	c.Use(Middle1())
	c.Use(Middle2())
	c.Use(Middle3())
	c.GET("/", func(c *Context) {
		fmt.Println("get function.")
	})
	c.Run()
}

func Middle1() func(c *Context) {
	return func(c *Context) {
		fmt.Println("middle-1-start")
		c.Next()
		fmt.Println("middle-1-end")
	}
}

func Middle2() func(c *Context) {
	return func(c *Context) {
		fmt.Println("middle-2-start")
		c.Abort() // 如果中断，那么洋葱中心的最终函数就不会执行
		c.Next()
		fmt.Println("middle-2-end")
	}
}

func Middle3() func(c *Context) {
	return func(c *Context) {
		fmt.Println("middle-3-start")
		c.Next()
		fmt.Println("middle-3-end")
	}
}

// Next()
// middle-1-start
// middle-2-start
// middle-3-start
// get function.
// middle-3-end
// middle-2-end
// middle-1-end

// Abort()
// middle-1-start
// middle-2-start
// middle-2-end
// middle-1-end
```

## 10. GET、POST、PUT、DELETE区别

都属于`Restful`的接口规范 (还有一个种方式就是`rpc`)

* get：获取服务器资源，类似于查询，不会修改数据
* post：向服务端发送数据，类似于插入，会新增服务端的数据状态
* put：向服务端发送数据，类似于更新，会改变服务端的数据状态，但是不会新增
* delete：请求服务端删除一个资源，类似于删除，会减少服务端的数据状态

比较：

* get VS post：get的参数放在请求url上，post的请求数据放在请求体中，get的请求会在浏览器产生缓存而post不会。一个是查询数据的方式，一个是发送数据的方式
* put VS post： 两者都是发送数据，但是post的url一般对应到一个集合资源上新增，而put的url一般对应到一个具体的对象上修改或更新

## 11.  `Restful`与`rpc`的区别

`restful`是面向资源的，多用于C/S架构； `rpc`是远程过程调用，多用于微服务

* 从本质上看rpc可以基于tcp或http，而restful是基于Http来实现的
* 从传输的数据量上来看，restful是http封装的所以数据量更大，而rpc更加轻量级也更快
* 因为一般使用restful的服务都是toC的，不明白请求的来源，http协议是各个框架都普遍支持的，所以可以在网关使用restful利用http来接收；而rpc一般用于公司内部自定义的协议，都是商量好的，知道数据模式，所以用tcp传输更加快速

* restful的API是面向资源的，对于一个资源的操作体现在GET、POST、PUT、DELETE上，同一个url可以有不同的操作；而rpc通常直接将操作动作放在请求url上

## 12. RPC框架要做到的最基本的三件事：

1. 服务端如何确定客户端要调用的函数；

   在远程调用中，客户端和服务端分别维护一个【ID->函数】的对应表， ID在所有进程中都是唯一确定的。客户端在做远程过程调用时，附上这个ID，服务端通过查表，来确定客户端需要调用的函数，然后执行相应函数的代码。

2. 如何进行序列化和反序列化；

   客户端和服务端交互时将参数或结果转化为字节流在网络中传输，那么数据转化为字节流的或者将字节流转换成能读取的固定格式时就需要进行序列化和反序列化，序列化和反序列化的速度也会影响远程调用的效率。

3. 如何进行网络传输（选择何种网络协议）；

   多数RPC框架选择TCP作为传输协议，也有部分选择HTTP。如gRPC使用HTTP2。不同的协议各有利弊。TCP更加高效，而HTTP在实际应用中更加的灵活。

## 13. session和cookie的介绍与区别

* **存储位置不同：**session存储在服务器上，cookie存储在客户端浏览器上
* **存储量不同：**session的大小没有限制，但是不会存放很多东西并会设置删除机制；cookie保存的数据<=4kb， 一个站点最多保存20个cookie
* **存储方式不同：**session能存储任何数据类型的数据，而cookie只能保管ASCII字符串并编码为unicode字符或二进制数据
* **安全性不同：**session安全性高对用户透明，cookie对客户端可见安全性低可能会被恶意分析
* **有效期不同：**session依赖于JSESSIONID的cookie，一般关闭窗口该session就会失效。cookie可以设置属性从而达到长期有效的作用
* **跨域不同：**session不支持跨域，而cookie支持

## 14. 基于session的认证与基于token的认证

### 传统的基于session认证

http是一种**无状态的协议**，不可能每次访问服务器资源都需要提供用户名和密码，所以一般就会在服务端保存session然后当服务端返回数据时，会保存为cookie在浏览器中以便下一次使用

缺点：

* session保存在服务端**内存**，如果用户量大的话就会导致服务端压力过大
* 一个session将用户与一个服务端绑定，不利于分布式场景
* cookie保存在浏览器有被攻击的风险

### 基于token的认证

token的认证方式也是无状态的，不将用户与服务端绑定。在每个用户第一次登陆的时候，服务端根据用户信息生成一个时效token，然后返回给用户，下一次再需要请求的时候，用户只需要在请求头中加入这个token就可以访问对应的服务。

## 15. JWT的组成部分/Token的一些其他问题

![UY7YbE](http://xwjpics.gumptlu.work/qinniu_uPic/UY7YbE.png)

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

**token被别人获取了是不是就可以访问你的页面了？**

是的，token就像是一个临时密码。解决方法：

* token时间不要设置过长
* 服务端中可以记录token和对应的IP，防止其他IP登陆

* 只读数据可以放宽，但是对于重要的数据操作接口可以加一层密码验证

**token会不会重复？**

如果在claim中加入时间戳的话一般不会

**token的验证？**

* 发送时会用一个密钥加密
* 接收时将收到的token解密，然后使用相同的签发人签名对比签名字段是否相同

## 16. Cookie泄漏问题怎么解决

* 给cookie添加`HttpOnly`属性，这样就cookie就只能在http请求中使用无法在脚本中使用
* 使用基于`Token`的认证方式
* Https

# 二、华为链项目

## 0. 项目目的是什么？解决了什么问题

* 区块链数字身份认证网络

  * 用户管理：用户的注册、登陆、授权（JWT）

  * 功能介绍：使用区块链保存每位车主的数字身份，用于后续的扣费交互验证

    注册流程：车主提交个人注册信息 => 取Hash存入区块链中  => 各个部门核对Hash初始化证书并签名（用自己的私钥） => 最终生成链上数字证书 

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220322174159942.png" alt="image-20220322174159942" style="zoom:50%;" />

* 异构一致性区块链支付账本

  * 扣费数字认证
  * 借助区块链的一致性算法实现ETC扣费的多方同步
  * OBU、RSU硬件上实现MQTT通信（第一版是借助libp2p的节点发现，然后通过建立socket连接提供服务）

* 技术栈

  * gin、gorm对象关系映射后端框架
    * 中间件：JWT、Cors跨域、自定义日志（分割、定时清除和链接最新的日志文件）
  * socket通信、MQTT

## 1. 项目的整体框架哪些功能

后端的框架：

```shell
├─  .gitignore
│  go.mod // 项目依赖
│  go.sum
│  latest_log.log #最新log日志软连接
│  LICENSE
│  main.go //主程序
│  README.md
│  tree.txt
│          
├─api         
├─config // 项目配置入口   
├─database  // 数据库备份文件（初始化）
├─log  // 项目日志
├─middleware  // 中间件
├─model // 数据模型层
├─routes
│      router.go // 路由入口    
├─static // 打包静态文件          
├─upload   
├─utils // 项目公用工具库
│  │  setting.go 
│  ├─errmsg   
│  └─validator         
```

1. 用户管理权限设置
2. 用户密码加密存储
3. 列表查询分页
4. 图片上传七牛云
5. 中间件实现：JWT 认证、跨域 cors、自定义日志（分割、定时清除和链接最新的日志文件）
6. 与区块链交互
7. 硬件上的pubsub通信

技术栈：gin框架开发，对象关系映射使用gorm

## 2. 后端项目中使用了哪些中间件, 怎样实现的？

Logger日志、Cors跨域、JWT

### Logger日志

使用了`logrus`包来记录日志文件，使用`rotatelogs`来分割、定时清除和链接最新的日志文件

```go
logWriter, _ := rotatelogs.New(
  //logFilePath+"_%Y_%m_%d.log",
  "log/log_%Y_%m_%d.log",                    //文件格式名
  rotatelogs.WithMaxAge(7*24*time.Hour),     //最大保存时间
  rotatelogs.WithRotationTime(24*time.Hour), //文件分割时间
  rotatelogs.WithLinkName(linkLogName),      //软链接最新的log文件
)
```

然后记录的步骤如下：

1. 获取当前时间
2. Next
3. 计算经过的时间`time.Since()`
4. 从请求头中获取并记录请求的信息（请求的方法、IP、访问代理工具、数据大小等）
5. 创建log实体``logger.WithFields``然后根据类型记录

### Cors跨域

https://segmentfault.com/a/1190000040485198

什么是跨域？

* 一个请求的URL包含`协议://域名:端口`只要这三者有一项不同，对于浏览器来说都是跨域
* 浏览器的跨域限制就是同源保护，避免一个域与另一个域内容直接进行交互，因为这样可能会导致XSS、CSFR等攻击

怎样解决跨域？

* 前端：通过nginx反向代理实现：因为服务器之间(B、C)不存在跨域问题
  * 正向代理：A->B->C, B就是一个代理服务器，将请求转发到C，重点在于A明确目的地址为C
  * 反向代理：A->B<-C, A只要访问B即可，B从C拿取资源返回给A，A无需知道C只要知道B即可
* 添加响应头解决跨域（项目中做法）：浏览器询问B，如果B允许其访问则实现跨域
  * access-control-allow-origin、access-control-max-age

使用gin自带的跨域工具`cors`

```go
func Cors() gin.HandlerFunc {
	return cors.New(
		cors.Config{
			AllowAllOrigins:        true,
			AllowMethods:           []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
			AllowBrowserExtensions: true,
			AllowHeaders:           []string{"*"},
			ExposeHeaders:          []string{"Content-Length", "Authorization", "Content-Type"},
			AllowCredentials:       true,
			MaxAge:                 12 * time.Hour,
		},
	)
}
```

### JWT

`JWT(Json web token)`是一种基于token的认证方式。

项目中申明了一个结构体实现了`JWT`的`Claim`接口， 使用了`JWT`的一些标准字段以及用户名：

```go
type Myclaim struct {
	Username           string `json:"username"`
	jwt.StandardClaims        //其中包含了jwt中间件的一些标准字段
}
```

然后获取当前的时间，设置了token到期的时间并签名（类似于加盐）返回最终的token

```go
const t = 24 * 2 //默认的有效时间 （单位小时）

//生成Token
func SetToken(username string) (string, int) {
	vaildTime := time.Now().Add(t * time.Hour) //计算有效时间
	newClaim := &Myclaim{
		Username: username,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: vaildTime.Unix(), //这里要传时间戳
			Issuer:    "xwj",            //签名人
		},
	}
	//生成Token对象
	//SigningMethodHS256签名模式使用标准模式
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, newClaim)
	//对token签名,类似于加盐
	tokenString, err := token.SignedString(tokenSignKey)
	if err != nil {
		fmt.Println(err)
		return "", errormsg.ERROR
	}
	return tokenString, errormsg.SUCCESS
}
```

检查token的正确性就是：

1. 将token string转换为claim
2. 断言为自定义的claim类型
3. 使用`token.Valid`验证是否正确

```go
//验证Token
func CheckToken(tokenString string) (*Myclaim, int) {
	token, err := jwt.ParseWithClaims(tokenString, &Myclaim{}, func(token *jwt.Token) (interface{}, error) {
		return tokenSignKey, nil
	})
	if err != nil {
		return nil, errormsg.ERROR
	}
	myclaim, code := token.Claims.(*Myclaim) //把token的Claim断言为Myclain类型
	if token.Valid && code {                 //如果验证有效就返回登录数据
		return myclaim, errormsg.SUCCESS
	} else {
		return nil, errormsg.ERROR
	}
}
```

中间件函数的实现过程：

1. 获取请求头中的`Authorization`字段
2. 检查一下token格式是否是`Bearer`
3. 检查token是否正确（用上面的函数）
4. c.Next()

## 3. 数据关系怎样定义与设计的

项目的数据关系很简单：

用户、注册信息、证书。后端只存储用户和注册信息是一个一对多的关系，所以注册信息中保留一个用户ID就可以了，证书保存在区块链中存储。

但是gorm相关的高级查询也有尝试过

用户分为普通用户和管理员，管理员就是一些注册机构

## 4. 怎样实现上传图片到七牛云返回链接

`github.com/qiniu/api.v7/v7 v7.8.1`使用七牛云提供的包开发

1. 先将两个密钥Ak，SK配置好, 生成上传到七牛云的token

   `qbox.NewMac(AccessKey, SecretKey)`

2. 然后配置好区域、是否启用https等

   ```go
   cfg := storage.Config{
     // 配置存储空间
     Zone:          &storage.ZoneHuanan, // 空间对应的机房
     UseCdnDomains: false,               // 上传是否使用CDN上传加速
     UseHTTPS:      false,               // 是否使用https域名
   }
   ```

3. 根据配置构建表单上传对象，上传

   ```go
   formUploader := storage.NewFormUploader(&cfg)
   ret := storage.PutRet{} //结果返回对象
   //开始上传
   err := formUploader.PutWithoutKey(context.Background(), &ret, upToken, file, fileSize, &putExtra)
   ```

   （有意思的是他是将结果对象的指针放在请求参数中）

4. 拼接文件的url：

   `url := ImgUrl + "/" + ret.Key`

## 5. 怎样实现错误处理

单独用一个文件编写了所有的错误类型的关系映射，用map[int]string保存，key为自定义错误的状态码，value是错误的提示输出，返回这样一个映射。运行过程中出现错误需要返回就对应返回

## 6. ORM是什么，项目中怎样使用gorm数据迁移

ORM即`Object-Relationl Mapping`对象关系映射，它的作用是**在关系型数据库和对象之间作一个映射**，这样，我们在具体的操作数据库的时候，就不需要再去和复杂的SQL语句打交道，只要像平时操作对象一样操作它就可以了 。

创建结构体的时候加入为了保证gorm对象关系映射，加入了gorm的tag：

```go
type User struct {
	//	ID        uint `gorm:"primarykey"`
	//	CreatedAt time.Time
	//	UpdatedAt time.Time
	//	DeletedAt DeletedAt `gorm:"index"`
	gorm.Model	//引入gorm自带的字段 见上方
	UserName string `gorm:"type:varchar(20); not null" json:"user_name"`
	PassWord string `gorm:"type:varchar(20); not null" json:"pass_word"`
	Role int `gorm:"type:int;" json:"role"`	//角色 权限管理
}
```

连接数据库的过程：

1. 创建数据库	

   ```go
   dbConfig := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
   		utils.DbUser, utils.DbPassWord, utils.DbHost, utils.DbPort, utils.DbName)
   db, err = gorm.Open(mysql.Open(dbConfig), &gorm.Config{})
   if err != nil {
     fmt.Println("连接数据库失败！请检查配置", err)
   }
   ```

2. 自动迁移自动创建数据表

   `db.AutoMigrate(&User{}, &RegistryInfo{})` （这里指定好需要迁移的结构体）

3. 获取数据库对象

   ```go
   // 获取通用数据库对象 sql.DB ，然后使用其提供的功能
   sqlDB, err := db.DB()
   ```

4. 配置数据库连接池(最大连接数、时长等)

## 7. 用户密码的加密存储怎样实现

没有采用Hash存储的方式，而是使用了golang的scrypt包，具体的使用方法就是:

1. 对于原密码加盐加密（加盐的目的是让密码变长，让彩虹分析更加困难，更加安全）

   > 彩虹攻击，是指攻击者存储了一个大的`密码->hash`字典表Rainbow Tables。相比于普通的字典表，Rainbow Tables经过了空间优化和查找优化

2. 然后将密文使用base64编码返回

```go
//这里采用的是scrypt的加密方法
//加密,不需要解密，只需要比对
func InfoEncrypto(originaldata string, salt string) string {
	//salt是加盐（一般取8字节）,data是原始数据
	//加盐的目的是让原始数据变长，难以解密
	//N is a CPU/memory cost parameter
	//r and p must satisfy r * p < 2³⁰. 不然报错
	//keyLen输出长度
	key, err := scrypt.Key([]byte(originaldata), []byte(salt)[:8], 1<<15, 8, 1, 32)
	if err != nil {
		fmt.Println("scrypt err = ", err)
		return ""
	}
	return base64.StdEncoding.EncodeToString(key)	//返回base64编码结果
}
```

在用户创建、登陆的时候经过这一道工序

在用户创建的时候的方式可以采用gorm的hock函数：

```go
// 添加一个钩子函数
// 在明文密码存储到数据库之前进行加密
//在GORM 中保存、删除操作会默认运行在事务上，
//因此在事务完成之前该事务中所作的更改是不可见的，如果您的钩子返回了任何错误，则修改将被回滚。
func (u *User)BeforeCreate(){
	//用户密码的加密转化
    //这里的盐就是用户的创建时间
	u.PassWord = utils.InfoEncrypto(u.PassWord, u.CreatedAt.String())
}
```

## 8. 怎样实现分页查询

```go
//Limit限制每页显示的个数,offset偏移量，Find查找
//Preload
err := db.Preload("Category").Limit(pageSize).Offset((pageNum - 1) * pageSize).Find(&as).Error
```















