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
> * 基础手册：https://perldoc.perl.org/perlintro#What-is-Perl?
> * https://www.w3cschool.cn/perl/perl-environment.html
> * https://www.runoob.com/perl/perl-special-variables.html

记录学习**Perl（5.34.0）**基础语法重点

> 源码安装Perl：
>
> 下载：https://www.perl.org/get.html
>
> 过程：https://cloud.tencent.com/developer/article/1889268

<!-- more -->

[TOC]

# 一、语言特点

* 一种脚本语言，动态语言，适合文本处理
* 核心面向：面向实际（易于使用、效率、编译）而不是美观（精简），代码比较冗余

# 二、基础使用与语法
## 2.1 文件后缀

按照惯例

* `.pm` 应该保存 Perl Module，也就是 Perl 模块。例如 Socket.pm
* `.pl `应该保存 Perl Library，也就是 Perl 库文件。例如 perldb.pl
* `.plx`应该保存 Perl 脚本。

实际上大家都习惯用 .pl 来保存 Perl 脚本。

## 2.2 命令工具使用

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

## 2.3 引号

* 双引号内容会转义，单引号内容不会转义（类似于shell）

  ```perl
  print "Hello, $name\n";     # works fine
  print 'Hello, $name\n';     # prints $name\n literally
  ```

* 如果想在双引号中使用双引号，为了避免转义的麻烦，可以使用以下方式：

  ```perl
  print qq{hello, "world"};   # qq就代表了两个双引号
  print q{hello, "world"};	# q代表一个引号
  ```

## 2.4 变量

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

### 2. perlvar预定义变量/特殊变量

常见整理：

