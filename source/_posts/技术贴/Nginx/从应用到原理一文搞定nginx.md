---
title: 从应用到原理一文搞定nginx
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-07-28 09:30:02
---

# 一、基本概念

## 是什么

Nginx ("engine x") 是一个高性能的<font color='#e54d42'>HTTP 和反向代理服务器</font> ,特点是占有内存少，并发能力强，事实上 nginx 的并发能力确实在同类型的网页服务器中表现较好，中国大陆使用 nginx 网站用户有:百度、京东、新浪、网易、腾讯、淘宝等

Nginx 专为性能优化而开发， 性能是其最重要的考量,实现上非常注重效率 ，能经受高负载的考验,有报告表明能支持高 达 50,000 个并发连接数。

<!-- more -->

## 反向代理

正向代理： 如果把局域网外的 Internet 想象成一个巨大的资源库，则局域网中的客户端要访问 Internet，则需要通过代理服务器来访问，这种代理服务就称为正向代理。

![image-20220113162345754](/Users/xwj/Library/Application Support/typora-user-images/image-20220113162345754.png)



> <font color='#39b54a'>在客户端配置代理服务器，通过代理服务器进行互联网的访问</font>

----

反向代理，其实**客户端对代理是无感知的**，因为**客户端不需要任何配置就可以访问**，我们只需要将请求发送到反向代理服务器，由反向代理服务器去选择目标服务器获取数据后，在返回给客户端，此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器 IP 地址。

![image-20220323103228036](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220323103228036.png)

## 负载均衡

面对大量请求的需求时，一般会使用集群来处理请求, 将请求分发到多个服务器上，将负载分发到不同的服务器，也就是我们所说的**负载均衡**

![image-20210728095504845](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728095504845.png)

## 动静分离

为了加快网站的解析速度，可以把动态页面和静态页面由不同的服务器来解析，加快解析速度。降低原来单个服务器的压力。

![image-20210728100009520](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728100009520.png)

# 二、安装Nginx

## 手动安装

官网下载：http://nginx.org 

安装一些依赖：（centos）

```shell
# 安装pcre
wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz
# 解压文件
./configure 
# 回到 pcre 目录下执行 
make && make install
# 安装zlib、openssl
yum -y install make zlib zlib-devel gcc-c++ libtool
# 安装nginx
# 解压缩 nginx-xx.tar.gz 包。
# 进入解压缩目录，执行
./configure
make && make install
```

安装之后启动，启动命令在`/usr/sbin/nginx`：

```shell
/usr/sbin/nginx   				# 启动
/usr/sbin/nginx -s stop 	# 关闭
/usr/sbin/nginx -s reload # 重启
ps -ef | grep nginx 			# 查看相关进程
```

## 包管理工具安装

`apt install nginx`

## Nginx文件位置

**Ubuntu**安装之后的文件结构大致为：

- 所有的配置文件都在`/etc/nginx`下，并且每个虚拟主机已经安排在了`/etc/nginx/sites-available`下
- 程序文件在`/usr/sbin/nginx`
- 日志放在了`/var/log/nginx`中
- 并已经在`/etc/init.d/`下创建了启动脚本nginx
- 默认的虚拟主机的目录设置在了`/var/www/nginx-default` (有的版本 默认的虚拟主机的目录设置在了`/var/www`, 请参考/etc/nginx/sites-available里的配置)

安装完nginx之后默认开启的是80端口，可以在浏览器输入对应的IP地址，将会自动访问80端口：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728102201816.png" alt="image-20210728102201816" style="zoom:50%;" />

如果无法访问，那么有可能是80端口被防火墙阻拦，开启的方式是：

```shell
# 查看开放的端口号
firewall-cmd --list-all
# 设置开放的端口号
firewall-cmd --add-service=http –permanent 
sudo firewall-cmd --add-port=80/tcp --permanent
# 重启防火墙
firewall-cmd –reload
```

# 三、Nginx常用命令

将你的nginx命令路径加入到全局变量中:

```shell
# centos的命令在：
/usr/local/nginx/sbin
# ubuntu的命令在：
/usr/sbin/nginx
```

```shell
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages durlng configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/share/nginx/)
  -c filename   : set configuration file (default: /etc/nginx/nginx.conf)
  -g directives : set global directives out of configuration file
```

```shell
/usr/sbin/nginx   				# 启动
/usr/sbin/nginx -s stop 	# 关闭
/usr/sbin/nginx -s reload # 重启
ps -ef | grep nginx 			# 查看相关进程
```

