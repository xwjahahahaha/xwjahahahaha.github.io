---
title: shell脚本编辑基础
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-12 13:46:13
---

# 一、Shell脚本基础

## 1.1 概念

一系列的shell命令的集合，还可以加入一些逻辑操作 （if else for）将这些命令放入一个脚本文件中

<!-- more -->

## 1.2 基本格式

命名格式：xxx.sh  (一般后缀为sh，也可不写)

书写格式：

```shell
# test.sh  #是shell脚本的注释
#！/bin/bash  # 指定解析shell脚本使用的命令解释器  /bin/sh也可以这是不写这一行的默认解释器
# 一系列的shell命令
ls 
pwd
echo "hello world"
```

## 1.3 shell脚本的执行

```shell
# shell脚本的编写完之后，必须添加执行权限
chmod u+x xxx.sh
# 执行脚本
./xxx.sh
sh xxx./sh
```

shell脚本从上至下依次执行，中途有错误则中断

## 1.4 shell脚本中的变量

* 变量的定义

  1. 普通变量(本地变量)

     ```shell
     # 定义变量，定义变量必须要赋值， =前后不允许有空格，有空格是比较（类似于==）
     temp=999
     # 普通变量只能在当前进程中使用
     ```

  2. 环境变量（一般大写）

     ```shell
     # 可以理解为全局变量，在当前操作系统中可全局访问
     # 分类
     - 系统自带
     例如： -PWD  -PATH
     - 用户自定义
     - 将普通变量提升为系统变量  前面加个set或export
     GOPATH=/home/zoro/go/src ->普通环境变量
     set GOPATH=/home/zoro/go/src ->系统环境变量
     export GOPATH=/home/zore/go/src ->系统环境变量
     ```

