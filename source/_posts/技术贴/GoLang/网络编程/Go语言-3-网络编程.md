---
title: Go语言-3-网络编程
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2020-12-19 22:17:17
---

> 学习原文自: https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/03.0.md
>
> build-web-application-with-golang
>
> GitHub 地址→https://github.com/astaxie/build-web-application-with-golang
>
> 自己学习之用,并且在其上做了笔记与标注,如有侵权,联系删除

# 一、golang网络编程

## 1.1 Web基础

### 1.理论基础知识

#### Web工作方式

对于普通的上网过程，系统其实是这样做的：

浏览器本身是一个客户端，当你输入URL的时候，首先浏览器会去请求DNS服务器，通过DNS获取相应的域名对应的IP，然后通过IP地址找到IP对应的服务器后，要求建立TCP连接，等浏览器发送完HTTP Request（请求）包后，服务器接收到请求包之后才开始处理请求包，服务器调用自身服务，返回HTTP Response（响应）包；客户端收到来自服务器的响应后开始渲染这个Response包里的主体（body），等收到全部的内容随后断开与该服务器之间的TCP连接。

![U2FfnS](http://xwjpics.gumptlu.work/qinniu_uPic/U2FfnS.jpg)

Web服务器的工作原理可以简单地归纳为：

- 客户机通过TCP/IP协议建立到服务器的TCP连接
- 客户端向服务器发送HTTP协议请求包，请求服务器里的资源文档
- 服务器向客户机发送HTTP协议应答包，如果请求的资源包含有动态语言的内容，那么服务器会调用动态语言的解释引擎负责处理“动态内容”，并将处理得到的数据返回给客户端
- 客户机与服务器断开。由客户端解释HTML文档，在客户端屏幕上渲染图形结果

需要注意的是客户机与服务器之间的通信是==**非持久连接**==的，也就是当服务器发送了应答后就与客户机断开连接，等待下一次请求。

#### URL和DNS解析

我们浏览网页都是通过URL访问的，那么URL到底是怎么样的呢？

URL(Uniform Resource Locator)是“统一资源定位符”的英文缩写，用于描述一个网络上的资源, 基本格式如下

```txt
scheme://host[:port#]/path/.../[?query-string][#anchor]
scheme         指定低层使用的协议(例如：http, https, ftp)
host           HTTP服务器的IP地址或者域名
port#          HTTP服务器的默认端口是80，这种情况下端口号可以省略。如果使用了别的端口，必须指明.
path           访问资源的路径
query-string   发送给http服务器的数据
anchor         锚
```

DNS(Domain Name System)是“域名系统”的英文缩写，是一种组织成域层次结构的计算机和网络服务命名系统，它用于TCP/IP网络，它从事将主机名或域名转换为实际IP地址的工作。DNS就是这样的一位“翻译官”，它的基本工作原理可用下图来表示。

更详细的DNS解析的过程如下，这个过程有助于我们理解DNS的工作模式

1. 在浏览器中输入www.qq.com域名，操作系统会先检查自己**==本地的hosts文件==**是否有这个网址映射关系，如果有，就先调用这个IP地址映射，完成域名解析。
2. 如果hosts里没有这个域名的映射，则查找==**本地DNS解析器缓存**==，是否有这个网址映射关系，如果有，直接返回，完成域名解析。
3. 如果hosts与本地DNS解析器缓存都没有相应的网址映射关系，首先会找**TCP/IP参数中设置的首选DNS服务器**，在此我们叫它==**本地DNS服务器**==，此服务器收到查询时，如果要查询的域名，包含在本地配置区域资源中，则返回解析结果给客户机，完成域名解析，此解析**具有权威性**。
4. 如果要查询的域名，不由本地DNS服务器区域解析，但该服务器**已==缓存==了此网址映射关系，则调用这个IP地址映射**，完成域名解析，此解析**不具有权威性**。
5. 如果本地DNS服务器本地区域文件与缓存解析都失效，则根据本地DNS服务器的设置（是否设置转发器）进行查询，如果**未用转发模式**，本地DNS就把请求发至 “**根DNS服务器”**，“根DNS服务器”收到请求后会判断这个域名(.com)是谁来授权管理，并会返回一个负责该**顶级域名服务器**的一个IP。本地DNS服务器收到IP信息后，将会联系负责.com域的这台服务器。这台负责.com域的服务器收到请求后，如果自己无法解析，它就会找一个管理.com域的下一级DNS服务器地址(qq.com)给本地DNS服务器。当本地DNS服务器收到这个地址后，就会找qq.com域服务器，重复上面的动作，进行查询，直至找到www.qq.com主机。
6. 如果用的是**转发模式**，此DNS服务器就会把请求转发至**上一级DNS服务器**，由上一级服务器进行解析，上一级服务器如果不能解析，或找根DNS或把转请求转至上上级，以此循环。不管是本地DNS服务器用是是转发，还是根提示，最后都是把结果返回给本地DNS服务器，由此DNS服务器再返回给客户机。

![FmEzx9](http://xwjpics.gumptlu.work/qinniu_uPic/FmEzx9.jpg)

#### HTTP协议

HTTP是一种让Web服务器与浏览器(客户端)通过Internet发送与接收数据的协议,它**建立在TCP协议之上**，一般采用TCP的**80端口**。它是一个**==请求、响应协议==**--客户端发出一个请求，服务器响应这个请求。在HTTP中，客户端总是通过建立一个连接与发送一个HTTP请求来发起一个事务。**服务器不能主动去与客户端联系**，也不能给客户端发出一个回调连接。**客户端与服务器端都可以提前中断一个连接**。例如，当浏览器下载一个文件时，你可以通过点击“停止”键来中断文件的下载，关闭与服务器的HTTP连接。

HTTP协议是==**无状态**==的，同一个客户端的这次请求和上次请求是没有对应关系，对HTTP服务器来说，它并不知道这两个请求是否来自同一个客户端。为了解决这个问题， Web程序引入了**Cookie机制**来维护连接的**可持续状态。**

> HTTP协议是建立在TCP协议之上的，因此TCP攻击一样会影响HTTP的通讯，例如比较常见的一些攻击：SYN Flood是当前最流行的DoS（拒绝服务攻击）与DdoS（分布式拒绝服务攻击）的方式之一，这是一种利用TCP协议缺陷，**发送大量伪造的TCP连接请求**，从而使得被攻击方资源耗尽（CPU满负荷或内存不足）的攻击方式。

##### <font color='#e54d42'>HTTP请求包（浏览器信息）</font>

我们先来看看Request包的结构, Request包分为3部分，第一部分叫**Request line（请求行）**, 第二部分叫**Request header（请求头）**,第三部分是**body（主体）**。header和body之间有个空行，请求包的例子所示:

* Request line（请求行）
* Request header (请求体)
* body (主体)

> ```
> GET /domains/example/ HTTP/1.1        				//请求行: 请求方法 请求URI HTTP协议/协议版本
> Host：www.iana.org                						//服务端的主机名
> User-Agent：Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.4 (KHTML, like Gecko) Chrome/22.0.1229.94 Safari/537.4            										//浏览器信息
> Accept：text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8    //客户端能接收的mine
> Accept-Encoding：gzip,deflate,sdch        		//是否支持流压缩
> Accept-Charset：UTF-8,*;q=0.5        				//客户端字符编码集
> //空行,用于分割请求头和消息体
> //消息体,请求资源参数,例如POST传递的参数
> ```

HTTP协议定义了很多与服务器交互的请求方法，最基本的有4种，分别是GET,POST,PUT,DELETE。一个URL地址用于描述一个网络上的资源，而HTTP中的**GET, POST, PUT, DELETE**就对应着对这个资源的**查，改，增，删**4个操作。我们最常见的就是GET和POST了。**GET一般用于获取/查询资源信息，而POST一般用于更新资源信息。**

通过fiddler抓包可以看到如下请求信息:

![ivjZx7](http://xwjpics.gumptlu.work/qinniu_uPic/ivjZx7.png)

![VzWQX6](http://xwjpics.gumptlu.work/qinniu_uPic/VzWQX6.png)

**==GET和POST的区别:==**

1. 我们可以看到GET请求消息体为空，POST请求带有消息体。
2. GET提交的数据会放在URL之后，以`?`分割URL和传输数据，参数之间以`&`相连，如`EditPosts.aspx?name=test1&id=123456`。POST方法是把提交的数据放在HTTP包的body中。
3. GET提交的数据大小有限制（因为浏览器对URL的长度有限制），而POST方法提交的数据没有限制。
4. GET方式提交数据，会带来安全问题，比如一个登录页面，通过GET方式提交数据时，用户名和密码将出现在URL上，如果页面可以被缓存或者其他人可以访问这台机器，就可以从历史记录获得该用户的账号和密码。

##### <font color='#e54d42'>HTTP响应包（服务器信息）</font>

> ```
> HTTP/1.1 200 OK                        //状态行
> Server: nginx/1.0.8                    //服务器使用的WEB软件名及版本
> Date:Date: Tue, 30 Oct 2012 04:14:25 GMT        //发送时间
> Content-Type: text/html                //服务器发送信息的类型
> Transfer-Encoding: chunked            //表示发送HTTP包是分段发的
> Connection: keep-alive                //保持连接状态
> Content-Length: 90                    //主体内容长度
> //空行 用来分割消息头和主体
> <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"... //消息体
> ```

Response包中的第一行叫做状态行，由HTTP协议版本号， 状态码， 状态消息 三部分组成。

**==状态码==**用来告诉HTTP客户端,HTTP服务器是否产生了预期的Response。HTTP/1.1协议中定义了5类状态码， 状态码由三位数字组成，第一个数字定义了响应的类别

- 1XX 提示信息 - 表示请求已被成功接收，继续处理
- 2XX 成功 - 表示请求已被成功接收，理解，接受
- 3XX 重定向 - 要完成请求必须进行更进一步的处理
- 4XX 客户端错误 - 请求有语法错误或请求无法实现
- 5XX 服务器端错误 - 服务器未能实现合法的请求

我们看下面这个图展示了详细的返回信息，左边可以看到有很多的资源返回码，200是常用的，表示正常信息，302表示跳转。response header里面展示了详细的信息。

![bewQN0](http://xwjpics.gumptlu.work/qinniu_uPic/bewQN0.png)



##### HTTP协议无状态和Connection: keep-alive的区别

无状态是指协议对于事务处理没有记忆能力，服务器不知道客户端是什么状态。从另一方面讲，打开一个服务器上的网页和你之前打开这个服务器上的网页之间没有任何联系。

HTTP是一个无状态的面向连接的协议，无状态不代表HTTP不能保持TCP连接，更不能代表HTTP使用的是UDP协议（面对无连接）。

<font color='#39b54a'>从HTTP/1.1起，默认都开启了Keep-Alive保持连接特性，简单地说，当一个网页打开完成后，**客户端和服务器之间用于传输HTTP数据的TCP连接不会关闭**，如果客户端再次访问这个服务器上的网页，**会继续使用这一条已经建立的TCP连接。**</font>

Keep-Alive不会永久保持连接，它有一个保持时间，可以在不同服务器软件（如Apache）中设置这个时间。

![I0BbrB](http://xwjpics.gumptlu.work/qinniu_uPic/I0BbrB.jpg)

上面这张图我们可以了解到整个的通讯过程，同时细心的读者是否注意到了一点，一个URL请求但是左边栏里面为什么会有那么多的资源请求(这些都是静态文件，**go对于静态文件有专门的处理方式**)。

这个就是浏览器的一个功能，<font color='#e54d42'>第一次请求url，服务器端返回的是html页面，然后浏览器**开始渲染HTML**：当解析到HTML DOM里面的**图片连接，css脚本和js脚本的链接**，浏览器就会**自动发起一个请求静态资源的HTTP请求**，获取相对应的静态资源，然后浏览器就会渲染出来，最终将所有资源整合、渲染，完整展现在我们面前的屏幕上。</font>

> 网页优化方面有一项措施是减少HTTP请求次数，就是把**尽量多的css和js资源合并在一起**，目的是尽量减少网页请求静态资源的次数，提高网页加载速度，同时减缓服务器的压力。

### 2.Go搭建一个Web服务器

Web是基于http协议的一个服务，Go语言里面提供了一个完善的**net/http**包，通过http包可以很方便的就搭建起来一个可以运行的Web服务。同时使用这个包能很简单地对Web的路由，静态文件，模版，cookie等数据进行设置和操作。

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"strings"
)

func main() {
	http.HandleFunc("/", SayHello)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}

func SayHello(w http.ResponseWriter, r * http.Request)  {
	r.ParseForm() // 解析参数, 默认不解析参数
	// Form contains the parsed form data, including both the URL
	fmt.Println(r.Form)
	fmt.Println("path", r.URL.Path)
	fmt.Println("scheme", r.URL.Scheme)
	fmt.Println(r.Form["url_long"])
	for k, v := range r.Form {
		fmt.Println("key", k)
		// strings.Join 就是把string合并到一个string
		fmt.Println("val:", strings.Join(v, ""))
	}
	fmt.Fprintf(w, "Hello xwj!")		//这个写入到w的是输出到客户端的
}
```

上面这个代码，我们build之后，然后执行web.exe,这个时候其实已经在9090端口监听http链接请求了。

在浏览器输入`http://localhost:9090`

可以看到浏览器页面输出了`Hello astaxie!`

可以换一个地址试试：`http://localhost:9090/?url_long=111&url_long=222`

看看浏览器输出的是什么，服务器输出的是什么？如下

```go
map[url_long:[111 222]]
path /
scheme 
[111 222]
key url_long
val: 111222
map[]
path /favicon.ico
scheme 
[]
```

> 如果你以前是PHP程序员，那你也许就会问，**我们的nginx、apache服务器不需要吗？Go就是不需要这些，**因为他直接就监听tcp端口了，做了nginx做的事情，然后**sayhelloName这个其实就是我们写的逻辑函数了**，**跟php里面的控制层（controller）函数类似**。
>
> 如果你以前是Python程序员，那么你一定听说过tornado，这个代码和他是不是很像，对，没错，Go就是拥有类似Python这样动态语言的特性，写Web应用很方便。
>
> 如果你以前是Ruby程序员，会发现和ROR的/script/server启动有点类似。

### 3.Go如何使得Web工作

Go的Web服务工作也离不开我们第一小节介绍的Web工作方式。

以下均是服务器端的几个概念:

* Request：用户请求的信息，用来解析用户的请求信息，包括post、get、cookie、url等信息

* Response：服务器需要反馈给客户端的信息

* Conn：用户的每次请求链接

* Handler：处理请求和生成返回信息的处理逻辑

Go实现Web服务的工作模式的流程图(http包执行流程):

![sBDSdL](http://xwjpics.gumptlu.work/qinniu_uPic/sBDSdL.png)

1. 创建Listen Socket, 监听指定的端口, 等待客户端请求到来。
2. Listen Socket接受客户端的请求, 得到Client Socket, 接下来通过Client Socket与客户端通信。
3. 处理客户端的请求, 首先从Client Socket读取HTTP请求的协议头, 如果是POST方法, 还可能要读取客户端提交的数据, 然后交给相应的handler处理请求, handler处理完毕准备好客户端需要的数据, 通过Client Socket写给客户端。

这整个的过程里面我们只要了解清楚下面三个问题，也就知道Go是如何让Web运行起来了

- 如何监听端口？
- 如何接收客户端请求？
- 如何分配handler？

前面小节的代码里面我们可以看到，Go是**通过一个函数`ListenAndServe`来处理这些事情的**，这个底层其实这样处理的：初始化一个server对象，然后调用了`net.Listen("tcp", addr)`，也就是底层用**==TCP协议==**搭建了一个服务，然后监控我们设置的端口。

```go
// 前面的关键代码:
// HandleFunc registers the handler function for the given pattern
http.HandleFunc("/", SayHello)								// 参数一: pattern, 参数二: 函数; 注册pattern的请求路由规则(serveMux中的映射关系)
err := http.ListenAndServe(":9090", nil)			// 参数一:端口, 参数二:handler; 监听端口; 参数二为空则创建一个默认的DefaultServeMux(路由器)				
```

下面代码来自Go的http包的源码，通过下面的代码我们可以看到整个的http处理过程：

```go
func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    var tempDelay time.Duration // how long to sleep on accept failure
  	// 循环监听请求
    for {
        rw, e := l.Accept()
        if e != nil {									// 错误处理
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                log.Printf("http: Accept error: %v; retrying in %v", e, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return e
        }
        tempDelay = 0
        c, err := srv.newConn(rw)				// 创建新链接
        if err != nil {
            continue
        }
        go c.serve()										// 为链接创建一个go程
    }
}
```

监控之后如何接收客户端的请求呢？上面代码(`ListenAndServe`)执行监控端口之后，调用了`srv.Serve(net.Listener)`函数，这个函数就是处理接收客户端的请求信息。这个函数里面起了一个`for{}`，首先通过Listener接收请求，其次==**创建一个Conn**，**最后单独开了一个goroutine**，==把这个请求的数据当做参数扔给这个conn去服务：`go c.serve()`。这个**==就是高并发体现了，用户的每一次请求都是在一个新的goroutine去服务，相互不影响。==**

那么如何具体分配到相应的函数来处理请求呢？conn首先会解析request:`c.readRequest()`,然后**获取相应的handler**:`handler := c.server.Handler`，也就是我们刚才**在调用函数`ListenAndServe`时候的第二个参数**，我们前面例子传递的是nil，也就是为空，那么默认获取**`handler = DefaultServeMux`**,那么这个变量用来做什么的呢？对，**这个变量就是一个<font color='#39b54a'>路由器</font>**，**它用来匹配url跳转到其相应的handle函数**，那么这个我们有设置过吗?有，我们调用的代码里面第一句不是**调用了`http.HandleFunc("/", sayhelloName)`嘛。这个作用就是注册了请求`/`的路由规则**，当请求uri为"/"，路由就会转到函数sayhelloName，**DefaultServeMux会调用`ServeHTTP`方法，这个方法内部其实就是调用`sayhelloName`本身，最后通过写入response的信息反馈到客户端**。

![SLygqL](http://xwjpics.gumptlu.work/qinniu_uPic/SLygqL.jpg)

### 4 Go的http包详解

Go的http有两个核心功能：Conn、ServeMux

#### Conn的goroutine

与我们一般编写的http服务器不同, Go为了实现高并发和高性能, **使用了goroutines来处理Conn的读写事件**, 这样每个请求都能保持独立，相互不会阻塞，可以高效的响应网络事件。这是Go高效的保证。

Go在等待客户端请求里面是这样写的：

```Go
c, err := srv.newConn(rw)
if err != nil {
    continue
}
go c.serve()
```

这里我们可以看到**==客户端的每次请求都会创建一个Conn，这个Conn里面保存了该次请求的信息，然后再传递到对应的handler，该handler中便可以读取到相应的header信息，这样保证了每个请求的独立性。==**

#### ServeMux的自定义

##### <font color='#fbbd08'>**路由器的实现:**</font>

我们前面小节讲述conn.server的时候，其实内部是调用了http包默认的**路由器**(<font color='#39b54a'>**DefaultServeMux**</font>)，通过路由器把本次请求的信息传递到了后端的处理函数。那么这个路由器是怎么实现的呢？

它的结构如下：

```Go
type ServeMux struct {
    mu sync.RWMutex   			//锁，由于请求涉及到并发处理，因此这里需要一个锁机制
    m  map[string]muxEntry  // 路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
    hosts bool 							// 是否在任意的规则中带有host信息
}
```

下面看一下muxEntry

```Go
type muxEntry struct {
    explicit bool   		// 是否精确匹配
    h        Handler 		// 这个路由表达式对应哪个handler
    pattern  string  		// 匹配字符串
}
```

接着看一下Handler接口的定义

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)  // 路由实现器
}
```

Handler是一个接口，但是前一小节中的`sayhelloName`函数并没有实现ServeHTTP这个接口，为什么能添加呢？原来在http包里面还定义了一个类型`HandlerFunc`,我们定义的函数`sayhelloName`就是这个HandlerFunc调用之后的结果，这个类型默认就实现了ServeHTTP这个接口，即**我们调用了HandlerFunc(f),强制类型转换f成为HandlerFunc类型，这样f就拥有了ServeHTTP方法。**

<font color='#39b54a'>**HandlerFunc的作用就是让函数F实现Handler接口中的ServeHTTP接口方法(把F变成了HandlerFunc类型)**</font>

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

##### <font color='#fbbd08'>**路由器分发请求:**</font>

**路由器**里面存储好了相应的路由规则之后，那么具体的**请求又是怎么分发**的呢？请看下面的代码，默认的路由器实现了`ServeHTTP`：

```Go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {											// * 号关闭连接
        w.Header().Set("Connection", "close")
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)												// 实现Handler接口
    h.ServeHTTP(w, r)															// 调用接口方法ServeHTTP
}
```

如上所示路由器接收到请求之后，如果是`*`那么关闭链接，不然调用`mux.Handler(r)`**==返回对应请求Pattern设置路由的处理Handler==**，然后执行`h.ServeHTTP(w, r)`也就是**==调用对应路由的handler的ServerHTTP接口==**，那么mux.Handler(r)怎么处理的呢？

```go
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    if r.Method != "CONNECT" {
        if p := cleanPath(r.URL.Path); p != r.URL.Path {
            _, pattern = mux.handler(r.Host, p)
            return RedirectHandler(p, StatusMovedPermanently), pattern		// 重定向Handler
        }
    }    
    return mux.handler(r.Host, r.URL.Path)							// 是请求链接的客户端请求
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    // Host-specific pattern takes precedence over generic ones 主机专一性的模式优先于一般的规则
    if mux.hosts {															// 是否在任意的规则中带有host信息
      h, pattern = mux.match(host + path)				
    }
    if h == nil {	
      	//match: Find a handler on a handler map given a path string.
        h, pattern = mux.match(path)						// 根据ServeMux中的m(map[string]muxEntry)与请求的url匹配获得muxEntry实体再返回其中的 h(Handler) 和 pattren
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}
```

原来他是**==根据用户请求的URL和路由器里面存储的map去匹配==**的，当匹配到之后**返回存储的handler**，调用这个**handler的ServeHTTP接口就可以执行到相应的函数了**。

通过上面这个介绍，我们了解了整个路由过程，**==Go其实支持外部实现的路由器 `ListenAndServe`的第二个参数就是用以配置<font color='#39b54a'>外部路由器</font>的==**，**它是一个Handler接口，即外部路由器只要实现了Handler接口就可以**,我们可以在自己实现的路由器的ServeHTTP里面实现自定义路由功能。

##### <font color='#fbbd08'>**自己实现一个简单的外部路由:**</font>

```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

// 自己创建一个简单的路由器
type MyMux struct {
}

func (p *MyMux) ServeHTTP(w http.ResponseWriter, r *http.Request)  {
	if r.URL.Path == "/abc" {
		SayHello2(w, r)
		return
	}
	fmt.Println("Http not found")
	http.NotFound(w, r)
	return
}

func SayHello2(w http.ResponseWriter, r *http.Request)  {
	fmt.Fprintf(w, "Hello ! xwj")
}

func main() {
	mux := &MyMux{}			// 创建路由实例
	err := http.ListenAndServe(":9090", mux)
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

#### Go代码执行整体流程

通过对http包的分析之后，现在让我们来梳理一下整个的代码执行过程。

- 首先调用Http.HandleFunc

  按顺序做了几件事：

  1 调用了DefaultServeMux的HandleFunc

  2 调用了DefaultServeMux的Handle

  3 往DefaultServeMux的map[string]muxEntry中增加对应的handler和路由规则<font color='#39b54a'>(注册路由)</font>

- 其次调用http.ListenAndServe(":9090", nil)

  按顺序做了几件事情：

  1 实例化Server

  2 调用Server的ListenAndServe()

  3 调用net.Listen("tcp", addr)监听端口

  4 启动一个for循环，在循环体中Accept请求

  5 对每个请求实例化一个Conn，并且开启一个goroutine为这个请求进行服务go c.serve()

  6 读取每个请求的内容w, err := c.readRequest()

  7 判断handler是否为空，如果没有设置handler（这个例子就没有设置handler），handler就设置为DefaultServeMux

  8 调用handler的ServeHttp

  9 在这个例子中，下面就进入到DefaultServeMux.ServeHttp

  10 根据request选择handler，并且进入到这个handler的ServeHTTP<font color='#39b54a'>(查询路由)</font>

  ```
    mux.handler(r).ServeHTTP(w, r)
  ```

  11 选择handler：

  A 判断是否有路由能满足这个request（循环遍历ServeMux的muxEntry）

  B 如果有路由满足，调用这个路由handler的ServeHTTP

  C 如果没有路由满足，调用NotFoundHandler的ServeHTTP

## 1.2 表单

表单是一个包含表单元素的区域。表单元素是允许用户在表单中（比如：文本域、下拉列表、单选框、复选框等等）输入信息的元素。表单使用表单标签（\）定义。

```
<form>
...
input 元素
...
</form>
```

Go里面对于form处理已经有很方便的方法了，在Request里面的有专门的form处理，可以很方便的整合到Web开发里面来.

HTTP协议是一种无状态<font color='#39b54a'>(不保存每次链接的连接者信息)</font>的协议，那么如何才能**辨别是否是同一个用户**呢？同时又如何**保证一个表单不出现多次递交**的情况呢？后面小节里面将对cookie(cookie是存储在客户端的信息，能够每次通过header和服务器进行交互的数据)等进行详细讲解。表单还有一个很大的功能就是能够上传文件，那么**Go是如何处理文件上传**的呢？针对大文件上传我们如何有效的处理呢？我们将一起学习Go处理文件上传的知识。

### 1.处理表单的输入

先来看一个表单递交的例子，我们有如下的表单内容，命名成文件login.gtpl(放入当前新建项目的目录里面)

```html
<html>
<head>
<title></title>
</head>
<body>
<form action="/login" method="post">
    用户名:<input type="text" name="username">
    密码:<input type="password" name="password">
    <input type="submit" value="登录">
</form>
</body>
</html>
```

上面递交表单到服务器的`/login`，当用户输入信息点击登录之后，会跳转到服务器的路由`login`里面，我们首先要判断这个是什么方式传递过来，POST还是GET呢？

http包里面有一个很简单的方式就可以获取，我们在前面web的例子的基础上来看看怎么处理login页面的form数据

```Go
package main

import (
    "fmt"
    "html/template"
    "log"
    "net/http"
    "strings"
)

func sayhelloName(w http.ResponseWriter, r *http.Request) {
    r.ParseForm()       	//解析url传递的参数，对于POST则解析响应包的主体（request body）
    //注意:如果没有调用ParseForm方法，下面无法获取表单的数据
    fmt.Println(r.Form) //这些信息是输出到服务器端的打印信息
    fmt.Println("path", r.URL.Path)
    fmt.Println("scheme", r.URL.Scheme)
    fmt.Println(r.Form["url_long"])
    for k, v := range r.Form {
        fmt.Println("key:", k)
        fmt.Println("val:", strings.Join(v, ""))
    }
    fmt.Fprintf(w, "Hello astaxie!") //这个写入到w的是输出到客户端的
}

func login(w http.ResponseWriter, r *http.Request) {
    fmt.Println("method:", r.Method) //获取请求的方法
    if r.Method == "GET" {
        t, _ := template.ParseFiles("login.gtpl")
        log.Println(t.Execute(w, nil))
    } else {
        //请求的是登录数据，那么执行登录的逻辑判断
        fmt.Println("username:", r.Form["username"])
        fmt.Println("password:", r.Form["password"])
    }
}

func main() {
    http.HandleFunc("/", sayhelloName)       //设置访问的路由
    http.HandleFunc("/login", login)         //设置访问的路由
    err := http.ListenAndServe(":9090", nil) //设置监听的端口
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

通过上面的代码我们可以看出**==获取请求方法是通过`r.Method`来完成的==**，这是个字符串类型的变量，返回GET, POST, PUT等method信息。

login函数中我们根据`r.Method`来判断是显示登录界面还是处理登录逻辑。**当GET方式请求时显示登录界面，其他方式请求时则处理登录逻辑，如查询数据库、验证登录信息等。**

当我们在浏览器里面打开`http://127.0.0.1:9090/login`的时候，出现如下界面

![kRUbj1](http://xwjpics.gumptlu.work/qinniu_uPic/kRUbj1.png)

如果你看到一个空页面，可能是你写的 login.gtpl 文件中有错误，请根据控制台中的日志进行修复。

我们输入用户名和密码之后发现在服务器端是不会打印出来任何输出的，为什么呢？默认情况下，**==Handler里面是不会自动解析form的，必须显式的调用`r.ParseForm()`后==**，你才能对这个表单数据进行操作。我们修改一下代码，在`fmt.Println("username:", r.Form["username"])`之前加一行`r.ParseForm()`,重新编译，再次测试输入递交，现在是不是在服务器端有输出你的输入的用户名和密码了。

**==`r.Form`里面包含了所有请求的参数==**，**比如URL中query-string、POST的数据、PUT的数据， ** **所以当你==在URL中的query-string字段和POST冲突时，会保存成一个slice==，里面存储了多个值**，Go官方文档中说在接下来的版本里面将会把POST、GET这些数据分离开来。

现在我们修改一下login.gtpl里面form的action值`http://127.0.0.1:9090/login`修改为`http://127.0.0.1:9090/login?username=astaxie`，再次测试，服务器的输出username是不是一个slice。服务器端的输出如下：

```go
method: POST
username: [31231 astaxie]
password: [312321]
```

`request.Form`是一个url.Values类型，里面存储的是对应的类似`key=value`的信息，下面展示了可以对form数据进行的一些操作:

```Go
v := url.Values{}
v.Set("name", "Ava")
v.Add("friend", "Jess")
v.Add("friend", "Sarah")
v.Add("friend", "Zoe")
// v.Encode() == "name=Ava&friend=Jess&friend=Sarah&friend=Zoe"
fmt.Println(v.Get("name"))
fmt.Println(v.Get("friend"))
fmt.Println(v["friend"])
```

> **Tips**: Request本身也提供了**FormValue()函数来获取用户提交的参数**。如r.Form["username"]也可写成r.FormValue("username")。**调用r.FormValue时会自动调用r.ParseForm，所以不必提前调用**。**r.FormValue只会返回同名参数中的第一个，若参数不存在则返回空字符串。**

### 2.验证表单的输入

开发Web的一个原则就是，**不能信任用户输入的任何信息**，所以验证和过滤用户的输入信息就变得非常重要，我们经常会在微博、新闻中听到某某网站被入侵了，存在什么漏洞，这些大多是因为网站对于用户输入的信息没有做严格的验证引起的，所以为了编写出安全可靠的Web程序，验证表单输入的意义重大。

我们平常编写Web应用主要有两方面的数据验证，**一个是在页面端的js验证(目前在这方面有很多的插件库，比如ValidationJS插件)**，一**个是在服务器端的验证**，我们这小节讲解的是如何在**==服务器端验证==**。

#### 必填字段

你想要确保从一个表单元素中得到一个值，例如前面小节里面的用户名，我们如何处理呢？Go有一个内置函数`len`可以获取字符串的长度，这样我们就可以通过**len来获取数据的长度**，例如：

```Go
if len(r.Form["username"][0])==0{			// 注意r.Form["username"]返回的是一个slice所以slice[0]才是参数
    //为空的处理
}
```

`r.Form`对不同类型的表单元素的留空有不同的处理， 对于空文本框、空文本区域以及文件上传，元素的值为空值,**而如果是未选中的复选框和单选按钮，则根本不会在r.Form中产生相应条目**，如果我们用上面例子中的方式去获取数据时程序就会报错。**所以我们需要通过`r.Form.Get()`来获取值，因为如果字段不存在，通过该方式获取的是空值**。但是通过`r.Form.Get()`只能获取单个的值，**如果是map的值，必须通过上面的方式来获取。**

#### 数字

你想要确保一个表单输入框中获取的只能是数字，例如，你想通过表单获取某个人的具体年龄是50岁还是10岁，而不是像“一把年纪了”或“年轻着呢”这种描述

如果我们是判断正整数，那么我们**先转化成int类型**，然后进行处理

```Go
getint,err:=strconv.Atoi(r.Form.Get("age"))
if err!=nil{
    //数字转化出错了，那么可能就不是数字
}

//接下来就可以判断这个数字的大小范围了
if getint >100 {
    //太大了
}
```

还有一种方式就是**正则匹配**的方式

```Go
if m, _ := regexp.MatchString("^[0-9]+$", r.Form.Get("age")); !m {
    return false
}
```

对于性能要求很高的用户来说，这是一个老生常谈的问题了，**他们认为应该尽量避免使用正则表达式，因为使用正则表达式的速度会比较慢**。但是在目前机器性能那么强劲的情况下，对于这种简单的正则表达式效率和类型转换函数是没有什么差别的。如果你对正则表达式很熟悉，而且你在其它语言中也在使用它，那么在Go里面使用正则表达式将是一个便利的方式。

> Go实现的正则是[RE2](http://code.google.com/p/re2/wiki/Syntax)，所有的字符都是UTF-8编码的。

#### 中文

有时候我们想通过表单元素获取一个用户的中文名字，但是又为了保证获取的是正确的中文，我们需要进行验证，而不是用户随便的一些输入。对于中文我们目前有两种方式来验证，可以使用 `unicode` 包提供的 `func Is(rangeTab *RangeTable, r rune) bool` 来验证，也可以使用正则方式来验证，这里使用最简单的正则方式，如下代码所示

```Go
if m, _ := regexp.MatchString("^\\p{Han}+$", r.Form.Get("realname")); !m {
    return false
}
```

#### 英文

我们期望通过表单元素获取一个英文值，例如我们想知道一个用户的英文名，应该是astaxie，而不是asta谢。

我们可以很简单的通过正则验证数据：

```Go
if m, _ := regexp.MatchString("^[a-zA-Z]+$", r.Form.Get("engname")); !m {
    return false
}
```

#### 电子邮件地址

你想知道用户输入的一个Email地址是否正确，通过如下这个方式可以验证：

```Go
if m, _ := regexp.MatchString(`^([\w\.\_]{2,10})@(\w{1,}).([a-z]{2,4})$`, r.Form.Get("email")); !m {
    fmt.Println("no")
}else{
    fmt.Println("yes")
}
```

#### 手机号码

你想要判断用户输入的手机号码是否正确，通过正则也可以验证：

```Go
if m, _ := regexp.MatchString(`^(1[3|4|5|8][0-9]\d{4,8})$`, r.Form.Get("mobile")); !m {
    return false
}
```

#### 下拉菜单

如果我们想要判断表单里面`<select>`元素生成的下拉菜单中是否有被选中的项目。**有些时候黑客可能会伪造这个下拉菜单不存在的值发送给你，那么如何判断这个值是否是我们预设的值呢？**

我们的select可能是这样的一些元素

```html
<select name="fruit">
<option value="apple">apple</option>
<option value="pear">pear</option>
<option value="banane">banane</option>
</select>
```

那么我们可以这样来验证

```Go
slice:=[]string{"apple","pear","banane"}

v := r.Form.Get("fruit")
for _, item := range slice {
    if item == v {
        return true
    }
}

return false
```

#### 单选按钮

如果我们想要判断radio按钮是否有一个被选中了，我们页面的输出可能就是一个男、女性别的选择，但是也可能一个15岁大的无聊小孩，一手拿着http协议的书，另一只手通过telnet客户端向你的程序在发送请求呢，你设定的性别男值是1，女是2，他给你发送一个3，你的程序会出现异常吗？因此我们也需要像下拉菜单的判断方式类似，判断我们获取的值是我们预设的值，而不是额外的值。

```html
<input type="radio" name="gender" value="1">男
<input type="radio" name="gender" value="2">女
```

那我们也可以类似下拉菜单的做法一样

```Go
slice:=[]int{1,2}

for _, v := range slice {
    if v == r.Form.Get("gender") {
        return true
    }
}
return false
```

#### 复选框

有一项选择兴趣的复选框，你想确定用户选中的和你提供给用户选择的是同一个类型的数据。

```html
<input type="checkbox" name="interest" value="football">足球
<input type="checkbox" name="interest" value="basketball">篮球
<input type="checkbox" name="interest" value="tennis">网球
```

对于复选框我们的验证和单选有点不一样，因为接收到的数据是一个slice

```Go
slice:=[]string{"football","basketball","tennis"}
a:=Slice_diff(r.Form["interest"],slice)
if a == nil{
    return true
}

return false
```

上面这个函数`Slice_diff`包含在我开源的一个库里面(操作slice和map的库)，https://github.com/astaxie/beeku

#### 日期和时间

你想确定用户填写的日期或时间是否有效。例如 ，用户在日程表中安排8月份的第45天开会，或者提供未来的某个时间作为生日。

Go里面提供了一个time的处理包，我们可以把用户的输入年月日转化成相应的时间，然后进行逻辑判断

```Go
t := time.Date(2009, time.November, 10, 23, 0, 0, 0, time.UTC)
fmt.Printf("Go launched at %s\n", t.Local())
```

获取time之后我们就可以进行很多时间函数的操作。具体的判断就根据自己的需求调整。

#### 身份证号码

如果我们想验证表单输入的是否是身份证，通过正则也可以方便的验证，但是身份证有15位和18位，我们两个都需要验证

```Go
//验证15位身份证，15位的是全部数字
if m, _ := regexp.MatchString(`^(\d{15})$`, r.Form.Get("usercard")); !m {
    return false
}

//验证18位身份证，18位前17位为数字，最后一位是校验位，可能为数字或字符X。
if m, _ := regexp.MatchString(`^(\d{17})([0-9]|X)$`, r.Form.Get("usercard")); !m {
    return false
}
```

上面列出了我们一些常用的服务器端的表单元素验证，希望通过这个引导入门，能够让你对Go的数据验证有所了解，特别是Go里面的正则处理。

### 3.预防跨站脚本

现在的网站包含大量的动态内容以提高用户体验，比过去要复杂得多。所谓**动态内容**，就是**根据用户环境和需要，Web应用程序能够输出相应的内容**。动态站点会受到一种名为**==“跨站脚本攻击”==（Cross Site Scripting, 安全专家们通常将其缩写成 XSS）**的威胁，而静态站点则完全不受其影响。<font color='#39b54a'>(服务器输出给客户端的数据被注入脚本,导致恶意脚本在浏览器中运行)</font>

攻击者通常会在有漏洞的程序中插入JavaScript、VBScript、 ActiveX或Flash以欺骗用户。一旦得手，他们可以盗取用户帐户信息，修改用户设置，盗取/污染cookie和植入恶意广告等。

对XSS最佳的防护应该结合以下两种方法：**一是验证所有输入数据**，有效检测攻击(这个我们前面小节已经有过介绍);**另一个是对所有输出数据进行适当的处理，以防止任何已成功注入的脚本在浏览器端运行。**

那么Go里面是怎么做这个有效防护的呢？Go的**==html/template==**里面带有下面几个函数可以帮你**转义**

- func HTMLEscape(w io.Writer, b []byte) //把b进行转义之后写到w
- func HTMLEscapeString(s string) string //转义s之后返回结果字符串
- func HTMLEscaper(args ...interface{}) string //支持多个参数一起转义，返回结果字符串

我们看4.1小节的例子

```Go
fmt.Println("username:", template.HTMLEscapeString(r.Form.Get("username"))) //输出到服务器端
fmt.Println("password:", template.HTMLEscapeString(r.Form.Get("password")))
template.HTMLEscape(w, []byte(r.Form.Get("username"))) //输出到客户端
```

如果我们输入的username是`<script>alert()</script>`,那么我们可以在浏览器上面看到输出如下所示：

![img](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/images/4.3.escape.png?raw=true)

**==Go的html/template包默认帮你过滤了html标签==**，但是有时候你只想要输出这个`<script>alert()</script>`看起来正常的信息，该怎么处理？请使用text/template。请看下面的例子：

```Go
import "text/template"
...
t, err := template.New("foo").Parse(`{{define "T"}}Hello, {{.}}!{{end}}`)
err = t.ExecuteTemplate(out, "T", "<script>alert('you have been pwned')</script>")
```

输出

```
Hello, <script>alert('you have been pwned')</script>!
```

或者使用template.HTML类型

```Go
import "html/template"
...
t, err := template.New("foo").Parse(`{{define "T"}}Hello, {{.}}!{{end}}`)
err = t.ExecuteTemplate(out, "T", template.HTML("<script>alert('you have been pwned')</script>"))
```

输出

```
Hello, <script>alert('you have been pwned')</script>!
```

转换成`template.HTML`后，变量的内容也不会被转义

转义的例子：

```Go
import "html/template"
...
t, err := template.New("foo").Parse(`{{define "T"}}Hello, {{.}}!{{end}}`)
err = t.ExecuteTemplate(out, "T", "<script>alert('you have been pwned')</script>")
```

转义之后的输出：

```
Hello, &lt;script&gt;alert(&#39;you have been pwned&#39;)&lt;/script&gt;!
```

### 4. 防止多次递交表单

不知道你是否曾经看到过一个论坛或者博客，在一个帖子或者文章后面出现多条重复的记录，这些大多数是因为用户重复递交了留言的表单引起的。由于种种原因，用户经常会重复递交表单。通常这只是鼠标的误操作，如双击了递交按钮，也可能是为了编辑或者再次核对填写过的信息，点击了浏览器的后退按钮，然后又再次点击了递交按钮而不是浏览器的前进按钮。当然，也可能是故意的——比如，在某项在线调查或者博彩活动中重复投票。那我们如何有效的防止用户多次递交相同的表单呢？

解决方案是**在表单中添加一个==带有唯一值的隐藏字段==**。在验证表单时，先检查带有该唯一值的表单是否已经递交过了。如果是，拒绝再次递交；如果不是，则处理表单进行逻辑处理。另外，**如果是采用了Ajax模式递交表单的话，当表单递交后，通过<u>javascript来禁用表单的递交按钮。</u>**

例如:

```html
<input type="checkbox" name="interest" value="football">足球
<input type="checkbox" name="interest" value="basketball">篮球
<input type="checkbox" name="interest" value="tennis">网球    
用户名:<input type="text" name="username">
密码:<input type="password" name="password">
<input type="hidden" name="token" value="{{.}}">
<input type="submit" value="登陆">
```

我们在模版里面增加了**一个隐藏字段`token`，这个值我们通过MD5(时间戳)来获取唯一值**，然后我们把这个**值存储到服务器端(session来控制，我们将在第六章讲解如何保存)**，以方便表单提交时比对判定。

```Go
func login(w http.ResponseWriter, r *http.Request) {
    fmt.Println("method:", r.Method) //获取请求的方法
    if r.Method == "GET" {
        crutime := time.Now().Unix()
        h := md5.New()																			// 创建md5实例
        io.WriteString(h, strconv.FormatInt(crutime, 10))		// 转换时间unix格式 => Int 给md5实例
        token := fmt.Sprintf("%x", h.Sum(nil))							//h.Sum(nil)计算hash

        t, _ := template.ParseFiles("login.gtpl")
        t.Execute(w, token)
    } else {
        //请求的是登陆数据，那么执行登陆的逻辑判断
        r.ParseForm()
        token := r.Form.Get("token")
        if token != "" {
            //验证token的合法性
        } else {
            //不存在token报错
        }
        fmt.Println("username length:", len(r.Form["username"][0]))
        fmt.Println("username:", template.HTMLEscapeString(r.Form.Get("username"))) //输出到服务器端
        fmt.Println("password:", template.HTMLEscapeString(r.Form.Get("password")))
        template.HTMLEscape(w, []byte(r.Form.Get("username"))) //输出到客户端
    }
}
```

其中的template简单解析: 详细template教程见:[Go template用法详解](https://www.cnblogs.com/f-ck-need-u/p/10053124.html)

* 在html文件中使用temlpate语法:

  ```
  {{.}}
  ```

  这部分是需要通过go的template引擎进行解析，然后替换成对应的内容。

* handler函数中使用`template.ParseFiles("login.gtpl")`，它会自动创建一个模板(关联到变量t上)，并解析一个或多个文本文件(不仅仅是html文件)

* 解析之后就可以使用`Execute(w, token)`去执行解析后的模板对象，执行过程是**合并、替换**的过程。例如html中的

  ```
  {{.}}
  ```

  会被替换成服务器端计算的token

上面的代码输出到页面的源码如下：

增加token之后在客户端输出的源码信息:

![bmpRdc](http://xwjpics.gumptlu.work/qinniu_uPic/bmpRdc.jpg)

我们看到token已经有输出值，你可以不断的刷新，可以看到这个值在不断的变化。这样就保证了每次显示form表单的时候都是唯一的，用户递交的表单保持了唯一性。

我们的解决方案可以防止非恶意的攻击，并能使恶意用户暂时不知所措，然后，它却不能排除所有的欺骗性的动机，对此类情况还需要更复杂的工作。

### 5.文件上传

要使表单能够上传文件，首先第一步就是要添加form的`enctype`属性，`enctype`属性有如下三种情况:

```
application/x-www-form-urlencoded   			表示在发送前编码所有字符（默认）
multipart/form-data      									不对字符编码。在使用包含文件上传控件的表单时，必须使用该值。
text/plain      													空格转换为 "+" 加号，但不对特殊字符编码。
```

所以，创建新的表单html文件, 命名为upload.gtpl, html代码应该类似于:

```html
<html>
<head>
    <title>上传文件</title>
</head>
<body>
<form enctype="multipart/form-data" action="/upload" method="post">
  <input type="file" name="uploadfile" />
  <input type="hidden" name="token" value="{{.}}"/>
  <input type="submit" value="upload" />
</form>
</body>
</html>
```

在服务器端，我们增加一个handlerFunc:

```Go
http.HandleFunc("/upload", upload)

// 处理/upload 逻辑
func upload(w http.ResponseWriter, r *http.Request) {
    fmt.Println("method:", r.Method) //获取请求的方法
    if r.Method == "GET" {
        crutime := time.Now().Unix()
        h := md5.New()
        io.WriteString(h, strconv.FormatInt(crutime, 10))
        token := fmt.Sprintf("%x", h.Sum(nil))

        t, _ := template.ParseFiles("upload.gtpl")
        t.Execute(w, token)
    } else {
        r.ParseMultipartForm(32 << 20)
        file, handler, err := r.FormFile("uploadfile")
        if err != nil {
            fmt.Println(err)
            return
        }
        defer file.Close()
        fmt.Fprintf(w, "%v", handler.Header)
        f, err := os.OpenFile("./test/"+handler.Filename, os.O_WRONLY|os.O_CREATE, 0666)  // 此处假设当前目录下已存在test目录
        if err != nil {
            fmt.Println(err)
            return
        }
        defer f.Close()
        io.Copy(f, file)
    }
}
```

通过上面的代码可以看到，处理文件上传我们需要调用`r.ParseMultipartForm`，里面的参数表示`maxMemory`，调用`ParseMultipartForm`之后，上传的文件存储在`maxMemory`大小的内存里面，如果文件大小超过了`maxMemory`，那么剩下的部分将存储在系统的临时文件中。我们可以通过`r.FormFile`获取上面的文件句柄，然后实例中使用了`io.Copy`来存储文件。

> 获取其他非文件字段信息的时候就不需要调用`r.ParseForm`，因为在需要的时候Go自动会去调用。而且`ParseMultipartForm`调用一次之后，后面再次调用不会再有效果。

通过上面的实例我们可以看到我们**上传文件主要三步处理**：

1. 表单中**增加`enctype="multipart/form-data”`**
2. 服务端调用`r.ParseMultipartForm`,**把上传的文件存储在<u>内存</u>和<u>临时文件</u>中**
3. **使用`r.FormFile`获取文件句柄**，然后对文件进行存储等处理。

文件`handler`是`multipart.FileHeader`,里面存储了如下结构信息

```Go
type FileHeader struct {
    Filename string
    Header   textproto.MIMEHeader
    // contains filtered or unexported fields
}
```

我们通过上面的实例代码打印出来上传文件的信息如下

![zkJnRK](http://xwjpics.gumptlu.work/qinniu_uPic/zkJnRK.jpg)

---

**模拟客户端上传文件:**

我们上面的例子演示了如何通过表单上传文件，然后在服务器端处理文件，其实**Go支持模拟客户端表单功能支持文件上传**，详细用法请看如下示例：

```Go
package main

import (
    "bytes"
    "fmt"
    "io"
    "io/ioutil"
    "mime/multipart"
    "net/http"
    "os"
)

func postFile(filename string, targetUrl string) error {
    bodyBuf := &bytes.Buffer{}
    bodyWriter := multipart.NewWriter(bodyBuf)

    //关键的一步操作
    fileWriter, err := bodyWriter.CreateFormFile("uploadfile", filename)
    if err != nil {
        fmt.Println("error writing to buffer")
        return err
    }

    //打开文件句柄操作
    fh, err := os.Open(filename)
    if err != nil {
        fmt.Println("error opening file")
        return err
    }
    defer fh.Close()

    //iocopy
    _, err = io.Copy(fileWriter, fh)
    if err != nil {
        return err
    }

    contentType := bodyWriter.FormDataContentType()
    bodyWriter.Close()

    resp, err := http.Post(targetUrl, contentType, bodyBuf)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    resp_body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return err
    }
    fmt.Println(resp.Status)
    fmt.Println(string(resp_body))
    return nil
}

// sample usage
func main() {
    target_url := "http://localhost:9090/upload"
    filename := "./astaxie.pdf"
    postFile(filename, target_url)
}
```

上面的例子详细展示了客户端如何向服务器上传一个文件的例子，客户端通过`multipart.Write`把文件的文本流写入一个缓存中，然后调用http的Post方法把缓存传到服务器。

> 如果你还有其他普通字段例如username之类的需要同时写入，那么可以调用multipart的WriteField方法写很多其他类似的字段。

## 1.3 访问数据库

Go没有内置的驱动支持任何的数据库，但是Go定义了database/sql接口，用户可以基于驱动接口开发相应数据库的驱动

### 1. database/sql接口

Go与PHP不同的地方是**Go官方没有提供数据库驱动**，而是**为开发数据库驱动定义了一些标准接口**，开发者可以根据定义的接口来开发相应的数据库驱动，这样做有一个好处，**==只要是按照标准接口开发的代码， 以后需要迁移数据库时，不需要任何修改==**。那么Go都定义了哪些标准接口呢？让我们来详细的分析一下 

#### sql.Register

这个存在于database/sql的函数是用来==**注册数据库驱动**==的，当第三方开发者开发数据库驱动时，都会实现init函数，在init里面会调用这个`Register(name string, driver driver.Driver)`完成本驱动的注册。

我们来看一下mymysql、sqlite3的驱动里面都是怎么调用的：

```Go
//https://github.com/mattn/go-sqlite3驱动
func init() {
    sql.Register("sqlite3", &SQLiteDriver{})
}

//https://github.com/mikespook/mymysql驱动
// Driver automatically registered in database/sql
var d = Driver{proto: "tcp", raddr: "127.0.0.1:3306"}
func init() {
    Register("SET NAMES utf8")
    sql.Register("mymysql", &d)
}
```

我们看到第三方数据库驱动都是通过调用这个函数来**注册自己的数据库驱动名称以及相应的driver实现**。**在database/sql内部通过一个map来存储用户定义的相应驱动。**

```Go
var drivers = make(map[string]driver.Driver)

drivers[name] = driver
```

因此通过database/sql的注册函数可以**同时注册多个数据库驱动，只要不重复。**

> 在我们使用database/sql接口和第三方库的时候经常看到如下:
>
> ```
>    import (
>        "database/sql"
>         _ "github.com/mattn/go-sqlite3"
>    )
> ```
>
> 新手都会被这个`_`所迷惑，其实这个就是Go设计的巧妙之处，我们在变量赋值的时候经常看到这个符号，它是用来忽略变量赋值的占位符，那么包引入用到这个符号也是相似的作用，<font color='#39b54a'>**这儿使用`_`的意思是引入后面的包名而不直接使用这个包中定义的函数，变量等资源。**</font>
>
> 我们在2.3节流程和函数一节中介绍过**init函数的初始化过程**，**包在引入的时候会自动调用包的init函数以完成对包的初始化。**因此，我们引入上面的数据库驱动包之后会自动去调用init函数，然后在init函数里面注册这个数据库驱动，这样我们就可以在接下来的代码中直接使用这个数据库驱动了。

#### driver.Driver

Driver是一个==**数据库驱动**==的接口，他定义了一个method： Open(name string)，这个方法**返回一个数据库的Conn接口。**

```Go
type Driver interface {
    Open(name string) (Conn, error)
}
```

==**返回的Conn只能用来进行一次goroutine的操作**==，也就是说不能把这个Conn应用于Go的多个goroutine里面。如下代码会出现错误

```Go
...
go goroutineA (Conn)  //执行查询操作
go goroutineB (Conn)  //执行插入操作
...
```

上面这样的代码可能会使**Go不知道某个操作究竟是由哪个goroutine发起的,从而导致数据混乱**，比如可能会把goroutineA里面执行的查询操作的结果返回给goroutineB从而使B错误地把此结果当成自己执行的插入数据。

**第三方驱动都会定义这个函数**，**它会解析name参数来获取相关数据库的连接信息，解析完成后，它将使用此信息来<u>初始化一个Conn并返回它</u>。**

#### driver.Conn

Conn是一个==**数据库连接的接口定义**==，他定义了一系列方法，**这个Conn只能应用在一个goroutine里面，不能使用在多个goroutine里面**，详情请参考上面的说明。

```Go
type Conn interface {
    Prepare(query string) (Stmt, error)
    Close() error
    Begin() (Tx, error)
}
```

**Prepare**函数**返回与当前连接相关的执行Sql语句的准备状态**，**可以进行查询、删除等操作。**

**Close函数关闭当前的连接，执行释放连接拥有的资源等清理工作。**因为驱动实现了database/sql里面建议的conn pool，所以你不用再去实现缓存conn之类的，这样会容易引起问题。

**Begin函数返回一个代表事务处理的Tx，通过它你可以进行查询,更新等操作，或者对事务进行回滚、递交。**

#### driver.Stmt

Stmt是一种==**准备好的状态**==，**和Conn相关联**，而且**只能应用于一个goroutine中，不能应用于多个goroutine。**

```Go
type Stmt interface {
    Close() error
    NumInput() int
    Exec(args []Value) (Result, error)
    Query(args []Value) (Rows, error)
}
```

Close函数**关闭当前的链接状态**，但是如果当前正在执行query，**query还是有效返回rows数据。**

**NumInput函数返回当前预留参数的个数**，当返回>=0时数据库驱动就会智能检查调用者的参数。当数据库驱动包不知道预留参数的时候，返回-1。

**<font color='#39b54a'>Exec函数执行Prepare准备好的sql，传入参数执行update/insert等操作，返回Result数据</font>**

<font color='#39b54a'>**Query函数执行Prepare准备好的sql，传入需要的参数执行select操作，返回Rows结果集**</font>

#### driver.Tx

**==事务处理==**一般就两个过程，**递交**或者**回滚**。数据库驱动里面也只需要实现这两个函数就可以

```Go
type Tx interface {
    Commit() error
    Rollback() error
}
```

这两个函数一个用来递交一个事务，一个用来回滚事务。

#### driver.Execer

这是一个**==Conn可选择实现的接口==**

```Go
type Execer interface {
    Exec(query string, args []Value) (Result, error)
}
```

如果这个接口没有定义，那么**在调用DB.Exec,就会首先调用Prepare返回Stmt，然后执行Stmt的Exec，然后关闭Stmt。**

#### driver.Result

这个是执行Update/Insert等操作返回的**==结果接口定义==**

```Go
type Result interface {
    LastInsertId() (int64, error)
    RowsAffected() (int64, error)
}
```

`LastInsertId`函数返回由数据库执行插入操作得到的**自增ID号。**

`RowsAffected`函数返回**query操作影响的数据条目数。**

#### driver.Rows

Rows是执行**==查询==**返回的**==结果集接口定义==**

```Go
type Rows interface {
    Columns() []string
    Close() error
    Next(dest []Value) error
}
```

Columns函数**返回查询数据库表的字段信息**，这个**返回的slice和sql查询的字段一一对应**，而不是返回整个表的所有字段。

Close函数用来**关闭Rows迭代器**。

Next函数**用来返回下一条数据**，把数据赋值给dest。**dest里面的元素必须是driver.Value的值除了string**，返回的数据里面所有的string都必须要转换成[]byte。如果最后没数据了，Next函数最后返回io.EOF。

#### driver.RowsAffected

RowsAffected其实就是一个**==int64的别名==**，但是**他实现了Result接口**，用来底层实现Result的表示方式

```Go
type RowsAffected int64

func (RowsAffected) LastInsertId() (int64, error)

func (v RowsAffected) RowsAffected() (int64, error)
```

#### driver.Value

Value其实就是一个**==空接口==**，他可以容纳任何的数据

```Go
type Value interface{}
```

drive的Value是驱动**必须能够操作的Value，Value要么是nil，要么是下面的任意一种**

```Go
int64
float64
bool
[]byte
string   [*]除了Rows.Next返回的不能是string.
time.Time
```

#### driver.ValueConverter

ValueConverter接口定义了如何**==把一个普通的值转化成driver.Value的接口==**

```Go
type ValueConverter interface {
    ConvertValue(v interface{}) (Value, error)
}
```

在开发的数据库驱动包里面实现这个接口的函数在很多地方会使用到，这个**ValueConverter有很多好处：**

- **转化driver.value到数据库表相应的字段**，例如int64的数据如何转化成数据库表uint16字段
- **把数据库查询结果转化成driver.Value值**
- **在scan函数里面如何把driver.Value值转化成用户定义的值**

#### driver.Valuer

Valuer接口定义了**==返回一个driver.Value的方式==**

```Go
type Valuer interface {
    Value() (Value, error)
}
```

很多类型都实现了这个Value方法，用来自身与driver.Value的转化。

通过上面的讲解，你应该对于驱动的开发有了一个基本的了解，**一个驱动只要实现了这些接口就能完成增删查改等基本操作了**，剩下的就是与相应的数据库进行数据交互等细节问题了，在此不再赘述。

#### database/sql

database/sql在database/sql/driver提供的接口基础上**定义了一些更高阶的方法**，用以简化数据库操作,同时内部还建议性地实现一个conn pool。

```Go
type DB struct {
    driver      driver.Driver
    dsn         string
    mu       sync.Mutex // protects freeConn and closed
    freeConn []driver.Conn
    closed   bool
}
```

我们可以看到Open函数返回的是DB对象，里面有一个freeConn，**它就是那个简易的连接池。**它的实现相当简单或者说简陋，就是当执行Db.prepare的时候会`defer db.putConn(ci, err)`,也就是把这个连接放入连接池，每次调用conn的时候会先判断freeConn的长度是否大于0，大于0说明有可以复用的conn，直接拿出来用就是了，如果不大于0，则创建一个conn,然后再返回之。





















## 1.4. Tcp编程

### 4.1 网络编程的两种模式

* **TCP socket 编程**，是网络编程的主流。之所以叫Tcp socket 编程，是因为底层是基于TCP/IP 协议的, 可以认为是**基于TCP自定义协议编程**. 比如: QQ 聊天[示意图]
* b/s 结构的**http 编程**，我们使用浏览器去访问服务器时，使用的就是http 协议，而http 底层依旧是用tcp socket 实现的。[示意图] 比如: 京东商城【这属于go web 开发范畴】

tcp socket 编程，简称**socket 编程**.下图为Golang socket 编程中客户端和服务器的网络分布:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129124110.png)

### 4.2 Socket编程小Demo 

#### 4.2.1 net包学习

**package net**

```
import "net"
```

net包提供了可移植的网络I/O接口，包括TCP/IP、UDP、域名解析和Unix域socket。

虽然本包提供了对网络原语的访问，大部分使用者只需要Dial、Listen和Accept函数提供的基本接口；以及相关的Conn和Listener接口。crypto/tls包提供了相同的接口和类似的Dial和Listen函数。

**Dial函数**和服务端建立连接：<font color='green'>（客户端）</font>

```go
conn, err := net.Dial("tcp", "google.com:80")
if err != nil {
	// handle error
}
fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
status, err := bufio.NewReader(conn).ReadString('\n')
// ...
```

Listen函数创建的<font color='green'>服务端</font>：

```go
ln, err := net.Listen("tcp", ":8080")
if err != nil {
	// handle error
}
for {
	conn, err := ln.Accept()
	if err != nil {
		// handle error
		continue
	}
	go handleConnection(conn)
}
```

type [Listener](https://github.com/golang/go/blob/master/src/net/net.go?name=release#266)

```go
type Listener interface {
    // Addr返回该接口的网络地址
    Addr() Addr
    // Accept等待并返回下一个连接到该接口的连接
    Accept() (c Conn, err error)
    // Close关闭该接口，并使任何阻塞的Accept操作都会不再阻塞并返回错误。
    Close() error
}
```

func [Listen](https://github.com/golang/go/blob/master/src/net/dial.go?name=release#261)

```
func Listen(net, laddr string) (Listener, error)
```

返回在一个本地网络地址laddr上监听的Listener。网络类型参数net必须是面向流的网络：

"tcp"、"tcp4"、"tcp6"、"unix"或"unixpacket"。参见Dial函数获取laddr的语法。

type [Conn](https://github.com/golang/go/blob/master/src/net/net.go?name=release#62)

```
type Conn interface {
    // Read从连接中读取数据
    // Read方法可能会在超过某个固定时间限制后超时返回错误，该错误的Timeout()方法返回真
    Read(b []byte) (n int, err error)
    // Write从连接中写入数据
    // Write方法可能会在超过某个固定时间限制后超时返回错误，该错误的Timeout()方法返回真
    Write(b []byte) (n int, err error)
    // Close方法关闭该连接
    // 并会导致任何阻塞中的Read或Write方法不再阻塞并返回错误
    Close() error
    // 返回本地网络地址
    LocalAddr() Addr
    // 返回远端网络地址
    RemoteAddr() Addr
    // 设定该连接的读写deadline，等价于同时调用SetReadDeadline和SetWriteDeadline
    // deadline是一个绝对时间，超过该时间后I/O操作就会直接因超时失败返回而不会阻塞
    // deadline对之后的所有I/O操作都起效，而不仅仅是下一次的读或写操作
    // 参数t为零值表示不设置期限
    SetDeadline(t time.Time) error
    // 设定该连接的读操作deadline，参数t为零值表示不设置期限
    SetReadDeadline(t time.Time) error
    // 设定该连接的写操作deadline，参数t为零值表示不设置期限
    // 即使写入超时，返回值n也可能>0，说明成功写入了部分数据
    SetWriteDeadline(t time.Time) error
}
```

Conn接口代表通用的面向流的网络连接。多个线程可能会同时调用同一个Conn的方法。

#### 4.2.2 demo要求

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129133540.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201129133522.png)



* 服务器端

  ```go
  func main() {
  	fmt.Println("服务器开始监听....")
  	//第一个参数: 网络协议名
  	//第二个参数: IP + 端口
  	listener, err := net.Listen("tcp", "0.0.0.0:8888")
  	if err != nil {
  		fmt.Println("服务器开启失败！", err)
  	}
  	//fmt.Println(listener)
  	defer listener.Close()	//演示关闭
  	//Accept函数始终帮助我们等待连接，而不会一下就结束
  	// Accept等待并返回下一个连接到该接口的连接
  	for  {
  		fmt.Println("等待客户端连接.....")
  		conn, err := listener.Accept()
  		if err != nil {
  			fmt.Println("连接失败！")
  		} else {
  			fmt.Printf("----------连接成功！客户端IP:%v----------\n", conn.RemoteAddr().String())
  		}
  		//这里启动协程为客户端服务
  		go Process(conn)		//每一个连接都是新的conn
  	}
  
  }
  
  
  //协程工作
  func Process(conn net.Conn)  {
  	defer conn.Close()	//延迟关闭
  	for  {
  		//创建一个新的切片
  		buf := make([]byte, 2048)
  		//等待客户端通过conn发送信息
  		//如果客户端没有write【发送】, 那么协程就阻塞在这！
  		//如果客户端宕机，这里的conn也会发现，并报错
  		//fmt.Printf("服务器在等待客户端[%v]的输入....\n", conn.RemoteAddr().String())
  		n, err := conn.Read(buf)
  		if err != nil {
  			fmt.Println("客户端已退出！连接结束", err)
  			break
  		}
  		//数据显示到服务器终端
  		//注意：
  		// 1. 使用Print而不是Println,因为客户端的\n也是读取过来的
  		// 2. buf[:n]必须要进行切片到n（n是读取到的字节数），不然缓冲buf的大小是2048，后面的字段可能未知
  		info := string(buf[:n])
  		fmt.Print(conn.RemoteAddr().String(), "说：", info)
  	}
  	fmt.Println("--------------连接结束-------------")
  }
  ```

* 客户端

  ```go
  func main() {
     //参数与服务器端一致
     conn, err := net.Dial("tcp", "127.0.0.1:8888")
     if err != nil {
        fmt.Println("连接失败！", err)
        return
     }
     //fmt.Println(conn)
     //1. 发送单行数据，然后退出
     reader := bufio.NewReader(os.Stdin) //os.Stdin终端输入
     for  {
        fmt.Println("请输入:")
        //从终端读取用户的数据，并发送给服务器
        line, err := reader.ReadString('\n')
        if err != nil {
           fmt.Println("读取输入失败！", err)
           return
        }
        if "exit" == strings.Trim(line, " \r\n"){ //去除掉回车换行空格与exit, Trim不会修改原字符串
           fmt.Println("Bye！")
           return
        }
        fmt.Println(line)
        //发送Line终端输入给服务器
        n, err := conn.Write([]byte(line))
        if err != nil {
           fmt.Println("向服务器写失败！", err)
           return
        }
        fmt.Printf("成功传输%d个字节\n", n)
     }
  }
  ```