# 四、Nginx的配置文件

位置:

* Centos: `/usr/local/nginx/conf/nginx.conf`

* Ubuntu: `/etc/nginx/nginx.conf`

## 全局块

​	从配置文件开始到 events 块之间的内容，主要会设置一些**影响 nginx 服务器整体运行的配置指令**，主要包括配 置运行 Nginx 服务器的用户(组)、允许生成的 worker process 数，进程 PID 存放路径、日志存放路径和类型以 及配置文件的引入等。

## events块

​	events 块涉及的指令主要**影响 Nginx 服务器与用户的网络连接**，常用的设置包括是否开启对多 work process 下的网络连接进行序列化，是否允许同时接收多个网络连接，选取哪种事件驱动模型来处理连接请求，每个 word process 可以同时支持的最大连接数等。

## http块

​	这算是 Nginx 服务器配置中最频繁的部分，**代理、缓存和日志定义等绝大多数功能和第三方模块的配置**都在这里。 需要注意的是:**http 块也可以包括 http 全局块、server 块**

### http块

http 全局块配置的指令包括**文件引入、MIME-TYPE 定义、日志自定义、连接超时时间、单链接请求数上限**等。

### server块

这块和**虚拟主机**有密切关系，虚拟主机从用户角度看，和一台独立的硬件主机是完全一样的，该技术的产生是为了 节省互联网服务器硬件成本。

每个 http 块可以包括多个 server 块，而<font color='#e54d42'>**每个 server 块就相当于一个虚拟主机**</font>。 而每个 server 块也分为全局 server 块，以及可以同时包含多个 locaton 块。

#### 全局 server 块 

最常见的配置是**本虚拟机主机的监听配置和本虚拟主机的名称或 IP 配置**。

#### location 块

一个 server 块可以配置多个 location 块。

这块的主要作用是**基于 Nginx 服务器接收到的请求字符串(例如 server_name/url-string)，对虚拟主机名称 (也可以是 IP 别名)之外的字符串(例如 前面的 /url-string)进行匹配，对特定的请求进行处理**。地址定向、数据缓 存和应答控制等功能，还有许多第三方模块的配置也在这里进行。

```ini
# 全局块

#user  nobody;
# worker_processes表示nginx并发处理的值,越大并发越大
worker_processes  1;			

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


# events块

events {
		# worker_connections表示每个work process支持的最大连接数为 1024.
    worker_connections  1024;
}



# http块

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
    		# 虚拟主机监听的端口号
        listen       80;	
        # 主机名称
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
        
        # location 块
        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```

# 五、Nginx配置实例-反向代理

## 实例一

示例目标：使用 nginx 反向代理，访问 `www.123.com `直接跳转到 127.0.0.1:8080 （这里就以tomcat为例）

### 准备工作

```shell
# 下载 apache-tomcat-7.0.70.tar.gz, 解压
tar -xvf apache-tomcat-7.0.70.tar.gz
# 启动
./apache-tomcat-7.0.70/bin/startup.sh
# Using CATALINA_BASE:   /r#ot/projects/nginx_test/apache-tomcat-7.0.70
# Using CATALINA_HOME:   /root/projects/nginx_test/apache-tomcat-7.0.70
# Using CATALINA_TMPDIR: /root/projects/nginx_test/apache-tomcat-7.0.70/temp
# Using JRE_HOME:        /usr/lib/jvm/java-8-openjdk-amd64/
# Using CLASSPATH:       /root/projects/nginx_test/apache-tomcat-7.0.70/bin/bootstrap.jar:/root/projects/nginx_test/apache-tomcat-7.0.70/bin/tomcat-juli.jar
# Tomcat started.

# 查看日志
tail -f ./apache-tomcat-7.0.70/logs/catalina.out 
```

启动后访问地址IP:8080,查看运行情况:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728135449998.png" alt="image-20210728135449998" style="zoom:35%;" />

实现的大致流程：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728140124558.png" alt="image-20210728140124558" style="zoom:50%;" />

### 修改本地Host

Windows的host文件如下图所示：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728140337727.png" alt="image-20210728140337727" style="zoom:50%;" />

mac和linux都在`/etc/hosts`文件中

在配置文件末尾加上你的IP映射的域名:

格式例如：`47.103.203.133 www.123.com`

**如果有购买了域名，那么本地可以不设置，对应解析到IP即可**