* `$_`: 包含了默认输入和模式匹配内容。

  * 在一些情况下不写，默认也会将其用作参数([详见](https://perldoc.perl.org/perlvar#NAME))

      ```perl
      foreach ('Google','Runoob','Taobao') {
          print;				# 等价于print $_;
          print "\n";
      }
      ```

  * `$_`：是一个全局变量

* `$!`: 即`$OS_ERROR`的简写，系统操作字符串

* `@_`: 传入子程序的所有参数

其他手动查询：

* https://www.runoob.com/perl/perl-special-variables.html
* https://perldoc.perl.org/perlvar#NAME

## 2.5 注释

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

## 2.6 条件判断、While、for、foreach、given(switch)

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
    next;		# 类似于continue
    last;		# 类似于break
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

```perl
given($age) {
    when($_ > 18) {
        say "you are more than 18";
        continue;
    }
    when($_ <= 18) {
        say "you are less than 18";
        continue;
    }
    default{}
}
```

## 2.7 文件操作

打开文件句柄

```perl
open(my $in,  "<",  "input.txt")  or die "Can't open input.txt: $!";    # 只读模式
open(my $out, ">",  "output.txt") or die "Can't open output.txt: $!";	# 写模式
open(my $log, ">>", "my.log")     or die "Can't open my.log: $!";		# 附加模式（没有会创建）
```

您可以使用`<>`运算符从打开的文件句柄中读取。在标量上下文中，它从文件句柄中读取一行，在列表上下文中，它读取整个文件，将每一行分配给列表的一个元素：

```perl
my $line  = <$in>;				# 读取一行
my @lines = <$in>;				# 读取全部
```

```perl
while (<$in>) {     # assigns each line in turn to $_
    print "Just read in this line: $_";					# 循环读取
}
```

`print()`也可以采用可选的第一个参数来指定要打印到的文件句柄：

```perl
open(my $out, ">>", "output.txt") or die "can't open output.txt:$!";
my @words = ("hello", "word");
my $word = "\nhahaha\n";
print $out @words;
print $out $word;
```

使用完文件句柄必须要`close`

```perl
close $in or die "$in: $!";
```

## 2.8 子程序/函数

```perl
sub average {
    print "传入的所有参数为：@_\n";			# @_就是传入的所有参数
    my $len = scalar(@_);
    print "参数的个数为：$len\n";
    my $sum = 0;
    for (@_) {
        $sum += $_;
    }
    my $avg = $sum / $len;
    return $avg  # 可以没有返回值
}

print average(1, 3, 6, 9, 12), "\n";
```

## 2.9 运算符

| 比较符 | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| lt     | 检查左边的字符串是否小于右边的字符串，如果是返回 true，否则返回 false。 |
| gt     | 检查左边的字符串是否大于右边的字符串，如果是返回 true，否则返回 false。 |
| le     | 检查左边的字符串是否小于或等于右边的字符串，如果是返回 true，否则返回 false。 |
| ge     | 检查左边的字符串是否大于或等于右边的字符串，如果是返回 true，否则返回 false。 |
| eq     | 检查左边的字符串是否等于右边的字符串，如果是返回 true，否则返回 false。 |
| ne     | 检查左边的字符串是否不等于右边的字符串，如果是返回 true，否则返回 false。 |
| cmp    | 如果左边的字符串大于右边的字符串返回 1，如果相等返回 0，如果左边的字符串小于右边的字符串返回 -1。 |

## 2.10 各类内置函数

https://perldoc.perl.org/perlfunc

## 2.11 state（静态变量）

```perl
sub increment {
    state $times = 0;
    $times ++;
    say "time is $times";
} 

increment();    #1
increment();    #2
```

只初始化一次

# 三、模块mod

## package

Perl 中每个包有一个单独的符号表，定义语法为：

```perl
package mypack;
```

此语句定义一个名为 **mypack** 的包，在此后定义的所有变量和子程序的名字都存贮在该包关联的符号表中，直到遇到另一个 **package** 语句为止。

每个符号表有其自己的一组变量、子程序名，各组名字是不相关的，因此可以在不同的包中使用相同的变量名，而代表的是不同的变量。

从一个包中访问另外一个包的变量，可通过" 包名 + 双冒号( :: ) + 变量名 " 的方式指定。

```perl
#!/usr/bin/perl
# 默认就在main包下
$index = 1;
print "包名: ", __PACKAGE__, ": $index\n";


package Foo;
$index = 10;
print "包名: ", __PACKAGE__, ": $index\n";

package main;   # 重置main包中的变量
$index = 100;
print "包名: ", __PACKAGE__, ": $index\n";
print "在包", __PACKAGE__, "中调用Foo包的变量: $Foo::index\n"; # 注意，调用包的变量必须是全局变量
```

Perl5 中用Perl包来创建模块。

Perl 模块是一个可重复使用的包，**模块的名字与包名相同**，定义的文件后缀为 **.pm**。

以下我们定义了一个模块 `Foo.pm`，代码如下所示：

```perl
#!/usr/bin/perl
 
package Foo;
sub bar { 
   print "Hello $_[0]\n" 
}
 
sub blat { 
   print "World $_[0]\n" 
}
1;
```

## Begin、End

Perl语言提供了两个关键字：**BEGIN，END**。它们可以分别包含一组脚本，用于程序体运行前或者运行后的执行。

语法格式如下：

```perl
BEGIN { ... }
END { ... }
```

- 每个 **BEGIN** 模块在 Perl 脚本载入和编译后但在其他语句执行前执行。
- 每个 **END** 语句块在解释器退出前执行。
- **BEGIN** 和 **END** 语句块在创建 Perl 模块时特别有用。

## 模块调用

模块的引用可以通过 **require** 函数或者**use**来调用

下面就是`main.pm`

```perl
package main;

require Foo;
Foo::bar("wold");
Foo::blat("bbbbb");
```

注意： 

* **@INC** 是 Perl 内置的一个特殊数组，它包含指向库例程/pm模块所在位置的目录路径。
* 所以我们默认执行的模块文件路径需要先加入到@INC，否则编译器找不到
* 

# 速查

* 关键字/内置函数：[perlfunc](https://perldoc.perl.org/perlfunc)
