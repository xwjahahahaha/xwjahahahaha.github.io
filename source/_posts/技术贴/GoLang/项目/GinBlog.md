---
title: GinBlog
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2020-12-25 22:47:39
---

# 一、项目简介

使用go做的全栈博客系统

<!-- more -->

# 二、Gin后端

## 2.1 快速开始

### 2.1.1 配置

* git或者gitee创建仓库
* 在本地下载创建的仓库
* `go mod init ginblog`创建mod包管理
* 下载Gin`go get -u github.com/gin-gonic/gin`

* 创建以下各级目录结构

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201225225648.png)

配置文件使用go-ini

安装:`go get gopkg.in/ini.v1`

编写以下配置文件:

config.ini

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226003818.png)

（按自己的配置）

在utils下创建server.go文件，加载配置文件

```go
var (
	AppMode string
	HttpPort string
	Db string
	DbHost string
	DbPort string
	DbUser string
	DbPassWord string
	DbName string
)

func init() {
	file, err := ini.Load("./config/config.ini")	//加载
	if err != nil {L
		fmt.Println("配置文件读取错误，请检查配置文件！")
	}
	LoadServer(file)
	LoadDB(file)
}

//加载server配置
func LoadServer(file *ini.File)  {
	//MustString的意思就是如果取不到值就取默认值
	AppMode = file.Section("server").Key("AppMode").MustString("debug")
	HttpPort = file.Section("server").Key("HttpPort").MustString(":3000")
}

//加载数据配置
func LoadDB(file *ini.File)  {
	Db = file.Section("database").Key("Db").MustString("mysql")
	DbHost = file.Section("database").Key("DbHost").MustString("xxxx")
	DbPort = file.Section("database").Key("DbPort").MustString("3306")
	DbUser = file.Section("database").Key("DbUser").MustString("xxxx")
	DbPassWord = file.Section("database").Key("DbPassWord").MustString("xxxx")
	DbName = file.Section("database").Key("DbName").MustString("ginblog")
}
```

注意这里的Load是**相对路径**

### 2.1.2 启动初始服务器

在routes下编写router.go文件，很类似于nodejs

```go
func InitRoute()  {
	//初始化gin
	gin.SetMode(utils.AppMode)	//设置服务器模式
	r := gin.Default()			//使用默认模式创建服务

	route := r.Group("api/v1")
	{
		route.GET("Hello", func(context *gin.Context) {
			context.JSON(http.StatusOK, gin.H{
				"msg":"ok",
			})
		})
	}
	r.Run(utils.HttpPort)
}
```

编写主函数main.go调用测试

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226010747.png)

访问结果如下：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226010806.png)

## 2.2 配置数据库、数据模型

使用[gorm官网](https://gorm.io/zh_CN/docs/index.html)

下载

`go get -u gorm.io/gorm`

`go get -u gorm.io/driver/sqlite`

在model下编写db.go

```go
var db *gorm.DB
var err error

func InitDb()  {
	//想要正确的处理 time.Time ，您需要带上 parseTime 参数
	dbConfig := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		utils.DbUser, utils.DbPassWord, utils.DbHost, utils.DbPort, utils.DbName)
	db, err = gorm.Open(mysql.Open(dbConfig), &gorm.Config{})
	if err != nil {
		fmt.Println("连接数据路失败！请检查配置", err)
	}
	//自动迁移
	//AutoMigrate 用于自动迁移您的 schema，保持您的 schema 是最新的。
	//AutoMigrate 会创建表，缺少的外键，约束，列和索引，并且会更改现有列的类型（如果其大小、精度、是否为空可更改）。
	//但 不会 删除未使用的列，以保护您的数据。
	err := db.AutoMigrate(&User{}, &Category{}, &Article{})
	if err != nil {
		fmt.Println("自动迁移失败！", err)
	}
	// 下面是gorm虚拟的数据库连接池的配置
	// 获取通用数据库对象 sql.DB ，然后使用其提供的功能
	sqlDB, err := db.DB()
	if err != nil {
		fmt.Println("获取通用数据库对象 sql.DB 错误！", err)
	}
	// SetMaxIdleConns 用于设置连接池中空闲连接的最大数量。
	sqlDB.SetMaxIdleConns(10)
	// SetMaxOpenConns 设置打开数据库连接的最大数量。
	sqlDB.SetMaxOpenConns(100)
	// SetConnMaxLifetime 设置了连接可复用的最大时间。
	// 注意这里的时间设置不要大于gin框架连接客户端的时间
	sqlDB.SetConnMaxLifetime(10 * time.Second)

}
```

编写用户、类别、文章三个对象类别

```go
//User.go
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


//Category.go
type Category struct {
	gorm.Model
	Name string `gorm:"type:varchar(20)" json:"name"`
}


//Article.go
type Article struct {
	Category Category `gorm:"-"`//类别
	gorm.Model
	Title string	`gorm:"type:varchar(20); not null" json:"title"`		//文章标题
	Desc string		`gorm:"type:varchar(200)" json:"desc"`		//文章描述
	Cid int			`gorm:"type:int; not null" json:"cid"`				//类别ID
	Content string	`gorm:"type:longtext" json:"content"`	//文章内容
	Img string 		`gorm:"type:varchar(100)" json:"img"`		//文章图片
	Author int		`gorm:"type:int" json:"author"`				//文章作者
}
```

对于文章的Category如果不写`gorm:"-"`的话会报错：<font color='red'>need to define a valid foreign key for relations or it need to implement the Valuer/Scanner interface</font>

加上`gorm:"-"`表示数据表迁移的时候（创建数据库表）忽略该字段，或者自己手动设置外键，格式`gorm:"foreignKey:xxxx"`但是这里忽略掉即可，后面再设计外键

详情见：https://stackoverflow.com/questions/63810512/invalid-field-found-for-struct-field-need-to-define-a-foreign-key-for-relation

main.go中添加`model.InitDb()  //初始化数据库`

启动项目，查看数据库中是否创建了表：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226090840.png)





