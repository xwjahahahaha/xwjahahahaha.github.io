---
title: kube-eagle-问题挑战总结
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-06-14 10:26:23
---

[TOC]

<!-- more -->

# 一、难点

## 1. Profile的数据怎么存、数据格式设计

当前profile数据的原始格式就是:

```shell
funcA;funcB;funC 2
funcD;funcE 5
```

> https://github.com/jlfwong/speedscope/wiki/Importing-from-custom-sources#brendan-greggs-collapsed-stack-format

这样原始的格式对于`speedscope`已经支持，但是存储的话最好还是使用json格式

所以目前采用的就是json存储的方式，大致的格式如下：

```json
export interface SampledProfile extends IProfile {
  type: ProfileType.SAMPLED

  // Name of the profile. Typically a filename for the source of the profile.
  name: string

  // Unit which all value are specified using in the profile. 单位
  unit: ValueUnit

  // The starting value of the profile. This will typically be a timestamp.
  // All event values will be relative to this startValue. 开始时间
  startValue: number

  // The final value of the profile. This will typically be a timestamp. This
  // must be greater than or equal to the startValue. This is useful in
  // situations where the recorded profile extends past the end of the recorded
  // events, which may happen if nothing was happening at the end of the
  // profile.
  endValue: number				// 结束时间

  // List of stacks
  samples: SampledStack[]	

  // The weight of the sample at the given index. Should have
  // the same length as the samples array.
  weights: number[]
}
```

## 2. Profile细粒度控制

不管是整个namespace还是一个deployment、pod最终对应到的都是进程

所以对于eagle-server要做的就是将请求的k8s定义的上层资源解析到对应主机的对应进程（通过client-go请求api-server）

* server的工作就是首先确定到pod层面的请求，然后通过查询pod所属的主机ip以及其所有的容器id，下发到各个主机
* 各个主机收到请求后通过docker-client将容器id转换为所有/主进程，然后交给profiler去执行

然后再将所有进程的结果汇总到server中，再由server负责绘制火焰图输出

## 3. 架构设计？

![image-20220803215529685](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220803215529685-20220806162554169.png)

* eagle-agent：作为Daemonset封装定义了不同类别的profiler(BPF、Async-profiler、py-spy)，根据程序种类进行profile；分别适配了裸机环境、容器docker环境、k8s环境的进程/容器/pod自动周期性采集
* eagle-server：接受客户端请求下发任务，聚合多个agent的profile结果，生成火焰图
* 数据存储：redis: 缓存任务队列，检测任务冲突; tsdb: 持久化运行存储profile数据，提供历史时序profile火焰图原始数据
* kubectl eagle：基于kubectl插件开发的eagle客户端，用于部署组建与服务交互

### 为什么这样设计？

一期到二期的主要是为了让历史数据持久化存储，这样可以持续监控整个集群，对于历史突发的高资源使用情况进行记录和排查

## 4. 容器环境下的挑战

容器化agent：

