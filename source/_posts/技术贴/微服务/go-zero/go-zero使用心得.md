---
title: go-zero使用心得
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2023-02-10 15:33:39
---

[TOC]


<!-- more -->

> 参考：
>
> * 

# 一、基础

## 1. api配置文件含义

API语法介绍：https://go-zero.dev/cn/docs/design/grammar/

### from与json区别

在编写接口的api文件时，其中`from`与`json`有不同的含义

如果是get类型的接口，那么Req结构体的字段就必须是from，如果是post那么就应该用json

```go
RuleGetOneReq {
  RuleId int64 `from:"rule_id"`
}
RuleGetOneResp {
  Rule Rule `json:"rule"`
}


@doc "查询一个规则"
@handler getOneRule
get /getOneRule(RuleGetOneReq) returns (RuleGetOneResp)
```

| tag key | 描述                                                         | 提供方  | 有效范围          | 示例            |
| ------- | ------------------------------------------------------------ | ------- | ----------------- | --------------- |
| json    | json序列化tag                                                | golang  | request、response | `json:"fooo"`   |
| path    | 路由path，如`/foo/:id`                                       | go-zero | request           | `path:"id"`     |
| form    | 标志请求体是一个form（POST方法时）或者一个query(GET方法时`/search?name=keyword`) | go-zero | request           | `form:"name"`   |
| header  | HTTP header，如 `Name: value`                                | go-zero | request           | `header:"name"` |

## 2. createTime、updateTime

根据文档：

> https://go-zero.dev/cn/docs/goctl/model#%E7%94%9F%E6%88%90%E8%A7%84%E5%88%99
>
> 默认规则
>
> 我们默认用户在建表时会创建createTime、updateTime字段(忽略大小写、下划线命名风格)且默认值均为`CURRENT_TIMESTAMP`，而updateTime支持`ON UPDATE CURRENT_TIMESTAMP`，对于这两个字段生成`insert`、`update`时会被移除，不在赋值范畴内，当然，如果你不需要这两个字段那也无大碍

所以我们只要在定义数据库Sql的时候定义这两个字段就可以了，在golang代码中无需赋值这两个字段，例如：

```sql
`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
```

## 3.  如何添加更多的CURD操作

默认`goctl model`生成的CURD代码可能无法满足需求，那
