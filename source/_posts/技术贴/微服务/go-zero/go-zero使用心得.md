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

# 一、api配置文件含义

## 1. from与json区别

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