## 2.3 架构错误处理与路由接口

后端实现错误字典，这样才能给前端去了解错误详情

这里的错误指的是业务逻辑上面的报错，要从后端返回给前端响应(例如用户名重复、查找失败等)，而不是指类似于404的请求错误

在Utils下创建errormsg文件夹，在其下创建errormsg.go

```go
//状态码参数
const (
	SUCCESS = 200
	ERROR = 500

	//code = 1000...表示用户模块错误
	ERROR_USERNAME_EXIST = 1001
	ERROR_PASSWORD_WRONG = 1002
	ERROR_USER_NOT_EXIST = 1003
	ERROR_TOKEN_NOT_EXIST = 1004
	ERROR_TOKEN_RUNTIME = 1005
	ERROR_TOKEN_WRONG = 1006
	ERROR_TOKEN_FORMAT_WRONG = 1007
	//code = 2000...表示文章模块错误

	//code = 3000...表示类别模块错误
)

//错误字典
var ErrorDist = map[int]string{
	SUCCESS : "Ok" ,
	ERROR : "Fail" ,

	ERROR_USERNAME_EXIST : "用户已存在",
	ERROR_PASSWORD_WRONG : "密码错误",
	ERROR_USER_NOT_EXIST : "用户不存在",
	ERROR_TOKEN_NOT_EXIST : "Token不存在",
	ERROR_TOKEN_RUNTIME : "Token已过期",
	ERROR_TOKEN_WRONG : "Token不正确",
	ERROR_TOKEN_FORMAT_WRONG : "Token格式错误",
}

//返回错误描述
func GetErrorDist(code int) string {
	return ErrorDist[code]
}
```

创建对应模块的api模型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226101332.png)

类似于User.go的方式，定义请求的上下文处理函数

```go
// 查询用户是否存在
func UserExist(c *gin.Context)  {
	
}
// 查询一个用户
func GetUser(c *gin.Context)  {

}
// 查询用户列表
func GetUsers(c *gin.Context)  {

}
// 新增一个用户
func AddUser(c *gin.Context)  {

}
// 编辑一个用户
func AlterUser(c *gin.Context)  {

}
// 删除一个用户
func DelUser(c *gin.Context)  {

}
```

然后修改route的初始化路由组

```go
routev1 := r.Group("api/v1")
	{
		//用户模块相关接口
		routev1.POST("user/add", v1.AddUser)
		routev1.GET("users", v1.GetUsers)
		routev1.PUT("user/:id", v1.AlterUser)
		routev1.DELETE("user/:id", v1.DelUser)
		//文章模块相关接口

		//类别模块相关接口

	}
```

使用v1标记api版本，对应的route也可以使用上述方式标记

**补记：**

[get、post、put、patch与delete之间的区别](https://www.cnblogs.com/sunyang-001/p/11014629.html)

1. \- get:从服务器端获取数据，请求**body在地址栏**上

2. \- post:向服务器端提交数据，**请求数据在报文body里**

   发送一个修改数据的请求，需求数据要从新创建

3. \- put:向服务器端提交数据，**请求数据在报文body里**

   发送一个修改数据的请求，需求数据更新（全部更新）

4. \- patch:向服务器端提交数据，**请求数据在报文body里**

    发送一个修改数据的请求，需求数据更新（部分更新）

5. \- delete:向服务器端提交数据，**请求数据在报文body里**

   发送一个删除数据的请求

## 2.4 编写用户模块，实现初步验证

### 2.4.1 新增用户

分别实现CheckUser和CreateUser

```go
//验证用户是否存在
func CheckUser(username string) int {
	var user User
	//查询，把满足条件的第一个结果赋值给user
	db.Select("id").Where("user_name = ?", username).First(&user)
	if user.ID > 0{
		return errormsg.ERROR_USERNAME_EXIST
	}
	return errormsg.SUCCESS
}

//创建用户
func CreateUser(data *User) int {
	err := db.Create(&data).Error
	if err != nil {
		return errormsg.ERROR // 500
	}
	return errormsg.SUCCESS
}
```

写好了模型的方法之后就可以写User对应的添加用户的控制器方法了

编辑v1/User.go

```go
var code int
// 新增一个用户
func AddUser(c *gin.Context)  {
	var data model.User            			//创建实例
	_ = c.ShouldBindJSON(&data) 			//绑定JSON赋值给User，就是接收前端的数据并转化为模型
	code = model.CheckUser(data.UserName)	//先检查当前用户是否存在
	if code == errormsg.SUCCESS{
		//成功就写入数据库
		model.CreateUser(&data)
	}
	c.JSON(http.StatusOK, gin.H{		//这里的http.StatusOK是网络的服务的状态码
		"status": code,
		"data": data,
		"message": errormsg.GetErrorDist(code),
	})
}
```

启动服务器，使用apipost测试

报错：<font color='red'>invalid character '-' in numeric literal</font>

错误原因：请求类型出错，很可能使用的是Form data，要使用Json

检查请求头并修改：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226125838.png)

