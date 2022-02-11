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

  * $?: 脚本执行完成之后的状态，成功==0 or 失败！=0

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

# 小技巧

**1.获取当前工作目录**

```shell
BASEDIR="$(dirname $(readlink -f "$0"))"  # 获取当前执行文件的绝对父目录路径
```

**2.不让命令有任何输出**

`[command] > /dev/null 2>&1`

https://www.cnblogs.com/ultranms/p/9353157.html

执行了这条命令之后，该条shell命令将不会输出任何信息到控制台，也不会有任何信息输出到文件中。

* `/dev/null`可以看做是一个黑洞，只写文件所有写入该文件的内容都会丢失，常用于禁止标准输出/输出
* `2>&1` : 1表示标准输出、2表示标准错误输出，`>&` 表示将标准错误输出重定向绑定到标准输出
* 整体：将标准输出、标准错误输出都重定向到了“黑洞”，也就是不处理到终端也不保存到文件，就是不管命令的所有输出 

**3.获取当前执行次脚本的目录**

```shell
SCRIPT_DIR=`realpath ${BASH_SOURCE[0]} | xargs dirname` 
```

