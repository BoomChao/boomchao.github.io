---
date : '2026-08-17T21:00:10+08:00'
draft : false
title : '初识 VictoriaMetrics'
tags : ["VictoriaMetrics", "云原生", "可观测性"]
categories: ["可观测"]
---

# 基本概念

来玩指标之前，我们需要弄清楚时序数据的几个基本概念

Q：什么是 series？又什么是 samples？

A：看下面这个图，一目了然；形象解释了一条时序数据的基本组成

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-11.png)

> 这里需要注意，`requests_total{path="/", code="200"}` and `requests_total{path="/", code="403"}` 这是两个不同的指标，因为 code 的 label 不一样

cardinality（也称为基数）就是指的 series 的数量，high cardinality（高基数）就是指的这个 unique time series 的数量非常多

那衡量一个时序数据库系统好不好就得看上面的 TIME SERIES 和 SAMPLE

对于 series，关键的指标如下 ：Active time series（[活跃时序数](https://docs.victoriametrics.com/victoriametrics/faq/#what-is-an-active-time-series)）、High churn rate（[高流失率](https://docs.victoriametrics.com/victoriametrics/faq/#what-is-high-churn-rate)）

分别对应的 VM 的指标如下

```yaml
活跃时序数: sum(max_over_time(vm_cache_entries{type="storage/hour_metric_ids"}[1h])) 
流失率: sum(rate(vm_new_timeseries_created_total{}[1m]))
```

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-13.png)

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-14.png)

对于 sample，其实就是 (Timestamp Value) 这种形式，对应抓取的指标如下

```yaml
sum(rate(vm_promscrape_scraped_samples_sum{}[1m]))
```

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-12.png)

大概每秒抓取 79w 个 sample



# vmagent

**问题描述**

今天遇到一个问题，线上用户配置了一个 vmpodscrape 的配置如下

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
.....
spec:
  namespaceSelector:
    matchNames:
    - project-ai
    - project-coding
  podMetricsEndpoints:
  - interval: 15s
    path: /metrics
    relabelConfigs:
    - sourceLabels:
      - __meta_kubernetes_pod_job_id
      targetLabel: job_id
    - replacement: engine
      targetLabel: metric_component
    - action: replace
      regex: (.*)
      replacement: $1:8000
      sourceLabels:
      - __meta_kubernetes_pod_ip
      targetLabel: __address__
  ......
  - interval: 15s
    - action: replace
      regex: (.*)
      replacement: $1:8001 ~ $1:8008        # 伪代码，表秒监听 8001～8008这九个端口
      sourceLabels:
      - __meta_kubernetes_pod_ip
      targetLabel: __address__
    - action: replace
      replacement: "8001"
      targetLabel: vllm_metric_port
  selector:
    matchLabels:
      user-label.active: true