（我已购买，此IP对应的域名就是aliyun.gumptlu.work）

### 修改配置文件

在http块下修改server如下所示：

```ini
server {
	# 因为我的80端口其他在使用，所以这里用90
  listen  90; 																
  server_name 47.103.203.133;
  
  location / { 
    root html;
    # 核心就是在此，代理转发到本地的8080端口
    proxy_pass http://127.0.0.1:8080;					
    index index.html index.htm;
  }
}
```

检查语法，重新启动nginx

```shell
nginx -t
nginx -s reload
```

查看效果：

(如果是修改本地Host以及80端口那么就是直接访问www.123.com)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728143844071.png" alt="image-20210728143844071" style="zoom:40%;" />

## 实例二

 实现效果:使用 nginx 反向代理，根据访问的**路径**跳转到不同端口的服务中

 nginx监听端口为90，

*  访问 http://IP:9001/edu/ 直接跳转到 127.0.0.1:8080

* 访问 http://IP:9001/vod/ 直接跳转到 127.0.0.1:8081

### 准备工作

准备两个tomcat服务器，一个8080（上一个案例部署）、一个8081

```shell
cp -r apache-tomcat-7.0.70 tomcat8081		# 复制一个8081
mv apache-tomcat-7.0.70 tomcat8080
```

![image-20210728144952498](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728144952498.png)

分别进入tomcat8080文件夹，启动tomcat

进入另一个8081，修改配置的端口：

`tomcat8081/conf/server.xml`

```xml
<Server port="8015" shutdown="SHUTDOWN">
  ...
  <Connector port="8081" protocol="HTTP/1.1"
             connectionTimeout="20000"
             redirectPort="8444" />
  ...
  <Connector port="8019" protocol="AJP/1.3" redirectPort="8444" />
  ...
```

启动tomcat8081

查看是否都已启动：

![image-20210728155654924](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728155654924.png)

分别在两个tomcat文件夹中添加两个页面文件

```shell
# tomcat8080
cd webapps 
mkdir edu && cd edu
vim index.html
# 添加如下内容：
# <h1>8080edu</h1>

# tomcat8081
cd webapps
mkdir vod && cd vod
vim index.html
# <h1>8081vod</h1>
```

测试一下：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728161344893.png" alt="image-20210728161344893" style="zoom:50%;" />

### 修改配置文件

`nginx.conf` 

```ini
server {
  listen  90;
  server_name 47.103.203.133;
  
  location ~/edu/ {
    proxy_pass http://127.0.0.1:8080;
  }
  location ~/vod/ {
    proxy_pass http://127.0.0.1:8081;
  }
}
```

访问测试：(前面的域名代替了服务器IP)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728162342320.png" alt="image-20210728162342320" style="zoom:50%;" />

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210728162357317.png" alt="image-20210728162357317" style="zoom:50%;" />



### location指令说明

该指令用于匹配 URL。 语法如下:

```shell
location [ = | ~ | ~* | ^~] url {

}
```

1. `=` :用于不含正则表达式的 url 前，要求请求字符串与 url 严格匹配，如果匹配成功，就停止继续向下搜索并立即处理该请求。

2. `~`:用于表示 url 包含正则表达式，并且区分大小写。

3. `~*`:用于表示 url 包含正则表达式，并且不区分大小写。

4. `^~`:用于不含正则表达式的 url 前，要求 Nginx 服务器找到标识 url 和请求字符串匹配度最高的 location 后，立即使用此location处理请求，而不再使用 location块中的正则 url 和请求字符串做匹配。

**注意: 如果 url 包含正则表达式，则必须要有 ~ 或者 ~* 标识。**

# 六、Nginx配置实例-负载均衡

目标/实现效果：浏览器输入地址http://IP/edu/index.html 实现负载均衡效果，将请求平均到8080和8081两个服务器上

## 准备工作

同样的准备两个tomcat服务器作为测试使用

在两个tomcat中都在`webapps`文件夹下创建`edu/index.html`文件 （做过反向代理的可以直接吧8080的edu文件夹复制到8081）

**为了区别起见，把8081的`indexl.html`文件内容改成`8081edu`**

## 修改配置

在`nginx.conf`文件中：

```ini
http {
	
	...
	
  # 负载均衡
  # 自定义名字，填写负载均衡的服务器列表weight代表权重
  upstream tomcatServer {
  server 127.0.0.1:8080 weight=1;
  server 127.0.0.1:8081 weight=1; 
  }

  ...

}
```

