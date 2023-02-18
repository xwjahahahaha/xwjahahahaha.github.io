---
title: Go坑与注意点
tags:
  - golang
categories:
  - technical
  - Golang
toc: true
declare: true
date: 2021-03-05 20:30:01
---

# 一、基础语法陷阱

<!-- more -->

## append陷阱

```go
	var array =[]int{1,2,3,4,5}// len:5,capacity:5
	var newArray=array[1:3]// len:2,capacity:4   (已经使用了两个位置，所以还空两位置可以append)
	fmt.Printf("%p\n",array) //0xc420098000
	fmt.Printf("%p\n",newArray) //0xc420098008 可以看到newArray的地址指向的是array[1]的地址，即他们底层使用的还是一个数组
	fmt.Printf("%v\n",array) //[1 2 3 4 5]
	fmt.Printf("%v\n",newArray) //[2 3]

	newArray[1]=9 //更改后array、newArray都改变了
	fmt.Printf("%v\n",array) // [1 2 9 4 5]
	fmt.Printf("%v\n",newArray) // [2 9]
	
	// 重点1 : 对切片append可能会导致切片引用的数组改变
	newArray=append(newArray,11,12)//append 操作之后，array的len和capacity不变,newArray的len变为4，capacity：4。因为这是对newArray的操作
	fmt.Printf("%v\n",array) //[1 2 9 11 12] //注意对newArray做append操作之后，array[3],array[4]的值也发生了改变
	fmt.Printf("%v\n",newArray) //[2 9 11 12]

	// 重点2 : append扩容可能会导致原底层数组的改变! 
	newArray=append(newArray,13,14) // 因为newArray的len已经等于capacity，所以再次append就会超过capacity值，
	// 此时，append函数内部会创建一个新的底层数组（是一个扩容过的数组），并将array指向的底层数组拷贝过去，然后在追加新的值。
	fmt.Printf("%p\n",array) //0xc420098000
	fmt.Printf("%p\n",newArray) //0xc4200a0000
	fmt.Printf("%v\n",array) //[1 2 9 11 12]
	fmt.Printf("%v\n",newArray) //[2 9 11 12 13 14]  他两已经不再是指向同一个底层数组y了
```

==**append操作可能会导致原本使用同一个底层数组的两个Slice变量变为使用不同的底层数组。**==

==**所以, 切片在作为函数传参时,要注意在函数中不能append越界**==

## 相对路径问题

### 二进制运行文件

go编译好的二进制文件的相对目录是根据**==当前执行路径==**

os.Getwd() 获取的是运行时你当前所在的路径

比如在/etc 目录下 运行/usr/main文件  最终通过Getwd获取到的值是  /etc
golang中的相对路径就是根据这个执行路径来相对的

### 文件操作Path的相对路径

当文件操作类的代码需要Path时，填写相对路径要==相对于工程的根目录而不是当前目录==

```go
//    ./是你当前的工程目录，并不是该go文件所对应的目录。
//    比如myProject/src/main/main.go
//    main.go里使用./,其路径不是myProject/src/main/，而是myProject/
 
//    下面是一个测试，上面的注释是对应的文件被创建到的目录
 
 
//E:\文档\Go\tire_api\A.txt
os.Create("./A.txt")
 
//E:\文档\Go\tire_api\src\A.txt
os.Create("./src/A.txt")
 
//E:\文档\Go\tire_api\src\main\A.txt
os.Create("./src/main/A.txt")
 
//E:\文档\Go\tire_api\src\main\log\A.txt
os.Create("./src/main/log/A.txt")
```

# 二、重点语法理解再记录

## 1. omitempty

https://blog.csdn.net/Edu_enth/article/details/124007406

具体的一般会在结构体字段的后面描述这个字段的信息，让struct与json解耦，不管go中字段的名字如何变化，只要后面json的名字不变，那么就无需改动：

```go
AAA {
  Name   string  `json:"name"`
  Desc   string  `json:"desc"`
  UserId int64   `json:"user_id"`
  Rules  []Rule `json:"omitempty,rules"`
}
```

添加`omitempty`的作用不是让Json -> 结构体时可以省略这个字段，而是当结构体->Json时如果没有此字段则省略

此时请求的Json如果是如下：

```json
{
    "name": "sg-001",
    "desc": "安全组1",
    "user_id": 111
}
```

那么会报错：

```shell
error: type mismatch for field rules
```

那么如果想让Json->结构体时，一个字段不传递就默认为空不会报错，那么上述场景的解决方法就是改为如下:

```go
AAA {
  ...
  Rules  []*Rule `json:"omitempty,rules"`
}
```

换结构体为其指针，此时如下请求就没问题了

```json
{
    "name": "sg-001",
    "desc": "安全组1",
    "user_id": 111,
    "rules": []
}
```



