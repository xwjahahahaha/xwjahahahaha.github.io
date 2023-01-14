---
title: 使用BCC制作exporter用于prometheus采集和grafana监控
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-19 14:36:22
---

[TOC]


<!-- more -->

> 参考：
>
> * https://blog.csdn.net/qq_34258344/article/details/115532126
> * https://github.com/cloudflare/ebpf_exporter

介绍了基本bcc-exporter的数据整个采集->可视化过程（重点在于exporter代码中bcc编写的注意事项），并且在最后详细的介绍了**延迟型数据/指标可视化–热点直方图heatmap的解析和使用**

# 一、安装基本依赖

安装`ebpf-exporter`, https://github.com/cloudflare/ebpf_exporter

```shell
go get -u -v github.com/cloudflare/ebpf_exporter/...					# 操作相当于go install
# 如果你的GOBIN已经在PATH中了，那么就不需要执行以下操作
cp -ip go/bin/ebpf_exporter /usr/local/bin/ebpf_exporter
```

测试安装是否成功：

```shell
ebpf_exporter --help
```

![image-20220719144614435](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220719144614435.png)

安装Prometheus、grafana，自行下载：

* https://prometheus.io/download/
* https://grafana.com/grafana/download?pg=get&plcmt=selfmanaged-box1-cta1

# 二、编写export的yaml文件

具体详细的规则见：https://github.com/cloudflare/ebpf_exporter

以收集NFS的延迟为例，具体的yaml文件可以编写为(`nfsdist.yaml`)：

```yaml
programs:
  - name: nfs_latency
    metrics:
      histograms:
        - name: nfs_latency_seconds
          help: nfs_latency traces NFS reads, writes, opens, and getattr
          table: dist
          bucket_type: exp2            # log2直方图
          bucket_min: 0
          bucket_max: 26                # 共28个槽(这里27个+Inf一个)，最大的区间为2^25~2^26（33.6s .. 67.1s）
          bucket_multiplier: 0.000001   # 乘数: bfp微妙=>秒
          labels:
            - name: operation
              size: 8
              decoders:
                - name: uint
                - name: static_map
                  static_map:
                    1: read
                    2: getattr
                    3: write
                    4: open
            - name: bucket
              size: 8
              decoders:
                - name: uint
    kprobes:
      nfs_file_read: trace_entry
      nfs_file_write: trace_entry
      nfs4_file_open: trace_entry
      nfs_getattr: trace_entry
    kretprobes:
      nfs_file_read: trace_read_return
      nfs_file_write: trace_write_return
      nfs4_file_open: trace_open_return
      nfs_getattr: trace_getattr_return
    code: |
      #include <uapi/linux/ptrace.h>
      #include <linux/fs.h>
      #include <linux/sched.h>
      #define OP_NAME_LEN 8

      // 27 buckets for latency
      const u8 max_latency_slot = 26;

      // 四种操作
      enum opration {
        NFT_READ = 1,
        NFT_GETATTR = 2,
        NFT_WRITE = 3,
        NFT_OPEN = 4
      };

      struct dist_key {
        u8 op;                                // 注意内存对齐
        u64 slot;
      };


      BPF_HASH(start, u32);                   // 记录时间差 pid => start_time
      // (max_latency_slot + 2) 是因为有28个槽
      // *4 是因为共有4类操作，每个28个槽
      BPF_HISTOGRAM(dist, struct dist_key, (max_latency_slot + 2) * 4);

      // time operation
      int trace_entry(struct pt_regs *ctx)
      {
        u64 pid_tgid = bpf_get_current_pid_tgid();
        u32 pid = pid_tgid >> 32;
        u32 tid = (u32)pid_tgid;
        u64 ts = bpf_ktime_get_ns();            // 记录开始的时间
        start.update(&tid, &ts);
        return 0;
      }

      static int trace_return(struct pt_regs *ctx, enum opration op)
      {
        // get pid
        u64 *tsp;
        u64 pid_tgid = bpf_get_current_pid_tgid();
        u32 pid = pid_tgid >> 32;
        u32 tid = (u32)pid_tgid;

        // fetch timestamp and calculate delta
        tsp = start.lookup(&tid);       // 获取过去的时间
        if (tsp == 0) {
          return 0;   // missed start or filtered
        }
        u64 delta = (bpf_ktime_get_ns() - *tsp) / 1000;

        // 将记录的delta经过log2映射到对应的整数桶
        u64 latency_slot = bpf_log2l(delta);

        // Cap latency bucket at max value
        if (latency_slot > max_latency_slot) {
            latency_slot = max_latency_slot;
        }

        // store as histogram
        struct dist_key key = {};
        key.op = op;
        key.slot = latency_slot;

        // 累加到直方图中
        dist.increment(key);
        start.delete(&tid);
        return 0;
      }

      int trace_read_return(struct pt_regs *ctx)
      {
        return trace_return(ctx, NFT_READ);
      }

      int trace_write_return(struct pt_regs *ctx)
      {
        return trace_return(ctx, NFT_WRITE);
      }

      int trace_open_return(struct pt_regs *ctx)
      {
        return trace_return(ctx, NFT_OPEN);
      }

      int trace_getattr_return(struct pt_regs *ctx)
      {
        return trace_return(ctx, NFT_GETATTR);
      }
```

