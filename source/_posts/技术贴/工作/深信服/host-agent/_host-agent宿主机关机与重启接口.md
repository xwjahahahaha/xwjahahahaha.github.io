---
title: _host-agent宿主机关机与重启接口
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-02-16 14:37:05
---

# 一、需求

* `acloud-keeper`服务 

  `acloud-keeper`运行与宿主机中的服务，在620_ARM版本中，此服务主要作用宿主机与容器之间的信息交互

* `host-agent`服务	

  `host-agent`运行在宿主机中的服务，在680版本中，其主要功能如下：

  1. 接替`acloud-keeper`具有的功能
  2. 屏蔽X86和ARM，以及不同OS之间的差异

* `container-cmd-proxy`服务	

  `container-cmd-proxy`运行在VT容器中的服务，在680版本中，主要用于充当`host-agent`的服务端，给`host-agent`提供内部api接口

<!-- more -->

`host-agent`

| 功能函数或api                  | 大致实现                                           | 使用点                                     | 680                      |
| ------------------------------ | -------------------------------------------------- | ------------------------------------------ | ------------------------ |
| /host-agent/v1/system/shutdown | 通过http通信host-agent,调用宿主机shutdown命令关机  | HCI关闭（安全关机r）                       | 新增                     |
| /host-agent/v1/system/poweroff | 通过http通信host-agent,调用宿主机poweroff命令关机  | HCI断开电源（强制关机）                    | 新增                     |
| /host-agent/v1/system/reboot   | 通过http通信host-agent,调用宿主机reboot命令重启    | HCI重启                                    | 新增                     |
| pullup_acloud                  | 通过操作host-agent服务来对容器进行启动与停止等操作 | 用于宿主机对容器的拉起操作                 | 删除，由容器管理服务处理 |
| rescue_with_operate_docker     | 定时检测3次容器都未拉起时，进入容器的兜底逻辑      | 容器兜底机制                               | 具体实现更改             |
| acloud-monitor                 | 每分钟对容器进行检测，判断容器是否正常             | 定时检测，容器不正常进入兜底机制，便于排障 | 维持                     |
| sync_network_profile           | 每分钟同步容器的网络配置                           | 用于兜底逻辑的使用                         | 维持                     |
| sync_acloud_passwd             | 每分钟同步HCI密码                                  | 用于兜底逻辑的使用                         | 维持                     |

# 二、关机逻辑

![image-20220216150021540](http://xwjpics.gumptlu.work/image-20220216150021540.png)


​	





​	




​	
​			


​		