注意的问题：

* **发起请求的tag（例如user_name）要与User结构体定义的json一样，数据库迁移生成的表字段名也是依据这个来的，所以在使用查询等语句的条件的时候需要和此字段的tag保持一致（真不行就直接去看数据库表字段）**

```go
//正确的响应字段
{
	"data": {
		"ID": 7,
		"CreatedAt": "2020-12-26T12:54:57.4914334+08:00",
		"UpdatedAt": "2020-12-26T12:54:57.4914334+08:00",
		"DeletedAt": null,
		"user_name": "xwj",
		"pass_word": "123",
		"role": 0
	},
	"message": "Ok",
	"status": 200
}

//重复用户名响应字段
{
	"data": {
		"ID": 0,
		"CreatedAt": "0001-01-01T00:00:00Z",
		"UpdatedAt": "0001-01-01T00:00:00Z",
		"DeletedAt": null,
		"user_name": "xwj",
		"pass_word": "123",
		"role": 0
	},
	"message": "用户已存在",
	"status": 1001
}
```

### 2.4.2 查找用户

编写User.go模型方法

```go
//查询所有用户
func GetUsers(pageSize int, pageNum int) []User {
	var users []User
	//Limit限制每页显示的个数,offset偏移量，Find查找
	err := db.Limit(pageSize).Offset((pageNum - 1) * pageSize).Find(&users).Error
	if err != nil && err != gorm.ErrRecordNotFound {
		//ErrRecordNotFound记录未找到不算是错误
		return nil
	}
	return users
}
```

编写V1/User.go的路由

```go
// 查询用户列表
func GetUsers(c *gin.Context)  {
	pageSize, _ := strconv.Atoi(c.Query("pagesize"))	//获取请求的单页大小
	pageNum, _ := strconv.Atoi(c.Query("pagenum"))		//获取请求的页数
	if pageSize == 0 || pageNum == 0{
		pageSize = -1  	//对于gorm的limit有一个机制就是当参数为-1时取消分页
		pageNum = 1		//对于offset同理。要满足Offset((pageNum - 1) * pageSize)中的值为-1
	}
	//查询
	users := model.GetUsers(pageSize, pageNum)
	//返回
	c.JSON(http.StatusOK, gin.H{
		"status": errormsg.SUCCESS,
		"data": users,
		"message": errormsg.GetErrorDist(errormsg.SUCCESS),
	})
}
```

测试：

`localhost:3000/api/v1/users`、 `localhost:3000/api/v1/users?pagesize=&pagenum=`   不传参为查询全部

`localhost:3000/api/v1/users?pagesize=4&pagenum=1` 传参代表分页

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226134536.png)

### 2.4.3 加密存储

这里不使用简单的Hash加密验证

而是采用新型的加盐加密算法以此来防止彩虹表的攻击

有两种加密的方案：