* 位置变量

  > 执行脚本的时候，可以给脚本传递参数，脚本内部使用这些参数就需要使用这些位置变量

  ```shell
  # 执行脚本
  ./test.sh aa bb cc dd ...
  ```

  * $0: 执行的文件名vi吗
  * $1: 第一个参数，aa
  * $2: 第二个参数，bb
  * ....

  ![](http://xwjpics.gumptlu.work/20201005150929.png)

  ![](http://xwjpics.gumptlu.work/20201005151017.png)

  ![](http://xwjpics.gumptlu.work/20201005150951.png)

  **注意多余的参数也传进去了，只是没有使用**

* 特殊变量

  * $#: 获取传递的参数的个数

  * $@: 给脚本传递的所有参数

  * $?: 脚本执行完成之后的状态，成功==0 or 失败!=0

  * $$: 脚本进程执行之后对应的进程ID

    ```shell
    echo "Hello World"
    echo "第一个参数: $0"
    echo "第二个参数: $1"
    echo "第三个参数: $2"
    echo "第四个参数: $3"
    echo "第五个参数: $4"
    echo "传递的所有参数的个数: $#"
    echo "传递的所有参数: $@"
    echo "脚本执行完的状态: $?"
    echo "脚本执行的进程编号: $$"
    ```

    

  ![](http://xwjpics.gumptlu.work/20201005151703.png)

  直接输入`echo $?`查看的是上一个进程的状态

* 普通变量取值

  ```shell
  # 变量定义
  value=123  #默认以字符串处理
  echo $value #打印变量前面要加$
  echo ${value} #这样也可以，但是相对麻烦
  ```

  ![](http://xwjpics.gumptlu.work/20201005152427.png)

* 取命令执行结果的两种方式

  把命令执行的结果用变量保存：

  ```shell
  var=$(shell命令)
  var=`shell命令`
  ```

  

![](http://xwjpics.gumptlu.work/20201005152702.png)

* 引号的使用

  ```shell
  # 双引号
  echo "xxxx $var"  #会将其中变量的值打印输出
  # 单引号
  echo 'xxxx $var'  #会原样输出，不会取变量值
  ```


## 1.5 条件判断和循环

* shell脚本的if条件判断

  ```shell
  # if语句
  if [ 条件判断 ];then
  	逻辑处理 -> shell命令
  	xxxx
  fi
  # =====================或
  if [ 条件判断 ]
  then
  	逻辑处理 -> shell命令
  	xxxx
  fi
  # if ... elif ... fi
  if [ 条件判断 ];then
  	逻辑处理 -> shell命令
  	xxxx
  elif [ 条件判断 ];then
  	shell命令
  	xxx
  elif [ 条件判断 ];then
      shell命令
      xxx
  else
  	shell命令
  fi
  ```

  注意：if和中括号中间有空格，中括号中的条件判断与中括号两边都有空格

  ```shell
  #!/bin/bash
  #对传递到脚本内部的文件名做判断
  if [ -d $1 ];then
          echo "$1 是一个目录"
  elif [ -s $1 ];then
          echo "$1 是一个文件,且文件不为空"
  else
          echo "$1 不是一个文件也不是一个目录"
  fi
  ```

  常用判断条件：

  * 关于文件属性

  ![](http://xwjpics.gumptlu.work/20201005154552.png)

  ![](http://xwjpics.gumptlu.work/20201005155043.png)

  * 关于字符串的判定

  ![](http://xwjpics.gumptlu.work/20201005155131.png)

  * 常见数值判定

    ![](http://xwjpics.gumptlu.work/20201005155245.png)

  * 逻辑运算操作符

    ![](http://xwjpics.gumptlu.work/20201005155338.png)

  

* shell脚本for循环

  ```shell
  # shell中有for、where
  # 语法： for 变量 in 集合; do; done
  #!/bin/bash
  # 对当前目录下的文件进行遍历
  list=$(ls)
  for var in $list;do
          echo "当前文件: $var"
  done
  ~     
  ```

## 1.6 shell脚本中的函数

```shell
# 没有函数的修饰，也没有参数也没有返回值
# 格式
funcName(){
	# 得到第一个参数
	arg1=$1
	# 得到第二个参数
	arg2=$2
	# 得到第三个参数
	arg3=$3
	函数体 ->一系列的shell命令  逻辑循环和判断
}
# 没有参数列表，但是可以传参
# 函数调用
funcName aa bb cc dd
# 函数调用之后的状态
0：成功
非0：失败

```

```shell
#!/bin/bash
# 判断传递进来的文件名是不是目录，如果不是就创建
# 定义函数
funcIsDir(){
        if [ -d $1 ];then
                echo "$1是一个目录"
        else
                # 创建这个目录
                mkdir $1
                if [ 0 -ne $? ];then
                        echo "目录创建失败"
                        exit
                fi
                echo "目录创建成功"
                ls -a
        fi
}
# 函数的调用
funcIsDir $1
```

# 二、技巧

## 1.获取当前工作目录

```shell
BASEDIR="$(dirname $(readlink -f "$0"))"  # 获取当前执行文件的绝对父目录路径
```

## 2.不让命令有任何输出

`[command] > /dev/null 2>&1`

https://www.cnblogs.com/ultranms/p/9353157.html

执行了这条命令之后，该条shell命令将不会输出任何信息到控制台，也不会有任何信息输出到文件中。

* `/dev/null`可以看做是一个黑洞，只写文件所有写入该文件的内容都会丢失，常用于禁止标准输出/输出
* `2>&1` : 1表示标准输出、2表示标准错误输出，`>&` 表示将标准错误输出重定向绑定到标准输出
* 整体：将标准输出、标准错误输出都重定向到了“黑洞”，也就是不处理到终端也不保存到文件，就是不管命令的所有输出 

## 3.获取当前执行次脚本的目录

```shell
SCRIPT_DIR=`realpath ${BASH_SOURCE[0]} | xargs dirname` 
```

## 4.shift获取下一个参数

https://blog.csdn.net/zhu_xun/article/details/24796235

shift命令用于对参数的移动(左移)，通常用于在不知道传入参数个数的情况下依次遍历每个参数然后进行相应处理（常见于Linux中各种程序的启动脚本） 

示例1:依次读取输入的参数并打印参数个数：
`run.sh`:

```shell
#!/bin/bash
while [ $# != 0 ];do
echo "第一个参数为：$1,参数个数为：$#"
shift
done
```

输入如下命令运行：`run.sh a b c d e f`

结果显示如下：
第一个参数为：a,参数个数为：6
第一个参数为：b,参数个数为：5
第一个参数为：c,参数个数为：4
第一个参数为：d,参数个数为：3
第一个参数为：e,参数个数为：2
第一个参数为：f,参数个数为：1
从上可知 `shift `命令每执行一次，变量的个数(`$#`)减一（之前的`$1`变量被销毁,之后的`$2`就变成了`$1`），而变量值提前一位, 同理，shift n后，前n位参数都会被销毁

## 5. if多个嵌套判断条件格式

如果有一个if语句有两个判断条件：

```shell
# 条件1和条件2都为真时执行
if [ 条件1 ] && [ 条件2 ]; then
    # 执行代码
fi
```

如果有其中的一个条件中嵌套了两个条件

```shell
# 当条件1为真且（条件2的子条件1为真或条件2的子条件2为真）时执行
if [ 条件1 ] && { [ 子条件1 ] || [ 子条件2 ]; }; then
    # 执行代码
fi
```

这里必须这么写，具体细节有两个：

* `{}`类似于括号，将两个表达式包揽起来，不能用`[]`
* 大括号表示分组表达式，其中的最后必须要有一分号;
* 大括号前后都必须留有一个空格

## 6. grep去除掉自己显示

`grep -v grep` 是一个用于过滤 `grep` 自身进程的常见命令组合。其用法主要在于当你使用 `ps` 命令查找某个进程时，想要排除 `grep` 命令本身。

`-v` 选项在 `grep` 命令中表示 "invert match"，即它会打印出不匹配给定模式的所有行。

因此，`grep -v grep` 会过滤掉所有包含 "grep" 的行，通常用于从进程列表中排除 `grep` 命令本身的进程。

## 7.巧妙的使用或逻辑操作

xxx || exit 1 

这种写法中的 `||` 是一个逻辑或操作符。在 bash 脚本中，这表示如果左边的命令（`xxx`）执行成功，则整个 `||` 表达式返回 true，不会再执行右边的命令，否则执行右边的命令

注意：左边执行成功为true，执行失败为false，与具体返回值关系不大

## 8. 如何调试shell脚本

1. **使用 `-v`（verbose）选项**: 你可以在运行脚本时使用 `-v` 选项，如 `bash -v script.sh`。这会打印脚本中执行的每一行命令。
2. **使用 `-x`（xtrace）选项**: `-x` 选项也可以用于脚本的调试，如 `bash -x script.sh`。这会在每条命令执行之前，先打印出来。
3. **在脚本中设置 `-x`**: 你可以在脚本中的任何位置添加 `set -x` 来开启跟踪，并用 `set +x` 来关闭跟踪。这样就可以只对脚本的一部分进行跟踪。
4. **使用 `trap` 命令**: `trap` 命令可以在接收到信号后执行相应的命令，常用于在脚本退出时清理临时文件。也可以用来在每个命令后打印堆栈跟踪，帮助查找错误。
5. **使用调试工具**: 有一些专门用于 shell 脚本调试的工具，如 `bashdb`，它提供了类似 gdb 的调试环境。

## 9.条件判断[]与[[]]

1. 字符串比较：在 `[...]` 中，如果你要比较的字符串中包含空格，那么你需要将其用引号括起来。例如`[ "$a" = "$b" ]`。但在 `[[...]]` 中，你不需要这么做。例如`[[ $a = $b ]]` (在 Bash shell 中，`=` 和 `==` 在 `[[ ... ]]` 结构中都可以用来进行字符串比较，他们是等价的)

2. 正则表达式：`[[...]]` 支持正则表达式，而 `[...]` 不支持

   ```shell
   str="123"
   if [[ $str =~ ^[0-9]+$ ]]; then 
       echo "Matched!"
   else 
       echo "Not Matched~"
   fi
   ```

3. 逻辑运算符：`[[...]]` 支持 `&&` 和 `||` 逻辑运算符，而 `[...]` 则需要 `-a` 和 `-o`

4. 文件名展开或者通配符：如果你的表达式中包含文件名展开或者通配符（例如`*`，`?`），`[[...]]` 会把它们当作字符串处理，而 `[...]` 会尝试展开它们

   ​	`?`

   ```shell
   $ str="Hello, world!"
   $ if [[ $str == Hello* ]]; then echo "Matched!"; else echo "Not matched!"; fi
   Matched!
   ```

   ​	`*`

   ```shell
   # 当前目录文件
   $ ls
   file1.txt file2.txt
   
   # 使用 [[ ... ]], 会当作单纯的字符串处理
   $ if [[ "file*.txt" == "file1.txt" ]]; then echo "Matched!"; else echo "Not matched!"; fi
   Not matched!
   
   # 使用 [ ... ]，会展开文件内容，直接出错
   $ if [ "file*.txt" == "file1.txt" ]; then echo "Matched!"; else echo "Not matched!"; fi
   -bash: [: file1.txt: binary operator expected
   ```

   在大多数情况下，推荐使用 `[[...]]`，因为它更健壮，也更易于理解。然而，如果你的脚本需要在不同的shell环境中运行，或者你正在写一个POSIX兼容的脚本，你应该使用 `[...]`，因为 `[[...]]` 是bash和一些其他现代shell的特性，而不是所有shell都支持

5. 
