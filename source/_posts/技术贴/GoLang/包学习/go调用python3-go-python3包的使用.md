---
title: go调用python3:go-python3包的使用
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-10-06 16:00:32
---

> 参考资料：
>
> https://zhuanlan.zhihu.com/p/150253406
>
> https://blog.csdn.net/skyztttt/article/details/8115086
>
> https://poweruser.blog/embedding-python-in-go-338c0399f3d5
>
> 包地址：
>
> python2：https://github.com/sbinet/go-python
>
> python3：https://github.com/DataDog/go-python3

<!-- more -->

Python是机器/深度学习御用开发语言，Golang是新时代后端开发语言。Python很适合算法写模型，而Golang很适合提供API服务，两位同志都红的发紫。

出于项目需求和兴趣，这里就介绍一下正确搅基的办法。

从网络中查询资料后，发现两个好用的包（见上方）

**因为python使用的是3.x，所以本文使用的是DataDog/go-python3版本， sbinet/go-python对应python2.x**

[TOC]

# 一、安装go-python3包

安装就有点麻烦。。。

> 我的环境
>
> 系统：MacOS Big Sur 11.6 M1芯片 arm64
>
> go：go version go1.16.5 darwin/arm64
>
> python：python3.9

命令: `go get github.com/DataDog/go-python3`

报错1:

```shell
# github.com/DataDog/go-python3
../../../go_projects/pkg/mod/github.com/!data!dog/go-python3@v0.0.0-20210805105248-03d93fb21b67/dict.go:141:13: could not determine kind of name for C.PyDict_ClearFreeList
```

原因：

**`DataDog/go-python3`只适用于`python3.7`版本！**，所以之前我的电脑安装的是3.9，3.9版本删除了`C.PyDict_ClearFreeList`函数，所以找不到。

