---
title: eBPF-6-bcctools专项分析-profile
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-24 10:55:38
---

[TOC]

<!-- more -->

> 参考：
>
> * https://linux.extremeoverclocking.com/man/8/bcc-profile
> * https://luobuda.github.io/2017/04/23/System-map%E5%92%8Ckallsyms%E6%96%87%E4%BB%B6/

## 1. 输出调用栈中的unknown

[bcc-profile(8)](https://linux.extremeoverclocking.com/man/8/bcc-profile)给出了解决`unknown`帧的部分方法:

* 先使用`perf`工具初步分析，然后在使用`profile`

  具体例子如下，然后检查`unknown frames`

  ```shell
  $ sudo perf record -F 49 -a -g 
  $ sudo perf script
  $ rm -rf perf.data
  ```

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220524104127586.png" alt="image-20220524104127586" style="zoom:50%;" />

  * `-F 49`： 表示使用`49`频率检查
  * `-a`: 所有`cpu`
  * `-g`: 调用栈包括内核、用户空间
  * ` sleep 1; perf script`：等待一秒，等`perf record`采集完写入到`perf.data`文件后，使用`perf script`将调用栈从该文件中读取输出

* 一般来说最为主要的原因为目标软件<font color='#e54d42'>没有使用帧指针(`frame pointers`)编译</font>, 所以采用帧指针的采样方式就无法获取到调用栈的某些部分；一个解决方式是使用编译器使用帧指针方式重新编译: `eg, gcc -fno-omit-frame-pointer, or Java's -XX:+PreserveFramePointer.`

* `“[unknow]”`帧的另一个原因是` JIT `编译器，它不使用传统的符号表。解决方法编写符号表写入到` /tmp/perf-PID.map` 文件，此文件将会被`profile`工具读取。
