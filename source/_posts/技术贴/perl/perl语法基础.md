---
title: perl语法基础
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-13 10:57:22
---

> 学习自：
>
> * https://perldoc.perl.org/perlintro#What-is-Perl?
> * https://www.w3cschool.cn/perl/perl-environment.html

记录学习**Perl（5.34.0）**基础语法重点

> 源码安装Perl：
>
> 下载：https://www.perl.org/get.html
>
> 过程：https://cloud.tencent.com/developer/article/1889268

<!-- more -->

# 一、语言特点

* 一种脚本语言，动态语言，适合文本处理
* 核心面向：面向实际（易于使用、效率、编译）而不是美观（精简），代码比较冗余

# 二、基础语法
## 命令工具使用

可以用perl直接执行perl代码

|     选项      |          描述           |
| :-----------: | :---------------------: |
| -d[:debugger] |  在调试模式下运行程序   |
|  -Idirectory  | 指定 @INC/#include 目录 |
|      -t       |      允许污染警告       |
|      -U       |     允许不安全操作      |
|      -w       |   允许很多有用的警告    |
|      -W       |      允许所有警告       |
|      -X       |      禁用使用警告       |
|  -e program   |   执行一行 perl 代码    |
|     file      |   执行 perl 脚本文件    |

## 注意点

* 双引号内容会转义，单引号内容不会转义（类似于shell）

  ```perl
  print "Hello, $name\n";     # works fine
  print 'Hello, $name\n';     # prints $name\n literally
  ```

## 变量

### 1. scalars、arrays、hashes

* scalars：代表单个值, 以`$`开头

  * 虽然类型可变，但是第一次使用前面必须加上一个`my`关键字，并且变量名前面必须有`$`
      
      `my`关键字也限制了这个变量的作用域在一个块`{}`中，如果不加那么就是一个全局变量
      
      ```perl
      my $animal = "camel";
      my $answer = 42;
      ```
      
  * `$_`表示默认变量

* arrays：一系列值，以`@`开头

  * 数组支持不同类型值，定义时前面必须加上`@`表示这是一个数组

    ```perl
    my @animals = ("camel", "llama", "owl");
    my @numbers = (23, 42, 69);
    my @mixed   = ("camel", 42, 1.23);
    ```

  * `$#array`表示数组的最后一个值的下标，所以`$#array+1`可以表示数组长度，但是在`if`语句的比较中可以直接使用`@array`表示数组长度：

    `if (@animals < 5) { ... }`

  * 不论是数组还是标量scalars，想要获取单个值都是使用`$`

    ```perl
    print $animals[0]; 
    print $answer;
    ```

  * 数组的切片

    ```perl
    @animals[0,2];                 # gives ("camel", "owl"); 逗号表示单独的两个下标
    @animals[0..2];                # gives ("camel", "llama", "owl"); ".."表示连续的切片
    @animals[1..$#animals];        # gives all except the first element 
    ```

  * 也可以用一些api函数对数组操作

    ```perl
    my @sorted    = sort @animals;
    my @backwards = reverse @numbers;
    ```

* hashes：无序的k/v对，以`%`开头

  ```perl
  my %fruit_color = ("apple", "red", "banana", "yellow");   	# 用逗号分隔
  my %fruit_color = (											# 或者=>
      apple  => "red",
      banana => "yellow",
  );
  $fruit_color{"apple"};           # gives "red"
  # 获取所有key和value
  my @fruits = keys %fruit_color;	
  my @colors = values %fruit_color;
  ```

  * 散列可以嵌套

    ```perl
    my $variables = {
        scalar  =>  {
                     description => "single item",
                     sigil => '$',
                    },
        array   =>  {
                     description => "ordered list of items",
                     sigil => '@',
                    },
        hash    =>  {
                     description => "key/value pairs",
                     sigil => '%',
                    },
    };
    # 多重引用用->获取值
    print "Scalars begin with a $variables->{'scalar'}->{'sigil'}\n";
    ```


### 2. perlvar预定义变量

* 变量命名规则：以字母/下划线开头，中间可以用数字、字母下划线；
  * 特殊在于：变量名中间可以使用特殊的`::`和`'`

## 注释

多行注释

```perl
=zhushi
这是一个多行注释
这是一个多行注释
这是一个多行注释
这是一个多行注释
=cut
```

注意：

* 开始的`=`之后必须跟一个字符

## 条件判断、While、for、foreach

```perl
if ( condition ) {
    ...
} elsif ( other condition ) {
    ...
} else {
    ...
}
```

```perl
# the traditional way
if ($zippy) {
    print "Yow!";
}

# the Perlish post-condition way  一条语句简写的方法
print "Yow!" if $zippy;
print "We have no bananas" unless $bananas;
```

```perl
while ( condition ) {
    ...
}
```

```perl
print "LA LA LA\n" while 1;          # loops forever
```

```perl
for ($i = 0; $i <= $max; $i++) {
    ...
}

# 一般使用foreach
foreach (@array) {
    print "This element is $_\n";
}

print $list[$_] foreach 0 .. $max;

# you don't have to use the default $_ either...
foreach my $key (keys %hash) {
    print "The value of $key is $hash{$key}\n";
}
```

## 内置函数

[perlfunc](https://perldoc.perl.org/perlfunc)







# 三、文件

按照惯例

* `.pm` 应该保存 Perl Module，也就是 Perl 模块。例如 Socket.pm
* `.pl `应该保存 Perl Library，也就是 Perl 库文件。例如 perldb.pl
* `.plx`应该保存 Perl 脚本。

实际上大家都习惯用 .pl 来保存 Perl 脚本。
