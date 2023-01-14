---
title: bcc环境构建问题总结
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-27 21:01:29
---

[TOC]


<!-- more -->

> 参考：
>
> * https://github.com/iovisor/bcc/issues/3881

# 一、BCC工具源码编译时遇到的问题总结

## 1. 编译`bcc`时`cmake Warning`

![image-20220519110150982](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519110150982.png)

原因：缺少依赖

解决：这些错误其实都不影响后面的编译（可能部分功能会影响，例如`debug`），同样会构建成功

> 相关参考`issue`:
>
> * https://github.com/iovisor/bcc/issues/3601
> * https://github.com/isage/lua-imagick/issues/16

```shell
sudo apt install libdebuginfod-dev    # 我尝试了并不能直接下载
sudo apt install libluajit-5.1-dev
sudo apt install arping netperf
```

## 2. make`编译阶段提示`getName()`函数不带参数，而`llvm-6.0`版本中的调用却带参数

![image-20220517202125777](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517202125777.png)

原因：使用的`llvm`的版本太低了，我的版本是`6.0.0`

![image-20220517202400162](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517202400162.png)

根据issue，bcc将逐渐不支持旧版本的`llvm`，所以现在最好升级到`11.0`以上([相关issue](https://github.com/iovisor/bcc/issues/3808))

同样的问题`issue`：https://github.com/iovisor/bcc/issues/3881

**解决方法（二选一）：**

1. 升级`llvm`的版本（一劳永逸）

   几种安装方式都行：

   > 参考自：https://zhuanlan.zhihu.com/p/102028114

   - 源码安装方式

     ```shell
     # 采用源码安装方式
     cd /usr/local
     wget https://github.com/llvm/llvm-project/releases/download/llvmorg-13.0.0/llvm-project-13.0.0.src.tar.xz
     tar xvf llvm-project-13.0.0.src.tar.xz
     mv llvm-project-13.0.0.src/ llvm
     cd llvm
     mkdir build && cd build
     # cmake生成编译信息
     cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS="clang;lldb" -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_BUILD_TYPE="Release" -DLLVM_INCLUDE_TESTS=OFF -DCMAKE_INSTALL_PREFIX="/usr/local/llvm" ../llvm
     # 编译
     make     # 注意：这里会非常慢，耐心等待
     # 安装
     make install
     ```

       其中因为我的`cmake`版本过低，所以要先更新一下`cmake`

       ![image-20220517205409738](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220517205409738.png)

       详细见：https://blog.csdn.net/u013925378/article/details/106945800

     ---

     `make`编译过程中到这一步就会失败：

     ![image-20220518111936362](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220518111936362.png)

     并且会导致我的服务器网络出现问题无法连接，最后被`kill`掉

     现象：这里的编译可能会重启服务器的网卡/网络，因为会导致与服务器的ssh链接断开

     原因：目前还未解决，尝试使用第二种方式

   - 使用官方脚本安装

     ```shell
     wget https://apt.llvm.org/llvm.sh
     chmod +x llvm.sh
     # 后面的参数是版本号
     sudo ./llvm.sh 11
     ```

2. download the release [0.24.0](https://github.com/iovisor/bcc/releases/tag/v0.24.0), it will compile successfully. （临时解决）

   ```shell
   wget https://github.com/iovisor/bcc/releases/download/v0.24.0/bcc-src-with-submodule.tar.gz
   tar xvf bcc-src-with-submodule.tar.gz 
   mkdir build && cd build
   cmake ..
   make
   sudo make install
   cmake -DPYTHON_CMD=python3 .. # build python3 binding
   # 将目录添加到目录堆栈顶，简单理解就是进入这个目录，和popd一起使用
   pushd src/python/   
   make
   # 安装到/usr/lib/python3/dist-packages/下
   sudo make install
   popd	# 返回栈顶目录
   ```

## 3. 编译`bcc`的时候`cmake ..`失败

![image-20220519191926700](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519191926700.png)

原因：检查`ls /usr/lib/llvm-13/lib/`下是否有如下所需要的依赖（`.a`后缀）

![image-20220519192513782](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220519192513782.png)

如果没有的话安装`sudo apt-get install libclang-13-dev` (中间的版本号对应你的`clang`版本)

## 4. make时报错 error: no matching function for call to

在0%的时候就开始报错：

`**error:** no matching function for call to ‘**std::__cxx11::basic_string<char>::replace(long unsigned int, long unsigned int, std::__cxx11::basic_string<char>&, int)**’`

![image-20220727210512089](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220727210512089.png)

下面的错误都是如此提示，全都是参数个数不对

首先怀疑的就是版本问题，发现上一步`cmake`使用的c/c++的编译器是`cc`和`c++`，而不是`g++`

![image-20220727210704080](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220727210704080.png)

解决：

```shell
export CC=/usr/local/bin/gcc
export CXX=/usr/local/bin/g++
cmake ..							# 在cmake之前指定好编译器
```

![image-20220727210838979](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220727210838979.png)