* [bcrypt](https://pkg.go.dev/golang.org/x/crypto/bcrypt)
* [scrypt](https://pkg.go.dev/golang.org/x/crypto/scrypt)

这里使用第二种scrypt

在Utils下创建crypt.go文件

编写如下加密函数

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

使用方法一： 修改model下的User

```go
//创建用户
func CreateUser(data *User) int {
	data.PassWord = utils.InfoEncrypto(data.PassWord, data.CreatedAt.String())		//对密码加密
	err := db.Create(&data).Error
	if err != nil {
		return errormsg.ERROR // 500
	}
	return errormsg.SUCCESS
}
```

使用方法二（不建议）：

修改mdoel/User.go模型文件

**这里可以使用钩子函数，但是<font color='red'>十分不建议使用</font>，因为第一有时候出现一些莫名其妙的问题，其次对阅读代码不友好！**

添加一个存储密码前的钩子函数

这个**函数不需要调用**，**实现了之后gorm框架会自动调用**

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

新增用户测试一下

报错：<font color='red'>Error 1406: Data too long for column 'pass_word' at row 1</font>

原因：加密后的密文太长，之前设置的password是varchar(20)太短了，改成255就可以了

成功响应的结果：

```json
{
	"message": "Ok",
	"status": 200
}
```

### 2.4.4 删除用户

User.go模型

```go
//删除用户
func DelUser(uid int) int {
	var user User
	err := db.Delete(&user, uid).Error
	if err != nil {
		return errormsg.ERROR
	}
	return errormsg.SUCCESS
}
```

User.go路由调用

```go
// 删除一个用户
func DelUser(c *gin.Context)  {
	uid, _ := strconv.Atoi(c.Param("id"))
	code := model.DelUser(uid)
	c.JSON(http.StatusOK, gin.H{
		"status": code,
		"data": nil,
		"message": errormsg.GetErrorDist(code),
	})
}
```

测试：

注意使用DELETE

![](http://xwjpics.gumptlu.work/qinniu_uPic/20201226194559.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226194632.png)

### 2.4.5 更新用户信息

```go
//修改用户信息（不包括密码）
func AlterUser(userID int, user *User) int {
	userInfoMap := make(map[string]interface{}, 10)
	// 仅更新部分字段
	userInfoMap["user_name"] = user.UserName
	userInfoMap["role"] = user.Role
	err := db.Model(user).Where("id = ?", userID).Updates(&userInfoMap).Error
	if err != nil {
		return errormsg.ERROR
	}
	return errormsg.SUCCESS
}
```

```go
// 编辑一个用户信息（不包括密码）
func AlterUser(c *gin.Context)  {
	var user model.User
	err := c.ShouldBindJSON(&user)
	uid, _ := strconv.Atoi(c.Param("id"))
	if err != nil {
		fmt.Println("ShouldBindJSON err = ", err)
	}
	//先检查修改的用户名目前是否是重名的
	code = model.CheckUser(user.UserName)
	if code == errormsg.SUCCESS{
		//执行修改
		code = model.AlterUser(uid, &user)  //注意这里的id是请求连接里面的参数，而不是在data包里面对象的id
	}
	c.JSON(http.StatusOK, gin.H{
		"status": code,
		"message": errormsg.GetErrorDist(code),
	})
}
```

测试;

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226202255.png)

重复名字：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201226202531.png)



## 2.5 编写分类、文章模块

分类模块就一个字段Name，一样的CRUD，与用户同理不在详述，文章模块也是，但是文章模块的分类查询涉及到关联查询。下面就是编写的代码

要使用gorm的预加载功能：

---

### **2.5.1 预加载外键**

GORM 允许在 `Preload` 的其它 SQL 中直接加载关系，例如：

```go
type User struct {
  gorm.Model
  Username string
  Orders   []Order
}

type Order struct {
  gorm.Model
  UserID uint
  Price  float64
}

// 查找 user 时预加载相关 Order
db.Preload("Orders").Find(&users)
// SELECT * FROM users;
// SELECT * FROM orders WHERE user_id IN (1,2,3,4);

db.Preload("Orders").Preload("Profile").Preload("Role").Find(&users)
// SELECT * FROM users;
// SELECT * FROM orders WHERE user_id IN (1,2,3,4); // has many
// SELECT * FROM profiles WHERE user_id IN (1,2,3,4); // has one
// SELECT * FROM roles WHERE id IN (4,5,6); // belongs to
```

这里就将查询所有文章修改为：

```go
type Article struct {
	Category Category `gorm:"foreignKey:Cid"`//类别
	gorm.Model
	Title string	`gorm:"type:varchar(20); not null" json:"title"`		//文章标题
	Desc string		`gorm:"type:varchar(200)" json:"desc"`		//文章描述
	Cid int			`gorm:"type:int; not null" json:"cid"`				//类别ID
	Content string	`gorm:"type:longtext" json:"content"`	//文章内容
	Img string 		`gorm:"type:varchar(100)" json:"img"`		//文章图片
	Author int		`gorm:"type:int" json:"author"`				//文章作者
}

//查询所有文章
func GetArticles(pageSize int, pageNum int) []Article {
	var as []Article
	//Limit限制每页显示的个数,offset偏移量，Find查找
	//Preload
	err := db.Preload("Category").Limit(pageSize).Offset((pageNum - 1) * pageSize).Find(&as).Error
	if err != nil && err != gorm.ErrRecordNotFound {
		//ErrRecordNotFound记录未找到不算是错误
		return nil
	}
	return as
}
```

注意：`gorm:"foreignKey:Cid"`修改了

查询出来的结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201227155237.png)

Category被预加载，所以根据Article的Cid外键就自动加载了Category的数据

### 2.5.2 查询某分类下的所有文章

```go
//查询某个分类下的所有文章
func GetArticlesInCategory(pageSize int, pageNum int, cid int) ([]Article, int) {
	var alist []Article
	err := db.Preload("Category").Limit(pageSize).Offset((pageNum-1)*pageSize).Where("Cid = ?", cid).Find(&alist).Error
	if err != nil {
		fmt.Println("GetArticlesInCategory err = ", err)
		return nil, errormsg.ERROR_CATEGORY_NOT_EXIST
	}
	return alist, errormsg.SUCCESS
}
```

