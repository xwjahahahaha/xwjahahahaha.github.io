---
title: Bazel构建系统的使用以及go项目实践案例
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-12 14:55:36
---

[TOC]

> 参考：
>
> * 官网文档：https://bazel.build/docs （官方文档写的真的垃圾）
> * https://zhuanlan.zhihu.com/p/95998597
> * https://www.jianshu.com/p/3a4a2b5f46de

因为`gVisor`项目中使用到了`Bazel`组件，所以在此记录简单学习其使用入门的重点

<!-- more -->

# 一、基本概念

## 1. `Bazel`是做什么的？使用场景

例如官网解释：

> Bazel 是一个开源构建和测试工具，类似于 Make、Maven 和 Gradle。它使用人类可读的简要构建语言。Bazel 支持多种语言的项目，并针对多个平台构建输出。Bazel 支持跨多个代码库和大量用户的大型代码库。

总结：一个**构建系统**的工具

使用场景：

* 构建、编译、测试项目的工作

## 2. 什么是构建系统？种类有哪些

这些基础知识来自于官网：https://bazel.build/basics

### 2.1 构建系统作用？

* 拥有构建系统可以帮助管理共享依赖项
* 构建系统还能提高设置的速度，以帮助工程师共享资源和结果

### 2.2 构建系统的目标？

* 将源代码转换为机器可读取的可执行二进制文件
* 自动化构建：无论是用于测试还是发布到生产环境，最好能够自动触发构建

### 2.3 为什么选择构建系统？为什么不直接使用编辑器、编译脚本？

区别/优势在于：

* 适合多语言：编译器只适合单一编程语言，当项目逐渐复杂就需要构建系统
* 编译依赖：当多语言多单元的代码依赖产生后，依赖的构建顺序、外部依赖的维护，这些繁琐的工作都可以交给构建系统 （这些要求`shell`脚本可以做到）
* 简单快速：构建系统编写简单且构建非常快速；而`shell`脚本的编写随着项目的扩大变得日益复杂，甚至需要超过维护项目业务代码本身的精力，并且`shell`脚本需要逻辑判断每个依赖项构建项目的速度并不快
* 适配性/拓展性：构建系统能够很好的适应多个操作系统，即使有细微的差异，这是`shell`脚本维护复杂的原因

### 2.4 基于任务的构建系统

* 核心：工作单元是任务，每个任务都是一个可以执行任何类型的逻辑的脚本，任务将其他任务指定为必须在其之前运行的依赖项。
* 实例：目前使用的大部分主流构建系统（例如 Ant、Maven、Gradle、Grunt 和 Rake）都是基于任务的
* 优势：都提供了模块化的方式编写构建脚本，来方便构建系统和管理依赖关系

* 缺点：
  * 并行构建的难度：因为任务是无法识别并行冲突的
  * 难以执行增量构建
  * 难以维护和调试脚本

### 2.5 基于工件(Artifact)的构建系统

* 核心：模块化的构建脚本不再是具体任务步骤，而是声明式清单，描述一组要构建的工件及其依赖项，以及一组有限的选项，它们会影响构建方式，基于工件的构建系统具有由系统定义的少量任务，工程师可以有限地配置任务。工程师仍然告知系统应构建什么，但构建系统会决定如何构建
* 实例：Bazel 是基于工件的构建系统 ，当工程师在命令行上运行 `bazel` 时，他们指定了一组要构建的目标（什么），而 Bazel 负责配置、运行和调度目标编译步骤（**工作原理**）

> 整个系统实际上是一个数学函数，将源文件（和编译器等工具）作为输入，并生成二进制文件作为输出

