---
title: go项目cli与配置文件：cobra与viper项目
tags:
  - golang
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-10-07 19:47:49
---

> 参考资料：
>
> * https://github.com/spf13/cobra
> * https://github.com/spf13/viper
>
> * https://github.com/spf13/cobra/blob/master/user_guide.md

作者[spf13](https://github.com/spf13)有两个明星项目—cobra & viper （眼睛蛇与蝮蛇）

![cobra和viper的代表图案](http://xwjpics.gumptlu.work/qinniu_uPic/zTvvSe.png)

能够帮助我们的go项目添加cmd应用以及读取初始的配置文件，并且两者还可以高效配合使用。

本文就记录了一下这两个项目的基本使用方式（也是个人使用的记录），方便快速上手，如果想更详细了解的话可以去官网查看详细的文档.

<!-- more -->

[TOC]

# 一、Cobra

Cobra是一个库，提供了一个简单的界面，可以创建类似于git和go工具的强大的现代CLI界面。  Cobra也是一个应用程序，它可以生成应用程序脚手架来快速开发基于Cobra的应用程序。

## 1.1 安装

```shell
go get -u github.com/spf13/cobra
```

## 1.2 使用

### 1.2.1 架构

项目一般架构：

* cmd： 目录下存放你的cli各种命令（根命令、子命令等）
* main.go: 项目入口函数
* Init.go: 初始化函数

```shell
  ▾ appName/
    ▾ cmd/
        add.go
        your.go
        commands.go
        here.go
        root.go
      main.go
      init.go
```

`root.go`编写go项目可执行程序的根命令(以`hugo`为例)

```go
var rootCmd = &cobra.Command{
  Use:   "hugo",
  Short: "Hugo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at http://hugo.spf13.com`,
  Run: func(cmd *cobra.Command, args []string) {
    // Do Stuff Here
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}
```

`main.go`一般只需要调用根目录的执行函数即可。

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

### 1.2.2 添加子命令

添加一个子命令`add`在`add.go`中：

```go
var add = &cobra.Command{
  Use:   "add [Person]",
  Short: "add one item",
  Long:  `add one item`,
  Args: cobra.ExactArgs(1),													// 指定该命令的参数输入个数 
  Run: func(cmd *cobra.Command, args []string) {		// RunE表示可以返回一个错误
    // todo
  },
}
```

一般在在项目的`init.go`中的`init`函数中初始化操作：在root命令下添加这个子命令(可以多行添加多个)

```go
func init() {
  cmd.rootCmd.AddCommand(add)
}
```

### 1.2.3 添加flag

flag有两种类型：

1. Persistent Flags: 持久型的Flag，即设置在根命令上，其本身以及所有子命令都会有这个全局Flag

   ```go
   rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")
   ```

2. Local Flags: 这种flag只应用在特定的命令上

   ```go
   localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
   ```

Flag 默认是可选的，但是有的命令我们希望其变为必选，可以这样操作：

```go
// Local Flags
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
// Persistent Flags
rootCmd.PersistentFlags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkPersistentFlagRequired("region")
```

# 二、Viper

Viper是一个完整的Go应用**配置解决方案**，包括12-Factor应用。它被设计用于在应用程序中工作，可以处理**所有类型**的配置需求和格式。

## 2.1 安装

```shell
go get github.com/spf13/viper
```

## 2.2 使用

### 2.2.1 设置默认值

一个好的项目需要在引入配置文件之前使用一些默认值，可以通过以下方式构建

```go
viper.SetDefault("ContentDir", "content")
viper.SetDefault("LayoutDir", "layouts")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```

### 2.2.2 读取配置文件

支持的配置文件格式：` JSON, TOML, YAML, HCL, envfile and Java properties config files`

Viper可以搜索多个路径，但目前单个Viper实例只支持单个配置文件。

具体的使用规则如下：

```go
// 加载配置文件之前，设置一些配置
viper.SetConfigName("config") // 配置文件名称
viper.SetConfigType("yaml") // 配置文件的拓展名
viper.AddConfigPath("/etc/appname/")   // 配置文件的路径
viper.AddConfigPath("$HOME/.appname")  // 可设置多个搜索路径
viper.AddConfigPath(".")               // 可选在当前工作空间中查找
err := viper.ReadInConfig() // 找到并加载配置文件
if err != nil { // 处理错误
	panic(fmt.Errorf("Fatal error config file: %w \n", err))
}
```

### 2.2.3 写配置文件

有时您希望存储在运行时所做的所有修改。为此，可以使用一些命令，每个命令都有自己的用途:

* WriteConfig——如果存在，则将当前viper配置写入预定义路径。如果没有预定义的路径，则返回错误。将覆盖当前配置文件，如果它存在。 

* SafeWriteConfig——将当前的viper配置写入预定义的路径。如果没有预定义的路径，则返回错误。将不会覆盖当前配置文件，如果它存在。

* WriteConfigAs—将当前viper配置写入给定的文件路径。将覆盖给定的文件，如果它存在。 
* SafeWriteConfigAs—将当前的viper配置写入给定的文件路径。如果给定的文件存在，则不会覆盖该文件。

所有标记为Safe的函数都不会覆盖任何文件，如果不存在，就创建，而默认行为是创建或截断。

```go
viper.WriteConfig() // 将viper中的配置保存到'viper.AddConfigPath()' and 'viper.SetConfigName'预定义的配置文件中
viper.SafeWriteConfig()
viper.WriteConfigAs("/path/to/my/.config")
viper.SafeWriteConfigAs("/path/to/my/.config") // will error since it has already been written
viper.SafeWriteConfigAs("/path/to/my/.other_config")
```

### 2.2.4 运行时读取配置文件

Viper支持让应用程序**在运行时实时读取配置文件的功能**。  需要重新启动服务器以使配置生效的日子已经一去不复返了，使用viper的应用程序可以在运行时读取配置文件的更新，不会错过任何信息。  

只需告诉viper实例watchConfig。也可以在**每次发生更改**时提供一个函数让Viper运行。

确保在调用WatchConfig()之前添加了所有的configPaths

```go
viper.OnConfigChange(func(e fsnotify.Event) {
	fmt.Println("Config file changed:", e.Name)
})
viper.WatchConfig()
```

### 2.2.5 从IO流中读取配置文件

```go
viper.SetConfigType("yaml") // or viper.SetConfigType("YAML")

// any approach to require this configuration into your program.
var yamlExample = []byte(`
Hacker: true
name: steve
hobbies:
- skateboarding
- snowboarding
- go
clothing:
  jacket: leather
  trousers: denim
age: 35
eyes : brown
beard: true
`)

viper.ReadConfig(bytes.NewBuffer(yamlExample))

viper.Get("name") // this would be "steve"
```

### 2.2.6 重设值与覆盖、键使用别名

```go
viper.Set("Verbose", true)
viper.Set("LogFile", LogFile)


// 注册
viper.RegisterAlias("loud", "Verbose")

viper.Set("verbose", true) // same result as next line
viper.Set("loud", true)   // same result as prior line

viper.GetBool("loud") // true
viper.GetBool("verbose") // true
```

### 2.2.7 Get值

Viper提供的所有Get值的方法：

- `Get(key string) : interface{}`
- `GetBool(key string) : bool`
- `GetFloat64(key string) : float64`
- `GetInt(key string) : int`
- `GetIntSlice(key string) : []int`
- `GetString(key string) : string`
- `GetStringMap(key string) : map[string]interface{}`
- `GetStringMapString(key string) : map[string]string`
- `GetStringSlice(key string) : []string`
- `GetTime(key string) : time.Time`
- `GetDuration(key string) : time.Duration`
- `IsSet(key string) : bool`
- `AllSettings() : map[string]interface{}`

需要注意的一件重要的事情是，如果没有找到，每个Get函数将返回一个零值。为了检查给定的键是否存在，提供了IsSet()方法。

获取多重嵌套的Key、数组：

```json
{
    "host": {
        "address": "localhost",
        "ports": [
            5799,
            6029
        ]
    },
    "datastore": {
        "metric": {
            "host": "127.0.0.1",
            "port": 3099
        },
        "warehouse": {
            "host": "198.0.0.1",
            "port": 2112
        }
    }
}

GetInt("host.ports.1") // returns 6029
```

> <font color='#39b54a'>注意： 如果有与上述方法中`.`取健相同，那么就使用该健</font>
>
> ```go
> {
>     "datastore.metric.host": "0.0.0.0",
>     "host": {
>         "address": "localhost",
>         "port": 5799
>     },
>     "datastore": {
>         "metric": {
>             "host": "127.0.0.1",
>             "port": 3099
>         },
>         "warehouse": {
>             "host": "198.0.0.1",
>             "port": 2112
>         }
>     }
> }
> 
> GetString("datastore.metric.host") // returns "0.0.0.0"
> ```

viper还支持远程在key/value数据库中读取配置，还可以读取环境变量等等高级功能，如果有需求的话可以看官方文档。

# 三、结合使用

结合使用其实也就是通过viper读取配置文件，然后给cobra的flag使用

```go
serverCmd.Flags().Int("port", 1138, "Port to run Application server on")
viper.BindPFlag("port", serverCmd.Flags().Lookup("port"))
```