```go
func GetArticlesInCategory(c *gin.Context)  {
	//获取连接中的cid
	cid, _ := strconv.Atoi(c.Query("id"))
	pageSize, _ := strconv.Atoi(c.Query("pagesize"))	//获取请求的单页大小
	pageNum, _ := strconv.Atoi(c.Query("pagenum"))		//获取请求的页数
	alist, code := model.GetArticlesInCategory(pageSize, pageNum, cid)
	c.JSON(http.StatusOK, gin.H{
		"status": code,
		"data": alist,
		"message": errormsg.GetErrorDist(code),
	})
}
```

## 2.6 登录

### 2.6.1 JWT

JWT就是做网站Token的中间件

下载`go get -u github.com/dgrijalva/jwt-go`

因为接口写好了，如果不设置Token，那么网站就会受到攻击，实现jwt的中间件来实现Token

```go
package middleware

import (
	"fmt"
	"ginblog/utils"
	"ginblog/utils/errormsg"
	"github.com/dgrijalva/jwt-go"
	"github.com/gin-gonic/gin"
	"net/http"
	"strings"
	"time"
)

var tokenSignKey = []byte(utils.TokenSignKey)

type Myclaim struct {
	Username string	 	`json:"username"`
	jwt.StandardClaims			//其中包含了jwt中间件的一些标准字段
}

const t = 10	//默认的有效时间 （单位小时）

//生成Token
func SetToken(username string) (string, int) {
	vaildTime := time.Now().Add(t * time.Hour)		//计算有效时间
	newClaim := &Myclaim{
		Username:       username,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: vaildTime.Unix(),	//这里要传时间戳
			Issuer:    "xwj",					//签名人
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

//验证Token
func CheckToken(tokenString string) (*Myclaim, int) {
	token, err := jwt.ParseWithClaims(tokenString, &Myclaim{}, func(token *jwt.Token) (interface{}, error) {
		return tokenSignKey, nil
	})
	if err != nil {
		return nil, errormsg.ERROR
	}
	myclaim, code := token.Claims.(*Myclaim)		//把token的Claim断言为Myclain类型
	if token.Valid && code {						//如果验证有效就返回登录数据
		return myclaim, errormsg.SUCCESS
	}else {
		return nil, errormsg.ERROR
	}

}


//实现jwt中间件,当用户调用有权限的接口时，对于用户使用的Token的逻辑判断函数
func JwtToken() gin.HandlerFunc {
	return func(c *gin.Context) {
		code := errormsg.SUCCESS
		tokenHeader := c.Request.Header.Get("Authorization")
		//验证Token头
		if tokenHeader == ""{
			//Token为空
			code = errormsg.ERROR_TOKEN_EMPTY
			c.JSON(http.StatusOK, gin.H{
				"status": code,
				"message": errormsg.GetErrorDist(code),
			})
			c.Abort()
			return
		}
		checkToken := strings.SplitN(tokenHeader, " ", 2)
		if len(checkToken) != 2 && checkToken[0] != "Bearer"{
			code = errormsg.ERROR_TOKEN_FORMAT_WRONG
			c.JSON(http.StatusOK, gin.H{
				"status": code,
				"message": errormsg.GetErrorDist(code),
			})
			c.Abort()
		}
		key, resCode := CheckToken(checkToken[1])			//格式无误后，检查Token
		if resCode == errormsg.ERROR {						//如果检测没有通过，那么就返回错误
			code = errormsg.ERROR_TOKEN_WRONG
			c.JSON(http.StatusOK, gin.H{
				"status": code,
				"message": errormsg.GetErrorDist(code),
			})
			c.Abort()
		}
		c.Set("username", key.Username)
		c.Next()
	}
}
```

### 2.6.2 登录验证

实现登录模块的密码验证，在model/User.go下

```go
//登录验证
func CheckLogin(userName ,passWord string) int {
	var user User
	// 查询用户的数据库中的密码
	db.Where("username = ?", userName).First(&user)
	// 未查询到当前用户
	if user.ID == 0{
		return errormsg.ERROR_USER_NOT_EXIST
	}
	// 判断密码是否相同
	if utils.InfoEncrypto(passWord, user.CreatedAt.String()) != user.PassWord {
		return errormsg.ERROR_PASSWORD_WRONG
	}
	// 判断用户的权限
	if user.Role != 0 {
		return errormsg.ERROR_USER_RIGHT_ERR
	}
	// 成功返回
	return errormsg.SUCCESS
}
```

在Api的V1中实现Login.go

```go
// 登录函数
func Login(c *gin.Context)  {
	// 获取用户客户端的输入
	var userInfo model.User
	var token string
	c.ShouldBindJSON(&userInfo)
	// 检查密码
	code := model.CheckLogin(userInfo.UserName, userInfo.PassWord)
	// 成功则生成Token返回
	if code == errormsg.SUCCESS {
		token, code = middleware.SetToken(userInfo.UserName)
		c.JSON(http.StatusOK, gin.H{
			"status": code,
			"data": token,
		})
	}else {
		// 否则就返回错误
		c.JSON(http.StatusOK, gin.H{
			"status": code,
			"data": "",
			"message": errormsg.GetErrorDist(code),
		})
	}
}

```

