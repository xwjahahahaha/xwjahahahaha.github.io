---
title: 一期profile调研报告
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-05-30 15:30:27
---

[TOC]


<!-- more -->

# 一、一期Profile调研情况以及最佳实践

## 1. 主要目标

* 适配多语言：c/c++、go、java的profile工具 （主要是`CPU Profile`并生成火焰图）
* 如果不能实现完全无侵入，那么需要提供一个最佳实践说明书
* 最终目标：类似于[pyroscope](https://demo.pyroscope.io/?name=hotrod.python.frontend%7B%7D)的`Profile`展示

## 2. 基础预备

关于调研相关概念等基础的预备知识记录

### 2.1 debuginfo基础相关

#### 1. debuginfo信息

`debuginfo`详细基本信息（来自`gdb`）: https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html

> `GDB `允许您将程序的调试信息放在与可执行文件本身分开的文件中，从而允许` GDB `自动查找和加载调试信息。因为debuginfo可能会很大，所以目前一般的方法会将其单独分离开来

##### 获取elf文件的debuginfo

```shell
g++ xxx -o xxx.out -gdwarf
readelf -wi xxx.out > debug_xxx.txt
```

##### bcc中的debuginfo的两种section

bcc支持`.note.gnu.build-id` or `.gnu_debuglink` sections in the ELF

详情见：https://github.com/iovisor/bcc/pull/967

#### 2. GDB的debuginfo使用

> 此部分来自于：
>
> * https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html

##### debug link方法

可执行文件包含一个`debug link`，该链接指定单独的`debug info`文件的名称。单独的调试文件的名称通常是`executable.debug`，其中`executable` 是对应的可执行文件的名称，不包括前导目录（ `e.g., ls.debug for /usr/bin/ls`）此外，调试链接为调试文件指定了一个 `32` 位循环冗余校验 (`CRC`) 校验和，`GDB `使用它来验证可执行文件和调试文件是否来自同一个构建

寻找方式：

* 在可执行目录中查找，然后在该目录的子目录中查找`.debug`，最后在每个全局调试目录下，在一个子目录中，其名称与可执行文件绝对文件名的前导目录相同。

##### build ID方法

可执行文件包含一个`Build ID`，一个唯一的位字符串，也存在于相应的调试信息文件中（这仅在某些操作系统上受支持，当对二进制文件和 `GNU Binutils `使用 `ELF `或` PE `文件格式时）调试信息文件的名称不是由`build ID `明确指定的，但可以从`build ID `计算出来;

寻找方式：

* `GDB `在每个全局调试目录的` .build-id `子目录中查找名为 `nn/nnnnnnnn.debug` 的文件，其中` nn `是`build id` 位串的前 2 个十六进制字符，而` nnnnnnnn `是位串的其余部分(真正的`build id`字符串是 `32 `个或更多的十六进制字符，而不是 `10 `个), `GDB `可以使用`build id`自动查询 `debuginfod `服务器，以便下载本地无法找到的单独调试文件; 

  > 关于debuginfo服务器见：[Debuginfod](https://sourceware.org/gdb/onlinedocs/gdb/Debuginfod.html#Debuginfod).

> 这两种方法基本与`bcc`的方法一致

##### 查询次序

假设您要求`GDB`进行调试 `/usr/bin/ls`，其中有一个指定文件的调试链接`ls.debug`，以及十六进制值为的`build ID `: `abcdef1234`。如果全局调试目录列表包括 `/usr/lib/`调试，然后`GDB`将按照指示的顺序查找以下调试信息文件：

- `-/usr/lib/debug/.build-id/ab/cdef1234.debug`
- `-/usr/bin/ls.debug`
- `-/usr/bin/.debug/ls.debug`
- `-/usr/lib/debug/usr/bin/ls.debug.`
- `debuginfod`服务器下载

### 2.2 System.map和kallsyms文件

#### 1. System.map

> 该文件是是一份内核符号表`kernel symbol table`，包含了内核中的变量名和函数名地址，在每次编译内核时，自动生成。
> 相关资料：
> [GNU Binutils wiki](https://en.wikipedia.org/wiki/GNU_Binutils) [GNU Binutils](https://www.gnu.org/software/binutils/) [What Are Symbols?](http://dirac.org/linux/system.map/)

#### 2. /proc/kallsysms

`/proc/kallsysms have symbols of dynamically loaded modules as well static code and system.map is symbol tables of only static code. `

`kallsyms`包含了`kernel image`和动态加载模块的符号表，函数如果被编译器内联（inline）或优化掉，则它在`/proc/kallsyms`有可能找不到。正在运行的内核可能和`System.map`不匹配，出现`System.map does not match actual kernel`，所以`/proc/kallsyms`才是参考的主要来源，我们应该通过`/proc/kallsyms`获得符号的地址。

`/proc/kallsyms`的形成过程为：

- [/scripts/kallsyms.c](https://github.com/torvalds/linux/blob/master/scripts/kallsyms.c) 生成`System.map`
- [/kernel/kallsyms.c](https://github.com/torvalds/linux/blob/master/kernel/kallsyms.c) 生成`/proc/kallsyms`
- [/scripts/kallsyms.c](https://github.com/torvalds/linux/blob/master/scripts/kallsyms.c) 解析`vmlinux(.tmp_vmlinux)`生成`kallsyms.S(.tmp_kallsyms.S)`，然后内核编译过程中将`kallsyms.S`(内核符号表)编入内核镜像`uImage`
- 内核启动后`./kernel/kallsyms.c`解析`uImage`形成`/proc/kallsyms`
- `/proc/kallsyms`包含了内核中的函数符号(包括没有`EXPORT_SYMBOL`)、全局变量(用`EXPORT_SYMBOL`导出的全局变量)

### 2.3 java profile相关

> 参考：
>
> * https://tech.meituan.com/2019/10/10/jvm-cpu-profiler.html

#### Java的编译过程是什么？

![img](http://xwjpics.gumptlu.work/qinniu_uPic/9df89177-114a-343a-bfe0-672739b33ed6.png)

大致为以下两个步骤：

1. 源文件由编译器编译成字节码（`ByteCode`）  
2. 字节码由java虚拟机解释运行。因为java程序既要编译同时也要经过JVM的解释运行，所以说Java被称为半解释语言（ `semi-interpreted language`）

JVM包含了`C++`编写的库（`libjvm`），功能包括：编译、线程管理、垃圾回收；`HotSpot`是使用最为广泛的`JVM`

#### JVM Agent是什么？

一个特殊的程序库，可以在java程序的启动阶段随命令行传递给JVM，然后其作为一个伴生库与目标JVM运行在同一个进程中

作用：提供固定的接口获取到JVM进程内的相关信息

agent相关的命令：

```shell
Plain Text
    -agentlib:<库名>[=<选项>]
                  加载本机代理库 <库名>, 例如 -agentlib:jdwp
                  另请参阅 -agentlib:jdwp=help
    -agentpath:<路径名>[=<选项>]
                  按完整路径名加载本机代理库
    -javaagent:<jar 路径>[=<选项>]
                  加载 Java 编程语言代理, 请参阅 java.lang.instrument
```

根据编写Agent的语言不同，引申出两个概念：

* `JVMTI Agent`: (`JVM Tool Interface`) Agent用C/C++/Rust编写
* `Java Agent`: 用java编写的

#### JVMTI Agent详述？

是JVM提供的一套标准的C/C++编程接口，是实现Debugger、Profiler、Monitor、Thread Analyser等工具的统一基础，在主流Java虚拟机中都有实现，当我们要基于JVMTI实现一个Agent时，需要实现如下入口函数：

```c
// $JAVA_HOME/include/jvmti.h
JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *vm, char *options, void *reserved);
```

使用C/C++实现该函数，并将代码编译为动态连接库（Linux上是.so），通过`-agentpath`参数将库的完整路径传递给Java进程，JVM就会在启动阶段的合适时机执行该函数。在函数内部，我们可以通过`JavaVM`指针(也就是第一个参数)参数拿到`JNI`和`JVMTI`的函数指针表，这样我们就拥有了与JVM进行各种复杂交互的能力

更多JVMTI相关的细节可以参考[官方文档](https://docs.oracle.com/en/java/javase/12/docs/specs/jvmti.html)

#### Java Agent详述？

使用C/C++的方式成本高且不好维护，所以JVM自身基于JVMTI封装了一套Java的Instrument API接口，允许使用Java语言开发Java Agent（只是一个jar包），大大降低了Agent的开发成本

在Java Agent中，我们需要在jar包的`MANIFEST.MF`中将`Premain-Class`指定为一个入口类，并在该入口类中实现如下方法：

```java
public static void premain(String args, Instrumentation ins) {
    // implement
}
```

这样打包出来的jar就是一个Java Agent，可以通过`-javaagent`参数将jar传递给Java进程伴随启动，JVM同样会在启动阶段的合适时机执行该方法

在该方法内部，参数`Instrumentation`接口提供了`Retransform Classes`的能力，我们利用该接口就可以**对宿主进程的`Class`进行修改**，实现方法耗时统计、故障注入、Trace等功能。`Instrumentation`接口提供的能力较为单一，仅与Class字节码操作相关，但由于我们现在已经处于宿主进程环境内，就可以利用JMX直接获取宿主进程的内存、线程、锁等信息。无论是Instrument API还是JMX，它们内部仍是统一基于JVMTI来实现。

更多Instrument API相关的细节可以参考[官方文档](https://docs.oracle.com/en/java/javase/12/docs/api/java.instrument/java/lang/instrument/package-summary.html)。

#### SafePoint Bias 安全点偏差是什么？

安全点的定义：

**Safepoint 可以理解成是在代码执行过程中的一些特殊位置**，当线程执行到这些位置的时候，**线程可以暂停**。在 SafePoint 保存了其他位置没有的**一些当前线程的运行信息，供其他线程读取**, 一般来说线程只有运行到了 SafePoint 的位置，他的**一切状态信息，才是确定的**

安全点偏差问题：

因为某些方法执行时间极短，但执行频率很高，真实占用了大量的CPU Time，**但Sampling Profiler的采样周期不能无限调小(会导致性能开销骤增)或者某些Profile方法只能在安全点采样**，所以会导致大量的样本调用栈中并不存在刚才提到的”高频小方法“，进而导致最终结果无法反映真实的CPU热点。更多Sampling相关的问题可以参考《[Why (Most) Sampling Java Profilers Are Fucking Terrible](https://psy-lob-saw.blogspot.com/2016/02/why-most-sampling-java-profilers-are.html)》

#### 字节码增强是什么？

对已经编译为字节码的java程序，通过工具让其代码具有本不具有的功能都属于字节码增强范畴。例如简化代码的Lombok，最常用的AOP技术以及各类框架类工具，如代码覆盖率Jacoco、链路追踪的Skywalking、性能分析工具Arthas等等

#### 基于Java Agent + JMX实现方式

以`Java Agent`为入口，进入目标JVM进程后开启一个`ScheduledExecutorService`，定时利用JMX的`threadMXBean.dumpAllThreads()`来导出所有线程的`StackTrace`，最终汇总并导出即可

```java
// com/uber/profiling/profilers/StacktraceCollectorProfiler.java
/*
 * StacktraceCollectorProfiler等同于文中所述CpuProfiler，仅命名偏好不同而已
 * jvm-profiler的CpuProfiler指代的是CpuLoad指标的Profiler
 */
// 实现了Profiler接口，外部由统一的ScheduledExecutorService对所有Profiler定时执行
@Override
public void profile() {
    ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
    // ...
    for (ThreadInfo threadInfo : threadInfos) {
        String threadName = threadInfo.getThreadName();
        // ...
        StackTraceElement[] stackTraceElements = threadInfo.getStackTrace();
        // ...
        for (int i = stackTraceElements.length - 1; i >= 0; i--) {
            StackTraceElement stackTraceElement = stackTraceElements[i];
            // ...
        }
        // ...
    }
}
```

#### 基于JVMTI + GetStackTrace实现

在更底层的C/C++层面，我们可以直接对接JVMTI接口，使用原生C API对JVM进行操作，功能更丰富更强大，但开发效率偏低。基于上节同样的原理开发CPU Profiler，使用JVMTI需要进行如下这些步骤：

1. 编写`Agent_OnLoad()`，在入口通过`JNI`的`JavaVM*`指针的`GetEnv()`函数拿到`JVMTI`的`jvmtiEnv`指针：

```c
// agent.c
JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *vm, char *options, void *reserved) {
    jvmtiEnv *jvmti;
    (*vm)->GetEnv((void **)&jvmti, JVMTI_VERSION_1_0);
    // ...
    return JNI_OK;
}
```

2. 开启一个线程定时循环，定时使用`jvmtiEnv`指针配合调用如下几个`JVMTI`函数：

```c
// 获取所有线程的jthread
jvmtiError GetAllThreads(jvmtiEnv *env, jint *threads_count_ptr, jthread **threads_ptr);
// 根据jthread获取该线程信息（name、daemon、priority...）
jvmtiError GetThreadInfo(jvmtiEnv *env, jthread thread, jvmtiThreadInfo* info_ptr);
// 根据jthread获取该线程调用栈
jvmtiError GetStackTrace(jvmtiEnv *env,
                         jthread thread,
                         jint start_depth,
                         jint max_frame_count,
                         jvmtiFrameInfo *frame_buffer,
                         jint *count_ptr);
```

主逻辑大致是：首先调用`GetAllThreads()`获取所有线程的“句柄”`jthread`，然后遍历根据`jthread`调用`GetThreadInfo()`获取线程信息，按线程名过滤掉不需要的线程后，继续遍历根据`jthread`调用`GetStackTrace()`获取线程的调用栈

3.在Buffer中保存每一次的采样结果，最终生成必要的统计数据即可

缺点：JVMTI的GetStackTrace()函数并不需要在Caller的安全点执行，但当调用GetStackTrace()获取其他线程的调用栈时，必须等待，直到目标线程进入安全点，所以还是会有安全点偏差问题

#### 基于JVMTI + AsyncGetCallTrace实现

为了避免安全点偏差问题而提出

假如我们拥有一个函数可以获取当前线程的调用栈且不受安全点干扰，另外它还支持在UNIX信号处理器中被异步调用，那么我们只需注册一个UNIX信号处理器，在Handler中调用该函数获取当前线程的调用栈即可。

由于UNIX信号会被发送给进程的随机一线程进行处理，因此最终信号会均匀分布在所有线程上，也就均匀获取了所有线程的调用栈样本。

`OracleJDK/OpenJDK`内部提供了这么一个函数——`AsyncGetCallTrace`

```c
// 栈帧
typedef struct {
  jint lineno;
  jmethodID method_id;
} AGCT_CallFrame;
// 调用栈
typedef struct {
  JNIEnv *env;
  jint num_frames;
  AGCT_CallFrame *frames;
} AGCT_CallTrace;
// 根据ucontext将调用栈填充进trace指针
void AsyncGetCallTrace(AGCT_CallTrace *trace, jint depth, void *ucontext);
```

`AsyncGetCallTrace`是`async`的，不受安全点影响，这样的话采样就可能发生在任何时间，包括Native代码执行期间、GC期间等，在这时我们是无法获取Java调用栈的，`AGCT_CallTrace`的`num_frames`字段正常情况下标识了获取到的调用栈深度，但在如前所述的异常情况下它就表示为负数，最常见的-2代表此刻正在GC。

由于`AsyncGetCallTrace`非标准JVMTI函数，因此我们无法在`jvmti.h`中找到该函数声明，且由于其目标文件也早已链接进JVM二进制文件中，所以无法通过简单的声明来获取该函数的地址，这需要通过一些Trick方式来解决。简单说，Agent最终是作为**动态链接库加载**到目标JVM进程的地址空间中，因此可以在`Agent_OnLoad`内通过`glibc`提供的`dlsym()`函数拿到当前地址空间（即目标JVM进程地址空间）名为“`AsyncGetCallTrace`”的符号地址。这样就拿到了该函数的指针，按照上述原型进行类型转换后，就可以正常调用了。

通过`AsyncGetCallTrace`实现`CPU Profiler`的大致流程：

1. 编写`Agent_OnLoad()`，在入口拿到`jvmtiEnv`和`AsyncGetCallTrace`指针，获取`AsyncGetCallTrace`方式如下:

```c
typedef void (*AsyncGetCallTrace)(AGCT_CallTrace *traces, jint depth, void *ucontext);
// ...
AsyncGetCallTrace agct_ptr = (AsyncGetCallTrace)dlsym(RTLD_DEFAULT, "AsyncGetCallTrace");
if (agct_ptr == NULL) {
    void *libjvm = dlopen("libjvm.so", RTLD_NOW);
    if (!libjvm) {
        // 处理dlerror()...
    }
    agct_ptr = (AsyncGetCallTrace)dlsym(libjvm, "AsyncGetCallTrace");
}
```

2. 在OnLoad阶段，我们还需要做一件事，即注册`OnClassLoad`和`OnClassPrepare`这两个Hook，原因是jmethodID是延迟分配的，使用AGCT获取Traces依赖预先分配好的数据。我们在OnClassPrepare的CallBack中尝试获取该Class的所有Methods，这样就使JVMTI提前分配了所有方法的jmethodID，如下所示：

```c
void JNICALL OnClassLoad(jvmtiEnv *jvmti, JNIEnv* jni, jthread thread, jclass klass) {}
void JNICALL OnClassPrepare(jvmtiEnv *jvmti, JNIEnv *jni, jthread thread, jclass klass) {
    jint method_count;
    jmethodID *methods;
    jvmti->GetClassMethods(klass, &method_count, &methods);
    delete [] methods;
}
// ...
jvmtiEventCallbacks callbacks = {0};
callbacks.ClassLoad = OnClassLoad;
callbacks.ClassPrepare = OnClassPrepare;
jvmti->SetEventCallbacks(&callbacks, sizeof(callbacks));
jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_CLASS_LOAD, NULL);
jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_CLASS_PREPARE, NULL);
```

3. 利用`SIGPROF`信号来进行定时采样：

```c
// 这里信号handler传进来的的ucontext即AsyncGetCallTrace需要的ucontext
void signal_handler(int signo, siginfo_t *siginfo, void *ucontext) {
    // 使用AsyncCallTrace进行采样，注意处理num_frames为负的异常情况
}

// ...

// 注册SIGPROF信号的handler
struct sigaction sa;
sigemptyset(&sa.sa_mask);
sa.sa_sigaction = signal_handler;
sa.sa_flags = SA_RESTART | SA_SIGINFO;
sigaction(SIGPROF, &sa, NULL);

// 定时产生SIGPROF信号
// interval是nanoseconds表示的采样间隔，AsyncGetCallTrace相对于同步采样来说可以适当高频一些
long sec = interval / 1000000000;
long usec = (interval % 1000000000) / 1000;
struct itimerval tv = {{sec, usec}, {sec, usec}};
setitimer(ITIMER_PROF, &tv, NULL);
```

4. 在`Buffer`中保存每一次的采样结果，最终生成必要的统计数据即可。

按如上步骤即可实现基于`AsyncGetCallTrace`的CPU Profiler，这是社区中目前性能开销最低、相对效率最高的CPU Profiler实现方式，在Linux环境下结合`perf_events`还能做到同时采样Java栈与Native栈，也就能同时分析Native代码中存在的性能热点。

更多关于`AsyncGetCallTrace`的内容，可以参考《[The Pros and Cons of AsyncGetCallTrace Profilers](http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)》。

#### HotSpot的Dynamic Attach机制

> **HotSpot**的正式发布名称为"Java HotSpot Performance Engine"，是[Java虚拟机](https://zh.m.wikipedia.org/wiki/Java虚拟机)的一个实现，包含了服务器版和桌面应用程序版，现时由[Oracle](https://zh.m.wikipedia.org/wiki/Oracle)维护并发布

JDK在1.6以后提供了`Attach API`，允许向运行中的`JVM`进程添加`Agent`，这项手段被广泛使用在各种Profiler和字节码增强工具中, 这样就避免了重新启动添加`agent`等参数

`Dynamic Attach`是HotSpot提供的一种特殊能力，它允许一个进程向另一个运行中的JVM进程发送一些命令并执行，命令并不限于加载Agent，还包括Dump内存、Dump线程等等。

#### .Djava.io.tmpdir参数的作用？

此参数只是当前java启动的进程的所需要临时目录的位置

在`perf-map-agent`方式中，attach的java进程加此参数是针对自己设置`tmp`目录，其依赖的对应`Pid`的`/tmp/hsperfdata_<user>/<pid>`文件的找寻方法不会变

## 3. 各语言Profile最佳使用方式调研

> 根据官方merge和测试代码：
>
> * https://github.com/iovisor/bcc/pull/967
> * https://github.com/iovisor/bcc/pull/967/files#diff-78f1fdebba737267c347b1815002d3190e4cc324c4086d1c347873284381b1ca

### 3.1 分类总结

#### 用户态语言profile

<font color='#e54d42'>针对不同语言的调用栈profile，无关语言本身，重点在于语言是如何在执行时转换为机器码的，这不是语言本身的属性，而是与其具体的实现方式有关。</font>例如java，其需要像流水线的方式从解释方式执行再到JIT编译方式执行java方法，在一个完全插桩的Java函数中，会遇到解释方式执行的部分java方法代码、编译方式的代码(JVM函数的C++代码)、JIT编译之后的部分Java方法代码；对于如何插桩这些不同的形式是有差别的需要分别实现的。

核心工作：

* 明确一个语言是用什么来运行的？其如何工作的？
* BPF怎样获取调用栈？默认方式？额外方式？ => <font color='#e54d42'>决定调用栈是否完整</font>
* 该语言程序的符号解析表在哪？如何生成的？怎样去获取？运行时会发生变化吗？ => <font color='#e54d42'>决定调用栈的可读性</font>

按照语言如何生成最终的机器码进行了如下分类(仅针对用户态程序)

| 语言类型        | 实例                     | 如何运行？                                                   | 符号表在哪？会变化吗？                                       | 建议编译参数                                                 | 额外符号表方式                                               |
| --------------- | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 编译型          | C/C++、Golang、Rust      | 源程序经过编译器编译为机器码保存在二进制`ELF`文件中直接运行  | `ELF`文件中(没有`strip`处理)；不会变化                       | `-fno-omit-frame-pointer`帧指针                              | `ELF`的`section`字段:`.gnu_debuglink` 、`.note.gnu.build-id`指向的`debugginfo`文件、`BTF`(仅支持内核) |
| 即时编译型(JIT) | Java、JavaScript、Nodejs | 源文件编译为字节码，在运行时再编译为对应的机器码，通常会从运行时的操作中接收反馈来指导优化编译（混合的方式，其实也算是解释型语言） | 因为是运行时编译，所以一般符号表放在`JIT`运行时的内存中并且这些映射关系也在不断变化 | 运行时`-XX:+PreserveFramePointer`加入解释器帧指针选项(`java`)； | `perf-map-agent`生成、动态`USDT`探针                         |
| 解释型          | Bash shell、python       | 不会将源文件整体编译为机器码，而是使用自身内置的子函数/解释器进行语法分析和一步步执行 | 解释器的内存映射表中，不断变化                               | 解释器启用帧指针                                             | 动态`USDT`                                                   |

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220603192806111.png" alt="image-20220603192806111" style="zoom:50%;" />

> <font color='#e54d42'>一些应用可以为JIT提供额外的符号映射表(`/tmp/perf-PID.map`)，但是这个不能使用在`uprobes`上，因为JIT编译器可能会在内存移动被`uprobe`插桩过的函数但是不会通知内核，从而导致恢复插桩前的指令的时候会写入错误的内存</font>

#### 内核调用栈profile

对于内核态的软件，内核会在[`/proc/kallsymc`](####2. /proc/kallsysms)中保存动态符号表，随着加载的内核模块的增加而增加；此外，内核本身函数名、变量名会在内核编译时保存在[`System.map`](####1. System.map)文件中

> 大部分的发行版内核在编译的时候都会开启`CONFIG_FRAME_POINTER=y`帧指针参数，如果没开启则使用BPF的方式可能会对内核栈的采集有影响

#### BPF的寻找调用栈方法

**为什么要使用BPF去Profile各个语言的调用栈而不是使用每个语言特定自己实现的工具？**

主要原因：BPF提供了一个软件用户栈与内核栈多个层次的调用串联关系，很多场景下内核事件反应了具体确定和量化的问题，而需要在用户栈中修改对应的代码（也有一些特定语言的调试工具实现了用户内核调用栈的串联跟踪）

> <font color='#39b54a'>BPF的优势在于插桩内核事件，并在内核上下文环境中获取到整个调用栈信息</font>
>
> <font color='#e54d42'>其实目前来说，BPF寻找调用栈的优势还是仅限于内核栈的完整性以及内核上下文与用户态上下文的一致性，对于不符合BPF调用栈回溯的用户态程序来说，BPF的Profile并不占优势</font>

BPF寻找调用栈的方式：

* (默认)帧指针
* 其他：调试信息(`debuginfo`)
* 还在准备支持中：`LBR`、`ORC`

### 3.2 C/C++（BPF方式）

#### 调研结论

* 默认gcc/g++编译器不会使用帧指针方式（为了节省一个寄存器用于通过寄存器）, 所以需要手动打开保证调用栈完整
* `ELF`文件中的符号表信息默认不会剔除，但是使用了`strip`就会被剔除

* `c++`与`c`几乎一样，`c++`在符号显示上可能会因为支持对象(`self`对象)而显示形式不同, 函数的参数也可能不会遵守处理器`ABI`

#### 用户最佳使用方式：

提供了检查当前执行文件符号表是否存在的方式以及在编译后单独剥离出`debuginfo`信息的两种方式（这里以`dummy`/`cc_dummy`为假定的`ELF`可执行二进制程序）

如果条件允许，最好重新使用帧指针参数编译底层`libc`等依赖库, 这样调用栈会更加完整

> 如果我们可以提供此已编译的底层依赖库最好

##### 检查符号表是否存在：

```shell
$ readelf -s dummy
```

![image-20220603201443105](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220603201443105.png)

如果没有`.symtab`这个`section`则一般说明该程序被`strip`过了，应用的主要本地符号表已经被清除

![image-20220603201850545](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220603201850545.png)

##### `.gnu_debuglink`方式：

```shell
# 编译原文件
$ g++ -fno-omit-frame-pointer -o dummy dummy.cc
# 剥离debuginfo信息到dummy.debug
$ objcopy --only-keep-debug dummy dummy.debug
# Add section .gnu_debuglink linking to <file> 
$ objcopy --add-gnu-debuglink=dummy.debug dummy
# 删除掉原文件symbol和section信息
$ strip dummy
```

注意点：

* 如果要指定其他目录，需要在`--add-gnu-debuglink`之前指定将`xxx.debug`放入对应目录，`--add-gnu-debuglink`也要指定到对应的目录，添加连接后就会在原文件中保存对应的文件目录：

  * 查看`link`是否存在方法：`readelf dummy -wk`

    ![image-20220527103317914](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220527103317914.png)

* `strip dummy`的作用:

  * 验证不借助自身的符号表和`section`即借助外部`debuginfo`的方式(不管是`.gnu_debuglink`还是`.note.gnu.build-id`这两个`section`)可以实现解析地址到函数名
  * 减小原文体积

##### `.note.gnu.build-id`方式

查看`ELF`文件的`build-id`，其在`note`区：

```shell
$ readelf -n cc_dummy
```

![image-20220603203125377](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220603203125377.png)

对应到文件：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220705172735035.png" alt="image-20220705172735035" style="zoom:50%;" />

```shell
$ g++ -fno-omit-frame-pointer -o dummy -Xlinker --build-id=0x123456789abcdef0123456789abcdef012345678 dummy.cc
# 剥离debuginfo信息到dummy.debug
$ objcopy --only-keep-debug dummy dummy.debug
$ mkdir -p /usr/lib/debug/.build-id/12
$ mv dummy.debug /usr/lib/debug/.build-id/12/3456789abcdef0123456789abcdef012345678.debug
# 删除掉原文件symbol和section信息
$ strip dummy
```

#### 列举`ELF`可插桩函数的方法(uprobes)

```shell
$ sudo bpftrace -l "uprobe:dummy"
```

![image-20220603203727284](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220603203727284.png)

### 3.3 go（BPF方式）

#### 调研结论

1. `go`的两个编译器`Go gc`以及`gccgo`都保留了帧指针(`Go1.7`之后)，并且最终二进制文件中也包含了相关符号
2. 因为`go`协程和动态栈管理方面的区别`uretprobes`目前在`golang`程序中使用并不安全，可能会导致目标程序崩溃

2. `go`源文件的 `.note.go.build-id`节，bcc目前不支持

   `go`的编译中使用的`buildid`的`section`为`.note.go.build-id`:

   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220527164021024.png" alt="image-20220527164021024" style="zoom:50%;" />

   经过测试，自己指定`build id`的编译方式无法通过测试:

   ```shell
   go build -o go_dummy -ldflags="-buildid 0x123456789abcdef0123456789abcdef012345678" go_dummy.go 
   ```

   ![image-20220527164439144](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220527164439144.png)

   测试结果：

   ![image-20220527164607749](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220527164607749.png)

   因为在`bcc`源码(`src/cc/bcc_elf.c`)中还未支持, 因为只查询了`.note.gnu.build-id`

   ```c++
   static int find_buildid(Elf *e, char *buildid) {
     Elf_Data *data = get_section_elf_data(e, ".note.gnu.build-id");
     if (!data || data->d_size <= 16 || strcmp((char *)data->d_buf + 12, "GNU"))
       return 0;
   
     char *buf = (char *)data->d_buf + 16;
     size_t length = data->d_size - 16;
     size_t i = 0;
     for (i = 0; i < length; ++i) {
       sprintf(buildid + (i * 2), "%02hhx", buf[i]);
     }
   
     return 1;
   }
   ```

   解决方案：

   * 修改bcc源码，添加`.note.go.build-id`方式
   * go编译添加`.note.gnu.build-id` (未测试)

#### 用户最佳使用方式：

##### 检查符号表是否存在:

```shell
$ readelf -s go_dummy
```

如果不存在则需要重新编译源文件

##### `.gnu_debuglink`方式：

```shell
$ go build -gcflags="-l" -o go_dummy go_dummy.go
objcopy --only-keep-debug go_dummy go_dummy.debug
objcopy --add-gnu-debuglink=go_dummy.debug go_dummy
strip go_dummy
```

### 3.4 java（Async-profiler）

#### 调研结论

##### 1. BCC分析java缺陷以及原因

Java调用栈难以跟踪的原因在与其编译方法的复杂性：

* Java方法首先被编译为字节码(JVM)，随后在解释器中运行
* 当执行频率超过阈值`-XX:CompileThreshold`时, 字节码会通过JIT即时编译为本地指令运行
* JVM本身的优化：会对Java方法进行剖析重新编译、动态改变内存中的位置

难以识别`Java方法`和`stack traces`

- JVM的JIT（`just-in-time`）没有给系统级`profiler`公开符号表
- JVM默认将帧指针寄存器（`frame pointer register`，借助`x86-64`上的`RBP`寄存器）作为通用寄存器，默认不开启帧指针
- 因为java虚拟机及时编译，所以如果要符号表内容也是在不断累加，要不断适应更新
- 不能使用`uprobe`插桩`java`程序，这可能会导致`java`进程的崩溃

除了这些缺陷BPF还有一些优势：

* 可以跟踪JVM的主库`libjvm`（因为是`C++`编写的），可用于实现分析java线程、加载类、编译方法、分配内存以及gc垃圾回收等

##### 2. 传统`Java Cpu Profile`方法总结

* `sampling`/采样方式

  基于对`StackTrace`的采样实现，核心步骤如下：

  1. 引入Profiler依赖，或直接利用Agent技术注入目标JVM进程并启动Profiler。
  2. 启动一个采样定时器，以固定的采样频率每隔一段时间（毫秒级）对所有线程的调用栈进行Dump。
  3. 汇总并统计每次调用栈的Dump结果，在一定时间内采到足够的样本后，导出统计结果，内容是每个方法被采样到的次数及方法的调用关系

  当前主流的实现方式与对比如下：

  | 方法名                                                       | 是否有[安全点偏差问题?](####SafePoint Bias 安全点偏差是什么？) | 限制                |
  | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------- |
  | [基于Java Agent + JMX实现方式](####基于Java Agent + JMX实现方式) | Y                                                            | 未明确              |
  | [基于JVMTI + GetStackTrace实现](###基于JVMTI + GetStackTrace实现) | Y                                                            | 未明确              |
  | [基于JVMTI + AsyncGetCallTrace实现](####基于JVMTI + AsyncGetCallTrace实现) | N                                                            | `OracleJDK/OpenJDK` |

* `Instrumentation`

  基于`Instrument API`，对所有必要的Class进行[字节码增强](####字节码增强是什么？)，在进入**每个**方法前进行埋点，方法执行结束后统计本次方法执行耗时，最终进行汇总

两种方法的对比：

| 方法名          | 优点                               | 缺点                                                         |
| --------------- | ---------------------------------- | ------------------------------------------------------------ |
| sampling        | 主流，对代码无侵入，性能开销较低   | 需要额外侵入一个线程;安全点(Safe Point)缺陷会导致统计结果偏差;JVM中额外引入的`agent.jar`可能会引入第三方依赖对业务造成污染 |
| Instrumentation | 绝对精准的方法调用次数以及调用时间 | 使用[字节码增强](####字节码增强是什么？)代码侵入大且性能消耗巨大 |
|                 |                                    |                                                              |

##### 3. 社区项目情况总结

| 项目名                                                       | 基本原理依赖                                                 | 特点                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Greys](https://github.com/oldmanpushcart/greys-anatomy)、[Arthas](https://github.com/alibaba/arthas)、[JVM-Sandbox](https://github.com/alibaba/jvm-sandbox)、[JVM-Profiler](https://github.com/uber-common/jvm-profiler) | [Java Agent + JMX](####基于Java Agent + JMX实现方式)         | 传统方式                                                     |
| [async-profiler](https://github.com/jvm-profiling-tools/async-profiler)、[Honest-Profiler](https://github.com/jvm-profiling-tools/honest-profiler) | [JVMTI + AsyncGetCallTrace](####基于JVMTI + AsyncGetCallTrace实现) | async-profiler使用`AsyncGetStackTrace`避免了采样点偏差，目前在社区中性能开销最低且相对效率最高; 可以`-javaagent`方式也可以`attach`方式 |
| [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html)(商用) | sampling和Instrumentation都可以(可选)                        | 可以实现[Dynamic Attach](####HotSpot的Dynamic Attach机制)而不需要重启java进程 |

##### 4. 当前解决方案

- 使用`JVMTIagent` [perf-map-agent](https://github.com/jvm-profiling-tools/perf-map-agent)，生成Java符号表，供perf和bcc读取（`/tmp/perf-PID.map`）。同时要加上`-XX:+PreserveFramePointerJVM `参数，让`perf`可以遍历基于帧指针（`frame pointer`）的堆栈
- 使用[async-profiler](https://github.com/jvm-profiling-tools/async-profiler)，该项目将perf的堆栈追踪和JDK提供的[AsyncGetCallTrace](http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)得到的栈结合了起来（增加识别准确性），同样能够获得`mixed-mode`火焰图。同时，此方法不需要启用帧指针，所以不用加上`-XX:+PreserveFramePointer`参数

#### 用户最佳使用方式：

以下操作均为可选，如果做了调用栈会更完整

Java默认不开启帧指针，运行Java时最好开启`-XX:+PreserveFramePointer`参数（可选，如果使用`perf-map-agent`方式）

并最好按如下提示安装对应`jdk`版本的调试包工具（在容器环境中）：

如果你是`OpenJDK`构建则需要安装`Debug Symbols`，因为`Oracle JDK `已经内置了`libjvm.so`, 但是`OpenJDK`还没有

```shell
sudo apt install openjdk-8-dbg
```

or for OpenJDK 11:

```shell
sudo apt install openjdk-11-dbg
```

On CentOS, RHEL and some other RPM-based distributions, this could be done with [debuginfo-install](http://man7.org/linux/man-pages/man1/debuginfo-install.1.html) utility:

```shell
sudo debuginfo-install java-1.8.0-openjdk
```

On Gentoo the `icedtea` OpenJDK package can be built with the per-package setting `FEATURES="nostrip"` to retain symbols.

测试安装是否成功：

The `gdb` tool can be used to verify if the debug symbols are properly installed for the `libjvm` library. For example on Linux:

```shell
$ gdb $JAVA_HOME/lib/server/libjvm.so -ex 'info address UseG1GC'
```

This command's output will either contain `Symbol "UseG1GC" is at 0xxxxx` or `No symbol "UseG1GC" in current context`.

#### 调用栈采集方式

##### `perf-map-agent`方式

> A java agent to generate `/tmp/perf-<pid>.map` files for just-in-time(JIT)-compiled methods for use with the [Linux `perf` tools](https://perf.wiki.kernel.org/index.php/Main_Page). 
>
> => 生成一个`Java`程序的符号快照

安装java:

```shell
sudo apt update
sudo apt install openjdk-11-jdk
```

安装`perf-map-agent`

> <font color='#e54d42'>注意：cmake时要求jdk-8，更高版本还未支持，见 https://github.com/jvm-profiling-tools/perf-map-agent/pull/83</font>

```shell
git clone https://github.com/jvm-profiling-tools/perf-map-agent.git
cd perf-map-agent
cmake .
make
```

安装`FlameGraph`

```shell
git clone https://github.com/brendangregg/FlameGraph
```

设置好`JAVA_HOME`和`perf-map-agent`的正确位置(根据自己安装的位置)：

```shell
# java
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
# perf_map_agent
export AGENT_HOME=/usr/lib/perf-map-agent
```

`FlameGraph`项目提供了[jmaps](https://github.com/brendangregg/FlameGraph/blob/master/jmaps)脚本(就在`FlameCraph`根目录下)，它会调用`perf-map-agent`为当前运行的所有`Java`进程生成符号表

```shell
sudo -E ./jmaps
Fetching maps for all java processes...
Mapping PID 580583 (user root):
wc(1):   2442   7741 143819 /tmp/perf-582656.map
```

> <font color='#e54d42'>要保持符号表都是最新的（间隔越小越好），在每次`profile`之前都需要先调用`jmaps`，因为一旦JVM对Java方法重新编译那么旧版本会失效; 对于大型的`java`程序这个过程可以需要占用1s的`CPU`时间</font>

`Profiling`:

```shell
sudo -E ./jmaps; sudo ./profile -df -p <pid> 10 > out.profile
# 如果要生成火焰图:
./flamegraph.pl --color=java --hash < out.profile > flamegraph.svg
```

如果为了避免JVM内联函数的影响, 可以通过添加`-u`参数实现对符号表内联的展开, 不过牺牲的是临时符号表文件变得更大了

```shell
sudo -E ./jmaps -u; sudo ./profile -df -p <pid> 10 > out.profile
# 如果要生成火焰图:
./flamegraph.pl --color=java --hash < out.profile > flamegraph.svg
```

###### 测试1-宿主机环境：

> 环境：
>
> * 测试`java`程序：`tomcat:9.0.63` （宿主机上运行）

直接使用`bcc-Profile`:

`sudo ./profile -U -p 582656`

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220530203314292.png" alt="image-20220530203314292" style="zoom:50%;" />

使用生成后:

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220530204022252.png" alt="image-20220530204022252" style="zoom: 43%;" />

###### 测试2-容器环境:

> 环境：
>
> * 测试`java`程序：`tomcat:9.0.63` （在容器上运行）

因为容器的隔离环境，所以在使用`sudo -E ./jmaps`的`attach`脚本时会报错找不到`socket`文件，可以复制`attach`依赖的环境(`perf-map-agent/out`)到容器中运行，然后再取出符号文件

> <font color='#e54d42'>风险1: attach程序使用的jdk如果与被attach的jdk不匹配，可能会导致问题</font>

> <font color='#e54d42'>下面这种方式并不好，因为exec对容器的侵入性太大，会给容器额外多出一个进程，如果容器原本资源使用率过大可能会导致容器崩溃</font>

见脚本：`jmaps-docker.sh`

使用方式：

```shell
sudo -E ./jmaps-docker <contianer id>; sudo ./profile -df -p <pid> 10 > out.profile
# 如果要生成火焰图:
./flamegraph.pl --color=java --hash < out.profile > flamegraph.svg
```

##### `async-profiler` 方式（非BPF）

> <font color='#e54d42'>版本限制：</font>
>
> * 宿主机的`glibc`

核心原理在于：`OracleJDK/OpenJDK`内部提供了这么一个函数——`AsyncGetCallTrace`，能够异步采集java进程的所有线程调用栈，并且不受安全点偏差的影响；`async-profiler`项目借助于此实现了对java程序同时采样Java栈与Native栈, 另外，也不需要为被profiling的Java进程设置`-XX:+PreserveFramePointer`参数

---

安装`async-profiler`

```shell
git clone git@github.com:jvm-profiling-tools/async-profiler.git
```

编译`jattach`的`attach`工具(加入到某个java线程中), 注意需要提前设置`JAVA_HOME`：

> 注意：`make`需要`openjdk 11`

```shell
make
```

使用：

```shell
Unrecognized option: --help
Usage: ./profiler.sh [action] [options] <pid>
Actions:
  start             start profiling and return immediately
  resume            resume profiling without resetting collected data
  stop              stop profiling
  dump              dump collected data without stopping profiling session
  check             check if the specified profiling event is available
  status            print profiling status
  list              list profiling events supported by the target JVM
  collect           collect profile for the specified period of time
                    and then stop (default action)
...
```

###### 测试1-宿主机环境:

> 环境：
>
> * 测试`java`程序：`tomcat:9.0.63` （宿主机上运行）

`sudo ./profiler.sh -d 10 666683`

![image-20220531200708016](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220531200708016.png)

###### 测试2-容器环境:

> 环境：
>
> * 测试`java`程序：`tomcat:9.0.63` （在容器上运行）

使用：

` sudo bash -x ./profiler.sh -d 10 --fdtransfer <pid>`

![image-20220606162701078](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220606162701078.png)

##### 未来方式

* `JVM`内建的符号转储支持：如果`JVM`支持类似于`perf-map-agent`方式设置一个线程直接不断将符号写入`/tmp/perf-PID`文件的话(可由外部信号触发)，那么Profile的性能就会大大提高
* `BPF`内核层面支持：使用内核内部的符号翻译改进调用栈信息的收集；目前社区还在讨论

### 3.5 Python (pyspy)

#### 调研结论

因为是解释型语言，运行时最终不会编译为机器码，而是通过自身内置的子函数/解释器进行语法分析和执行

解释器编译时如果启用了栈指针，那么可以基于栈指针进行工作，<font color='#e54d42'>但是调用栈只能体现解释器的内部运作而不能体现用户函数</font>

对于`java`语言来说，`jdk`提供方实现了例如`AsyncGetCallTrace`这样的方式，可以将编译器/解释器的原始调用栈还原为用户程序调用栈

#### 社区方案总结

目前的Profile方案基本就是需要借助于**对函数源代码有侵入**：

* `cProfile、profile`

  ```python
  import cProfile, pstats, StringIO
  pr = cProfile.Profile()
  pr.enable()
  # ... do something ...
  pr.disable()
  s = StringIO.StringIO()
  sortby = 'cumulative'
  ps = pstats.Stats(pr, stream=s).sort_stats(sortby)
  ps.print_stats()
  print s.getvalue()
  ```

* `hotshot`

  ```python
  >>> import hotshot, hotshot.stats, test.pystone
  >>> prof = hotshot.Profile("stones.prof")
  >>> benchtime, stones = prof.runcall(test.pystone.pystones)
  >>> prof.close()
  >>> stats = hotshot.stats.load("stones.prof")
  >>> stats.strip_dirs()
  >>> stats.sort_stats('time', 'calls')
  >>> stats.print_stats(20)
  ```

* 除此之外还有一种采样方案就是`Instrumentation`, 此类依靠`USDT`来实现程序调用栈剖析，但是这样的采样方式因为要对几乎每个函数插桩，所以性能消耗非常大，所以只适合某些单独函数的调用次数分析、trace调用栈分析等，不适合大规模的Profile; 

> [How to trace a Python application with eBPF/BCC](https://ish-ar.io/python-ebpf-tracing/)

##### Py-spy项目

https://github.com/benfred/py-spy

基本原理：

* 通过[process_vm_readv](http://man7.org/linux/man-pages/man2/process_vm_readv.2.html) 系统调用读取python进程内存

  > py-spy通过在Linux上使用process_vm_readv系统调用、在OSX上使用vm_read调用或在Windows上使用ReadProcessMemory调用直接读取python程序的内存来工作

* 计算Python程序的调用堆栈：通过查看全局`PyInterpreterState`变量来获得解释器中运行的所有`Python`线程，然后在每个线程中遍历每个`PyFrameObject`来获得调用堆栈

* 由于Python ABI在不同版本之间会发生变化，我们使用`rust`的`bindgen`为我们关心的每个`Python`解释器类生成不同的`rust`结构，并使用这些生成的结构来计算`Python`程序中的内存布局

容器化需要开启的权限：

* `- SYS_PTRACE`因为需要`process_vm_readv`这样的系统调用

副作用：

* 默认会阻塞python进程，但是可以通过`--nonblocking`选项不终止python进程，然而带来的影响就是采样的结果可能不会准确

## 5. profile内核限制

测试虚拟机环境

> * 内核版本：`5.13.0-41-generic`
> * `OS`：`Ubuntu 21.10`
> * `Arch`: `x86_64`
> * `llvm/clang`: `13.0.0` 

无关语言，profile本身需要支持的条件：

* `Stack trace`  内核版本最低要求>=4.6  [`d5a3b1f69186`](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=d5a3b1f691865be576c2bffa708549b8cdccda19)

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220524211954649-20220530153203655.png" alt="image-20220524211954649" style="zoom:33%;" />

## 6. 初步设计

### 如何获取debuginfo信息？

运行在宿主机NS环境下，怎样获取`Pod`中的执行进程的`debuginfo`信息？

初步逻辑：

进程cmd、cwd => 找到二进制位置 => 同级目录下(要求)找到`.debug`文件

> 最好情况：不用权限或最小的权限获得容器内的文件

文件关系：

* 找文件逻辑 => 在映射`Union FS`中寻找容器目录
* 找容器Pid

> 自己的想法：
>
> * `WORKDIR`确定工作目录即二进制文件所在目录 (可选)
> * `docker exec -w `: 通过`objcopy`帮助生成`.debug`文件（前提没被`strip`）
> * `docker cp`: 辅助到宿主机获取`.debug`文件

### 如何判断不同语言(c/c+、go、java)？

初步逻辑：

> file使用版本: `5.39`

通过`file`命令判断:

![image-20220531105133743](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220531105133743.png)

### 二进制文件elf是否自带debuginfo?BCC是否能识别?

默认自带，经过测试如果自带debug信息的二进制文件bcc可识别

# 二、错误记录

## 2.1 attach java进程失败

错误描述：

![image-20220531142841144](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220531142841144.png)

> 注: 被attach的tomcat程序我是用容器启动的，而需要attach的程序在宿主机环境

原因：

* 就是报错的原因: 需要attach的程序找不到正在运行的java程序创建的`/tmp/hsperfdata_<user_name>/<pid>`这个`socket`文件, 找不到的原因有两个：

  * 需要attach的程序的用户名与被attach的用户名不一致，导致查找的路径不对所以找不到，可见`/tmp/hsperfdata_*`下的多个用户：

    ![image-20220531145922609](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220531145922609.png)

  * 因为被attach的java程序运行在容器中，生成的临时文件也在容器fs中并且其`pid`为`1`，所以在宿主机环境下，无法找到此文件也无法`pid`匹配正确（因为宿主机视角下检测到容器进程的`pid`是不一样的）

解决方法：复制文件到容器中

## 2.2 async-profiler

### 1. 环境描述

* Linux ubuntu-impish 5.13.0-44-generic #49-Ubuntu SMP Wed May 18 13:28:06 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

* async-profiler git commit: `58a3cb0d252a115cbb0ec55d3073ddc068d7022b`
* tomcat：9.0

### 2. 问题描述

容器中跑的`tomcat java`进程，然后在宿主机上使用`attach`的方式（使用`jattach`工具）时出现错误：

```shell
Failed to inject profiler into 22491
```

(`22491`是宿主机视角下tomcat容器的`pid`)

### 3. 排查思路

> 参考：
>
> * https://github.com/jvm-profiling-tools/async-profiler/issues/245

因为错误输出非常少，所以先看这个`profiler.sh`脚本的所有命令：

```shell
sudo -E bash -x ./profiler.sh -d 10 22491
```

```shell
+ set -eu
+ SCRIPT_BIN=./profiler.sh
+ '[' -h ./profiler.sh ']'
+++ dirname ./profiler.sh
++ cd .
++ pwd -P
+ SCRIPT_DIR=/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler
+ JATTACH=/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/jattach
+ FDTRANSFER=/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/fdtransfer
+ USE_FDTRANSFER=false
+ PROFILER=/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so
+ ACTION=collect
+ DURATION=60
+ FILE=
...
+ shift
+ '[' 0 -gt 0 ']'
+ '[' 22491 = '' ']'
+ '[' true = true ']'
+ FILE=/tmp/async-profiler.37876.22491
+ LOG=/tmp/async-profiler-log.37876.22491
++ uname -s
+ UNAME_S=Linux
+ '[' Linux = Linux ']'
+ ROOT_PREFIX=/proc/22491/root
+ case $ACTION in
+ fdtransfer
+ '[' false = true ']'
+ jattach start,file=/tmp/async-profiler.37876.22491,
+ set +e
+ /home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/jattach 22491 load /home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so true start,file=/tmp/async-profiler.37876.22491,,log=/tmp/async-profiler-log.37876.22491
Connected to remote JVM
JVM response code = -1
/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so was not loaded.
/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so: cannot open shared object file: No such file or directory

+ RET=255
+ '[' 255 -ne 0 ']'
+ '[' 255 -eq 255 ']'
+ echo 'Failed to inject profiler into 22491'
Failed to inject profiler into 22491
+ '[' Linux = Darwin ']'
+ LD_PRELOAD=/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so
+ /bin/true
+ mirror_log
+ '[' -f /tmp/async-profiler-log.37876.22491 ']'
+ '[' -f /proc/22491/root/tmp/async-profiler-log.37876.22491 ']'
+ exit 255
```

关键点在于执行`jattach`：

```shell
/home/vagrant/.../async-profiler/build/jattach 22491 load /home/vagrant/.../async-profiler/build/libasyncProfiler.so true start,file=/tmp/async-profiler.37876.22491,,log=/tmp/async-profiler-log.37876.22491
Connected to remote JVM
JVM response code = -1
/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so was not loaded.
/home/vagrant/eBPF/tools_study/profile_tools/java_perf_map_test/async-profiler/build/libasyncProfiler.so: cannot open shared object file: No such file or directory
```

在看`jattach`的执行步骤，使用`strace`排查：

```shell
sudo -E strace -s999 /home/vagrant/.../async-profiler/build/jattach 22491 load /home/vagrant/.../async-profiler/build/libasyncProfiler.so true start,file=/tmp/async-profiler.37876.22491,,log=/tmp/async-profiler-log.37876.22491
```

部分输出：

```shell
...
write(3, "start,file=/tmp/async-profiler.37876.22491,,log=/tmp/async-profiler-log.37876.22491\0", 84) = 84
read(3, "-1\n/home/vagrant/.../async-profiler/build/libasyncProfiler.so was not loaded.\n/home/vagrant/.../async-profiler/build/libasyncProfiler.so: cannot open shared object file: No such file or directory\n", 8191) = 288
write(1, "JVM response code = -1\n/home/vagrant/.../async-profiler/build/libasyncProfiler.so was not loaded.\n/home/vagrant/.../async-profiler/build/libasyncProfiler.so: cannot open shared object file: No such file or directory\n", 308JVM response code = -1
/home/vagrant/.../async-profiler/build/libasyncProfiler.so was not loaded.
/home/vagrant/.../async-profiler/build/libasyncProfiler.so: cannot open shared object file: No such file or directory
) = 308
read(3, "", 8192)                       = 0
```

看来是被`attach`的`JVM`进程的问题：

根据https://github.com/jvm-profiling-tools/async-profiler#troubleshooting ：

The connection with the target JVM has been established, but JVM is unable to load profiler shared library. **Make sure the user of JVM process has permissions to access `libasyncProfiler.so`** by exactly the same absolute path. For more information see [#78](https://github.com/jvm-profiling-tools/async-profiler/issues/78).

根据上面的重点，就可知道问题的原因在于传递的`libasyncProfiler.so`此文件的路径参数，在被`attach`的进程中必须能访问到的，而容器环境下宿主机上的绝对路径容器是一定无法访问的，所以出现了找不到

根据此思路在运行`jattach`之前，将此`.so`文件复制到容器的`WorkDir`中, 然后修改传入的参数为`./libasyncProfiler.so`即可

但是又出现了新的问题：

```shell
./libasyncProfiler.so was not loaded.
/lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found (required by ./libasyncProfiler.so)
```

`objdump -T build/libasyncProfiler.so | grep GLIBC_2.33 ` 查看`.so`文件的动态符号表:

```shell
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.33  stat
```

> **GNU C库**，又名**glibc**，是GNU计划所实现的C标准库。尽管其名字中带有“C库”，但它现在也直接支持C++（以及间接支持其他编程语言） —维基百科

分别在宿主机和容器中查看`glibc`版本

宿主机：

```shell
$ ldd --version
ldd (Ubuntu GLIBC 2.35-0ubuntu3) 2.35
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

容器：

```shell
$ ldd --version
ldd (Debian GLIBC 2.31-13+deb11u3) 2.31
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

在容器中查看`libasyncProfiler.so` 文件

```shell
$ ldd libasyncProfiler.so 
./libasyncProfiler.so: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found (required by ./libasyncProfiler.so)
./libasyncProfiler.so: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./libasyncProfiler.so)
	linux-vdso.so.1 (0x00007fff441f7000)
	libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f7e9b530000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f7e9b516000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7e9b351000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f7e9b20d000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f7e9b750000)
```

所以解决方案有两个：降低宿主机版本 OR 升级容器版本

经过和项目人员沟通：https://github.com/jvm-profiling-tools/async-profiler/issues/601

官方提供了在低版本内核编译好的`libasyncProfiler.so`，测试后可行

> 记录关于动态链接的基础：
>
> 动态链接器`ld-linux.so`按照下面的顺序来搜索需要的动态共享库对于elf格式的可执行程序，是由`ld-linux.so*`来完成的，它先后搜索:
>
> * elf文件的`DT_RPATH`段（不可控）
> * 环境变量`LD_LIBRARY_PATH` 
> * `/etc/ld.so.cache`文件列表 
> * `/lib/`和`/usr/lib `目录找到库文件后载入内存

然后出现了新的问题：

```shell
Connected to remote JVM
JVM response code = 0
return code: 200

[WARN] Kernel symbols are unavailable due to restrictions. Try
  sysctl kernel.kptr_restrict=0
  sysctl kernel.perf_event_paranoid=1
[WARN] perf_event_open for TID 1 failed: Operation not permitted
[WARN] perf_event_open for TID 10 failed: Operation not permitted
[WARN] perf_event_open for TID 11 failed: Operation not permitted
[WARN] perf_event_open for TID 12 failed: Operation not permitted
[WARN] perf_event_open for TID 13 failed: Operation not permitted
[WARN] perf_event_open for TID 14 failed: Operation not permitted
[WARN] perf_event_open for TID 15 failed: Operation not permitted
[WARN] perf_event_open for TID 16 failed: Operation not permitted
[WARN] perf_event_open for TID 17 failed: Operation not permitted
...
[WARN] perf_event_open for TID 38 failed: Operation not permitted
[WARN] perf_event_open for TID 39 failed: Operation not permitted
[WARN] perf_event_open for TID 40 failed: Operation not permitted
[WARN] perf_event_open for TID 41 failed: Operation not permitted
[WARN] perf_event_open for TID 42 failed: Operation not permitted
[WARN] perf_event_open for TID 43 failed: Operation not permitted
[WARN] perf_event_open for TID 44 failed: Operation not permitted
[ERROR] No access to perf events. Try --fdtransfer or --all-user option or 'sysctl kernel.perf_event_paranoid=1'
```

原因：在容器环境下无法给每个线程打开`perf_event`事件(因为没有给`perf_event_open`系统调用授权)，所以无法在`jvm`中实现调用栈的追踪，此外系统内核的符号表文件也收到`kernel.kptr_restrict`的限制；

解决：`async-profiler`官方给出了解决思路：https://github.com/jvm-profiling-tools/async-profiler#profiling-java-in-a-container

所以采用第二种方式`–fdtransfer`, 解决。

# 三、基本原理

介绍依赖技术的基本原理与工作流程

## 3.1 BPF-Profile的基本原理

[eBPF-3-profile的源码解析](/Users/xwj/blog/source/_posts/技术贴/eBPF/eBPF-3-profile的源码解析.md)

## 3.2 async-profiler的基本原理

### 1. fdtransfer基本原理

此方法出现的原因：

为了解决对于容器隔离环境下，attach到目标java程序中，但是目标java程序的容器环境并不会开启`perf_event_open`这样系统调用的特权以及访问`/proc/kallsyms`内核动态符号表的特权所以出现的解决方法

文件：

* 服务端文件: `src/fdtransfer`
* 客户端文件: `src/fdtransferClinet_linux.cpp`、`src/fdtransferClinet.h`

核心思路：将容器中JVM进程的系统调用以及内核符号表的请求通过socket转发给宿主机上的`fdtransfer`服务器，由宿主机来实现这些请求

1. 进入容器的`net NS`环境
2. 开启socket 服务器并监听（宿主机进程）
   * 进入容器`Pid NS`，然后fork子进程，调用`accept`和`serverRequests`
3. 处理两类请求：
   1. `case PERF_FD`:  帮助JVM进程中管理的所有线程调用`perf_event_open`系统调用
   2. `case KALLSYMS_FD`：复制一份`/proc/kallsyms`文件并把其`fd`传送给客户端
