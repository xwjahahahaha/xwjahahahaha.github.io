---
title: go语言gob序列化指针无法寻址报错
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-09-11 15:28:18
---

# 一、错误描述

使用gob序列化时出现报错：` gob: unaddressable value of type *big.Float`

场景：

序列化一个数据结构中包含了一个map字段，其中value为`big.Float`这种结构体类型

```go
type SideBlock {
  ...
  Varphi map[string]big.Float
  ...
}

// 序列化

func (b *SideBlock) Serialization() []byte {
	var res bytes.Buffer
	encoder := gob.NewEncoder(&res)
	err := encoder.Encode(b)
	if err != nil {
		logger.Error("Serialization err : ", err)
	}
	return res.Bytes()
}
```

<!-- more -->

go版本： `go version go1.16.5 darwin/arm64`

OS: `Mac OS BigSur 11.5.2`

# 二、错误原因

在[issue](https://github.com/golang/go/issues/32251)中找到了答案：

他是使用`url.URL`作为Map的键出现了问题

问题在于`marshalBinary`的指针接收器，如果想使用该结构体作为key或value，那么我们需要自己实现一下特殊字段的`MarshalBinary`方法

> <font color='#e54d42'>个人理解： 当一个结构体的某个Map字段中有另一个结构体而不是改结构体的指针，那么序列化时就是该结构体的副本而不是本身，所以难以寻址</font>

# 三、解决方法

方法一：

https://play.golang.org/p/dK7SsqahtAd

```go
package main

import (
	"encoding/gob"
	"fmt"
	"io/ioutil"
)

type T struct {
}

func (t *T) String() string { return "" }

func (t T) MarshalBinary() (text []byte, err error) {
	return []byte(t.String()), nil
}

func main() {
	m := make(map[T]int)
	m[T{}] = 0
	e := gob.NewEncoder(ioutil.Discard)
	fmt.Println(e.Encode(m))
}
```

方法二：

把`big.Float`变为指针即字段变为`Varphi map[string]*big.Float`