* 基本结构：

  build 文件（通常名为 `BUILD`）在 Bazel 中如下所示：

  ```go
  java_binary(
      name = "MyBinary",
      srcs = ["MyBinary.java"],
      deps = [
          ":mylib",
      ],
  )
  java_library(
      name = "mylib",
      srcs = ["MyLibrary.java", "MyHelper.java"],
      visibility = ["//java/com/example/myproduct:__subpackages__"],
      deps = [
          "//java/com/example/common",
          "//java/com/example/myproduct/otherlib",
      ],
  )
  ```

  在 Bazel 中，`BUILD` 文件用于定义目标 - 这里的两类目标分别为 `java_binary` 和 `java_library`。每个目标都对应一个可由系统创建的工件：二进制目标会生成可直接执行的二进制文件，而库目标会生成可由二进制文件或其他库使用的库。每个目标都包含：

  - `name`：命令行和其他目标引用目标的方式
  - `srcs`：要编译成目标的工件的源文件
  - `deps`：必须在此目标之前构建并链接到该目标的其他目标

  依赖项要么位于同一软件包中（如 `MyBinary` 的 `:mylib` 依赖项），要么位于同一源层次结构中的其他软件包中（如 `mylib` 的依赖项）日期：`//java/com/example/common`）

  与基于任务的构建系统一样，您可以使用 Bazel 命令行工具执行构建。如需构建 `MyBinary` 目标，请运行 `bazel build :MyBinary`。在干净的代码库中首次输入该命令后，Bazel 会执行以下操作：

  1. 解析工作区中的每个 `BUILD` 文件，以创建工件之间的依赖关系图。
  2. 使用图确定 `MyBinary` 的传递依赖项；也就是说，`MyBinary` 所依赖的每个目标，以及这些目标所依赖的每个目标，递归。
  3. 按顺序构建每个依赖项。首先，Bazel 会构建没有其他依赖项的每个目标，并跟踪仍需为每个目标构建哪些依赖项。一旦目标的所有依赖项均已构建完毕，Bazel 就会开始构建该目标。此过程会持续进行，直到 `MyBinary` 的每个传递依赖项构建完毕。
  4. 构建 `MyBinary` 以生成最终的可执行二进制文件，该代码可链接第 3 步中构建的所有依赖项。