```

其实粗看这个配置，发现好像也没什么问题，就是监听两个 ns 下的带有固定 label 的 pod 的端口，然后采集 metric

但是这种配置上到生产环境简直是指标的灾难，问题如下

vmpodscrape 是针对 pod 的采集配置，没有任何的健康检查机制，也不校验目标端口是否真的有数据，这就导致我只要集群有 pod 打上这个标签，**vmagnet 就一定会去这个pod上的 8000～8008 端口轮询去查数据，无论这些端口有没有数据**

比如我现在一个训练任务起了 200 个pod，但是没有一个 pod 的端口暴露指标了，这就意味着我的 target 一共有 200\*8 = 1600 个 target，那每一次采集会触发 1600 次 connection timeoue，这简直是灾难



实际部署上线后，vmagent 疯狂打错误日志，都是抓取失败超时的场景

查看 down 掉的 target 指标，整个约 5k 左右的数量，而且是瞬间增加

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image.png)

查看实际的超时指标，最高的超时数量甚至达到了 14k，这个对 vmagent 简直是灾难

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-1.png)

实际生产过程中发现扩容 vmagent 的数量都不好使，因为这种情况本质上是并发打满了，默认的并发是 2 \* 当前的 CPU 核数，几个关键参数的作用如下图

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-2.png)

上面这个图是针对 push 指标的场景，其实从这个图就可以看出，push 的写入方不应该是 vminsert，应该是过 vmagent 来 push，因为在这里可以做流控的策略

maxConcurrency 叫做并发数：取值是当前的核数 \* 2

maxQueueDuration 是队列等待时间：并发数慢了处理不过来就先入队列，最长等待时间默认是 1 min

maxInsertRequestSize 是最大的写入体积（对push这种场景）：超过这个体积抓取直接丢弃，默认是 32MB

maxScrapeSize 是最大的抓取体积（对应pull这种场景）：超过这个体积抓取直接丢弃，这个值默认是 16Mb

介绍几个有用的指标如下

```yaml
vm_promscrape_scrape_requests_total        # 抓取请求总数
vm_promscrape_max_scrape_size_exceeded_errors_total  # 超过抓取最大限制的请求总数
vm_promscrape_scrapes_timed_out_total      # 超时的抓取请求总数
vm_promscrape_scrapes_failed_total         # 失败的抓取请求总数
```

注意上面的失败的抓取总数其实里面包含了超时的抓取数，是子集的关系



理论上都不应该用 vmpodscrape 来抓指标，理由如下

vmpodscrape 不确认 pod 的状态，只要是 pod 在集群存在就会进行抓取，这就会导致如果pod是处理 initalizing 状态或者 failed 以及 completed 的状态，这种情况明显会触发超时，拖慢 vmagent 的效率

那怎么避免呢？推荐使用 vmservicescrape&#x20;

vmservicescrape  用的是 k8s 原生的 sd\_discovery 机制，我们查看原始 chart 包里面的定义

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-3.png)

其实你会发现这里有三种类型，endpoints 和 endpointslices 就不说了，endpoints 已经过时

那下面的 service 怎么理解呢，service 主要是探活 / 黑盒探测——你不关心是哪个 Pod 应答,只想知道"这个 Service 整体通不通"；比如配合 blackbox-exporter 做 HTTP 探测,或者某些对外表现为单一逻辑实例的组件

采集指标首选的是 endpointslices，这个通过一个环境变量把这个开关打开启用即可

```yaml
victoria-metrics-operator:
  enabled: true
  env:
  - name: VM_VMSERVICESCRAPEDEFAULT_ENFORCEENDPOINTSLICES
    value: true
```

这就意味着，我如果使用的是 vmservicescrape，那其实抓取的是 endpointslices，这个对应的是每个后端pod的真实 IP，由于service的特殊机制，一般 service 下面的 pod都会配置 ReadinessProbe，那不健康的 pod 自动就会从 endpointslices 里面剔除掉了，自然 vmagent 也就不会去抓取指标了，很大程度上减少了超时现象的发生



如果非要使用 vmpodscrape 的方式，一定要加上 POD 的 ready 状态的检测，也就是 relabel 加上下面的这个配置 source label 的配置，

```yaml
relabelConfigs:
- action: keep
  sourceLabels:
    - __meta_kubernetes_pod_ready   # 这个 label是纯用来过滤的
  regex: "true"                     # 不会真的写到 kube_pod_labes这个指标里面去
```



# vmselect

&#x20;vmselect 的工作位置如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-4.png)

参考官网的这张图，你会发现 vmselect 其实就是上游的各个查询承接器，负责接收上游的比如 grafana、vmui、alerting 等的各种查询请求



我们先从 vmselect 的两个参数说起

```yaml
-search.maxConcurrentRequests     # 这个参数代表同时执行的查询请求数量
-search.maxQueueDuration          # 这个代表上面的同时查询数量满了之后，后续请求的排队超时时间
                                  # 默认数值是 30s
