---
title: golang的编译与运行
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 19:58:38
---

> * https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-compile-intro

# 一、go程序的编译与运行

## 1. build和run

`go build`命令有许多选项：

```shell
$ go help build
...
-n 不执行地打印流程中用到的命令
-x 执行并打印流程中用到的命令，要注意下它与-n选项的区别
-work 打印编译时的临时目录路径，并在结束时保留。默认情况下，编译结束会删除该临时目录。
...
```

我们创建一个最简单的go文件：

<!-- more -->

```go
package main

import "fmt"

func main() {
	fmt.Println("hello")
}
```

执行`go build -n test.go`

输出如下信息：

```shell
$ go build -n test.go

#
# command-line-arguments
#

mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile fmt=/Users/xwj/sdk/go1.16.5/pkg/darwin_arm64/fmt.a
packagefile runtime=/Users/xwj/sdk/go1.16.5/pkg/darwin_arm64/runtime.a
EOF
cd /Users/xwj/projects/go_projects/src/go_Advance_text/main
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -shared -p main -complete -buildid 2N2BhTZO3OKXdy5gIUjY/2N2BhTZO3OKXdy5gIUjY -goversion go1.16.5 -D _/Users/xwj/projects/go_projects/src/go_Advance_text/main -importcfg $WORK/b001/importcfg -pack ./test.go
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/buildid -w $WORK/b001/_pkg_.a # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=$WORK/b001/_pkg_.a
......
packagefile path=/Users/xwj/sdk/go1.16.5/pkg/darwin_arm64/path.a
EOF
mkdir -p $WORK/b001/exe/
cd .
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=0Jkx3V8tc3QCM4Z59Smq/2N2BhTZO3OKXdy5gIUjY/2N2BhTZO3OKXdy5gIUjY/0Jkx3V8tc3QCM4Z59Smq -extld=clang $WORK/b001/_pkg_.a
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/buildid -w $WORK/b001/exe/a.out # internal
mv $WORK/b001/exe/a.out test
```

过程看起来很乱，仔细观看下来可以发现主要由几部分组成，分别是：

- 创建临时目录，mkdir -p $WORK/b001/
- 查找依赖信息，cat >$WORK/b001/importcfg << 'EOF'… 
- 执行源代码编译，/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/compile -o ... 
- 收集链接库文件，cat >$WORK/b001/importcfg.link << 'EOF'
- 生成可执行文件，/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/link -o…
- 移动可执行文件，mv $WORK/b001/exe/a.out test

其实整个过程类似于c/c++：

**创建临时目录=>查找依赖=>编译=>收集链接裤文件(连接)=>可执行文件=>移动到当前目录下**

`go run`命令也有这些参数，我们可以再执行一下`go run -x test.go`

```shell
$ go run -x test.go  
WORK=/var/folders/s9/6mpqpft95y1dsj6hk8b_syth0000gn/T/go-build587868206
mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile fmt=/Users/xwj/sdk/go1.16.5/pkg/darwin_arm64/fmt.a
packagefile runtime=/Users/xwj/sdk/go1.16.5/pkg/darwin_arm64/runtime.a
EOF
cd /Users/xwj/projects/go_projects/src/go_Advance_text/main
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -shared -p main -complete -buildid FxrIcYVgVhEP6J6UDglp/FxrIcYVgVhEP6J6UDglp -dwarf=false -goversion go1.16.5 -D _/Users/xwj/projects/go_projects/src/go_Advance_text/main -importcfg $WORK/b001/importcfg -pack ./test.go
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/buildid -w $WORK/b001/_pkg_.a # internal
cp $WORK/b001/_pkg_.a /Users/xwj/Library/Caches/go-build/fe/fe6e260767f5c790cc2fab1dbc9b4ae0b2800717737785de94110ef68a2cb27b-d # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=$WORK/b001/_pkg_.a
...
packagefile path=/Users/xwj/sdk/go1.16.5/pkg/darwin_arm64/path.a
EOF
mkdir -p $WORK/b001/exe/
cd .
/Users/xwj/sdk/go1.16.5/pkg/tool/darwin_arm64/link -o $WORK/b001/exe/test -importcfg $WORK/b001/importcfg.link -s -w -buildmode=exe -buildid=PMQmOSMwG-dF6Sbf84Ef/FxrIcYVgVhEP6J6UDglp/HlysvLj3tn4evhT3AOuk/PMQmOSMwG-dF6Sbf84Ef -extld=clang $WORK/b001/_pkg_.a
$WORK/b001/exe/test
hello
```

`run`的流程就很清楚了，最后一步有所改变：

**创建临时目录=>查找依赖=>编译=>收集链接裤文件(连接)=>可执行文件=>执行**

那么能否拿到这个临时生成的可执行文件？默认是不行的，在`go run`最后会把临时目录删除。我们可以使用--`work`保留这个目录。演示过程如下：

```shell
$ go run -x --work hello.go
WORK=/var/folders/s9/6mpqpft95y1dsj6hk8b_syth0000gn/T/go-build846171112
...
$WORK/b001/exe/test
Hello World
```

