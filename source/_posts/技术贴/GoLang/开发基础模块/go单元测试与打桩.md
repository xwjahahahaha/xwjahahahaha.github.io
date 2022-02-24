---
title: go单元测试与打桩
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-02-11 14:18:28
---




<!-- more -->

# 一、单元测试

testing 测试框架

Go 语言中自带有一个轻量级的测试框架**testing** 和自带的**go test** 命令来实现**单元测试**和**性能测试**，testing 框架和其他语言中的测试框架类似，可以基于这个框架写针对相应函数的测试用例，也可以基于该框架写相应的压力测试用例。通过单元测试，可以解决如下问题:

1) 确保每个函数是可运行，并且运行结果是正确的
2) 确保写出来的代码性能是好的，
3) <u>单元测试</u>能及时的发现程序设计或实现的<u>逻辑错误</u>，使问题及早暴露，便于问题的定位解决，<u>而**性能测试**的重点在于发现程序设计上的一些问题，让程序能够在**高并发**的情况下还能保持稳定</u>

## 1.1 一般使用

testing 提供对 Go 包的自动化测试的支持。通过 `go test` 命令，能够自动执行如下形式的任何函数：

```go
import "testing"
func TestXxx(*testing.T)
```

其中 Xxx 可以是任何字母数字字符串（**但第一个字母不能是小写 [a-z]**），用于识别测试例程。

在这些函数中，使用 Error, Fail 或相关方法来发出失败信号。

注意：**<font color='red'>函数开头的Test是固定的写法</font>**，<font color='red'>后面的Xxx第一个字母要大写</font>

使用例子：

```go
import "testing"

func TestAddUpper(t *testing.T)  {
	res := AddUpper(10)
	if res != 11{
		t.Fatalf("代码计算错误%v\n", res)
	}
	//如果正确就输出日志
	t.Logf("代码正确%v\n",res)
}
```

没有主函数也可以执行

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201118160839.png)

## 1.2 大致原理/过程

test框架大致的原理

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201118161512.png)

## 1.3 注意事项

1) **测试用例文件名必须以<font color='red'>_test.go </font>结尾。**比如`cal_test.go` , `cal `不是固定的。
2) 测试用例函数必须以Test 开头，一般来说就是Test+被测试的函数名，比如TestAddUpper
3) **TestAddUpper(t *tesing.T) 的形参类型必须是*testing.T** 
4) 一个测试用例文件中，可以有多个测试用例函数，比如TestAddUpper、TestSub
5) 运行测试用例指令:

​		(1)` cmd>go test`  [如果运行正确，无日志，错误时，会输出日志]

​		(2) `cmd>go test -v `[**运行正确或是错误，都输出日志**]

6) 当出现错误时，**可以使用t.Fatalf 来格式化输出错误信息**，并退出程序
7) **t.Logf 方法可以输出相应的日志**
8) 测试用例函数，并没有放在main 函数中，也执行了，这就是测试用例的方便之处[原理图].
9) PASS 表示测试用例运行成功，FAIL 表示测试用例运行失败
10) **测试单个文件，一定要带上被测试的原文件 `go test -v cal_test.go cal.go`**
11) **测试单个方法 `go test -v -test.run TestAddUpper`**
12) **单元测试的时候不加载init函数**

# 二、打桩

> 参考：
>
> * https://blog.csdn.net/ruanrunxue/article/details/112447535

## 2.1 什么是打桩

当我们测试主要函数时，不希望其中包含的次要函数对环境的依赖导致主要函数难以测量，具体来说比如当你需要对依赖了很多第三方库或者跟平台、环境强相关的模块进行单元测试时，仅仅使用`go test`往往不能使测试用例正常的执行结束，更别说达到代码验证的目的了。例如：

```go
// 判断/opt/container/config.properties文件中是否包含str字符串
func isConfigFileContain(str string) bool {
	file, err := os.Open("/opt/container/config.properties")
	if err != nil {
		panic(err)
	}
	content, err := ioutil.ReadAll(file)
	if err != nil {
		panic(err)
	}
	return strings.Contains(string(content), str)
}
```

需要访问文件，但是并不是每个环境中有这个文件，所以测试起来较为麻烦

一般有两种解决方式：

* 在本地创建文件，测试，然后修改文件，再测试  （外部打桩）
* 让测试用例在调用`os.Open("/opt/container/config.properties")`时能够按照我们的预想返回相应的结果。（内部打桩）

**对于次要测试对象，我们通常只会关注主要测试对象和次要测试对象之间的交互，比如是否被调用、调用参数、调用的次数、调用的结果等，至于次要测试对象是如何执行的，这些细节过程我们并不关注**。

明显上面第二种解决方式更好；所以以上例子的测试应该如下：

```go
var fileStr = `xxxxxxxx`

func TestIsConfigFileContain(t *testing.T) {
	ast := assert.New(t)
	openPatch := gomonkey.ApplyFunc(os.Open, func(name string) (*os.File, error) {
        return &os.File{}, nil
	})
    readPatch := gomonkey.ApplyFunc(ioutil.ReadAll, func(r io.Reader) ([]byte, error) {
        return []byte(fileStr), nil
	})
	defer openPatch.Reset()
    defer readPatch.Reset()

	ans := isConfigFileContain()
	ast.Equal(true, ans)
}
```

## 2.2 常见问题

### 两次调用

测试的一个函数中的两个子函数分别又调用了同一个子函数：

```go
func test() {
    a()
    b()
}

func a() {
    c()
}

func b() {
    c()
}
```

这样我们在测试函数test时对函数c进行打桩的时候并不希望其返回的是同一个值，应该分别a、b两种情况

解决方式：引入一个计数器，当第二次调用的时候放回不同的值

``` go
func Test（t *testing）{
    ...
    count := 0
    patch := gomonkey.ApplyFunc(c, func() (string, error) {
        if count == 0 {
            return aStr, nil
            count++
        }else {
            return bStr, nil
        }
	})
    defer patch.Reset()
    ...
} 
```