### 2.6.3 分权限管理接口

需要token的分为一类，不需要的分为另一类

```go
func InitRoute()  {
	//初始化gin
	gin.SetMode(utils.AppMode)	//设置服务器模式
	r := gin.Default()			//使用默认模式创建服务

	auth := r.Group("api/v1")
	auth.Use(middleware.JwtToken())
	//以下接口都需要授权token才能使用
	{
		//用户模块相关接口

		auth.PUT("user/:id", v1.AlterUser)
		auth.DELETE("user/:id", v1.DelUser)

		//文章模块相关接口
		auth.POST("article/add", v1.AddArticle)
		auth.PUT("article/:id", v1.AlterArticle)
		auth.DELETE("article/:id", v1.DelArticle)
		//类别模块相关接口
		auth.POST("category/add", v1.AddCategory)
		auth.PUT("category/:id", v1.AlterCategory)
		auth.DELETE("category/:id", v1.DelCategory)
	}
	public := r.Group("api/v1")
	//以下都是不需要Token的公共接口
	{
		// 用户相关
		public.POST("user/add", v1.AddUser)							//新增用户
		public.GET("users", v1.GetUsers)
		public.GET("user", v1.GetUser)
		public.POST("login", v1.Login)								//注意，用户登录不需要权限
		// 文章相关
		public.GET("articles", v1.GetArticles)						//获取所有的文章
		public.GET("article", v1.GetArticle)
		public.GET("article/byCategory", v1.GetArticlesInCategory)	//获取单分类下的所有文章
		// 类别相关
		public.GET("categories", v1.GetCategories)
		public.GET("category", v1.GetCategory)
	}
	err := r.Run(utils.HttpPort)
	if err != nil {
		log.Panic(err)
	}
}
```

## 2.7 文件上传

目前因为网站带宽等原因，一般服务器不会把网站的静态资源访问与服务器本站放在一起，这样会影响加载的速度。一般都会上传文件到第三方服务中。

使用第三方的七牛云作为云端存储

注册七牛云，并完成以下配置：

* 创建存储空间
* 指定域名，没有自己的域名就使用其提供的测试域名（30天免费）
* 使用自己域名的记得配置解析与Channel

进入个人页面获取密钥AK与SK：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210118095516.png)

进入七牛云的开发者中心，看提供的api文档

添加`github.com/qiniu/api.v7/v7 v7.8.1`到go mod 文件中，然后再命令行中输入`go mod download`下载，如果不行的话就使用IDE手动下载

services/Upload.go

```go
package services

import (
	"context"
	"fmt"
	"ginblog/utils"
	"ginblog/utils/errormsg"
	"github.com/qiniu/api.v7/v7/auth/qbox"
	"github.com/qiniu/api.v7/v7/storage"
	"mime/multipart"
)

var AccessKey = utils.Ak
var SecretKey = utils.Sk
var Bucket = utils.Bucket
var ImgUrl = utils.QiniuServer

//文件上传(数据流上传方式)
func UploadFile(file multipart.File, fileSize int64) (string, int) {
	putPolicy := storage.PutPolicy{
		Scope: Bucket,
	}
	mac := qbox.NewMac(AccessKey, SecretKey)
	//生成上传的Token
	upToken := putPolicy.UploadToken(mac)

	cfg := storage.Config{
		// 配置存储空间
		Zone: &storage.ZoneHuanan,	// 空间对应的机房
		UseCdnDomains: false,		// 上传是否使用CDN上传加速
		UseHTTPS: false,			// 是否使用https域名
	}
	// 可选配置 这里不需要额外的配置
	putExtra := storage.PutExtra{}
	// 构建表单上传的对象
	formUploader := storage.NewFormUploader(&cfg)
	ret := storage.PutRet{}			//结果返回对象

	//开始上传
	err := formUploader.PutWithoutKey(context.Background(), &ret, upToken, file, fileSize, &putExtra)
	if err != nil {
		fmt.Println("formUploader.PutWithoutKey err : ", err)
		return "", errormsg.ERROR_UPLOADFILE_ERROR
	}
	// 拼接图片外链
	url := ImgUrl +  "/" + ret.Key
	//返回图片外部链接
	return url, errormsg.SUCCESS
}
```

v1/Upload.go

```go
func Upload(c *gin.Context)  {
	file, fileHeader, err := c.Request.FormFile("file")
	if err != nil {
		fmt.Println(err)
	}
	//fmt.Println("file:", file, "fileHeader:", fileHeader)
	fileSize := fileHeader.Size
	// 调用上传函数
	url, code := services.UploadFile(file, fileSize)
	// 返回给客户端结果
	c.JSON(http.StatusOK, gin.H{
		"status": code,
		"data": url,
		"message": errormsg.GetErrorDist(code),
	})
}
```

## 2.8 日志处理

使用日志包中间件https://pkg.go.dev/github.com/sirupsen/logrus

下载： `go get -u github.com/sirupsen/logrus`