* 需要开启如下权限：

  * apparmor：取消加载apparmor限制的配置，为了能够访问`/sys/fs/ebpf `和` /sys/kernel/debug/tracing`

    ```yaml
    annotations:
            container.apparmor.security.beta.kubernetes.io/eagle-agent: "unconfined"
    ```

  * namespace隔离：

    pid、network不与宿主机隔离，因为对于pid来说避免转换，对于网络来说拥有宿主机网络对`Async_profiler`更有用

    ```yaml
    spec:
          hostPID: true
          hostNetwork: true
    ```

  * 以root权限运行，目前bpf的所有程序都需要访问内核相关的头文件或者一些特权调用，所以必须使用root运行，还要和docker-server交互

    ```yaml
    securityContext:
              runAsUser: 0    # 以root权限运行,因为要与docker-server交互
    ```

  * capability 能力：

    * 网络相关：
      * `NET_ADMIN`: 创建网络相关的bpf程序的时候(`BPF_PROG_TYPE_CGROUP_SKB`)，当需要`attach`的时候要创建`bpf_link`(调用`link_create`函数)，bpf会检查是否有此权限`CAP_NET_ADMIN`
      * `NET_RAW`: 打开一些原生的`socker`，需要这样的权限
    * 系统相关：
      * `SYS_ADMIN`：因为内部调用`bpf_get_map_fd_by_id()`需要此权限，`python-bcc`在将`map`数据结构的id转换为fd的时候明确需要此能力
      * `SYSLOG`：地址转换的时候需要访问内核符号表`/proc/kallsyms`
      * `SYS_PTRACE`：当使用`ptrace`的时候就会使用此权限，py-spy也会使用此权限
      * `SYS_RESOURCE`：设置一些软资源限制的能力（[`setrlimit`系统调用](https://man7.org/linux/man-pages/man2/getrlimit.2.html)）
      * `SYS_MODULE`：加载一些内核头模块
    * 进程通信相关：
      * `IPC_LOCK`：进程之间通信不会收到内存软限制的影响

    ```yaml
    capabilities:
      add:
        - NET_ADMIN
        - NET_RAW
    
        - SYS_ADMIN
        - SYSLOG
        - SYS_PTRACE
        - SYS_RESOURCE
        - SYS_MODULE
    
        - IPC_LOCK
    ```

  * 需要挂载的宿主机目录：

    * `/lib/modules`:  内核头文件映射的地方
    * `/sys/kernel/debug`：tracing文件交互
    * `/sys/fs/cgroup`、`/sys/fs/ebpf`这些文件系统
    * `/var/run/docker.sock`: 在容器里也需要使用docker-client

    ```yaml
    volumeMounts:
      - name: host
      mountPath: /host
      - name: run
      mountPath: /run
      - name: modules
      mountPath: /lib/modules
      - name: debugfs
      mountPath: /sys/kernel/debug
      - name: cgroup
      mountPath: /sys/fs/cgroup
      - name: bpffs
      mountPath: /sys/fs/ebpf
      - name: docker-socket
      mountPath: /var/run/docker.sock
      - name: source-linux-header
      mountPath: /usr/src
    ```

对于async-profiler，bpf开启的权限足够async-profiler使用了，所以不需要额外再开启多余的权限

## 5. 怎样识别不同进程所使用的编程语言

* 首先通过`/proc/pid/comm`获取该进程的启动命令，例如`java xxxx`，那么就可以判定为java程序，python同理

* 如果无法判断（例如go、c/c++），那么就进一步通过其执行文件`exe`的绝对路径（如果进程在容器中，还需要在最前面加上`/path`挂载的前置路径），通过`file`命令读取`elf`文件获取elf描述信息来区分：
  * 如果有`Go BuildID`，那么就是go程序
  * 其他就默认为`c/c++`程序

```go
func judgeProcessProgType(pid int, cid string) ProgType {
	// 1. judge comm
	stdout, stderr, err := execcmd.RunCommand(exec.Command("cat", fmt.Sprintf("/proc/%d/comm", pid)))
	if err != nil {
		log.Errorf("RunCommand error: %v", err)
		if len(stderr) > 0 {
			log.Errorf("stderr: %s", stderr)
		}

		return UNKNOWN
	}

	if res := getTypeByName(strings.Trim(stdout, "\n")); res != UNKNOWN {
		return res
	}

	// 2. judge exec path
	exePath := utils.GetProcessInfoByPid(uint(pid), "exe")
	if len(strings.Trim(exePath, "\n")) == 0 {
		return UNKNOWN
	}

	// judge elf file is golang?
	if inContainer {
		// in container need add /host prefix, because mount / (in host) on /host (in container)
		exePath, err = getExeFullPathInContainer(uint(pid), cid)
		if err != nil {
			log.Errorf("getExeFullPathInContainer error: %v", err)
			return UNKNOWN
		}
	}

	fileRes := utils.FileElf(exePath)
	if len(strings.Trim(fileRes, "\n")) == 0 {
		return UNKNOWN
	}

	if isGo := goProgReg.MatchString(fileRes); isGo {
		return GO
	}

	return UNKNOWN
}
```

```go
func FileElf(path string) string {
	res, _, err := execcmd.RunCommand(exec.Command("file", path))
	if err != nil {
		log.Error(err)
		return ""
	}

	if len(res) == 0 {
		log.Info("get executable elf file information is empty")
		return ""
	}

	return res
}
```

## 6. 构建Profile容器环境需要适配各类主机内核版本

不能像官网一样根据`uname -a`简单的构建镜像，因为这只与当前构建主机的内核版本相关，分布式下的其他主机的内核版本可能不一致

具体问题可见下面的[2.1](##2.1 新的主机环境BPF脚本运行出错)

解决的思路：不同unix系列对应不同的`Dockerfile`，在容器运行时将对应主机的linux头文件目录挂载进来

## 7. bpf profile的基本原理等问题

更多问题见调研内容，其中包括如下问题：

* 不同编程语言profile的挑战？最佳实践？

* java profiler项目的对比，Async-profiler的优势点？

# 二、遇到的问题

## 2.1 新的主机环境BPF脚本运行出错

操作描述：

`Unable to find kernel headers. Try rebuilding kernel with CONFIG_IKHEADERS=m (module) or installing the kernel development package for your running kernel version.`

![image-20220618104828346](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220618104828346.png)

原因：

* 相关issue：https://github.com/iovisor/bcc/issues/3038
* 在`/lib/modules/5.13.0-44-generic/build`下没找到内核头文件，这是BPF程序需要的

解决：

* 因为在容器中跑BPF，先进入容器然后检查`/lib/modules/5.13.0-44-generic/build`，发现有该文件，但是与`48`不同的是，该`build`（是一个符号链接）指向的内核文件并没有安装, 所以导致了BPF程序的加载失败

  ![image-20220618110122918](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220618110122918.png)

* DaemonSet容器的Dockerfile中指明了安装`linux-headers-$(uname -r)`, 但是这台主机是`5.13.0-44-generic`版本的内核，容器环境的`uname -a`是根据构建容器的主机环境内核版本，所以导致了这个错误

* 解决方法：

  不同unix系列对应不同的Dockerfile，在容器运行时将对应主机的linux头文件目录挂载进来


## 2.2 新主机获取docker存储引擎出错，导致agent无法启动

错误描述：

`ERRO[0000]/eagel/agent/service/profile.go:402 xwj/eagel-agent/service.runCommandBase() docker: error while loading shared libraries: libltdl.so.7: cannot open shared object file: No such file or directory `

![image-20220622210932922](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220622210932922.png)

原因：就是报错原因，docker找不到这个共享库

解决方法：

1. 挂载`usr/lib/x86_64-linux-gnu/libltdl.so.7`到容器

2. 构建镜像的时候额外安装保证容器环境中存在：`apt-get install -y libltdl7 `

   https://github.com/moby/moby/issues/37531

## 2.3 使用client-go客户端访问很慢

问题描述

出现日志：`I0624 11:22:03.204919 1273822 request.go:601] Waited for 1.198427306s due to client-side throttling, not priority and fairness, request: GET:https://10.219.204.206:6443/apis/events.k8s.io/v1beta1`

原因：

* 客户端被设置了限流，参考：参考：https://blog.csdn.net/u012803274/article/details/119964785

先找到问题代码的位置：在`client-go/rest/request.go`:

```shell
func (r *Request) tryThrottleWithInfo(ctx context.Context, retryInfo string) error {
	if r.rateLimiter == nil {
		return nil
	}

	now := time.Now()

	err := r.rateLimiter.Wait(ctx)
	if err != nil {
		err = fmt.Errorf("client rate limiter Wait returned an error: %w", err)
	}
	latency := time.Since(now)

	var message string
	switch {
	case len(retryInfo) > 0:
		message = fmt.Sprintf("Waited for %v, %s - request: %s:%s", latency, retryInfo, r.verb, r.URL().String())
	default:
		message = fmt.Sprintf("Waited for %v due to client-side throttling, not priority and fairness, request: %s:%s", latency, r.verb, r.URL().String())
	}

	if latency > longThrottleLatency {			// 关键点在此
		klog.V(3).Info(message)
	}
	if latency > extraLongThrottleLatency {
		// If the rate limiter latency is very high, the log message should be printed at a higher log level,
		// but we use a throttled logger to prevent spamming.
		globalThrottledLogger.Infof("%s", message)
	}
	metrics.RateLimiterLatency.Observe(ctx, r.verb, *r.URL(), latency)

	return err
}
```

这里有一个client的限流器判断

解决：

我们可以在创建客户端的restConf修改配置：

```go
// Maximum burst for throttle.
// If it's zero, the created RESTClient will use DefaultBurst: 10.
Burst int

// Rate limiter for limiting connections to the master from this client. If present overwrites QPS/Burst
RateLimiter flowcontrol.RateLimiter
```

从默认配置来看，默认 QPS 是5，最大 QPS 是10，如果设置了 RateLimiter，将会覆盖 QPS 和 Burst 参数

```go
restConf, err = clientcmd.BuildConfigFromFlags("", kubeConfigPath)
if err != nil {
  fmt.Fprintf(os.Stderr, "clientcmd.BuildConfigFromFlags error: %v", err)
  return
}
// 设置限流(防止出现限流提示,影响速度)
restConf.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(1000, 1000)
```



