---
date : '2026-08-24T19:00:00+08:00'
draft : false
title : '怎么判断你的指标是否健康'
tags : ["VictoriaMetrics", "云原生", "可观测性"]
categories: ["可观测"]
---

# 从一次 Top 10 高基数排查说起

今天和大家讨论一个很有意思的问题，就是不管你用 VictoriaMetrics 还是 Prometheus，怎么判断你的指标是否健康呢？

可能大家脱口而出：直接看高基数就行。但只看高基数真的就可以吗？我今天就发现一个例子：一个测试集群指标估计打得很高，然后让 AI 给我扫一下 Top 10 的高基数指标。

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vm/health/images/vm-health-top10-cardinality.png)

当时就很困惑：这些应该都是 node-exporter 的采集器指标，照理来说配置不变的话，每个集群的采集量级都是一样的。为什么新增一个集群后，原来 VmStorage 的配置就扛不住了？而且这 Top 10 的指标理论上我也没办法减少，比如上面的 `node_cpu_seconds_total`，需要通过这个指标看 Pod 的利用率，这个少不了的。

# 基数膨胀的两种形态

其实真正影响指标存储、也就是基数膨胀的形态，有两种：一种是**纵向膨胀**，一种是**横向膨胀**。

|            | 纵向膨胀                                   | 横向膨胀                     |
| ---------- | -------------------------------------- | ------------------------ |
| 形态         | 少数 metric \* 巨量的 label 组合                | 巨量的 metric 名 \* 少量的 series |
| 元凶         | Label 组合太多，塞了 trace\_id、request\_id 等   | 采集器把动态信息编码进指标名           |
| Top 10 能否发现 | 一眼就能发现                                 | 无法发现                     |
| 发现方式       | 看 Top 10 的高基数                          | 看 metric 名总数             |

# 指标体检的四个方向

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vm/health/images/vm-health-diagram.png)

## 看活跃时序数

先看活跃时序数（active time series），因为它决定了 vmstorage 的内存。

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vm/health/images/vm-health-active-series.png)

## 看 metric 名总数

```bash
curl https://test.com/vm/select/0/prometheus/api/v1/label/name/values | jq '.data|length'
```

理论上 metric 名的总数不会过万，否则就可能存在随机生成的指标名。

## 看 total vs active 的比值

```bash
curl https://test.com/vm/select/0/prometheus/api/v1/status/tsdb
```

比如这次返回的结果如下：

```json
{
  "status": "success",
  "isPartial": false,
  "data": {
    "totalSeries": 266120454,
    "totalLabelValuePairs": 3343916706,
    "seriesCountByMetricName": [
      {
        "name": "node_cpu_seconds_total",
        "value": 7734719
      },
      {
        "name": "node_cpu_scaling_governor",
        "value": 5686309
      },
      {
        "name": "node_ib_counters",
        "value": 3946914
      },
      {
        "name": "cilium_node_connectivity_latency_seconds",
        "value": 2137454
      },
      {
        "name": "node_cpu_guest_seconds_total",
        "value": 1935394
      },
      {
        "name": "cilium_node_connectivity_status",
        "value": 1068722
      },
      {
        "name": "container_network_transmit_bytes_total",
        "value": 1017607
      },
      {
        "name": "ray_tasks",
        "value": 702725
      },
      {
        "name": "container_network_transmit_packets_dropped_total",
        "value": 622790
      },
      {
        "name": "container_network_receive_bytes_total",
        "value": 567563
      }
    ],
    "seriesCountByLabelName": [
      {
        "name": "cluster",
        "value": 266193882
      },
      {
        "name": "category",
        "value": 266193656
      },
      {
        "name": "job",
        "value": 266143875
      },
      {
        "name": "__name__",
        "value": 266120454
      },
      {
        "name": "instance",
        "value": 266101072
      },
      {
        "name": "prometheus",
        "value": 266028899
      },
      {
        "name": "namespace",
        "value": 258841792
      },
      {
        "name": "pod",
        "value": 257621308
      },
      {
        "name": "container",
        "value": 244661601
      },
      {
        "name": "endpoint",
        "value": 237018420
      }
    ],
    "seriesCountByFocusLabelValue": [],
    "seriesCountByLabelValuePair": [
      {
        "name": "prometheus=monitor/vm",
        "value": 266028899
      },
      {
        "name": "namespace=monitor",
        "value": 222422535
      },
      {
        "name": "endpoint=metrics",
        "value": 219018632
      },
      {
        "name": "container=node-exporter",
        "value": 214212231
      },
      {
        "name": "service=vm-prometheus-node-exporter",
        "value": 213836553
      },
      {
        "name": "job=node-exporter",
        "value": 213836547
      }
    ],
    "labelValueCountByLabelName": [
      {
        "name": "__name__",
        "value": 26045
      },
      {
        "name": "address",
        "value": 25018
      },
      {
        "name": "instance",
        "value": 21676
      },
      {
        "name": "pod",
        "value": 19171
      },
      {
        "name": "device",
        "value": 11971
      },
      {
        "name": "resource",
        "value": 11852
      },
      {
        "name": "mountpoint",
        "value": 11678
      },
      {
        "name": "interface",
        "value": 11063
      },
      {
        "name": "port",
        "value": 11028
      }
    ]
  }
}
```

这里就能看到哪些指标的 series 是最多的。

## 看流失率（churn rate）

```promql
sum(rate(vm_new_timeseries_created_total{}[1m]))
```

如果这个曲线一直在上升，就说明有大量指标在新增写入；正常情况下曲线是持平的，而且数据量很小，因为理论上不会有那么多新增的指标写入，除非旧的 series 被清理掉。

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vm/health/images/vm-health-churn-rate.png)