（The C.PyDict_ClearFreeList function [has been removed](https://docs.python.domainunion.de/3/whatsnew/3.9.html#id3) from the Python C API with Python 3.9)

https://github.com/DataDog/go-python3/issues/38

解决：

使用`anaconda`创建一个python3.7的环境：（如果不会使用`anaconda`可以自行百度，不难）

```shell
conda create -n fs_py37 python=3.7
conda activate fs_py37  
```

找到环境下`lib/pkgconfig`目录, 我的是：(一般都在conda安装中的envs下)

`/Users/xwj/opt/anaconda3/envs/fs_py37/lib/pkgconfig`

然后重置环境, go get:

```shell
export PKG_CONFIG_PATH=/Users/xwj/opt/anaconda3/envs/fs_py37/lib/pkgconfig
go get github.com/DataDog/go-python3 
```

报错2（不是m1 mac的可能没问题）:

```shell
# github.com/DataDog/go-python3
ld: warning: ignoring file /Users/xwj/opt/anaconda3/envs/fs_py37/lib/libpython3.7m.dylib, building for macOS-arm64 but attempting to link with file built for macOS-x86_64
Undefined symbols for architecture arm64:
  "_PyBool_FromLong", referenced from:
      __cgo_a0c8609171f9_Cfunc_PyBool_FromLong in _x002.o
     (maybe you meant: __cgo_a0c8609171f9_Cfunc_PyBool_FromLong)
			.....
```

原因：

anaconda安装的python3.7版本（amd64）与go的arm64架构不匹配（垃圾M1）

解决：

要么换python架构，要么换go的架构，反正两者架构要一致才可以

尴尬的是，目前M1还是不支持python@3.7: https://doesitarm.com/formula/python@3.7/

所以只能安装一个go的amd64版本也就是intel的x86 (以后也可能用得到)：

1. 官网下载压缩包：https://golang.google.cn/dl/

   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/QvtNdL.png" alt="QvtNdL" style="zoom:67%;" />

   注意选x86架构而不是ARM64

2. 将解压的文件夹放到一个你喜欢的地方，我是`/Users/xwj/sdk/go_x86_1.17.1`

3. 进入这个文件夹修改`go`执行文件名字，防止和本来的arm架构的`go`重名，并且提醒自己使用的go环境

   ```shell
   cd go_x86_1.17.1/bin
   mv go gox86
   ```

4. 添加配置文件：

   `vim ~/.zshrc`，我的配置文件如下

   ```shell
   # Go arm64
   # export GOROOT="/Users/xwj/sdk/go1.17.1"
   # Go x86-64
   export GOROOT="/Users/xwj/sdk/go_x86_1.17.1"
   export GOPROXY=https://goproxy.cn,direct
   export GOPATH="/Users/xwj/projects/go_projects"
   export GO111MODULE="auto"
   export PATH=$PATH:$GOROOT/bin:$GOPATH/src:$GOPATH/bin
   ```

   `source ~/.zshrc`, 重新打开一个终端

   **想使用哪个就将另一个注释掉再source一下即可（重开终端）**

5. 测试：

   ![image-20211006170400449](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211006170400449.png)

   这样go环境就可以切换了，目前先使用amd64的尝试

下面再重新get测试一下：

```shell
export PKG_CONFIG_PATH=/Users/xwj/opt/anaconda3/envs/fs_py37/lib/pkgconfig
gox86 get github.com/datadog/go-python3  
```

![6561pH](http://xwjpics.gumptlu.work/qinniu_uPic/6561pH.png)

大功告成！

# 二、基本使用

## 2.1 调用自定义python

目录总体结构：

![image-20211006213827430](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211006213827430.png)

测试的python文件

`hello.py`

```python
a = 10
def SayHello(xixi):
    return xixi + "haha"
```

可以将go调用python的过程分为以下5步：

1. 初始化python环境

2. 引入模块py对象

3. 使用该模块的变量与函数
4. 解析结果
5. 销毁python3运行环境

以下五步的所有程序都在`main.go`文件中

```go
package main

import (
	"fmt"
	"github.com/datadog/go-python3"
	"os"
)

func init() {
	// 1. 初始化python环境
	python3.Py_Initialize()
	if !python3.Py_IsInitialized() {
		fmt.Println("Error initializing the python interpreter")
		os.Exit(1)
	}
}

func main() {
	// 2. 设置本地python import 的路径
	p := "/Users/xwj/opt/anaconda3/envs/fs_py37/lib/python3.7/site-packages"
	InsertBeforeSysPath(p)
	// 3. 导入hello模块
	hello := ImportModule("./hello", "hello")
	// pyObject => string 解析结果
	helloRepr, err := pythonRepr(hello)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[MODULE] repr(hello) = %s\n", helloRepr)
	// 4. 获取变量
	a := hello.GetAttrString("a")
	aString, err := pythonRepr(a)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[VARS] a = %#v\n", aString)
	// 5. 获取函数方法
	SayHello := hello.GetAttrString("SayHello")
	// 设置调用的参数（一个元组）
	args := python3.PyTuple_New(1)	// 创建存储空间
	python3.PyTuple_SetItem(args, 0, python3.PyUnicode_FromString("xwj"))	// 设置值
	res := SayHello.Call(args, python3.Py_None)	// 调用
	fmt.Printf("[FUNC] res = %s\n", python3.PyUnicode_AsUTF8(res))
	// 6. 调用第三方库sklearn
	sklearn := hello.GetAttrString("sklearn")
	skVersion := sklearn.GetAttrString("__version__")
	sklearnRepr, err := pythonRepr(sklearn)
	if err != nil {
		panic(err)
	}
	skVersionRepr, err := pythonRepr(skVersion)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[IMPORT] sklearn = %s\n", sklearnRepr)
	fmt.Printf("[IMPORT] sklearn version =  %s\n", skVersionRepr)
	//// 7. 结束环境
	python3.Py_Finalize()
}

// InsertBeforeSysPath
// @Description: 添加site-packages路径即包的查找路径
// @param p
func InsertBeforeSysPath(p string){
	sysModule := python3.PyImport_ImportModule("sys")
	path := sysModule.GetAttrString("path")
	python3.PyList_Append(path, python3.PyUnicode_FromString(p))
}

// ImportModule
// @Description: 倒入一个包
// @param dir
// @param name
// @return *python3.PyObject
func ImportModule(dir, name string) *python3.PyObject {
	sysModule := python3.PyImport_ImportModule("sys") 	// import sys
	path := sysModule.GetAttrString("path")            // path = sys.path
	python3.PyList_Insert(path, 0, python3.PyUnicode_FromString(dir)) // path.insert(0, dir)
	return python3.PyImport_ImportModule(name)            // return __import__(name)
}

// pythonRepr
// @Description: PyObject转换为string
// @param o
// @return string
// @return error
func pythonRepr(o *python3.PyObject) (string, error) {
	if o == nil {
		return "", fmt.Errorf("object is nil")
	}
	s := o.Repr()
	if s == nil {
		python3.PyErr_Clear()
		return "", fmt.Errorf("failed to call Repr object method")
	}
	defer s.DecRef()

	return python3.PyUnicode_AsUTF8(s), nil
}

// PrintList
// @Description: 输出一个List
// @param list
// @return error
func PrintList(list *python3.PyObject) error {
	if exc := python3.PyErr_Occurred(); list == nil && exc != nil {
		return fmt.Errorf("Fail to create python list object")
	}
	defer list.DecRef()
	repr, err := pythonRepr(list)
	if err != nil {
		return fmt.Errorf("fail to get representation of object list")
	}
	fmt.Printf("python list: %s\n", repr)
	return nil
}
```

编译成可执行文件`test`并运行：

```shell
gox86 build ./
./test
```

运行时报错：`dyld: Library not loaded: @rpath/libpython3.7m.dylib`

![UFgcZm](http://xwjpics.gumptlu.work/qinniu_uPic/UFgcZm.png)

原因：

该运行文件没有链接到需要匹配的python lib库

解决：

首先查看该文件的依赖文件：

`otool -L /Users/xwj/projects/blockchain_projects/FedSharing/fed_sharing/mainchain/miner/test/./test`

![mYYj86](http://xwjpics.gumptlu.work/qinniu_uPic/mYYj86.png)

可以看到有两个依赖，第一个无法加载，可能就是路径有问题

在此之前，我们可以简单了解一下`@executable_path、@loader_path、@rpath`三者都是什么意思：

* @executable_path：就是可执行程序的路径
* @loader_path：可以通过这个path来设置动态库的install path name
* @rpath：它是run path的缩写。本质上它不是一个明确的path，甚至可以说它不是一个path。它只是一个变量，或者叫**占位符**。这个变量通过XCode中的run path选项设置值，或者通过`install_name_tool`的`-add_rpath`设置值。设置好run path之后，所有的@rpath都会被替换掉

我们并没有设置@rpath，所以导致失败。

所以在这里我们给其设置一个@rpath即之前本地conda创建的库，这样就可以链接上了。

找到库的路径，我的是`/Users/xwj/opt/anaconda3/envs/fs_py37/lib/libpython3.7m.dylib` (就是在你创建的环境名下的lib中)

然后对这个可执行文件`test`运行以下命令添加：

`install_name_tool -add_rpath /Users/xwj/opt/anaconda3/envs/fs_py37/lib ./test    `

> <font color='#39b54a'>如果你的可执行程序希望在多个机器上都适配，那么可以添加一个链接，把lib放到项目相对路径下，使用@executable_path指向这个相对路径</font>

再次运行即可

输出如下：

![image-20211006215622130](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211006215622130.png)

## 2.2 调用import的第三方库包

这里以`sklearn`为例

在hello.py中添加`import sklearn`

在conda环境下下载：` conda install scikit-learn -y`

如果使用IDE记得将IDE中python环境设置与conda activate的环境一致

在`main.go`中添加:

```go
func main() {
  ...
  // 6. 调用第三方库sklearn
	sklearn := hello.GetAttrString("sklearn")
	skVersion := sklearn.GetAttrString("__version__")
	sklearnRepr, err := pythonRepr(sklearn)
	if err != nil {
		panic(err)
	}
	skVersionRepr, err := pythonRepr(skVersion)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[IMPORT] sklearn = %s\n", sklearnRepr)
	fmt.Printf("[IMPORT] sklearn version =  %s\n", skVersionRepr)
  ...
}	
```

运行结果如下：

![image-20211006230529317](http://xwjpics.gumptlu.work/qinniu_uPic/image-20211006230529317.png)

基本的使用可以了之后就可以自由的使用go去调用python函数了，还有更多的功能（例如设置python变量的值等）都可以在https://docs.python.org/3.7/c-api/index.html查看！

# 三、内存管理

虽然go和python都是自动管理内存的，但是当我们使用`PyObject`的时候我们需要手动提示Python runtime该`PyObject`是否还需要，否则就可能发生内存的泄漏。

每个`PyObject`都有一个引用计数，如果它降为0,Python的垃圾收集器知道它可以释放内存。

当我们不再需要一个对象时，有时我们需要减少`ref count`，有时我们需要增加它，以确保内存不会被过快释放。

每一个`PyObject`都有两个函数:

* `.DecRef()`: 减少引用计数
* `.IncRef()`: 增加引用计数

当我们调用Python C API函数时，输入和返回的`PyObjects`会发生以下三种已定义的情况:

1. Return value: “New” reference

   大多数在当我们创建一个新的Obecjt的时候(i.e. `PyList_new`)

   一旦你不再需要`PyObject`，你可以**减少**引用计数`.DecRef()`，或者把它交给需要的人(例如，另一个使用它的函数，或者把它作为返回值传递给调用者，然后减值)。否则就会出现内存泄漏。

   ```go
   pObject := python3.PyUnicode_FromString(p)
   defer pObject.DecRef()
   ```

2. The function “steals” a reference to an item

   大多数在函数设置set items的时候，例如`PyList_setItem`, 这时候我们不需要做任何事

3. Return value: “Borrowed” reference

   两个指针指向同一个内存空间

   大多数函数get items的时候，例如`PyList_GetItem`

   如果你想继续处理这个对象/指针，就**增加**引用计数(并注意以后它会以某种方式减少)。

例子：

```go
pylist := python3.PyList_New(len(data)) //retval: New reference, gets stolen later
for i := 0; i < len(data); i++ {
  item := python3.PyFloat_FromDouble(data[i]) //ret val: New reference, gets stolen later
  ret := python3.PyList_SetItem(pylist, i, item)
  if ret != 0 {
    if python3.PyErr_Occurred() != nil {
      python3.PyErr_Print()
    }
    item.DecRef()
    pylist.DecRef()
    return nil, fmt.Errorf("error setting list item")
  }
}
```

当我们心创建一个`PyObject`时因为后续还要使用(`stolen`加入到List中)，所以我们先不`.DecRef()`，当`pylist.DecRef()`的时候会把其中所有的元素都一起`.DecRef()`

现在我们需要修改之前的`main.go`代码，防止内存的泄漏：

```go
package main

import (
	"fmt"
	"github.com/datadog/go-python3"
	"os"
)

func init() {
	// 1. 初始化python环境
	python3.Py_Initialize()
	if !python3.Py_IsInitialized() {
		fmt.Println("Error initializing the python interpreter")
		os.Exit(1)
	}
}

func main() {
	// 7. 结束环境(提前defer)
	defer python3.Py_Finalize()
	// 2. 设置python import 的路径
	p := "/Users/xwj/opt/anaconda3/envs/fs_py37/lib/python3.7/site-packages"
	InsertBeforeSysPath(p)
	// 3. 导入hello模块
	hello := ImportModule("./hello", "hello")
	defer hello.DecRef()
	// pyObject => string 解析结果
	helloRepr, err := pythonRepr(hello)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[MODULE] repr(hello) = %s\n", helloRepr)
	// 4. 获取变量
	a := hello.GetAttrString("a")
	defer a.DecRef()
	aString, err := pythonRepr(a)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[VARS] a = %#v\n", aString)
	// 5. 获取函数方法
	SayHello := hello.GetAttrString("SayHello")
	defer SayHello.DecRef()
	// 设置调用的参数（一个元组）
	args := python3.PyTuple_New(1)	// 创建存储空间
	defer args.DecRef()
	input := python3.PyUnicode_FromString("xwj")		// input不需要DecRef，因为DecRef args的时候就一起DecRef了
	python3.PyTuple_SetItem(args, 0, input)	// 设置值
	res := SayHello.Call(args, python3.Py_None)	// 调用
	fmt.Printf("[FUNC] res = %s\n", python3.PyUnicode_AsUTF8(res))
	// 6. 调用第三方库sklearn
	sklearn := hello.GetAttrString("sklearn")
	defer sklearn.DecRef()
	skVersion := sklearn.GetAttrString("__version__")
	defer skVersion.DecRef()
	sklearnRepr, err := pythonRepr(sklearn)
	if err != nil {
		panic(err)
	}
	skVersionRepr, err := pythonRepr(skVersion)
	if err != nil {
		panic(err)
	}
	fmt.Printf("[IMPORT] sklearn = %s\n", sklearnRepr)
	fmt.Printf("[IMPORT] sklearn version =  %s\n", skVersionRepr)
}

// InsertBeforeSysPath
// @Description: 添加site-packages路径即包的查找路径
// @param p
func InsertBeforeSysPath(p string){
	sysModule := python3.PyImport_ImportModule("sys")
	path := sysModule.GetAttrString("path")
	pObject := python3.PyUnicode_FromString(p)
	defer pObject.DecRef()
	python3.PyList_Append(path, pObject)
}

// ImportModule
// @Description: 倒入一个包
// @param dir
// @param name
// @return *python3.PyObject
func ImportModule(dir, name string) *python3.PyObject {
	sysModule := python3.PyImport_ImportModule("sys") 	// import sys
	path := sysModule.GetAttrString("path")            // path = sys.path
	dirObject := python3.PyUnicode_FromString(dir)
	defer dirObject.DecRef()
	python3.PyList_Insert(path, 0, dirObject) // path.insert(0, dir)
	return python3.PyImport_ImportModule(name)            // return __import__(name)
}

// pythonRepr
// @Description: PyObject转换为string
// @param o
// @return string
// @return error
func pythonRepr(o *python3.PyObject) (string, error) {
	if o == nil {
		return "", fmt.Errorf("object is nil")
	}
	s := o.Repr()		// 获取对象转换为可读
	if s == nil {
		python3.PyErr_Clear()
		return "", fmt.Errorf("failed to call Repr object method")
	}
	defer s.DecRef()
	return python3.PyUnicode_AsUTF8(s), nil
}

// PrintList
// @Description: 输出一个List
// @param list
// @return error
func PrintList(list *python3.PyObject) error {
	if exc := python3.PyErr_Occurred(); list == nil && exc != nil {
		return fmt.Errorf("Fail to create python list object")
	}
	defer list.DecRef()
	repr, err := pythonRepr(list)
	if err != nil {
		return fmt.Errorf("fail to get representation of object list")
	}
	fmt.Printf("python list: %s\n", repr)
	return nil
}
```

# 四、Pylist迭代器

```go
pylist := python3.PyList_New(len(data)) //retval: New reference, gets stolen later
defer pylist.DecRef()
// 获取迭代器
seq := pylist.GetIter() //ret val: New reference
//...
defer seq.DecRef()

// 迭代器有next函数，能够返回下一个item
tNext := seq.GetAttrString("__next__") //ret val: new ref
//...
defer tNext.DecRef()

pylistLen := pylist.Length()
for i := 1; i <= pylistLen; i++ {
  item := tNext.CallObject(nil) //ret val: new ref
  // do something
  if item != nil {
    item.DecRef()
  }
}
```

# 五、多个goroutine调用python

要点： **保证python在一段时间中只给一个线程调用**

```python
# foo.py
import sys

def print_odds(limit=10):
    """
    Print odds numbers < limit
    """
    for i in range(limit):
        if i%2:
            sys.stderr.write("{}\n".format(i))

def print_even(limit=10):
    """
    Print even numbers < limit
    """
    for i in range(limit):
        if i%2 == 0:
            sys.stderr.write("{}\n".format(i))

```
```go
// We’ll try to print odd and even numbers concurrently from Go, using two different goroutines (thus involving threads):
package main

import (
    "sync"

    "github.com/sbinet/go-python"
)

func main() {
    // PyEval_InitThreads() 创建全局解释器GIL并且会锁定他
    python.Initialize()

    var wg sync.WaitGroup
    wg.Add(2)

    fooModule := python.PyImport_ImportModule("foo")
    odds := fooModule.GetAttrString("print_odds")
    even := fooModule.GetAttrString("print_even")

    // 保存当前锁定的状态，然后释放GIL锁
    state := python.PyEval_SaveThread()

    go func() {
      	// 获取当前的GIL并锁定
        _gstate := python.PyGILState_Ensure()
        odds.Call(python.PyTuple_New(0), python.PyDict_New())
      	// 释放当前状态
        python.PyGILState_Release(_gstate)

        wg.Done()
    }()

    go func() {
      	// 获取当前的GIL并锁定
        _gstate := python.PyGILState_Ensure()
        even.Call(python.PyTuple_New(0), python.PyDict_New())
      	// 释放当前状态
        python.PyGILState_Release(_gstate)

        wg.Done()
    }()

    wg.Wait()

    // 当我们不在需要GIL的时候，在关闭python环境之前将状态重新存储并锁定GIL，然后关闭
    python.PyEval_RestoreThread(state)
    python.Finalize()
}
```

总共就是三个步骤：

1. Save the state and lock the GIL.
2. Do Python.
3. Restore the state and unlock the GIL.

# 六、注意事项

## 6.1 多次调用内存消耗

循环调用一个go-python函数(i.e.`demo`)内存会上升一段然后再保持平稳，理论上说没有内存泄露应该是一条直线

```go
for {
  demo(oModule)
}
```

![rwZcyk](http://xwjpics.gumptlu.work/qinniu_uPic/rwZcyk.png)

出现这样的原因是go的自动内存回收机制而不是出现了内存泄漏

## 6.2 重复导包

```go
python3.Py_Initialize()
python3.PyRun_SimpleString("import numpy")
python3.Py_Finalize()
python3.Py_Initialize()
python3.PyRun_SimpleString("import numpy")
// segfault happening here
```

有一些python拓展包（i.e. `numpy`）如果像上面这样重复导入会报错

## 6.3 支持的Python

在删除了`PyEval_ReInitThreads`函数的绑定后，在Win、Mac、Linux上成功地在Python 3.8上运行了go-python3，该函数已从Python 3.8开始的Python C API中删除。我还没有用Python 3.9进行测试。

所以，结论是只当前支持python 3.7

