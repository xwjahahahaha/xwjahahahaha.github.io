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

中间件就是**包装原始应用并并添加一些额外的功能**，也就是在请求的过程中新增一层，最大的特点就是封装好之后可以实现简单的拔插。例如在一个http请求的处理过程中加入例如log记录、权限管理等中间操作，一个语句就可以使用

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

### ？参数获取：

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

# 二、项目

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
5. 中间件实现：JWT 认证、跨域 cors、自定义日志
6. 与区块链交互
7. 硬件上的pubsub通信

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/0QHPvA.png" alt="0QHPvA" style="zoom:50%;" />

技术栈：gin框架开发，对象关系映射使用gorm

## 2. 项目中使用了哪些中间件, 怎样实现的？

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

JWT(Json web token)是一种基于token的认证方式。

项目中申明了一个结构体实现了JWT的Claim接口， 使用了JWT的一些标准字段以及用户名：

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

4. 配置数据库连接池(最大连接数、时常等)

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