## 编写注意事项

关于yaml的基本label见ebpf_exporter官方的说明，下面是几个官方没介绍的重点

### 1. 注意attach对应版本的内核函数

例如，因为使用的是`NFS4`, 所以如果直接使用`nfs_file_open`这样的系统调用是不可行的，因为新版本更新了系统的调用

如果发现attach的函数没有采集到数据，就需要考虑是否attach错了函数

### 2. 怎样理解BPF_HISTOGRAM(name [, key_type [, size ]])的第三个参数size？

从bcc的官方使用手册中可以看到基本设定：https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#4-bpf_histogram

Syntax: `BPF_HISTOGRAM(name [, key_type [, size ]])`

Creates a histogram map named `name`, with optional parameters.

Defaults: `BPF_HISTOGRAM(name, key_type=int, size=64)`

For example:

```
BPF_HISTOGRAM(dist);
```

This creates a histogram named `dist`, **which defaults to 64 buckets indexed by keys of type int.**

This is a wrapper macro for `BPF_TABLE("histgram", ...)`.

Methods (covered later): map.increment().

也就是说size是直方图中桶的数量的设定，那么这个桶的数量就与`key_type`的种类数相关(默认是64)

例如：

```c
...
// 27 buckets for latency, max range is 33.6s .. 67.1s
const u8 max_latency_slot = 26;
// Key for latency measurements map
struct latency_key_t {
  u8 op;
  u64 slot;
};

// Histograms to record latencies (2 is the number of ops in reclaim_op)
BPF_HISTOGRAM(xfs_reclaim_latency, struct latency_key_t, (max_latency_slot + 2) * 2);

// Possible reclaim operations
enum reclaim_op {
  S_COUNT = 1,
  S_FREE  = 2
};
...
```

这里`(max_latency_slot + 2) * 2`计算的解释如下：

* 看`latency_key_t`结构体的两个参数，首先是`slot`, 已经定义了其最大为26，从0～26共有27个槽，再加上`+Inf`（表示最大槽到无穷大），所以一共有28个槽
* `*2`的原因在注释中也解释了，因为`op`（也就是操作有两种，见下面的`reclaim_op`），所以每一种有28个，共两种就需要`*2`，这就是总槽数

类似的还可以看到：官方工具cpudist中https://github.com/iovisor/bcc/blob/c1a767b34fc800436f7fe90f2e5614b2c27bc376/tools/cpudist.py

![image-20220721103926046](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220721103926046.png)

为了得到最大槽数，还需要找到最大的pid

### 3. 注意内存对齐

对于上面的例子中, `dist_key`结构：

```c
struct dist_key {
  u8 op;                                // 注意内存对齐
  u64 slot;
};
```

对应了上面的label：

```yaml
labels:
- name: operation
	size: 8
	decoders:
		- name: uint
		- name: static_map
			static_map:
				1: read
				2: getattr
				3: write
				4: open
- name: bucket
	size: 8
decoders:
	- name: uint
```

为什么两个要设置8的字节大小呢，因为`u64`占据8字节，所以为了内存对齐，即使`u8`是一个字节，也需要`pendding`填充7个字节，所以两个都需要是8字节，否则启动就会`decode`报错，形如：

```shell
2022/07/21 03:31:52 Error getting table "dist" values for metric "nfs_latency_seconds" of program "nfs_latency": error decoding labels: total size of key []byte{0x4, 0x0, 0x0, 0x0, 0x91, 0xcf, 0x11, 0x0, 0x5, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0} is 16 bytes, but wehave labels to decode 20
```

所以，在bcc的c代码中，结构体的字段要注意内存对齐，一般的做法就是按字节大小，从小到大排序，例如:

```c
struct dist_key {
  u8 op;                                
  u32 xxx;
  u64 slot;
};
```

## 启动

使用docker启动（注意宿主机上需要安装对应内核版本的linux-header，因为exporter的启动依赖头文件）

```shell
sudo docker run -it --rm --privileged \
    -v $(pwd)/nfsdist.yaml:/root/go/bin/nfs/nfsdist.yaml \
    -v /lib/modules/4.19.0-0.bpo.9-amd64/build:/lib/modules/4.19.0-0.bpo.9-amd64/build \
    -v /lib/modules/4.19.0-0.bpo.9-amd64/source:/lib/modules/4.19.0-0.bpo.9-amd64/source \
    -v /sys/kernel/debug:/sys/kernel/debug \
    -p 9435:9435 \
    ebpf_exporter \
    /bin/bash
```

启动：`cd /root/go/bin/ && ./ebpf_exporter --config.file=./nfs/nfsdist.yaml`

访问`:9435`接口：

![](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720134415914.png)

# 三、配置prometheus、grafana