打印了临时目录路径`WORK`，通过mv命令我们就可以把run生成的test文件拷贝到当前目录，如下所示：

```shell
mv /var/folders/s9/6mpqpft95y1dsj6hk8b_syth0000gn/T/go-build846171112 ./ 
```

![ztrAut](http://xwjpics.gumptlu.work/qinniu_uPic/ztrAut.png)

执行一下`test`:

```shell
$ ./go-build846171112/b001/exe/test 
hello
```

得到了预期的输出

## 2. 编译过程

### 1. 预备知识

#### 抽象语法树AST

[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)（Abstract Syntax Tree、AST），是源代码语法的结构的一种抽象表示，它用树状的方式表示编程语言的语法结构[1](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-compile-intro/#fn:1)。其中每个节点代表源代码中的一个元素，每一颗子树代表一个语法元素

例如`2*3+7`, 编译器的语法分析阶段会生成如下图所示的抽象语法树

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/jG4Fvj.png" alt="抽象语法树" style="zoom:67%;" />

编译器在执行完语法分析之后会输出一个抽象语法树，这个抽象语法树会辅助编译器进行**语义分析**，我们可以用它来确定语法正确的程序是否存在一些类型不匹配的问题。

#### 静态单赋值SSA

[静态单赋值](https://en.wikipedia.org/wiki/Static_single_assignment_form)（Static Single Assignment、SSA）是中间代码的特性，如果中间代码具有静态单赋值的特性，那么每个变量就只会被赋值一次[2](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-compile-intro/#fn:2)。

```go
x := 1
x := 2
y := x
```

经过简单的分析，我们就能够发现上述的代码第一行的赋值语句 `x := 1` 不会起到任何作用。下面是具有 SSA 特性的中间代码，我们可以清晰地发现变量 `y_1` 和 `x_1` 是没有任何关系的，所以在机器码生成时就可以省去 `x := 1` 的赋值，通过减少需要执行的指令优化这段代码。

```go
x_1 := 1
x_2 := 2
y_1 := x_2
```

静态单赋值是一个**代码优化**的方法, 所以属于编译器后端的一部分。除了SSA还有很多其他的编译优化方法

#### 指令集

[指令集](https://en.wikipedia.org/wiki/Instruction_set_architecture)[4](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-compile-intro/#fn:4)，很多开发者在都会遇到在本地开发环境编译和运行正常的代码，在生产环境却无法正常工作，这种问题背后会有多种原因，而不同机器使用的不同指令集可能是原因之一。

使用`uname -m`可以查看自己机器的指令集信息

x86 是目前比较常见的指令集，除了 x86 之外，还有 arm 等指令集，苹果最新 Macbook 的自研芯片就使用了 arm 指令集，不同的处理器使用了不同的架构和机器语言，所以很多编程语言为了在不同的机器上运行需要将源代码根据架构翻译成不同的机器代码。

**复杂指令集计算机（CISC）**和**精简指令集计算机（RISC）**是两种遵循不同设计理念的指令集，从名字我们就可以推测出这两种指令集的区别：

- 复杂指令集：通过增加指令的类型减少需要执行的指令数；
- 精简指令集：使用更少的指令类型完成目标的计算任务；

没有绝对的优劣，特定场景特定使用

### 2. 编译原理与核心过程

逻辑上过程可以分为四个阶段：

**1. 词法与语法分析、2. 类型检查和AST转换、3. 通用SSA生成、 4.机器代码生成**

#### 词法与语法分析

* **词法分析**的作用就是解析源代码文件，将文件中的字符串序列转换为Token序列，方便后面的解析，执行词法分析的程序称为**词法解析器(lexer)**

  例如：`package`, `json`, `import`, `(`, `io`, `)`, …

* 语法分析的输入就是词法分析的输出Token序列，语法分析器按规定好的文法(Grammar)解析Token序列构建抽象语法树(AST)，执行语法分析的程序就是**语法解析器**

  例如：

  ```json
  "json.go": SourceFile {
      PackageName: "json",
      ImportDecl: []Import{
          "io",
      },
      TopLevelDecl: ...
  }
  ```

  每个AST都对应着一个单独的go文件

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/6aqjZr.png" alt="6aqjZr" style="zoom:67%;" />

解析过程中的任何错误都会被解析器发现并将消息打印到标准输出上，整个编译过程也会终止。

#### 类型检查

拿到一组文件的抽象语法树之后，go的编译器会对语法树中的定义和使用的类型进行检查，类型检查会按照以下的顺序分别验证和处理不同类型的节点：

1. 常量、类型和函数名及类型；
2. 变量的赋值和初始化；
3. 函数和闭包的主体；
4. 哈希键值对的类型；
5. 导入函数体；
6. 外部的声明；

通过对抽象语法树的遍历，会对每个子树的类型进行验证，保证节点不存在类型错误。如果出现错误就会暴露出来。

除了类型的检查还会展开和改写一些内建的函数，例如make 关键字在这个阶段会根据子树的结构被替换成 [`runtime.makeslice`](https://draveness.me/golang/tree/runtime.makeslice) 或者 [`runtime.makechan`](https://draveness.me/golang/tree/runtime.makechan) 等函数。

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20211103200313757.png" alt="image-20211103200313757" style="zoom:67%;" />

#### 中间代码生成

当我们将源文件转换成了抽象语法树、对整棵树的语法进行解析并进行类型检查之后，就可以认为当前文件中的代码**不存在语法错误和类型错误的问题**了。Go 语言的编译器就会将输入的抽象语法树转换成**中间代码**。

在类型检查之后，编译器会通过 [`cmd/compile/internal/gc.compileFunctions`](https://draveness.me/golang/tree/cmd/compile/internal/gc.compileFunctions) 编译整个 Go 语言项目中的全部函数放入编译队列中等待goroutine的调用，并发执行的goroutine会将所有函数对应的抽象语法树转换成对应的中间代码

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/fRr6Zl.png" alt="fRr6Zl" style="zoom:67%;" />

go语言的中间代码会使用静态单赋值SSA进行代码优化，所以在这一阶段我们能够分析出代码中的无用变量和片段并对代码进行优化

#### 机器码的生成

Go 语言源代码的 [`src/cmd/compile/internal`](https://github.com/golang/go/tree/master/src/cmd/compile/internal) 目录中包含了很多机器码生成相关的包，**不同类型的 CPU 分别使用了不同的包生成机器码**，其中包括 amd64、arm、arm64、mips、mips64、ppc64、s390x、x86 和 wasm，其中比较有趣的就是 WebAssembly（Wasm）[7](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-compile-intro/#fn:7)了。

```go
$ GOARCH=wasm GOOS=js go build -o lib.wasm main.go
```

我们可以使用上述的命令将 Go 的源代码编译成能够在浏览器上运行 WebAssembly 文件，当然除了这种新兴的二进制指令格式之外，Go 语言经过编译还可以运行在几乎全部的主流机器上，不过它的兼容性在除 Linux 和 Darwin 之外的机器上可能还有一些问题.

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/nZFOIH.png" alt="nZFOIH" style="zoom:67%;" />

#### 编译器入口

Go 语言的编译器入口在 [`src/cmd/compile/internal/gc/main.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/gc/main.go) 文件中，其中 600 多行的 [`cmd/compile/internal/gc.Main`](https://draveness.me/golang/tree/cmd/compile/internal/gc.Main) 就是 Go 语言编译器的主程序，该函数会先获取命令行传入的参数并更新编译选项和配置，随后会调用 [`cmd/compile/internal/gc.parseFiles`](https://draveness.me/golang/tree/cmd/compile/internal/gc.parseFiles) 对输入的文件进行词法与语法分析得到对应的抽象语法树：

```go
func Main(archInit func(*Arch)) {
	...
	lines := parseFiles(flag.Args())
  ...
}
```

得到抽象语法树后会分**九个阶段**对抽象语法树进行更新和编译，就像我们在上面介绍的，抽象语法树会经历类型检查、SSA 中间代码生成以及机器码生成三个阶段：

1. 检查常量、类型和函数的类型；
2. 处理变量的赋值；
3. 对函数的主体进行类型检查；
4. 决定如何捕获变量；
5. 检查内联函数的类型；
6. 进行**逃逸分析**；
7. 将闭包的主体转换成引用的捕获变量；
8. 编译顶层函数；
9. 检查外部依赖的声明；

对整个编译过程有一个顶层的认识之后，我们重新回到词法和语法分析后的具体流程，在这里编译器会对生成语法树中的节点执行类型检查，除了常量、类型和函数这些顶层声明之外，它还会检查变量的赋值语句、函数主体等结构：

```go
  for i := 0; i < len(xtop); i++ {
      n := xtop[i]
      if op := n.Op; op != ODCL && op != OAS && op != OAS2 && (op != ODCLTYPE || !n.Left.Name.Param.Alias) {
        xtop[i] = typecheck(n, ctxStmt)
      }
    }

	for i := 0; i < len(xtop); i++ {
		n := xtop[i]
		if op := n.Op; op == ODCL || op == OAS || op == OAS2 || op == ODCLTYPE && n.Left.Name.Param.Alias {
			xtop[i] = typecheck(n, ctxStmt)
		}
	}
	...
```

类型检查会遍历传入节点的全部子节点，这个过程会展开和重写 `make` 等关键字，在类型检查会改变语法树中的一些节点，不会生成新的变量或者语法树，这个过程的结束也意味着源代码中已经不存在语法和类型错误，中间代码和机器码都可以根据抽象语法树正常生成。

```go
initssaconfig()

	peekitabs()

	for i := 0; i < len(xtop); i++ {
		n := xtop[i]
		if n.Op == ODCLFUNC {
			funccompile(n)
		}
	}

	compileFunctions()

	for i, n := range externdcl {
		if n.Op == ONAME {
			externdcl[i] = typecheck(externdcl[i], ctxExpr)
		}
	}

	checkMapKeys()
}
```

在主程序运行的最后，编译器会将顶层的函数编译成中间代码并根据目标的 CPU 架构生成机器码，不过在这一阶段也有可能会再次对外部依赖进行类型检查以验证其正确性。

