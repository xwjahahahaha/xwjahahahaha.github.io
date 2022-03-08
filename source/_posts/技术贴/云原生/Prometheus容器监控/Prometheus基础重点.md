---
title: Prometheus官方文档学习纪要
tags:
  - hide
categories:
  - technical
  - Prometheus
toc: true
declare: true
date: 2022-01-10 16:42:05：
---

> 参考自：
>
> * 官网文档https://prometheus.io/docs/introduction/first_steps/
>* https://prometheus.io/docs/guides/go-application/
> * https://blog.csdn.net/easylife206/article/details/102480764
> * https://blog.csdn.net/u014029783/article/details/80001251

记录学习与使用prometheus的重点，建议先过一遍官方文档：

# 一、基本介绍

Prometheus（Go语言开发）是由`SoundCloud`开发的开源监控告警系统和时序列数据库。从字面上理解，**Prometheus由两个部分组成，一个是监控告警系统，另一个是自带的时序数据库（TSDB）**。

<!-- more -->

![image-20220110193658682](http://xwjpics.gumptlu.work/image-20220110193658682.png)

Prometheus server从各个`exporter`拉取数据，或者间接地通过网关`Pushgateway`拉取数据，它默认本地存储抓取的所有数据。采集到的数据有两个去向，一个是可视化，另一个是告警。`PromQL`和其他`API`可视化地展示收集的数据，通过`Alertmanager`提供告警能力。

对比其他监控方案：

* `Ceilometer`：OpenStack中用来做计量计费功能的一个组件，后来又逐步发展增加了部分监控采集、告警的功能。
* `Nightingale`：（夜莺，简称n9e，Go语言开发）是滴滴开源的一个企业级监控解决方案。能达到企业级别的需求

**Prometheus的优缺点：**

优点：

* 简单轻量、自带时序服务器
* 处理能力强：指标都存储在Server本地数据库中，单个实例可以处理数百万的 Metrics
* 灵活的数据模型：引入了 Tag，给不同的接收端打好标识，属于多维数据模型，聚合统计更方便
* 强大的查询语句：`PromQL` 允许在同一个查询语句中，对多个 `Metrics `进行加法、连接等操作。
* 生态与社区：很好兼容k8s，社区有很多常见应用的`exporter`可以使用

缺点：

* 自带的时序服务器不支持**降采样**

  > 即通过降低数据精度，来达到存储更长时间历史数据的目的。比如，12个5秒精度的点，通过计算均值，合并成1个1分钟精度的点。

* 中心化`server`不支持拓展，容灾问题（社区目前已经有一些高可用的方案）

启动：

* 携带配置文件启动：`./prometheus --config.file prometheus.yml`

# 二、配置文件

## 1. 基本结构

```yaml
# my global config
global:
  scrape_interval:     15s				# 获取目标数据的频率
  evaluation_interval: 15s				# 评估规则的频率

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

```

* `global` : 全局共享配置
* `alerting`：和告警相关的配置
* `rule_files`: Prometheus服务器加载的规则，并通过这些规则去评估（`evaluation_interval`）
* `scrape_configs` : 配置Prometheus需要抓取/监控的资源点（其自身也是http接口，所以自身已经加入到了评估）

## 2. 其他配置

# 三、核心概念

## 1. 数据模型Data model

* Prometheus 从根本上将所有数据存储为**时间序列**：属于同一指标和同一组标记维度的时间戳值流
* 每一个时间序列都被其**指标名**和**标签**（k/v键值对）标记
* **指标名(metric name)**：一般用于指定被测量系统的一般性特征（例如http服务器的被请求次数等）
  * 由数字、字母、`_`、`:`组成，其中`:`是留给用户规则定义的指标

* **标签(label)**：所有同一个指标名的所有标签就定义了这个指标的各项数据维度，这些数据维度也用于查询时的筛选
  * 更改、添加、修改一个指标名的任意标签都会创建一个新的时间序列
  * 由数字、字母、`_`、`:`组成，其中`_`是供内部使用的指标
  * 空标签值得标签可以认为是不存在的标签
* 一般的结构： `<metric name>{<label name>=<label value>, ...}`，例如`api_http_requests_total{method="POST", handler="/messages"}`

## 2. 指标类型 Metric types

四类：

* `Counter`: 计数器，不断累计数量单调递增，只有在重启/重置时变为0, 例如可以用一个计数器去统计当前请求的数量
  * 注意，不要使用计数器统计会递减的指标
* `Gauge`：测量指标，表示可以任意增加或减少的一种度量，测量指标通常用于测量值，例如温度或当前内存使用情况

* `Histogram` : 直方图，直方图对观察结果进行采样（通常是请求持续时间或响应大小等），并将它们计入可配置的存储桶中。 它还提供所有观察值的总和。

* `Summary `: 概略图与直方图类似，摘要对观察结果进行采样（通常是请求持续时间和响应大小等）,虽然它还提供了观察总数和所有观察值的总和，但它计算了滑动时间窗口上的可配置分位数。

Histogram和Summary使用的频率较少，两种都是基于采样的方式。另外有一些库对于这两个指标的使用和支持程度不同，有些仅仅实现了部分功能。

这两个类型对于某一些业务需求可能比较常见，比如查询单位时间内：总的响应时间低于300ms的占比，或者查询95%用户查询的门限值对应的响应时间是多少。 

**使用Histogram和Summary指标的时候同时会产生多组数据，\_count代表了采样的总数，\_sum则代表采样值的和。 _bucket则代表了落入此范围的数据。**

下面是使用historam来定义的一组指标，计算出了平均五分钟内的查询请求小于0.3s的请求占比总量的比例值。

```shell
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (job) /
sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

## 3. 任务和实例 JOBS AND INSTANCES

* 在Prometheus中能够获取到指标数据的终端都称之为**实例 instances**，一般对应一个进程
* 具有相同目的的实例集合，例如为可扩展性或可靠性而复制的流程，称为**作业 job**。

* 例如，四个复制的实例总体是一个job：

  ```shell
  job: api-server
      instance 1: 1.2.3.4:5670
      instance 2: 1.2.3.4:5671
      instance 3: 5.6.7.8:5670
      instance 4: 5.6.7.8:5671
  ```

* Prometheus抓取目标指标数据的时候会默认添加一些标签：
  * `job`：目标所属的作业名称
  * `instance` : 被抓取的URL(<http>:<port>)

# 四、编写客户端

目前prometheus提供的编写获取指标的客户端sdk库已经非常丰富，包含了很多语言：[10 languages already supported](https://prometheus.io/docs/instrumenting/clientlibs) ，虽然语言不同但是都需要遵循一套基本规则:

https://prometheus.io/docs/instrumenting/writing_clientlibs/

# 五、编写自己的Export

https://blog.csdn.net/easylife206/article/details/102480764

## 1. Exporter的作用

对于自己linux机器上的一些指标（大量服务、硬件、网络等），我们期望将其暴露给prometheus处理，但是其格式并不是prometheus期望的指标格式。

prometheus期望的一般指标格式：

```shell
# HELP http_requests_total The total number of HTTP requests.	
# TYPE http_requests_total counter	
http_requests_total{method="post",code="200"} 1027	
http_requests_total{method="post",code="400"}    3
```

特定系统某些信息的格式:

* 通过`/proc`目录下这样的文件系统暴露系统信息；

- Redis 的监控信息需要通过 INFO 命令获取;
- 路由器等硬件的监控信息需要通过 `SNMP**` 协议获取;

所以一般的流程为：

<img src="http://xwjpics.gumptlu.work/image-20220124163016951.png" alt="image-20220124163016951" style="zoom:67%;" />

对于上面那些常见的情形, 社区早就写好了成熟的` Exporter`, 它们就是` node_exporter`, `redis_exporter` 和 `snmp_exporter`.

**整合监控：**exporter可以用于将其他服务的监控接入到prometheus中，这样就可以通过prometheus进行统一的监控，其还提供了promQL这样强大的数据处理功能

## 2. Export编写准则

https://prometheus.io/docs/instrumenting/writing_exporters/ 详细的介绍了一些准则，总结如下：

- 做到开箱即用(默认配置就可以直接开始用)

  - 高度可配置化：即可以配置只获取自己需要的指标而避免浪费性能

- 推荐使用 `YAML `作为配置格式

- 指标使用下划线命名

- 为指标提供 `HELP String` (指标上的` # HELP `注释, 事实上这点大部分 exporter 都没做好)

- 为 Exporter 本身的运行状态提供指标

  - 首先, 所有的 Prometheus 抓取目标都有一个 `up` 指标用来表明这个抓取目标能否被成功抓取. 因此, 假如 exporter 挂掉或无法正常工作了, 我们是可以从相应的 up 指标立刻知道并报警的.
  - 但是up是通过指标接口返回200来粗粒度确定的，如果用一个export同时监听多个模块，其中某些模块指标无法返回这时候就需要降低粒度，每类指标都设置一个`up`

- 可以提供一个落地页

  - 什么是落地页：用过 node_exporter 的会知道, 当访问它的主页, 也就是根路径 / 时, 它会返回一个简单的页面, 这就是 exporter 的落地页(Landing Page).
  - 落地页什么都可以放, 我认为最有价值的是放文档和帮助信息(或者放对应的链接). 而文档中最有价值的莫过于对于每个指标项的说明, 没有人理解的指标没有任何价值.

- Label的设置应该保证唯一性和可读性：

  - 唯一性：key应该是区分度很高的键，这样Value才很难重叠，避免修改Value创建新的时间序列而导致统计问题

    > 比方说某台 ECS 的名字变了, 那么在 Prometheus 内部就会重新记录一个时间序列, 造成额外的开销和部分 PromQL 计算的问题, 比如下面的示意图:
    >
    > ```shell
    > 序列A {id="foo", name="旧名字"} ..................	
    > 序列B {id="foo", name="新名字"}                   .................
    > ```

  - 可读性： 唯一性的例外，当有些标签必须表达某些信息时不需要考虑其区分性，比如IP这样的标签，我们希望可以通过ID去查询到IP，不然只有ID没办法排查，所以要增加可读性降低唯一性.

    但这样的问题有更好的解决方法：info指标(Info Metric). 单独暴露一个指标, 用 label 来记录实例的”额外信息”, 比如:

    > ```shell
    > ecs_info{id="foo", name="DIO", os="linux", region="hangzhou", cpu="4", memory="16GB", ip="188.188.188.188"} 1
    > ```

    这类指标的值习惯上永远为 1, 它们并记录实际的监控值, 仅仅记录 ecs 的一些额外信息. 而在使用的时候, 我们就可以通过 `PromQL` 的 `“Join”(group_left) `语法将这些信息加入到最后的查询结果中:

    > ```shell
    > # 这条 PromQL 将 aliyun_meta_rds_info 中记录的描述和状态从添加到了 aliyun_acs_rds_dashboard_MemoryUsage 中	
    > aliyun_acs_rds_dashboard_MemoryUsage 	
    >     * on (instanceId) group_left(DBInstanceDescription,DBInstanceStatus) 	
    >     aliyun_meta_rds_info
    > ```

    **一般的编写思路：Label记录唯一性的值 + Info指标记录详细可读性值**

## 3. Export实例

### 1. 一个简单的go Export

https://prometheus.io/docs/guides/go-application/

prometheus提供了官方的go客户端开发包，我们可以直接使用，首先下载依赖

```shell
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promauto
go get github.com/prometheus/client_golang/prometheus/promhttp
```

注册自定义的go指标的示例：

添加一个`myapp_processed_ops_total `计数器，用于计算迄今为止已处理的操作数。每 2 秒，计数器加一

```go
package main
import (
        "net/http"
        "github.com/prometheus/client_golang/prometheus/promhttp"
        "github.com/prometheus/client_golang/prometheus"
        "github.com/prometheus/client_golang/prometheus/promauto"
        "time"
)

func recordMetrics() {
        go func() {
                for {
                        // Counter两秒递增一次
                        opsProcessed.Inc()
                        time.Sleep(2 * time.Second)
                }

        }()

}

var (
        // 创建一个Counter类型的指标
        opsProcessed = promauto.NewCounter(prometheus.CounterOpts{
                Name: "myapp_processed_opt_total",
                Help: "The total number of processed events",
        })

)

func main() {
        recordMetrics()
        http.Handle("/metrics", promhttp.Handler())
        http.ListenAndServe(":2112", nil)
}
```

在另一个终端输入`curl http://localhost:2112/metrics`查看指标中查看我们自定义的指标

```shell
# HELP myapp_processed_opt_total The total number of processed events
# TYPE myapp_processed_opt_total counter
myapp_processed_opt_total 21
```

当然我们可以在本地的prometheus中配置这个数据抓取指标

```yaml
scrape_configs:
- job_name: myapp
  scrape_interval: 10s
  static_configs:
  - targets:
    - localhost:2112
```

### 2. 统计CPU温度与磁盘失败次数案例

> 源自：https://blog.csdn.net/u014029783/article/details/80001251

* 创建指标

  ```go
  var (
  	// gauge类型
  	cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
  		Name: "cpu_temperature_celsius",
  		Help: "Current temperature of the CPU",
  	})
  	// counter类型
  	hdFailures = prometheus.NewCounterVec(
  		prometheus.CounterOpts{
  			Name: "hd_errors_total",
  			Help: "Number of hard-disk errors.",
  		}, []string{"divice"},
  	)
  )
  ```

* 注册指标

  ```go
  func init() {
  	// 注册指标
  	// Metrics have to be registered to be exposed:
  	prometheus.MustRegister(cpuTemp)
  	prometheus.MustRegister(hdFailures)
  }
  ```

  使用`prometheus.MustRegister`是将数据直接注册到`Default Registry`，就像上面的运行的例子一样，这个`Default Registry`不需要额外的任何代码就可以将指标传递出去。注册后既可以在程序层面上去使用该指标了，这里我们使用之前定义的指标提供的API（`Set`和`With().Inc`）去改变指标的数据内容

* 设置指标值

  ```go
  func main() {
  	// 直接设置温度
  	cpuTemp.Set(65.3)
  	// 对应标签的数值+1,with返回的是一个counter实例
  	hdFailures.With(prometheus.Labels{"device":"/dev/sda"}).Inc()
  	// The Handler function provides a default handler to expose metrics
  	// via an HTTP server. "/metrics" is the usual endpoint for that.
  	http.Handle("/metrics", promhttp.Handler())
  	log.Fatal(http.ListenAndServe(":8080", nil))
  }
  ```

  这里只是模拟的直接写入，实际情况应该是不断从一个系统中获取这对应的指标，这样监控才是有意义的。其中With函数是传递到之前定义的`label=”device”`上的值。

### 3.自定义一个采集器Collector实例

> 源自：https://blog.csdn.net/u014029783/article/details/80001251

所有的指标类型都是实现了`Collector`采集器这个接口：

```go
type Collector interface {
    // 用于传递所有可能的指标的定义描述符
    // 可以在程序运行期间添加新的描述，收集新的指标信息
    // 重复的描述符将被忽略。两个不同的Collector不要设置相同的描述符
    Describe(chan<- *Desc)

    // Prometheus的注册器调用Collect执行实际的抓取参数的工作，
    // 并将收集的数据传递到Channel中返回
    // 收集的指标信息来自于Describe中传递，可以并发的执行抓取工作，但是必须要保证线程的安全。
    Collect(chan<- Metric)
}
```

了解了接口的实现后，我们就可以写自己的实现了，先定义结构体，这是一个**集群的指标采集器**，每个集群都有自己的`Zone`,代表集群的名称。另外两个是保存的采集的指标。

```go
type ClusterManager struct {
	Zone string
	OOMCountDesc *prometheus.Desc		// OOM错误计数
	RAMUUsageDesc *prometheus.Desc		// RAM使用指标信息
}
```

实现我们的采集工作，此处为模拟：

```go
func (c *ClusterManager) ReallyExpensiveAssessmentOfTheSystemState() (
	oomCountByHost map[string]int, ramUsageByHost map[string]float64,  //返回两个参数，map键是集群中的主机名
) {
	// 模拟两个采集数据
	oomCountByHost = map[string]int{
		"foo.example.org": int(rand.Int31n(1000)),
		"bar.example.org": int(rand.Int31n(1000)),
	}
	ramUsageByHost = map[string]float64{
		"foo.example.org": rand.Float64() * 100,
		"bar.example.org": rand.Float64() * 100,
	}
	return
}
```

下面实现自定义集群采集器的`Describe`接口方法，传递指标描述符到channel

```go
func (c *ClusterManager) Describe(ch chan <- *prometheus.Desc) {
	ch <- c.OOMCountDesc		// 传入两个Describe
	ch <- c.RAMUUsageDesc
}
```

再实现`Collect`数据采集方法：抓取数据然后转格式，最后返回到`chan`中, 并且传递的同时绑定原先的指标描述符。指标的类型（一个Counter和一个Guage）

```go
func (c *ClusterManager) Collect(ch chan<- prometheus.Metric) {
	oomCountByHost, ramUsageByHost := c.ReallyExpensiveAssessmentOfTheSystemState()
	for host, oomCount := range oomCountByHost {
		ch <- prometheus.MustNewConstMetric(  	// MustNewConstMetric 返回一个固定值指标
			c.OOMCountDesc,
			prometheus.CounterValue,	// 表明指标的类型Counter
			float64(oomCount),
			host,						// 设置主机名的host标签的值
		)
	}
	for host, ramUsage := range ramUsageByHost {
		ch <- prometheus.MustNewConstMetric(
			c.RAMUsageDesc,
			prometheus.GaugeValue,		// 表明指标的类型GauageValue
			ramUsage,
			host,
		)
	}
}
```

最后实现创建这个自定义采集器类的方法

```go
func NewClusterManager(zone string) *ClusterManager {
	return &ClusterManager{
		Zone: zone,
		OOMCountDesc: prometheus.NewDesc(
			"clustermanager_oom_crashes_total",		// 指标的名称
			"Number of OOM crashes.",					// Help
			[]string{"host"},								// 变化的标签
			prometheus.Labels{"zone": zone},				// 固定值的标签
		),
		RAMUsageDesc: prometheus.NewDesc(
			"clustermanager_ram_usage_bytes",		// 指标的名称
			"RAM usage as reported to the cluster manager.",	// Help
			[]string{"host"},								// 变化的标签
			prometheus.Labels{"zone": zone},				// 固定值的标签
		),
	}
}
```

注册集群收集器类：

```go
func main() {
   managerDB := NewClusterManager("db")
   managerCA := NewClusterManager("ca")
   // Since we are dealing with custom Collector implementations, it might
   // be a good idea to try it out with a pedantic registry.
   // 注册自定义的集群收集器
   reg := prometheus.NewPedanticRegistry()
   reg.MustRegister(managerDB)
   reg.MustRegister(managerCA)
}
```

如果直接执行上面函数，不会获取任何的参数，因为程序将自动推出，我们并未定义`http`接口去暴露数据出来，因此数据在执行的时候还需要定义一个`httphandler`来处理`http`请求。完整的

```go
func main() {
	managerDB := NewClusterManager("db")
	managerCA := NewClusterManager("ca")
	// Since we are dealing with custom Collector implementations, it might
	// be a good idea to try it out with a pedantic registry.
	// 注册自定义的集群收集器
	reg := prometheus.NewPedanticRegistry()
	reg.MustRegister(managerDB)
	reg.MustRegister(managerCA)

	gatherers := prometheus.Gatherers{
		prometheus.DefaultGatherer,
		reg,
	}

	h := promhttp.HandlerFor(gatherers,
		promhttp.HandlerOpts{
			ErrorHandling: promhttp.ContinueOnError,
		})
	http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		h.ServeHTTP(w, r)
	})
	fmt.Println("Start server at :8080")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		fmt.Errorf("error occur when start server %v", err)
		os.Exit(1)
	}
}
```

**其中`prometheus.Gatherers`用来定义一个采集数据的收集器集合，可以`merge`多个不同的采集数据到一个结果集合**，**这里我们传递了缺省的`DefaultGatherer`，所以他在输出中也会包含go运行时指标信息**。同时包含reg是我们之前生成的一个注册对象，用来自定义采集数据。

`promhttp.HandlerFor()`函数传递之前的`Gatherers`对象，并返回一个`httpHandler`对象，这个`httpHandler`对象可以调用其自身的`ServeHTTP`函数来接手`http`请求，并返回响应。其中**`promhttp.HandlerOpts`定义了采集过程中如果发生错误时，继续采集其他的数据。**

promhttp.HandlerFor()函数传递之前的Gatherers对象，并返回一个httpHandler对象，这个httpHandler对象可以调用其自身的ServHTTP函数来接手http请求，并返回响应。其中promhttp.HandlerOpts定义了采集过程中如果发生错误时，继续采集其他的数据。

启动后测试获取指标`curl localhost:8080/metrics`:

```shell
# HELP clustermanager_oom_crashes_total Number of OOM crashes.
# TYPE clustermanager_oom_crashes_total counter
clustermanager_oom_crashes_total{host="bar.example.org",zone="ca"} 318
clustermanager_oom_crashes_total{host="bar.example.org",zone="db"} 887
clustermanager_oom_crashes_total{host="foo.example.org",zone="ca"} 81
clustermanager_oom_crashes_total{host="foo.example.org",zone="db"} 81
# HELP clustermanager_ram_usage_bytes RAM usage as reported to the cluster manager.
# TYPE clustermanager_ram_usage_bytes gauge
clustermanager_ram_usage_bytes{host="bar.example.org",zone="ca"} 15.651925473279125
clustermanager_ram_usage_bytes{host="bar.example.org",zone="db"} 43.771418718698015
clustermanager_ram_usage_bytes{host="foo.example.org",zone="ca"} 6.563701921747622
clustermanager_ram_usage_bytes{host="foo.example.org",zone="db"} 66.45600532184905
```

