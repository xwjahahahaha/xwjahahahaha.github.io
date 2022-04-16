---
title: looklook开发重点记录
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-10 17:59:42
---

# 一、整体架构

项目架构

![gozerolooklook](http://xwjpics.gumptlu.work/qinniu_uPic/gozerolooklook.png)

<!-- more -->

业务架构

![gozerolooklook](http://xwjpics.gumptlu.work/qinniu_uPic/go-zero-looklook-service.png)

# 二、组件/技术栈

- k8s
- go-zero
- nginx网关
- filebeat
- kafka
- go-stash
- elasticsearch
- kibana
- prometheus
- grafana
- jaeger
- go-queue
- asynq
- asynqmon
- dtm
- docker
- docker-compose
- mysql
- redis
- modd
  - 功能：用于修改配置文件时无需重启服务，自动帮我更新
- jenkins
- gitlab
- harbor

# 三、网关设计/鉴权服务

使用`nginx`作为统一网关，将鉴权功能放在`nginx`:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410181046981.png" alt="image-20220410181046981" style="zoom: 33%;" />

使用`auth_request`模块作为统一鉴权，在API层次就不再鉴权(如果是资产相关的业务则会在API做第二次鉴权，保证一个安全性)

```json
server{
    listen 8081;
    access_log /var/log/nginx/looklook.com_access.log;
    error_log /var/log/nginx/looklook.com_error.log;

    location /auth {
	    internal;
        proxy_set_header X-Original-URI $request_uri;
	    proxy_pass_request_body off;
	    proxy_set_header Content-Length "";
	    proxy_pass http://looklook:8001/identity/v1/verify/token;
    }

    location ~ /usercenter/ {
       auth_request /auth;
       auth_request_set $user $upstream_http_x_user;
       proxy_set_header x-user $user;

       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header REMOTE-HOST $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_pass http://looklook:8002;
   }

   location ~ /travel/ {
       auth_request /auth;
       auth_request_set $user $upstream_http_x_user;
       proxy_set_header x-user $user;

       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header REMOTE-HOST $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_pass http://looklook:8003;
   }


    location ~ /order/ {
       auth_request /auth;
       auth_request_set $user $upstream_http_x_user;
       proxy_set_header x-user $user;

       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header REMOTE-HOST $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_pass http://looklook:8004;
   }

    location ~ /payment/ {
       auth_request /auth;
       auth_request_set $user $upstream_http_x_user;
       proxy_set_header x-user $user;

       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header REMOTE-HOST $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_pass http://looklook:8005;
   }
}
```

* 在需要JWT鉴权的服务中，我们可以通过JWT的token传递关键的值

  例如查询用户的信息：` http://127.0.0.1:8888/usercenter/v1/user/detail`，请求的流程如下：

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410203105305.png" alt="image-20220410203105305" style="zoom:50%;" />

  具体的获取方法如下：

  ```go
  func (l *DetailLogic) Detail(req types.UserInfoReq) (*types.UserInfoResp, error) {
  	userId := ctxdata.GetUidFromCtx(l.ctx)
  	......
  }
  ```

* 整体流程如下：

  ![image-20220410212503402](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410212503402.png)