```

`maxConcurrentRequests` 这个数值如果不显示设置，则默认取值是 `min(cores*2, 16)`；注意这个 cores 取的是 limit，而不是 request 的值

注意这就说明单纯提高 vmselect 的核心数是没有用的，如果不显示设置这个值，这就从侧面反映了官方建议每个 vmselct 的核数不超过 8 core，如果设置超过 8 core，则实际的可用最大并发数还是 16

比如，我下面这个云上的集群部署的两个 pod，我把 request 和 limit 的 CPU 都设置成 9 core

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-5.png)

但是实际发现最大的可用并发数还是 32（单个最大是 16，下面曲线是求和就是 32），而不是 2\*9\*2 = 36

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-6.png)

介绍如下几个关键指标代表 vmselect 是否过载

* vm\_concurrent\_select\_current：有多少请求正在被处理

* vm\_concurrent\_select\_capacity：允许的最大并发数

* vm\_concurrent\_select\_limit\_reached\_total：达到上限的 vmselect 的数量

* vm\_concurrent\_select\_limit\_timeout\_total：排队请求有多少已经超时的数量

所以现在看这个开源的看板，就会发现红线就是最大并发数，蓝线就是当前处理的请求数

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-7.png)



那这就引入另外一个问题：既然我的并发数默认是由我的 **副本数\*2\*核数** 决定的，那岂不是我的查询 QPS 永远不会超过这个数值了？

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-8.png)

我们仔细看上面这两张图

* 左边是 QPS：代表每秒处理的请求数；这里实际是代表的入站请求速率（incoming request rate），客户端发过来的速率完全可以 > 16/s，超出部分会进队列（这时候是由 maxQueueDuration 控制最长排队时间）

* 右边是并发数（maxConcurrentRequests）：代表同一时刻正在处理的请求

一定要明确一个概念：并发数 ≠ QPS，这两个是不同维度的度量；QPS = 并发数 / 平均响应时间

所以你会发现区别本质还是在于时间和时刻的理解：同样的请求数，这一时刻在处理，就是最大并发数；这一时间段在处理，那总数/时间段就是 QPS





高可用

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-9.png)

正常来说 vminsert 都会进行双写来保证高可用，比如图中所示，那这样每个数据实际都会存储到两个 vmstorage 里面去，这样给查询带来一个问题，我 vmselect 查询岂不是会有重复数据？

vmselect 是这样解决的，首先查询肯定是从所有的 vmstorage 里面查到，查到之后会按照时间戳进行去重的逻辑，有一个默认的参数是 `dedup.minScrapeInterval=1ms`，这个参数代表如果多个数据点具有相同的时间戳（或者时间戳间隔在 1ms以内）时，vmselect 只保留最新的数据点，以此来进行去重



内存限制：`-memory.allowedPercent` 其实是一个通用的参数(vminsert、vmstorage 都有)，主要作用是限制 vmselect 的占用的内存上限（注意是软限制，也有可能会超过这个限制）

其实很好奇为什么容器里面部署了有cgroup 的 limit 限制还需要这个东西呢？理由如下

1. vm 不只是只有容器版，还有单机版，没有这个限制单机版部署的内容容量怎么设置呢

2. 即使是容器部署，如果依靠 cgroup 的 limit 机制，需要注意的是 k8s 的 OOMKill 是灾难性的，整个 vmselect 的实例全部挂掉，所有查询都会失败，OOM 之后还需要 重新拉起➕预热；而 VM 自己设置这个限制就能够保证在 OOMKill 之前及时踩住刹车，从而进行内存释放，比如清理 cache、kill 单个查询等这些来降级保护大部分查询依然能够成功



# vmstorage

下面来解决 vmstorage 这边的几个常见的问题

## 常见问题

### **TSID 究竟是什么？**

不得不又放出这张图如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-10.png)

vm 其实对每个指标都有 canonical metric name，这个就是规范的度量名称；这个名称的组成就是metric name 和已经排过序的 labels 的组合，比如下面这两个指标

```plain&#x20;text
metric_name{instance="host",job="app"}
metric_name{job="app",instance="host"}
```

其实只是 label 的顺序不一样，但是在 vmstorage 里面的规范度量名称（也就是 canonical metric name）是一样的

其实也可以注意到，TSID 就是每个 TIME SERIES 都是唯一一个

我们查看源码的 TSID 的结构

```go
// TSID is unique id for a time series.
//
// Time series blocks are sorted by TSID.
//
// All the fields except MetricID are optional. They exist solely for better
// grouping of related metrics.
// It is OK if their meaning differ from their naming.
type TSID struct {
    // MetricGroupID is the id of metric group.
    //
    // MetricGroupID must be unique.
    //
    // Metric group contains metrics with the identical name like
    // 'memory_usage', 'http_requests', but with different
    // labels. For instance, the following metrics belong
    // to a metric group 'memory_usage':
    //
    //   memory_usage{datacenter="foo1", job="bar1", instance="baz1:1234"}
    //   memory_usage{datacenter="foo1", job="bar1", instance="baz2:1234"}
    //   memory_usage{datacenter="foo1", job="bar2", instance="baz1:1234"}
    //   memory_usage{datacenter="foo2", job="bar1", instance="baz2:1234"}
    MetricGroupID uint64

    // JobID is the id of an individual job (aka service).
    //
    // JobID must be unique.
    //
    // Service may consist of multiple instances.
    // See https://prometheus.io/docs/concepts/jobs_instances/ for details.
    JobID uint32

    // InstanceID is the id of an instance (aka process).
    //
    // InstanceID must be unique.
    //
    // See https://prometheus.io/docs/concepts/jobs_instances/ for details.
    InstanceID uint32

    // MetricID is the unique id of the metric (time series).
    //
    // All the other TSID fields may be obtained by MetricID.
    MetricID uint64
}
```

你会发现其实一个 TSID 下面包含了四个 ID：`MetricGroupID、JobID、InstanceID、MetricID`

查看源码这几个ID 的生成逻辑

```go
func generateTSID(dst *TSID, mn *MetricName) {
    dst.MetricGroupID = xxhash.Sum64(mn.MetricGroup)
    // Assume that the job-like metric is put at mn.Tags[0], while instance-like metric is put at mn.Tags[1]
    // This assumption is true because mn.Tags must be sorted with mn.sortTags() before calling generateTSID() function.
    // This allows grouping data blocks for the same (job, instance) close to each other on disk.
    // This reduces disk seeks and disk read IO when data blocks are read from disk for the same job and/or instance.
    // For example, data blocks for time series matching `process_resident_memory_bytes{job="vmstorage"}` are physically adjacent on disk.
    if len(mn.Tags) > 0 {     # 仅第一个tag才算作 JobID
        dst.JobID = uint32(xxhash.Sum64(mn.Tags[0].Value))
    }
    if len(mn.Tags) > 1 {     # 第一个tag之后的的都算作InstanceID
        dst.InstanceID = uint32(xxhash.Sum64(mn.Tags[1].Value))
    }
    dst.MetricID = generateUniqueMetricID()
}