其实也就是在默认配置中添加一个对应的采集接口：

```yaml
# my global config
global:
  scrape_interval: 5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 5s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
    
  # 采集ebpf_export的数据
  - job_name: 'ebpf_nfs_deplay'
    static_configs:
      - targets: ["localhost:9435"]    

```

开启prometheus：

`./prometheus --config.file="prometheus.yml"`

打开9090端口，查看target，发现正常：

![image-20220720141746929](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720141746929.png)

![image-20220720142801169](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720142801169.png)

关于grafana和prometheus的更多配置见：

http://kerneltravel.net/blog/2021/ljr-ebpf10/

# 四、延迟型数据的可视化-heatmap

> 参考：
>
> * https://grafana.com/blog/2020/06/23/how-to-visualize-prometheus-histograms-in-grafana/
> * https://towardsdatascience.com/prometheus-histograms-with-grafana-heatmaps-d556c28612c7

基本概念：

* 桶：柱状图一般会将数据分割为不同的间隔（可以为固定间隔，也可以为递增倍数的间隔，例如2的指数），并在单一的区间内统计次数或者其他计数操作形成柱子的高度
* heatmap：热力图，类似于直方图，但随着时间的推移，每个时间切片都代表自己的直方图。它不是使用条形高度作为频率的表示，而是使用单元格，并根据桶中值的数量为单元格着色。一般来说配合prometheus得直方图类型指标数据使用

> 为什么有了柱状图还需要热力图？
>
> * 因为热力图比柱状图多了时间序列切片，更能反应时间序列上的变化，直方图的问题在于无法看到随时间分布的任何趋势或变化
>
>   可见如下对比：
>
>   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720205633317.png" alt="image-20220720205633317" style="zoom:50%;" />
>
>   <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720205559674.png" alt="image-20220720205559674" style="zoom:50%;" />

## 源数据/prometheus直方图数据

Prometheus 直方图由三个元素组成： `xxx_count`计数样本数；`xxx_sum`总结所有样本的值；最后是一组`xxx_bucket`带有标签的多个桶，该标签`le`包含<font color='#e54d42'>所有</font>样本的计数，其值小于或等于标签中包含的数值`le` （关于标签详细见：https://prometheus.io/docs/concepts/metric_types/#histogram）

> 上面强调了“所有”这个词，因为 Prometheus 将样本放在它适合的所有桶中，而不仅仅是它适合的第一个桶。换句话说，桶的计数是累积的

一个简单的例子为：图片托管网站接收大小从几字节到几兆字节不等的图片，我们将存储桶设置为 64 字节到 16MB 之间的指数规模（每个存储桶的大小是前一个存储桶的四倍）

```shell
uploaded_image_bytes_bucket{le="64"}
uploaded_image_bytes_bucket{le="256"}
uploaded_image_bytes_bucket{le="1024"}
uploaded_image_bytes_bucket{le="4096"}
uploaded_image_bytes_bucket{le="16384"}
uploaded_image_bytes_bucket{le="65536"}
uploaded_image_bytes_bucket{le="262144"}
uploaded_image_bytes_bucket{le="1048576"}
uploaded_image_bytes_bucket{le="4194304"}
uploaded_image_bytes_bucket{le="16777216"}
uploaded_image_bytes_bucket{le="+Inf"}
uploaded_image_bytes_total
uploaded_image_bytes_count
```

> **为什么prometheus的直方图数据要累计？**
>
> * 虽然每个桶都单独计算累计更加直观（符合直方图的思路），但是如果需要统计时就比麻烦了，例如需要知道（在上面的例子中）*上传了多少小于（或等于）1MB 的文件？*可以立即在`uploaded_image_bytes_bucket{le="1048576"}`桶的计数中得到答案，而不需要从一开始的`64`的桶逐个累加，当然还有其他计算的方便，例如*256KB 和 1MB 之间的存储桶中有多少个文件？* 则可以通过`uploaded_image_bytes_bucket{le="1048576"} - ignoring(le) uploaded_image_bytes_bucket{le="262144"}`计算得到

## grafana的heatmap支持

对于此类源数据，grafana提供了hearmap的支持：https://github.com/grafana/grafana/pull/11087

可以继续使用直方图附带热点，但是这样还是无法体现时间变化趋势，如下图：

![image-20220720211322194](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720211322194.png)

Grafana还有一个Heatmap**面板**。当我们设置了直方图条规后，如果我们将面板类型切换为**热**图

具体步骤如下：

* 选择使用`heatmap`
* 数据格式一定要选择*时间序列桶(time series buckets)*
* 查询格式要设置为`sum(increase(xxxx_bucket[$__interval])) by (le)`,  `increase`更改我们的查询以显示每个直方图块的增加而不是存储桶的总数， `sum by (le)`	是根据桶分组统计
* 最后（可选），可以将**Query options > Max data points**设置为 25（改变每个单元格宽度，热图已经包含了很多信息，当分辨率太高时，它们会降低浏览器的速度）

![image-20220720215611294](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220720215611294.png)