* 优点：

  * 并行简单：因为一个工件的构建是使用系统编译器而不是用户自定义的脚本，这样就让并行安全问题得到解决，效率也就显著提升

    > 例如：由于 Bazel 知道每个目标仅生成一个 Java 库，因此它只需运行 Java 编译器（而不是任意用户定义的脚本）， 这样它就知道并行运行这些步骤是安全的。（有点类似于基于事件的并发）

  * 快速：如果输入没有更改，那么就可以重复使用输出；每次只能重新构建最小的工件集，同时保证不会生成过时的构建

    > 例如：如果 `MyBinary.java` 发生变化，Bazel 会知道重新构建 `MyBinary`，但会重复使用 `mylib`。如果 `//java/com/example/common` 的源文件发生更改，Bazel 知道要重新构建该库，即 `mylib` 和 `MyBinary`，但会重复使用 `//java/com/example/myproduct/otherlib`

  * 平台独立性/移植性：为了适应不同平台，对于构建项分为两种配置：

    - **主机配置**：构建期间运行的构建工具
    - **目标配置**：构建您最终请求的二进制文件（根据上面的构建工具）

  * 隔离环境：用rootFS文件隔离系统(类似于docker)在构建的时候创建沙盒隔离环境，避免同时写入一个文件的冲突, 比如编译 golang 项目时，不会依赖你本机的 GOPATH，从而做到同样源码、跨环境编译、输出相同，即构建的确定性

    例如，本地存在两个待构建的项目A和B，它们都依赖于第三方库D。不幸的是，它们分别依赖了不同的版本实现v1和v2

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513092801247.png" alt="image-20220513092801247" style="zoom:33%;" />

    归功于`Bazel`良好的隔离性，每个项目的工作区都是独立的。它们将所有依赖控制在各自的工作区，避免了第三方库的名字和版本冲突，

    <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513092829309.png" alt="image-20220513092829309" style="zoom: 33%;" />

  * 扩展构建系统：Bazel自带一些常用编程语言的构建工具，也支持拓展（[添加自定义规则](https://bazel.build/rules/rules)）

  * 确定性外部依赖项：确保外部依赖文件的更新于本地同步，Bazel通过构建hash值表进行比对是否更改

### 2.6 分布式构建

在分布式系统多台机器下的跨机器构建系统

https://bazel.build/basics/distributed-builds

### 2.7 依赖项管理 

可以帮助我们生成依赖项的关系图

https://bazel.build/basics/dependencies

查看依赖：https://bazel.build/docs/query-how-to

# 二、安装

`mac`的安装：

可以安装`bazelisk`工具，他会帮你自动下载/同步最新的`bazel`

```shell
brew install bazelisk
# 或者直接安装
brew install Bazel
```

其他平台安装：https://bazel.build/install

但是安装`bazelisk`后出现了问题：拉取最新版`bazel`超时

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513113138391.png" alt="image-20220513113138391" style="zoom:50%;" />

原因：被墙了

解决方法在于：

参考`issue`: https://github.com/bazelbuild/bazelisk/issues/220

在`~/.zshrc`中加入：

```shell
export BAZELISK_BASE_URL=https://github.com/bazelbuild/bazel/releases/download
export USE_BAZEL_VERSION=5.1.0.  # 换成你想要的版本
```

但是这样的坏处就是，`bazelisk`失去了原来同步最新版的功能

# 三、基本结构

## 1. 工作区 `WORKSPACE` 

* 每个工作区都有一个名为 `WORKSPACE` 的文本文件，该文件可能为空，也可能包含构建输出所需的[外部依赖项](https://bazel.build/docs/external)的引用。包含名为 `WORKSPACE` 的文件的目录被视为工作区的根，也是项目的根目录

* **代码仓库(Repository)**：一般地，当前工作区所在目录称为是**代码仓库(Repository)**，它包含了所有待构建的源文件、数据和构建脚本。当前代码仓库由匿名的`@`标识，而外部依赖的代码仓库由`@external_repo`标识

  例如，在当前代码仓库下，目标`//cub/base:placement_test`等价于`@//cub/base:placement_test`；为了简化，一般略去`@`前缀。

* 外部依赖的代码仓库：必须在当前代码仓库的`WORKSPACE`中显式地通过`http_repository`声明外部依赖的名称`@xunit_cut`，并在当前代码仓库中使用`@xunit_cut`引用该外部依赖的代码仓库(`xunit_cut`只是个例子)

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/webp.jpg" alt="img" style="zoom: 50%;" />

  

## 2. 软件包`BUILD`

### 1. 基本概念

* 代码组织的主要单元是软件包。 包就是目录，软件包是相关文件的集合 +有关如何使用这些文件生成输出工件的**规范/规则**（即`BUILD`文件或 `BUILD.bazel`），软件包定义为包含名为 `BUILD`（或 `BUILD.bazel`）的文件的**目录**。

* 一个包包括其目录中的所有文件，以及它下面的所有子目录，除了那些本身包含 `BUILD`文件的子目录（或者说如果某个子目录自身包含`BUILD`文件，则独立成为一个包，它并不隶属于上一级的父包），例如：

  ```shell
  src/my/app/BUILD
  src/my/app/app.cc
  src/my/app/data/input.txt
  src/my/app/tests/BUILD
  src/my/app/tests/test.cc
  ```

  `app.cc`, `input.txt`属于包`src/my/app`,而`test.cc`属于`src/my/app/tests`包

* 特殊地(常见于遗留系统中)，如果`BUILD`文件与`WORKSPACE`都在根目录中，此时该包名为空，而不是一个点号`.`

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513094040406.png" alt="image-20220513094040406" style="zoom:33%;" />

### 2. 软件包名称规范

`package-name:target-name`

软件包的名称是包含其 `BUILD` 文件的目录的名称，相对于包含代码库的顶级目录。例如：`my/app`

软件包名称必须完全由从 `A`-`Z`、`a`–`z`、`0`–`9`、'`/`' 集合中提取的字符组成“`-`”、“`.`”和“`_`”，且不能以斜杠开头

虽然 Bazel 支持工作区的根软件包（例如 `//:foo`）中的目标，但最好将该软件包留空，以使所有有意义的软件包都具有描述性名称

软件包名称不得包含子字符串 `//`，也不能以斜杠结尾

对于目录结构对其模块系统非常重要的语言（如 `Java`），请务必选择使用该语言的有效标识符的目录名称

### 3. `BUILD`文件规范

该文件主要针对其所在文件夹进行**依赖解析**（label）和**目标定义**（bazel target）

> Bazel 的之前版本用的文件名是 *BUILD* ，但是在一些大小写不区分的系统上，它很容易跟 build 文件混淆，因此后来改为了显式的 *BUILD.bazel* 。如果项目中同时存在两者，Bazel 更倾向于使用后者。对于所有的新项目，都推荐使用显式的 *BUILD.bazel*。

#### 基本规范

根据定义，每个软件包都包含一个 `BUILD` 文件，这是一个简短的程序。`BUILD` 文件使用命令式语言 [Starlark](https://github.com/bazelbuild/starlark/) 。

一般来说，顺序很重要：例如，变量必须在使用之前定义。但是，大多数 `BUILD` 文件**仅包含构建规则的声明**，并且这些语句的相对顺序是不重要的；

为了促进代码和数据之间的完全分离，`BUILD` 文件不能包含函数定义、`for` 语句或 `if` 语句（但允许使用列表理解和 `if` 表达式），可以在 `.bzl` 文件中声明函数。此外，`BUILD` 文件中还不允许使用 `*args` 和 `**kwargs` 参数；最重要的是，Starlark 中的程序无法执行任意 I/O 操作。这种不变性使得对 `BUILD` 文件的解释具有封闭性，仅依赖于一组已知的输入，这对于确保构建可重现非常重要

#### 加载扩展程序

Bazel 扩展程序是一些以 `.bzl` 结尾的文件。使用 `load` 语句从扩展导入符号。

```go
load("//foo/bar:file.bzl", "some_library")
```

这段代码会加载文件 `foo/bar/file.bzl` 并将 `some_library` 符号添加到环境中。这可用于加载新规则、函数或常量（例如，字符串或列表）您可以通过在调用 `load` 时使用其他参数来导入多个符号。参数必须是字符串字面量（无变量），并且 `load` 语句必须出现在顶层，它们不能位于函数正文中。

`load` 的第一个参数是一个标识 `.bzl` 文件的[标签](https://bazel.build/concepts/labels)。如果是相对标签，则会根据包含当前 `bzl` 文件的软件包（而非目录）进行解析。`load` 语句中的相对标签应使用前导 `:`。

支持别名：

```go
load("//foo/bar:file.bzl", library_alias = "some_library")
```

可以在一个 `load` 语句中定义多个别名，参数列表可以包含别名和常规符号名称

```go
load(":my_rules.bzl", "some_rule", nice_alias = "some_other_rule")
```

在 `.bzl` 文件中，以 `_` 开头的符号不会被导出，无法从其他文件加载。可见性不会影响加载（您无需使用 `exports_files` 公开 `.bzl` 文件）。

#### 构建规则的类型

- **`*_binary` 规则以给定语言构建可执行程序**。构建后，可执行文件将位于构建工具的二进制输出树中，对应于规则的标签名称

  在某些语言中，此类规则还会创建一个 `runfiles `目录，其中包含属于规则的 `data` 属性中提及的所有文件，或其传递依赖项的任一规则；这组文件汇集在一个位置，以便轻松部署到生产环境中。

- **`*_test` 规则是 `*_binary` 规则的专用项，用于自动化测试**。测试就是在成功时返回零的程序。

  与二进制文件一样，测试也包含 `runfiles` 树，测试下的文件是测试运行时可以合法打开的唯一文件。例如，程序 `cc_test(name='x', data=['//foo:bar'])` 可能会在执行期间打开并读取 `$TEST_SRCDIR/workspace/foo/bar`。（每种编程语言都有自己的效用函数来访问 `$TEST_SRCDIR` 的值，但它们都等同于直接使用环境变量。）如果未能遵守此规则，当测试在远程测试主机上执行时，将导致测试失败。

- `*_library` 规则用于指定给定编程语言中单独编译的模块。库可以依赖于其他库，而二进制文件和测试可以依赖于具有预期单独编译行为的库。

## 3. 目标`target`

### 1. 基本概念

* 软件包是目标的容器（一个软件包里可以有0个或多个目标），在软件包的 `BUILD` 文件中定义（即一个包里面`BUIL`定义的都是目标）。 大部分目标是两种主要类型之一：**文件和规则**（还有一种：包集合/包组`Package Group`）
* 文件分为两种：
  * 源文件：也就是编写的源代码（输入）
  * 生成的文件：也就是从源文件生成的（输出）
  * 常通过 `srcs`, `hdrs` 等属性表示
* 规则：
  * 作用：每个规则实例指定一组输入与输出的**关系**
  * 输入：源文件，也可能是其他规则的输出，常通过` deps `属性表示
  * 特点：规则生成的文件即输出总是与规则本身属于同一个软件包，无法将文件生成到另一个包中（但是规则的输入可以来自另一个包）

* 包组：
  * 作用：限制某些规则的可访问性(隔离)
  * 软件包组由 `package_group` 函数定义。有三个属性：包含的软件包列表、名称以及包含的其他软件包组。
  * 引用他们的唯一方法来自规则的 `visibility` 属性或 `package` 函数的 `default_visibility` 属性；它们不会生成或使用文件。

### 2. 目标名称规范

`package-name:target-name`

`target-name` 是软件包中**目标**的名称。目标分为文件和规则：

* 规则的名称是 `BUILD` 文件中规则的声明中 `name` 属性的值；

* 文件名是相对于包含 `BUILD` 文件的目录的路径名称；

目标名称必须完全由集合中的字符组成`a`-`z` 、`A` -`Z` 、`0` -`9`以及标点符号`!%-@^_"#$&'()*-+,;<=>?[]{|}~/.`组成

文件名必须是常规形式的**相对路径名**，这意味着它们不能以斜杠开头或结尾（例如，`/foo` 和 `foo/` 均禁止使用），也不能包含多个连续斜杠作为路径分隔符（例如 `foo//bar`）。同样，高级引用 (`..`) 和当前目录引用 (`./`) 也被禁止

### 3. 规则

#### 基本概念

规则指定了输入和输出之间的关系，以及构建输出的步骤

`BUILD` 文件通过调用 *rules* 规则来声明 *targets*目标, 例如使用`cc_binary`规则来申明一个目标`my_app`

```go
cc_binary(
    name = "my_app",
    srcs = ["my_app.cc"],
    deps = [
        "//absl/base",
        "//absl/strings",
    ],
)
```

每个规则调用都有一个 `name` 属性（必须是有效的[目标名称](https://bazel.build/concepts/labels#target-names)），用于在 `BUILD` 文件的软件包中声明目标。

每条规则都有一组属性, 给定规则的适用属性，以及每个属性的重要性和语义都是规则种类的函数

每个属性都有一个名称和类型。属性可以具有的一些常见类型包括整数、标签、标签列表、字符串列表、输出标签、输出标签列表

`srcs` 属性具有“标签列表”类型；其值（如果存在）是一个标签列表，每个标签都是目标的名称，即规则的输入。

详细规则：https://bazel.build/reference/be/overview

#### 自定义规则

如果你的项目有一些复杂构造逻辑、或者一些需要复用的构造逻辑，那么可以将这些逻辑以函数形式保存在 .bzl 文件，供 WORKSPACE 或者 BUILD 文件调用。其语法跟 Python 类似：

```python
def third_party_http_deps():
    http_archive(
        name = "xxxx",
        ...
    )

    http_archive(
        name = "yyyy",
        ...
    )
```

## 4. 标签`label`

为了引用一个依赖，`Bazel` 使用 `label` 语法对所有的包进行唯一标识,

所有目标都仅属于一个软件包。每个目标都存在一个全局的名字，称为标签 (`label`)，一个标签如下：

```shell
# @myrepo是代码库名称
@myrepo//my/app/main:app_binary
# 缩写
//my/app/main:app_binary
```

注意：标签一定要与传统的目录路径区分开来

标签由如下三个部分组成；其中，`@repo_name`, `@package_name`, `:`, `target_name`在一些特殊场景下都可省略。

![img](http://xwjpics.gumptlu.work/qinniu_uPic/webp-20220513105757712.jpg)

> 比如，go 中常用的一个日志库 `logrus` 的 label 为：
>
> ```python
> @com_github_sirupsen_logrus//:go_default_library
> ```
>
> 如果是本项目中的包路径，可以将 `//` 之前的 `workspace` 名字省去

* 标签的第一部分是代码库名称 `@myrepo//`。在典型情况下，标签引用使用其的同一代码库(在本工作区内，完全可略去`@repo_name`。但是，如果引用外部依赖，`@repo_name`则是必须的)，则该代码库标识符可以缩写为 `//`

* 标签的第二部分是非限定软件包名称 `my/app/main`，即相对于代码库根目录的软件包路径

  * 完全限定软件包名：`@myrepo//my/app/main`（也就是加上了代码库名）

  * 如果在同一个软件包下，则可以省略包名称，例如

    ```go
    java_binary(
        name = "MyBinary",
        srcs = ["MyBinary.java"],
        deps = [
          ":mylib",			// 省略@myrepo//my/app/main:mylib为:mylib，因为mylib在同一个软件包下
        ],
    )
    java_library(
        name = "mylib",
        srcs = ["MyLibrary.java", "MyHelper.java"],
        visibility = ["//java/com/example/myproduct:__subpackages__"],
        deps = [
            "//java/com/example/common",
            "//java/com/example/myproduct/otherlib",
        ],
    )
    ```

    可以省略为：(带不带冒号均可)

    ```shell
    app_binary
    :app_binary
    ```

* 标签中冒号后的部分，`app_binary`是非限定**目标**名称

  * 当它与软件包路径的最后一个组件匹配时，可以省略它和冒号

    ```shell
    //my/app/lib
    //my/app/lib:lib
    ```

* 标签总是以代码库标志符开始，请不要混淆软件包名:

  ```shell
  //my/app    # 这个以代码库缩写//开头，所以是一个标签
  my/app			# 这个就是软件包名
  ```

* 注意，`//my/app`并不是指代这个`my/app`软件包或者这个软件包下的所有目标，而是指`//my/app:app`，他只是这个目标`app`的缩写

* 文件目标的标签是相对于软件包根目录（包含 `BUILD` 文件的目录）的文件路径。例如：

  * `input.txt`文件位于代码库的 `my/app/main/testdata` 子目录中（`testdata`只是一个目录而不是一个包）, 其标签为:

    ```shell
    //my/app/main:testdata/input.txt			# testdata/input.txt 仅仅只是一个目录而不是一个包
    ```

* **相对标签**不能用于引用其他软件包中的目标, 例如：

  例如，如果源代码树包含软件包 `my/app` 和软件包 `my/app/testdata`（这两个目录都有自己的 `BUILD` 文件），后一个软件包中包含一个文件名为“`testdepot.zip`”，则在`//my/app:BUILD`中引用此文件的方式为：

  ```shell
  testdata/testdepot.zip    # 这样写是错误的，因为不是同一个包，不能用相对标签的方式
  //my/app/testdata:testdepot.zip		# 这样是正确的，从根开始开始引用
  ```

* 主代码库`@//`: 以 `@//` 开头的标签是对主代码库的引用，它在外部代码库中仍然有效。因此，当从外部代码库引用 `@//a/b/c` 时，它与 `//a/b/c` 不同。 前者引用主代码库，而后者在外部代码库本身中查找 `//a/b/c`

  > 代码以*代码库*形式组织。包含 `WORKSPACE` 文件的目录是主代码库的根目录，也称为 `@`

## 5. 其他文件

### 1. 配置项` .bazelrc`

对于`Bazel`来说，如果某些构建动作都需要某个参数，就可以将其写在此配置中，从而省去每次敲命令都重复输入该参数。举个 `Go` 的例子：构建、测试和运行时可能都需要`GOPROXY`，则可以配置如下：

```go
# set GOPROXY
test --action_env=GOPROXY=https://goproxy.io
build --action_env=GOPROXY=https://goproxy.io
run --action_env=GOPROXY=https://goproxy.io
```

# 四、使用`Bazel`创建golang项目

## 1. `bazel`命令基本使用

https://bazel.build/docs/user-manual

```shell
Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  aquery              Analyzes the given targets and queries the action graph.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  coverage            Generates code coverage report for specified test targets.
  cquery              Loads, analyzes, and queries the specified targets w/ configurations.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  license             Prints the license of this software.
  mobile-install      Installs targets to mobile devices.
  print_action        Prints the command line args for compiling a file.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  sync                Syncs all repositories specified in the workspace file
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.
```

## 2. rules_go 和 bazel gazelle

构建 golang 项目还需要了解两个概念：`rules_go` 和 `bazel gazelle`

### rules_go

[rules_go](https://link.zhihu.com/?target=https%3A//github.com/bazelbuild/rules_go) 是一个 Bazel 的扩展包，Bazel 可以编译 Go。它由一系列的 rule 构成，包括 `go_libray\go_binary\go_test`，支持 `vendor`、交叉编译；可以方便集成 `protobuf `、`cgo `、`gogo`、`nogo`等工具。

它会在` Bazel` 的沙箱中进行编译，不依赖本地 `GOROOT/GOPATH`，而是自动下载对应 `Go` 版本，从而可以在不同平台上进行一致性的编译。

### bazel gazelle

https://github.com/bazelbuild/bazel-gazelle#bazel-rule

`Gazelle `是一个自动生成 `Bazel `编译文件工具，包括给 `WORKSPACE `添加外部依赖、扫描源文件依赖自动生成` BUILD.bazel `文件等。

`Gazelle` 原生支持` Go `和 `protobuf`，当然可以通过扩展来支持其他语言和规则。

`Gazelle` 可以使用 `bazel `命令结合 [gazelle rule](https://link.zhihu.com/?target=https%3A//github.com/bazelbuild/bazel-gazelle%23bazel-rule) 运行，也可以下载使用单独的 Gazelle 的命令行工具。

- **自动添加外部依赖**

用 `bazel run //:gazelle update-repos repo-uri` 可以从 `go.mod` 导入对应依赖包。

比如想要往项目中增加 Kafka 的 segmentio 的 go client 包，只需要在项目根目录下执行命令： `bazel run //:gazelle update-repos github.com/segmentio/kafka-g`

`Gazelle` 便会自动增加一条依赖到 `WORKSPACE` 文件：

```go
go_repository(
    name = "com_github_segmentio_kafka_go",
    importpath = "github.com/segmentio/kafka-go",
    sum = "h1:Mv9AcnCgU14/cU6Vd0wuRdG1FBO0HzXQLnjBduDLy70=",
    version = "v0.3.4",
)
```

- **自动生成构建文件**

`Gazelle` 能够自动生成每个目录下的 `BUILD.bazel` 文件，只需要简单的两步：

1. 在项目根目录的 `BUILD.bazel` 中配置**加载**并**配置** `Gazelle`：

```go
load("@bazel_gazelle//:def.bzl", "gazelle")
# gazelle:prefix your/project/url    
gazelle(name = "gazelle") 
```

需要注意的是 `#` 后面的内容对于` Bazel` 而言是注释，对于 `Gazelle `来说却是一种语法，会被 `Gazelle `运行时所使用。当然 `Gazelle` 除了可以通过 `bazel rule` 运行，也可以单独在命令行中执行。

在根目录下执行 `bazel run //:gazelle`

## 3. 使用bazel实现一个golang的hello word

### 编译运行

创建go的运行文件

```go
package main

import "fmt"

func main() {
	fmt.Println("hello world")
}
```

创建`WORKSPACE`文件

（此部分不需要手敲，在https://github.com/bazelbuild/bazel-gazelle#id5有）

```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
# 设置rule_go和gazelle的外部依赖
http_archive(
    name = "io_bazel_rules_go",
    sha256 = "f2dcd210c7095febe54b804bb1cd3a58fe8435a909db2ec04e31542631cf715c",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_go/releases/download/v0.31.0/rules_go-v0.31.0.zip",
        "https://github.com/bazelbuild/rules_go/releases/download/v0.31.0/rules_go-v0.31.0.zip",
    ],
)

http_archive(
    name = "bazel_gazelle",
    sha256 = "5982e5463f171da99e3bdaeff8c0f48283a7a5f396ec5282910b9e8a49c0dd7e",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-gazelle/releases/download/v0.25.0/bazel-gazelle-v0.25.0.tar.gz",
        "https://github.com/bazelbuild/bazel-gazelle/releases/download/v0.25.0/bazel-gazelle-v0.25.0.tar.gz",
    ],
)

# 加载go_rules_dependencies、gazelle_dependencies两个函数使用
load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies", "go_repository")

############################################################
# Define your own dependencies here using go_repository.
# Else, dependencies declared by rules_go/gazelle will be used.
# The first declaration of an external repository "wins".
############################################################

go_repository(
    name = "org_golang_x_mod",
    importpath = "golang.org/x/mod",
    sum = "h1:kQgndtyPBW/JIYERgdxfwMYh3AVStj88WQTlNDi2a+o=",
    version = "v0.6.0-dev.0.20220106191415-9b9b3d81d5e3",
    build_external = "external",
)

go_rules_dependencies()

go_register_toolchains(version = "1.18") # 指定go版本（会在沙盒中运行）

gazelle_dependencies()
```

创建`BUILD.bazel`文件

```python
load("@io_bazel_rules_go//go:def.bzl", "go_binary")

go_binary(
    name = "test",
    srcs = ["main.go"],
    importpath = "test",
    visibility = ["//visibility:private"],
)
```

整个目录结构：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513162746528.png" alt="image-20220513162746528" style="zoom: 67%;" />

执行编译、运行：

```shell
bazel run //:test
```

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513162957589.png" alt="image-20220513162957589" style="zoom: 67%;" />

### 使用gazelle自动生成`BUILD.bazel`文件

上面虽然加载了外部`gazelle`依赖，但是并没有使用其自动生成的功能，因为一旦项目庞大起来，不可能所有的`BUILD`全都手写

在项目的根目录的`BUILD.bazel`中配置加载并配置`Gazelle`

```python
load("@bazel_gazelle//:def.bzl", "gazelle")
# gazelle:prefix test  
gazelle(name = "gazelle") 
```

> 需要注意的是 # 后面的内容对于`Bazel`而言是注释，**对于`Gazelle`来说却是一种语法**，会被`Gazelle`运行时所使用。当然`Gazelle`除了可以通过`bazel rule`运行，也可以单独在命令行中执行。

创建t1和t2两个文件夹，写入两个文件

```go
package t1

import "fmt"

func Test1() {
	fmt.Println("test t1")   // 另一个就是t2
}
```

文件结构：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513164321998.png" alt="image-20220513164321998" style="zoom:67%;" />

在根目录下面执行`bazel run //:gazelle`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513202927769.png" alt="image-20220513202927769" style="zoom:67%;" />

> 报错：`missing strict dependencies:xxx import of "golang.org/x/xerrors"` (你可能不会出错，因为上面的`WORKFILE`已经解决了)
>
> ![image-20220513203000286](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513203000286.png)
>
> 原因：缺少硬依赖`golang.org/x/xerrors`
>
> 解决：在`WORKFILE`文件中加入依赖
>
> ```go
> ############################################################
> # Define your own dependencies here using go_repository.
> # Else, dependencies declared by rules_go/gazelle will be used.
> # The first declaration of an external repository "wins".
> ############################################################
> 
> go_repository(
>     name = "org_golang_x_mod",
>     importpath = "golang.org/x/mod",
>     sum = "h1:kQgndtyPBW/JIYERgdxfwMYh3AVStj88WQTlNDi2a+o=",
>     version = "v0.6.0-dev.0.20220106191415-9b9b3d81d5e3",
>     build_external = "external",
> )
> 
> go_rules_dependencies()
> ```
>
> 详情见：https://github.com/bazelbuild/bazel-gazelle/issues/1217

检查目录：

红色框的就是`BUILD`文件

![image-20220513210941557](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513210941557.png)

其中根目录`BUILD.bazel`：

```python
load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library")
load("@bazel_gazelle//:def.bzl", "gazelle")

# gazelle:prefix test
gazelle(name = "gazelle")

go_library(
    name = "test_lib",
    srcs = ["main.go"],
    importpath = "test",
    visibility = ["//visibility:private"],
)

go_binary(
    name = "test",
    embed = [":test_lib"],
    visibility = ["//visibility:public"],
)

```

`t1/BUILD.bazel`：

```python
load("@io_bazel_rules_go//go:def.bzl", "go_library")

go_library(
    name = "t1",
    srcs = ["main.go"],
    importpath = "test/t1",
    visibility = ["//visibility:public"],
)
```

### 引用其他包

现在在`main`中调用`t2`

```go
package main

import (
	"fmt"
	"test/t2"			// 这里的路径就是对应t2的importpath
)

func main() {
	fmt.Println("hello world")
	t2.Test2()
}
```

此时需要手动重新生成一下目标关系:

`bazel run //:gazelle`

发现根目录下的`BUILD`变为:

```python
load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library")
load("@bazel_gazelle//:def.bzl", "gazelle")

# gazelle:prefix test
gazelle(name = "gazelle")

go_library(
    name = "test_lib",
    srcs = ["main.go"],
    importpath = "test",
    visibility = ["//visibility:private"],
    deps = ["//t2"],					# 多了这个依赖关系
)

go_binary(
    name = "test",
    embed = [":test_lib"],
    visibility = ["//visibility:public"],
)
```

最后`bazel run //:test`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220513215924633.png" alt="image-20220513215924633" style="zoom:67%;" />