// Returns local unique MetricID.
func generateUniqueMetricID() uint64 {
    // It is expected that metricIDs returned from this function must be dense.
    // If they will be sparse, then this may hurt metric_ids intersection
    // performance with uint64set.Set.
    return nextUniqueMetricID.Add(1)
}

// This number mustn't go backwards on restarts, otherwise metricID
// collisions are possible. So don't change time on the server
// between VictoriaMetrics restarts.
var nextUniqueMetricID = func() *atomicutil.Uint64 {
    var n atomicutil.Uint64
    n.Store(uint64(time.Now().UnixNano()))
    return &n
}()
```

除了 MetricID 是按照纳秒的时间戳来标识，其他都是按照 hash 算法来做哈希生成  int 类型的整数

其中的三个字段（`MetricGroupID`、`JobID`、`InstanceID`）都不是用来唯一标识时间序列的，真正的唯一标识是 `MetricID`。它们存在的唯一目的是 控制数据在磁盘上的物理排列顺序，从而优化查询时的磁盘 IO；源代码如下

```go
// Less return true if t < b.
func (t *TSID) Less(b *TSID) bool {
    // Do not compare MetricIDs here as fast path for determining identical TSIDs,
    // since identical TSIDs aren't passed here in hot paths.
    if t.MetricGroupID != b.MetricGroupID {
        return t.MetricGroupID < b.MetricGroupID
    }
    if t.JobID != b.JobID {
        return t.JobID < b.JobID
    }
    if t.InstanceID != b.InstanceID {
        return t.InstanceID < b.InstanceID
    }
    return t.MetricID < b.MetricID
}
```

可以发现比较 TSID 的时候，是先比较 MetricGroupID、再比较 JobID、然后再是 InstanceID，最后才是 MetricID

我们举个例子来直观的理解 TSID 里面这三者的关系

先回顾一下 prometheus 里面的[ job 和 instance 的概念](https://prometheus.io/docs/concepts/jobs_instances/)

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-15.png)

但在 VictoriaMetrics 里并不真的只对应 `job` 和 `instance` 标签。它们实际上取的是 tag 排序后的第 1 个和第 2 个 tag 的值；假设有这些时间序列：

```sql
http_requests_total{job="api", instance="10.0.0.1:8080", method="GET"}
http_requests_total{job="api", instance="10.0.0.1:8080", method="POST"}
http_requests_total{job="api", instance="10.0.0.2:8080", method="GET"}
http_requests_total{job="web", instance="10.0.0.3:8080", method="GET"}
```

排序后 Tags\[0] 都是 job（job-like），Tags\[1] 都是 instance（instance-like）；于是：

| 系列                                | MetricGroupID                 | JobID       | InstanceID            |
| --------------------------------- | ----------------------------- | ----------- | --------------------- |
| job=api, instance=.1, method=GET  | hash("http\_requests\_total") | hash("api") | hash("10.0.0.1:8080") |
| job=api, instance=.1, method=POST | 同上                            | 同上          | 同上                    |
| job=api, instance=.2, method=GET  | 同上                            | 同上          | hash("10.0.0.2:8080") |
| job=web, instance=.3, method=GET  | 同上                            | hash("web") | hash("10.0.0.3:8080") |

前两条在磁盘上紧挨在一起，当你查 `http_requests_total{job="api", instance="10.0.0.1:8080"}` 时，只需要顺序读一小段磁盘，几乎不需要随机 seek

明白上面的 TSID 的原理，现在看下面官方的blog的这张图

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-16.png)

就能明白这个过程究竟是发生了些什么，怎么从 metric name  到 TSID.MetricID



### 什么是 IndexDB?

上面我们回答了什么是 TSID，其实你可以看到 TSID 就是将人类可读的 metric name 转化成了计算机可读的 TSID 这样一个时间戳，但是你是否好奇我现在要查询请求是怎么和这个TSID关联起来的

这就是 IndexDB负责的内容：负责把 metirc name 和 TSID 的这层映射关系维护起来，方便查询

因为 vmstorage 其实是不存储 metric name 的，数据落盘到vmstorage之后存储的形式全部如下所示

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-17.png)

只存储 TSID 和属于这个 TSID 的时间范围的 samples&#x20;

我们以一个具体的查询为例子，看看查询的流程是什么样的

1. 我现在在 Grafana 上要查询 `sum_over_time(node_cpu_seconds_total{mode="idle"}[5m])`，那请求链路基本如下

   ![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-18.png)

2. 请求先过 vmselect，vmselect 把聚合函数之外的内容丢给下游的 vmstorage 来做数据索取

   ![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/victoria-metrics/vm/images/VictoriaMetrics-image-19.png)

   Vmstorage 先利用 IndexDB 将 metric name 转化成 TSID（比如上图中的 60, 12, 32 等就是不同的 series对应的 TSID），然后再用 TSID 去查询对应的 Main Storage 里面的实际存储的 sample 数据，比如我们这边的时间范围就是当前时刻的\[5m]区间



# 参考文档

[vm官网](https://docs.victoriametrics.com/victoriametrics/cluster-victoriametrics/)

[vmselect-how-it-works](https://victoriametrics.com/blog/vmselect-how-it-works/)