```ini
server {
  listen  9001;
  server_name 47.103.203.133;
   
  # 负载均衡
  # 名称要与自定义的一致
  location / {
   	proxy_pass http://tomcatServer;
   	index index.html index.htm;
  }
}
```

重启nginx

`nginx -t && nginx -s reload`

测试：

效果：不断访问`http://aliyun.gumptlu.work:9001/edu/index.html`，会随机出现`8081edu`和`8080edu`中的一个

因为是一样的权重，所以两者出现的概率是一样的大小

## 负载分配策略

### 1. 轮询(默认)

每个请求按时间顺序**逐一分配**到不同的后端服务器，如果后端服务器 down 掉，能自动剔除。 

### 2. weight权重

默认为1, 权重越高被分配的客户端越多

指定轮询几率，weight 和访问比率成正比，**用于后端服务器性能不均**的情况。 即上面的方式

### 3. IP_Hash

每个请求按访问 ip 的 hash 结果分配，这样**每个访客固定访问一个后端服务器**，可以**解决 session 的问题**。 例如:

```ini
upstream tomcatServer {
	ip_hash;
	server 127.0.0.1:8080 weight=1;
	server 127.0.0.1:8081 weight=1; 
}
```



> <font color='#39b54a'>根据客户端的IP取Hash将其与访问的服务器进行绑定，让一个客户端随后的访问都固定在这个服务器上</font>

### 4. Fair

按后端**服务器的响应时间来分配请求**，响应时间**短的优先分配**。

```ini
upstream tomcatServer {
	server 127.0.0.1:8080 weight=1;
	server 127.0.0.1:8081 weight=1; 
	fair;
}
```

# 七、Nginx配置实例-动静分离

动静分离：严格意义上说应该是**动态请求跟静态请求分开**，可以理解成使用 Nginx 处理静态页面，Tomcat 处理动态页面, 一般有两种方式：

* 一种是纯粹把静态文件独立成单独的域名，放在独立的服务器上，也是目前主流推崇的方案;

* 另外一种方法就是动态跟静态文件混合在一起发布，通过 nginx 来分开

通过 location 指定不同的后缀名实现不同的请求转发。

通过 expires 参数设置，可以使 浏览器缓存过期时间，减少与服务器之前的请求和流量。

具体 Expires 定义:是给一个资 源设定一个过期时间，也就是说无需去服务端验证，直接通过浏览器自身确认是否过期即可， 所以不会产生额外的流量。此种方法非常适合不经常变动的资源。(如果经常更新的文件， 不建议使用 Expires 来缓存)

> <font color='#39b54a'>如设置3d, 表示在这 3 天之内访问这个 URL，发送 一个请求，比对服务器该文件最后更新时间没有变化，则不会从服务器抓取，返回状态码 304，如果有修改，则直接从服务器重新下载，返回状态码 200。</font>

## 7.1 准备工作

创建一个静态资源文件夹`stable_data`：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803150120122.png" alt="image-20210803150120122" style="zoom:67%;" />

a.html:

```html
<h1>我是静态文件Html</h1>
```

## 7.2 配置

```ini
# 将nginx的用户设置为root
user root

http {
	server {
    listen  9001;
    server_name 47.103.203.133;
    # 静态资源的绝对根目录
    root /root/projects/nginx_test/stable_data;
    # 指定字符集防止乱码
	  charset gbk,utf-8;
    # 配置两个静态资源
    location /www/ {
			# 自动索引目录
      autoindex on; 
      # 显示出文件的确切大小，单位是bytes
      autoindex_exact_size on; 
      # 改为on后，显示的文件时间为文件的服务器时间
      autoindex_localtime on; 
      # 防止文件名乱码
      charset utf-8;
    }

    location /image/ {
      # 是否自动索引  
      autoindex on;
       # 显示出文件的确切大小，单位是bytes
      autoindex_exact_size on; 
      # 改为on后，显示的文件时间为文件的服务器时间
      autoindex_localtime on; 
    }
	}
}

```

重启nginx

测试:

访问`http://aliyun.gumptlu.work:9001/image/`可以看到自动索引的效果：

（你需要把上面的域名换成你的nginx服务器的IP）

![image-20210803145945729](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803145945729.png)

