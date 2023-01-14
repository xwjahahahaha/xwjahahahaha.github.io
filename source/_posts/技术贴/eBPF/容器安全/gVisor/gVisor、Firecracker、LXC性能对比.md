---
title: gVisor、Firecracker、LXC性能对比
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-06-26 22:16:15
---

[TOC]


<!-- more -->

> 参考：
>
> * [SDN x Cloud Native Meetup #46 WebinarFirecracker 與 gVisor 的三兩事](https://www.youtube.com/watch?v=c31fg5gQwHI)
> * [《Blending Containers and Virtual Machines: A Study of Firecracker and gVisor》](https://dl.acm.org/doi/10.1145/3381052.3381315)

# 容器安全技术如何选择？不同人的角度？

开发人员：

* 关注哪个性能最好

平台架构师：

* 全新的架构最小化依赖宿主机

内核工程师：

* 最小化支持改动，最好与上层开发人员无交集

# 性能对比

对比对象：

* 传统Linux Container
* Firecracker
* gVisor

指标：

* 性能
* 与内核绑定关系，可移植性

|                     | 系统调用数量(原本总共有350个)                                |
| ------------------- | ------------------------------------------------------------ |
| Linux Container     | 屏蔽了44个，需要开启权限才可以使用                           |
| Firecracker         | 白名单只有36个，并且如果参数不合规也不可调用                 |
| gVisor(11/2019版本) | sentry已支持237个系统调用代理<br />但是在host networking模式下只允许53个<br />不在host networking模式下只允许68个 |

  访问内核代码量的对比：

![image-20220626211134517](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626211134517.png)

* 因为Firecracker需要使用KVM启动VM，所以在`/virt`、`/arch`等目录都有大量的代码使用
* gVisor项目需要使用KVM来拦截系统调用，所以占了一部分的代码

在CPU方面三者没有多大的差异：

![image-20220626211345362](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626211345362.png)

网络部分：

![image-20220626211542919](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626211542919.png)

* gVisor基本与传统container有相同的网络调用覆盖，作者疑问点在于sentry不是说可以实现网络代理吗，为什么还是会大量使用宿主机网络代码

对于上图的最后一个部分，作者发现大量的代码调用都是包的检查即下左图：

![image-20220626211903489](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626211903489.png)

* 右图就再详细比较了三者对这一块代码的差异，gVisor比LXC少了一点的原因可能在于sentry本身做了一些缓存从而减轻了调用次数，而Firecracker全部调用都是0是因为借助虚拟机内的网络机制没走宿主机

网络传输效能：

![image-20220626212249710](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626212249710.png)

可以明显观察到Firecracker性能和直接主机互通差不多达到10G网卡的效能，但是gVisor的两种模式效能却非常低

 封包延迟：

![image-20220626212832973](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626212832973.png)

* 对于封包延迟来说，LXC明显优于两外两种，明显的原因在于，首先Firecracker时要经过两层内核的封装和解封，而gVisor需要经过sentry的网络栈封装和解封

总结：

* Firecracker： 低内核代码覆盖并且有很优异的性能
* gVisor：低效能并且很高度依赖宿主机内核代码，许多网络处理在sentry但是一些检查还是发生在宿主机内核，换来的是更加好一点的隔离性
* LXC：性能最好，但是隔离性低

内存对比：

![image-20220626213739357](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626213739357.png)

* 代码覆盖上来看gVisor与LXC基本差不多，Firecracker因为在虚拟机环境所以不太一样
* gVisor的sentry层对mmap的系统调用进行了筛选，类似于做了一个缓存，所以gVisor这里性能会优于LXC，减少了呼叫内核的次数

测试申请内存空间和释放空间以及写入的对比：

![image-20220626215543368](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626215543368.png)

* 明显的是gVisor的延迟在小空间的情景下相比于其他耗时非常大，但是随着空间的大小的增加逐渐递减，所以使用gVisor的话最好一次性要较大的空间

![image-20220626215807521](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626215807521.png)

* 同样，写入的情况下，gVisor性能也会差一点

文件读写：

![image-20220626220144307](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626220144307.png)

* gVisor和LXC在ext4的基础上还添加了overlayfs的机制(见图四)

![image-20220626220449994](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626220449994.png)

* Firecracker的俩中模式效能非常好甚至超过了宿主机效能，原因在于虚拟机中的写入磁盘在宿主机的角度来看只是写到到了内存，并没有真正的刷到磁盘中，只是虚拟机中的容器以为写完了，所以性能非常接近写入内存
* gVisor写入小文件时效能较差

![image-20220626221027161](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220626221027161.png)

* 读文件方面：根据Firecracker的文件所在位置不同可以分为三个种类，在宿主机内存，宿主机磁盘，虚拟机内存，总的来说因为在Firecracker的直接读取内存文件（不管是宿主机还是虚拟机）都会更快一些

总的来说，这篇论文针对的是gVisor2019年的版本，不知道这两年是否有所改进，但是其性能消耗上确实不尽人意，所以：
* 如果是使用gVisor建议将资源管道直接开大一点，这样性能效率会好一些，资源需求大的业务场景也更加合适
* 如果追求效率，目前的微内核解决方案效率上应该都是比较优秀的