### 2.8.1 实现日志中间件

修改router

```go
func InitRoute() {
	//初始化gin
	gin.SetMode(utils.AppMode) //设置服务器模式
	r := gin.New()             
	r.Use(middleware.Logger())
	r.Use(gin.Recovery())

	auth := r.Group("api/v1")
	auth.Use(middleware.JwtToken())
    .....
}
```

书写自定义中间件middleware/logger

```go
// log中间件

func Logger() gin.HandlerFunc {
	//实例化logger
	logger := logrus.New()
	return func(c *gin.Context) {
		startTime := time.Now()            //开始时间
		c.Next()                           //c.next就是先执行其他中间件，执行完毕之后就会返回这里执行下面的语句
		spendTime := time.Since(startTime) //从开始时间经过的时间
		//转换单位到毫秒
		//Ceil取绝对值, Nanoseconds 以纳秒返回持续的时间，纳秒到毫秒除以10的六次方
		spendTimeMs := fmt.Sprintf("%d ms", int64(math.Ceil(float64(spendTime.Nanoseconds())/1000000.0)))
		//获取一系列信息
		hostName, err := os.Hostname() //主机名
		if err != nil {
			hostName = "unKnow"
		}
		statusCode := c.Writer.Status()    //请求代码
		clientIp := c.ClientIP()           //客户端IP
		userAgent := c.Request.UserAgent() //用户访问代理工具
		dataSize := c.Writer.Size()        //数据大小
		if dataSize < 0 {
			dataSize = 0
		}
		method := c.Request.Method //用户调用的方法
		path := c.Request.URL      //用户请求的URl
		entry := logger.WithFields(logrus.Fields{
			"HostName":  hostName,
			"Status":    statusCode,
			"SpendTime": spendTimeMs,
			"IP":        clientIp,
			"cliAgent":  userAgent,
			"DataSize":  dataSize,
			"Method":    method,
			"Path":      path,
		})
		//返回内部错误
		if len(c.Errors) > 0 {
			entry.Error(c.Errors.ByType(gin.ErrorTypePrivate).String())
		}
		if statusCode >= 500 {
			entry.Error() //大于500是错误
		} else if statusCode >= 400 {
			entry.Warn() //大于400是警告
		} else {
			entry.Info()
		}
	}
}
```

测试结果

`time="2021-01-19T17:10:13+08:00" level=info DataSize=43 HostName=5P6E60SSTPVAIK2 IP="::1" Method=POST Path=/api/v1/user/add SpendTime="97 ms" Status=200 cliAgent="ApiPOST Runtime +https://www.apipost.cn"`

### 2.8.2 实现日志写入文件

修改Logger文件

```go
func Logger() gin.HandlerFunc {
	logFilePath := "log/log.log"
	file, err := os.OpenFile(logFilePath, os.O_RDWR | os.O_CREATE, 0755)
	if err != nil {
		fmt.Println("OpenLogFile err : ", err)
	}
	//实例化logger
	logger := logrus.New()
	logger.Out = file //输出到文件
    ....
}
```

### 2.8.3 时间分割与定时清除

使用两个go的包： 

* rotatelogs  	`go get -u github.com/lestrrat-go/file-rotatelogs`

* lfshook    `go get -u github.com/rifflock/lfshook`

并且设置了**最新日志的软链接**：

```go
func Logger() gin.HandlerFunc {
    	linkLogName := "LatestLog.log" //最新的日志名
        //实例化logger
        logger := logrus.New()
        //按时间分割以及定时清除
        logger.SetLevel(logrus.DebugLevel)
        //设置文件保存格式
        logWriter, _ := rotatelogs.New(
            //logFilePath+"_%Y_%m_%d.log",
            "log/log_%Y_%m_%d.log",                    //文件格式名
            rotatelogs.WithMaxAge(7*24*time.Hour),     //最大保存时间
            rotatelogs.WithRotationTime(24*time.Hour), //文件分割时间
            rotatelogs.WithLinkName(linkLogName),      //软链接最新的log文件
        )

        writeMap := lfshook.WriterMap{
            logrus.InfoLevel:  logWriter,
            logrus.DebugLevel: logWriter,
            logrus.FatalLevel: logWriter,
            logrus.WarnLevel:  logWriter,
            logrus.ErrorLevel: logWriter,
            logrus.PanicLevel: logWriter,
        }
        Hook := lfshook.NewHook(writeMap, &logrus.TextFormatter{
            TimestampFormat: "2006-01-02 15:04:05", //格式化时间
        })
        logger.AddHook(Hook)
    ....
}
```

## 2.9 跨域配置

### 2.9.1 修改新增用户验证

修改model/User

主要使用gin自带的Validator验证器