> 出现错误：**403 forbid**
>
> 日志报错："/root/projects/nginx_test/stable_data/www/index.html" failed (13: **Permission denied**)
>
> 原因：新创建的文件夹`/root/projects/nginx_test/stable_data`没有给访问权限或者是centos系统的selinux没有关闭或者nginx的用户设置为root
>
> 解决方案：检查配置文件`user root`是否有，对于自己创建的静态文件夹添加权限`chmod -R 777 stable_data`
>
> https://blog.csdn.net/taokeng/article/details/103705342

访问index

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803150152900.png" alt="image-20210803150152900" style="zoom:67%;" />

> <font color='#e54d42'>注意： 整个过程只用到了nginx与tomcat等其他后端服务无关，当删除文件后资源也就无法访问</font>

# 八、配置高可用Nginx集群

当单台nginx服务器宕机后，整个网络集群服务都会失效，所以需要高可用的配置：

![image-20210803151107276](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803151107276.png)

解决办法：多台nginx服务器

**对外暴露一个虚拟IP，使用Keepalived实现宕机检测, 一旦宕机就将虚拟IP绑定到热备Nginx服务器上**

![image-20210803151403263](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803151403263.png)



## 准备工作

配备两台Nginx服务器：

`10.211.55.3` 和 `10.211.55.4`分别安装好Nginx，访问对应的80端口，查看是否正常

在两台机子上都安装`keepalived`

Ubuntu: `sudo apt install keepalived`

Centos: `yum install keepalived -y`

## 配置(主从配置)

对于Keeplived的配置文件位置在`/etc/keepalived/keepalived.conf`中，如果没有则手动创建

```yaml
# 全局配置
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 10.211.55.3	# 你的IP
   smtp_connect_timeout 30
   router_id LVS_DEVEL		# 访问到主机，一般写主机IP或者对应的域名
}

# 检测脚本配置
vrrp_script chk_http_port {   # 检测脚本
	script "/usr/local/src/nginx_check.sh" 	# 脚本路径，注意改成你的路径
	interval 2 	#	(检测脚本执行的间隔) 
	weight -20		# 当脚本中的条件成立，当前主机权重的变换. 例如主服务器宕机则权重减少20
}

# 虚拟IP的配置
vrrp_instance VI_1 {
  state MASTER # 备份服务器上将 MASTER 改为 BACKUP
  interface eth0 # 网卡名称，根据主机的名称修改
  virtual_router_id 51 # 主、备机的 virtual_router_id 必须相同 
  priority 100 # 主、备机取不同的优先级，主机值较大，备份机值较小 
  advert_int 1	# 心跳检测时间间隔, 单位秒
  authentication {	# 密码验证权限
  	auth_type PASS
  	auth_pass 1111
  }
   virtual_ipaddress {
		10.211.55.10 # VRRP H 虚拟的IP地址 注意在一个网段, 也可以绑定多个
	} 
}
```

修改配置文件权限：

`chmod 644 keepalived.conf`

脚本文件`nginx_check.sh`内容：

Centos：

```shell
#!/bin/bash
A=`ps -C nginx –no-header |wc -l` 
if [ $A -eq 0 ];then
	/usr/local/nginx/sbin/nginx
	sleep 2
	if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
      killall keepalived
  fi
fi
```

Ubuntu：

```shell
#!/bin/bash
A=`ps -C nginx –no-header |wc -l` 
if [ $A -eq 0 ];then
	/usr/sbin/nginx
	sleep 2
	if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
      killall keepalived
  fi
fi
```

```shell
# 添加可执行权限
chmod +x nginx_check.ch
```

脚本的逻辑就是每两秒检测一下你的nginx是否启动，不同的系统nginx启动程序位置不同, 如果检测到nginx启动失败则关闭所有的keepalived

同样的对另一台机器做同样的操作（记得修改IP和BACKUP）

启动两台机器的nginx和keepalived

nginx之前已经启动，启动keepalived：

```shell
sudo systemctl start keepalived.service 
sudo systemctl status keepalived.service 	# 查看启动状态
```

主Nginx服务器显示：

![image-20210804143846068](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804143846068.png)

`ip a`查看ip地址绑定情况：

![image-20210804150259338](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804150259338.png)

从Nginx服务器：

![image-20210804144004427](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804144004427.png)

这样就代表启动成功！

> 错误：keepalived启动失败，`journalctl -u keepalived.service`查看错误日志：
>
> Configuration file '/etc/keepalived/keepalived.conf' is not a regular non-executable file
>
> 原因：conf配置文件权限不对
>
> 解决：`chmod 644 keepalived.conf` 然后重新启动

## 测试

