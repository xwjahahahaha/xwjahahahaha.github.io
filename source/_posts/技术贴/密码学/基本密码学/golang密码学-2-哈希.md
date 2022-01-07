---
title: golang密码学-2-hash
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2020-12-23 21:53:22
---

# 一、Hash算法实现

```go
// isHex bool 判断当前的字符串是不是16进制的字符串，还是直接的字符串
func HashString(text string, method string, isHex bool, times int) string {
	if times < 1{
		log.Panic("Must run more than once!")
	}
	var hashInstance hash.Hash
	switch strings.ToUpper(method) {
	case "MD4":
		hashInstance = md4.New()						//创建MD4实例
	case "MD5":
		hashInstance = md5.New()  						//创建MD5实例
	case "SHA1":
		hashInstance = sha1.New()						//创建sha1实例
	case "RIPEMD160":
		hashInstance = ripemd160.New()
	case "SHA256":
		hashInstance = sha256.New()
	case "SHA512":
		hashInstance = sha512.New()
	default:
		log.Panic("not have this hash method!")
	}
	var bytes []byte
	if isHex{											//如果是16进制的字符串，则需要decode
		decoBytes, err := hex.DecodeString(text)		//把16进制字符串转换为字节数组
		if err != nil {
			log.Panic("hex.DecodeString err = ", err)
		}
		bytes = decoBytes[:]
	}else {
		bytes = []byte(text)[:]
	}
	hashInstance.Write(bytes)     			//输入
	bytes = hashInstance.Sum(nil) 		//输出结果追加到参数的后面，参数设置为nil就是不追加
	//多次调用
	for i:=0; i<times-1; i++{
		hashInstance.Reset()				//resets the Hash to its initial state. 变为初始状态
		hashInstance.Write(bytes)			//将上一次的输出再次输入，注意：第一次之后的输出都是16进制字符串，所以后面的都是用hex方式
		bytes = hashInstance.Sum(nil)	//再次求一次
	}
	return fmt.Sprintf("%x", bytes)	//转换为16进制的字符串
}
```

<!-- more -->

对应的测试：

```go
func TestMd4(t *testing.T)  {
	const input = "123456"
	const expectOutputHex = "9aae92f9cb8ff3744c51e7f621d21096"
	const expectOutputStr = "585028aa0f794af812ee3be8804eb14a"
	//测试普通字符串
	res := Hash.HashString(input, "md4", false)
	if res == expectOutputStr{
		t.Log("bingo")
	}else {
		t.Fatal("calculate err ， err result = ", res)
	}
	//测试16进制字符串
	res = Hash.HashString(input, "md4", true)
	if res == expectOutputHex{
		t.Log("bingo")
	}else {
		t.Fatal("calculate err ， err result = ", res)
	}
}
```

# Tips

# 1.requested hash function #9 is unavailable

使用密码的时候，提示类似如上的函数标号错误，网上找的方法在引包前面加上_都不管用

我的解决方法：因为直接下载新的包需要翻墙，所以就去github上手动下载

地址：https://github.com/golang/crypto

解压，然后放在这个目录`$GOPATH/src/goalng.org/crypto`**注意，就放在这个目录，如果没有该文件夹就创建**

然后引用就可以了