```go
type User struct {
	//	ID        uint `gorm:"primarykey"`
	//	CreatedAt time.Time
	//	UpdatedAt time.Time
	//	DeletedAt DeletedAt `gorm:"index"`
	gorm.Model //引入gorm自带的字段 见上方
	// validate:"required,min=4,max=12,gte=2" require代表不能为空，min与max表示最大最小,gte表示大于等于, label字段解释
	UserName string `gorm:"type:varchar(20); not null" json:"user_name" validate:"required,min=4,max=12" label:"用户名"`
	PassWord string `gorm:"type:varchar(255); not null" json:"pass_word" validate:"required,min=6,max=20" label:"密码"`
	//role 1 为管理员  2 为一般用户
	Role int `gorm:"type:int;DEFAULT:2" json:"role" validate:"required,gte=2" label:"角色码"` //角色 权限管理
}
```

在Utile下新建validate

```go
package validator

import (
	"fmt"
	"ginblog/utils/errormsg"
	"github.com/go-playground/locales/zh_Hans_CN"
	unTrans "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	zhTrans "github.com/go-playground/validator/v10/translations/zh"
	"reflect"
)

func Validate(data interface{}) (string, int) {
	//创建验证实例
	validate := validator.New()
	//转换为中文
	uni := unTrans.New(zh_Hans_CN.New())
	trans, _ := uni.GetTranslator("zh_Hans_CN") //获取中文翻译方法
	err := zhTrans.RegisterDefaultTranslations(validate, trans)
	if err != nil {
		fmt.Println("Validate RegisterDefaultTranslations err : ", err)
	}
	//替换字段原值为label值
	validate.RegisterTagNameFunc(func(field reflect.StructField) string {
		label := field.Tag.Get("label")
		return label
	})
	//验证
	err = validate.Struct(data) //断言传入的空接口是一个结构体
	if err != nil {
		fmt.Println("", err)
		for _, v := range err.(validator.ValidationErrors) {
			return v.Translate(trans), errormsg.ERROR
		}
	}
	return "", errormsg.SUCCESS
}

```

新增用户方法v1/User

```go
// 新增一个用户
func AddUser(c *gin.Context) {
	var data model.User            //创建实例
	err := c.ShouldBindJSON(&data) //绑定JSON赋值给User，就是接收前端的数据并转化为模型
	if err != nil {
		fmt.Println("ShouldBindJSON err = ", err)
	}
	//对新增用户做验证
	msg, code := validator.Validate(&data)
	if code != errormsg.SUCCESS {
		c.JSON(http.StatusOK, gin.H{
			"status":  code,
			"message": msg,
		})
		return
	}
	code = model.CheckUser(data.UserName) //先检查当前用户是否存在
	if code == errormsg.SUCCESS {
		//成功就写入数据库
		model.CreateUser(&data)
	}
	c.JSON(http.StatusOK, gin.H{ //这里的http.StatusOK是网络的服务的状态码
		"status":  code,
		"message": errormsg.GetErrorDist(code),
	})
}
```

### 2.9.2 跨域参数配置

使用GIn框架自带的包cors   

下载包： `go get -u github.com/gin-contrib/cors`

详细的跨域知识介绍

基本概念： https://www.jianshu.com/p/f880878c1398  

cors跨域： http://www.ruanyifeng.com/blog/2016/04/cors.html

创建中间件文件Cors：

```go
func Cors() gin.HandlerFunc {
	return func(c *gin.Context) {
		cors.New(cors.Config{
			AllowAllOrigins: true,                                        //允许全部跨域
			AllowMethods:    []string{"*"},                               //允许的方法
			AllowHeaders:    []string{"Origin"},                          //
			ExposeHeaders:   []string{"Content-Length", "Authorization"}, //Authorization
			//AllowCredentials: true,                                        //是否允许cookie请求，项目未使用cookie所以不用
			//AllowOriginFunc: func(origin string) bool {
			//	return origin == "https://github.com"
			//},
			MaxAge: 12 * time.Hour, //域请求的保持时间
		})
	}
}
```

跨域出现错误:

![QCjs5k](http://xwjpics.gumptlu.work/qinniu_uPic/QCjs5k.png)

1. 检查中间件Cors是否添加

2. 可能是老版本gin两次请求,第一次options,第二次post/get导致的问题,解决方案:

   https://blog.csdn.net/u010918487/article/details/82686293

   gin使用cors中间件后没这个问题

### 2.9.3 分页的总数

在查询所有的用户、文章、类别时需要使用查找总数量来实现分页展示，所以添加一个字段total以及在查询数据库字段的时候使用Count统计总数，在接口中向客户端返回。

以下是文章的修改，其他同理

```go
//查询所有文章
func GetArticles(pageSize int, pageNum int) ([]Article, int, int) {
	var as []Article
	var total int64
	//Limit限制每页显示的个数,offset偏移量，Find查找
	//Preload
	err := db.Preload("Category").Limit(pageSize).Offset((pageNum - 1) * pageSize).Find(&as).Count(&total).Error
	if err != nil && err != gorm.ErrRecordNotFound {
		//ErrRecordNotFound记录未找到不算是错误
		return nil, errormsg.ERROR_ARTICLE_NOT_EXIST, 0
	}
	return as, errormsg.SUCCESS, int(total)
}
```

# 三、Vue前端

目前看不懂，学了VUE之后再来。。。。