预期效果： 访问虚拟IP，应该显示主Nginx服务器，停止主Nginx服务器，再访问虚拟IP应该显示从Nginx服务器

1. 访问虚拟IP地址：

   ![image-20210804144454014](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804144454014.png)



2. 停止主服务器

   ```shell
   sudo nginx -s stop
   ps -ef | grep nginx
   sudo systemctl stop keepalived.service
   sudo systemctl status keepalived.service 
   ```

   再次访问, 依旧可用！:

   ![image-20210804144857169](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804144857169.png)

以上就是简单的高可用nginx服务器配置实验，对于nginx后面反向代理的所有应用服务器也可以配置负载均衡，让整体的服务更加高可用





# 九、Nginx原理浅析与高级优化配置

## 9.1 原理

### master与work进程

启动nginx，观察一下进程你会发现会有一个master和多个work进程，如图：

![5psUde](http://xwjpics.gumptlu.work/qinniu_uPic/5psUde.png)

nginx实际运行的进程结构如图：

![image-20210804151040206](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804151040206.png)



当客户端访问master后，nginx采用多个work争抢的方式去提供服务，每个work可能连接着其他服务例如java的tomcat：





![NOy2By](http://xwjpics.gumptlu.work/qinniu_uPic/NOy2By.png)

**master-workers 的机制的好处**

* 方便`nginx -s reload`重启时的热部署

  当上一个配置链接了第一个work，重新启动后其他work链接新的配置争抢提供最新的服务，老work随后也会更新，方便热部署**服务不中断**

* 不需要锁机制

  对于每个 worker 进程来说，独立的进程，不需要加锁，所以省掉了锁带来的开销， 同时在编程以及问题查找时，也会方便很多。

* 降低风险

  采用独立的进程，可以让互相之间不会 影响，一个进程退出后，其它进程还在工作，服务不会中断，master 进程则很快启动新的 worker 进程。当然，worker 进程的异常退出，肯定是程序有 bug 了，异常退出，会导致当 前 worker 上的所有请求失败，不过不会影响到所有请求，所以降低了风险。

**需要设置多少个 worker**

Nginx 同 redis 类似都采用了 io 多路复用机制，每个 worker 都是一个独立的进程，但每个进 程里只有一个主线程，通过**异步非阻塞**的方式来处理请求， 即使是千上万个请求也不在话 下。每个 worker 的线程可以把一个 cpu 的性能发挥到极致。所以 **worker 数和服务器的 cpu 数相等**是最为适宜的。设少了会浪费 cpu，设多了会造成 cpu 频繁切换上下文带来的损耗。

**连接数 worker_connection**

> 发起一个请求一个work需要多少多少连接数？
>
> 答案：2个或者4个
>
> 解释：当访问静态资源时，nginx的work本身即可处理这样的请求，所以客户端到work一个work返回数据到客户端一个，总共两个；当访问的服务需要借助其他服务的时候(也即使用**反向代理**)，就可能是四个了，例如nginx不支持java，那么对数据库的操作时就要多两个链接：work和tomcat，tomcat返回work，所以也可能是四个

> work最大并发数的计算：
>
> 例如四个work，每个work的最大可连接数是1024，总共可连接4*1024个连接
>
> 那么并发数就是`总连接数/2或4`

这个值是表示每个 worker 进程所能建立连接的最大值，所以，一个 nginx 能建立的最大连接数，应该是 `worker_connections * worker_processes`。当然，这里说的是最大连接数，对于 HTTP 请求本地资源来说，能够支持的最大并发数量是 `worker_connections * worker_processes`，如果是支持 http1.1 的浏览器每次访问要占两个连接，所以普通的静态访问最大并发数是:` worker_connections * worker_processes /2`，而如果是 HTTP 作为反向代理来说，**最大并发数量**应该是`worker_connections * worker_processes/4`。因为作为反向代理服务器，每个并发会建立与客户端的连接和与后端服务的连接，会占用两个连接。

## 9.2 优化配置

![image-20210804153130048](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210804153130048.png)

### 基本配置补充

选择需要的配置添加即可

```yaml
########### 每个指令必须有分号结束。#################
user administrator administrators;  #配置用户或者组作为nginx启动用户，一般不建议root。
worker_processes 2;  #允许生成的work进程数，默认为1，一般与cpu内核数量一致
worker_cpu_affinity 0001 0010 0100 1000;　　#此项配置为开启多核CPU，对你先弄提升性能有很大帮助, nginx默认是不开启的,1为开启，0为关闭，因此先开启第一个倒过来写，
# 第一位0001（关闭第四个、关闭第三个、关闭第二个、开启第一个）
# 第二位0010（关闭第四个、关闭第三个、开启第二个、关闭第一个）
# 第三位0100（关闭第四个、开启第三个、关闭第二个、关闭第一个）
# 后面的依次类推, 那么如果是16核或者8核cpu，就注意为00000001、00000010、00000100，总位数与cpu核数一样。
pid /nginx/pid/nginx.pid;   #指定nginx进程运行文件存放地址
error_log log/error.log debug;  #制定日志路径，级别。这个设置可以放入全局块，http块，server块，级别可以为：debug|info|notice|warn|error|crit|alert|emerg
worker_rlimit_nofile 65535;　　　　#这个值为nginx的worker进程打开的最大文件数，如果不配置，会读取服务器内核参数（通过ulimit -a查看），如果内核的值设置太低会让nginx报错（too many open file），但是在此设置后，就会读取自己配置的参数不去读取内核参数

events {
		use epoll;　　　　#事件驱动模型, 客户端线程轮询方法、内核2.6版本以上的建议使用，可选项：epoll,select|poll|kqueue|epoll|resig|/dev/poll|eventport
    accept_mutex on;   	#设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;  	#设置一个进程是否同时接受多个网络连接，默认为off
    worker_connections  1024;    #设置一个work的最大连接数，默认为512
}
http {
    include       mime.types;   	#文件扩展名与文件类型映射表
    default_type  application/octet-stream; #默认文件类型，默认为text/plain
    server_tokens  off;　　　　#为错误页面上的nginx版本信息，建议关闭，提升安全性
    server_names_hash_bucket_size 128;	# 一些大小限制
    client_header_buffer_size 32k;
    large_client_header_buffers 4 32k;
    client_max_body_size 8m;
    access_log off; 			#取消服务日志    
    log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义日志格式
    access_log log/access.log myFormat;  #combined为日志格式的默认值
    tcp_nopush     on;　　#告诉nginx在数据包中发送所有头文件，而不是一个一个的发
    tcp_nodelay    on;		# 防止网络堵塞
    sendfile on;   #允许sendfile方式传输文件,sendfile可以再磁盘和tcp socket之间互相copy数据，默认为off，可以在http块，server块，location块。
    sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    keepalive_timeout 65;  #连接超时时间，默认为75s，可以在http，server，location块。
    error_page 404 https://www.baidu.com; #错误页404指定
    autoindex on; #开启目录列表访问，适合下载服务器，默认关闭。
		
		# 反向代理相关，location中也可设置
		proxy_intercept_errors on;
		proxy_redirect off;
		proxy_set_header X-Real-IP $remote_addr;
  	#后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
		#以下是一些反向代理的配置，可选。
    proxy_set_header Host $host;
    client_max_body_size 10m; #允许客户端请求的最大单文件字节数
    client_body_buffer_size 128k; #缓冲区代理缓冲用户端请求的最大字节数，
    proxy_connect_timeout 90; #nginx跟后端服务器连接超时时间(代理连接超时)
    proxy_send_timeout 90; #后端服务器数据回传时间(代理发送超时)
    proxy_read_timeout 90; #连接成功后，后端服务器响应时间(代理接收超时)
    proxy_buffer_size 4k; #设置代理服务器（nginx）保存用户头信息的缓冲区大小
    proxy_buffers 4 32k; #proxy_buffers缓冲区，网页平均在32k以下的设置
    proxy_busy_buffers_size 64k; #高负荷下缓冲大小（proxy_buffers*2）
    proxy_temp_file_write_size 64k; #设定缓存文件夹大小，大于这个值，将从upstream服务器传
    
    #FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
    fastcgi_intercept_errors on;
    fastcgi_connect_timeout 1300;
    fastcgi_send_timeout 1300;
    fastcgi_read_timeout 1300;
    fastcgi_buffer_size 512k;
    fastcgi_buffers 4 512k;
    fastcgi_busy_buffers_size 512k;
    fastcgi_temp_file_write_size 512k;

		#gzip是告诉nginx采用gzip后的数据来传输文件，会大量减少我们的发数据的量
    #gzip模块设置
		gzip on; #开启gzip压缩输出
		gzip_min_length 1k; #最小压缩文件大小
		gzip_buffers 4 16k; #压缩缓冲区
		gzip_http_version 1.0; #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
		gzip_comp_level 2; #压缩等级
		gzip_types text/plain application/x-javascript text/css application/xml;
		#压缩类型，默认就已经包含text/html，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
		gzip_vary on;
		limit_zone crawler $binary_remote_addr 10m; #开启限制IP连接数的时候需要使用
		
		
		# 负载均衡
    upstream mysvr {   
      server 127.0.0.1:7878;
      server 192.168.10.121:3333 backup;  #热备
    }

    server {
        keepalive_requests 120; #单连接请求上限次数。
        listen       4545;   #监听端口
        server_name  127.0.0.1;   #监听地址,IP或者域名       
        location  ~*^.+$ {       #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
           #root path;  #根目录
           #index vv.txt;  #设置默认页
           proxy_pass  http://mysvr;  #请求转向mysvr 定义的服务器列表
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip   
           expires 10d;   # 静态资源缓存时间
        } 
    }
}
```

### nginx配置https

```yaml
#server端基本配置
ssl_certificate      ssl/ca/io.com.pem;　　　　#这个为购买的https证书，供应商会生成
ssl_certificate_key  ssl/ca/io.com.key;
ssl_session_timeout  5m;
ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
#启用TLS1.1、TLS1.2要求OpenSSL1.0.1及以上版本，若您的OpenSSL版本低于要求，请使用 ssl_protocols TLSv1;
ssl_ciphers  HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM;
ssl_prefer_server_ciphers   on;

server {
    listen 80;
    listen 443 ssl spdy;
    server_name io.123.com;
    include      ssl/io.com;　　　　　　#注意看下一个文件
    location / {
        proxy_pass http://lb_io;
        if ($scheme = http ) {
        	return 301 https://$host$request_uri;　　　　#此项配置为转换为https的基本配置
        }
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
 
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
 
    access_log /data/logs/nginx/access/niuaero.log main;
}
```

### nginx配置反爬虫

```php
#以下内容添加nginx虚拟主机配置location里，proxypass之后
if ($http_user_agent ~* (Scrapy|Curl|HttpClient)) { 
     return 403; 
} 
 
#禁止指定UA及UA为空的访问 
if ($http_user_agent ~ "WinHttp|WebZIP|FetchURL|node-superagent|java/|FeedDemon|Jullo|JikeSpider|Indy Library|Alexa Toolbar|AskTbFXTV|AhrefsBot|CrawlDaddy|Java|Feedly|Apache-HttpAsyncClient|UniversalFeedParser|ApacheBench|Microsoft URL Control|Swiftbot|ZmEu|oBot|jaunty|Python-urllib|lightDeckReports Bot|YYSpider|DigExt|HttpClient|MJ12bot|heritrix|EasouSpider|Ezooms|BOT/0.1|YandexBot|FlightDeckReports|Linguee Bot|^$" ) { 
     return 403;              
} 
 
#禁止非GET|HEAD|POST方式的抓取 
if ($request_method !~ ^(GET|HEAD|POST)$) { 
    return 403; 
} 
```

### 高并发参数

如果是高并发架构，需要在nginx的服务器上添加如下的内核参数

这些参数追加到`/etc/sysctl.conf`,然后执行`sysctl -p` 生效。

```yaml
#每个网络接口接收数据包速度比内核处理速度快的时候，允许发送队列数目数据包的最大数
net.core.netdev_max_backlog = 262144

#调节系统同时发起的tcp连接数
net.core.somaxconn = 262144

#该参数用于设定系统中最多允许存在多少TCP套接字不被关联到任何一个用户文件句柄上，主要目的为防止Ddos攻击
net.ipv4.tcp_max_orphans = 262144

#该参数用于记录尚未收到客户端确认信息的连接请求的最大值
net.ipv4.tcp_max_syn_backlog = 262144

#nginx服务上建议关闭（既为0）
net.ipv4.tcp_timestamps = 0

#该参数用于设置内核放弃TCP连接之前向客户端发送SYN+ACK包的数量，为了建立对端的连接服务，服务器和客户端需要进行三次握手，第二次握手期间，内核需要发送SYN并附带一个回应前一个SYN的ACK，这个参数主要影响这个过程，一般赋予值为1，即内核放弃连接之前发送一次SYN＋ACK包。
*net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1*
```

# Tips

### 启动出现：nginx: [error] open() “/run/nginx.pid” failed (2: No such file or directory)

```shell
nginx -c [你的nginx.conf文件的绝对路径]
nginx -s reload
```
