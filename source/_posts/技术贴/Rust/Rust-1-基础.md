---
title: Rust-1-基础
tags:
  - rust
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-04-25 10:11:55
---

# 一、学习资源/参考

> https://www.rust-lang.org/zh-CN/learn/get-started
>
> https://www.jianshu.com/p/876b1cca26d8
>
> https://doc.rust-lang.org/book/

<!-- more -->

# 二、基本概念

![D1PvzH](http://xwjpics.gumptlu.work/qinniu_uPic/D1PvzH.png)

### Rustup：Rust安装器和版本管理工具

安装 Rust 的主要方式是通过 Rustup 这一工具，它既是一个 **Rust 安装器又是一个版本管理工具。**

### Cargo：Rust 的构建工具和包管理器

您在安装 Rustup 时，**也会安装 Rust 构建工具和包管理器**的最新稳定版，即 **Cargo**。Cargo 可以做很多事情：

- `cargo build` 可以构建项目
- `cargo run` 可以运行项目
- `cargo test` 可以测试项目
- `cargo doc` 可以为项目构建文档
- `cargo publish` 可以将库发布到 [crates.io](https://crates.io/)。

# 三、基本准备

## 安装

`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

## 换源

创建一个`config`文件,可以在以下目录:

- 用户目录`.cargo`文件夹
- 与`Cargo.toml`同级目录`.cargo`文件夹下创建`config`文件
- 环境变量 `CARGO_HOME` 指定的目录中 如： `D:\rust\cargo\config.toml`

文件内容:

```toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
# 指定镜像
replace-with = 'sjtu'

# 清华大学
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"

# 中国科学技术大学
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"

# 上海交通大学
[source.sjtu]
registry = "https://mirrors.sjtug.sjtu.edu.cn/git/crates.io-index"

# rustcc社区
[source.rustcc]
registry = "https://code.aliyun.com/rustcc/crates.io-index.git"
```

