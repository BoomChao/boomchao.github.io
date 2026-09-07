---
date : '2026-09-07T20:00:00+08:00'
draft : false
title : 'vlog初探'
tags : ["VictoriaLogs", "云原生", "可观测性"]
categories: ["可观测"]
---

# 整体架构

vlog 简称为 VictoiraLogs，是 VM 公司推出的一款日志收集工具，性能极高

需要注意的是 vlog 支持很多的写入源，本质是一个集中化的日志存储和查询以及写入组件；整体架构如下图

> 声明一点：vlog 没有倒排索引的概念，采用的是stream➕列式存储来加速查询

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-10.png)

整体总结就一句话：每个文件都有一个对下个文件的指向，所以能快速从海量的日志存储中迅速锁定到目标所有的那几个字节部分



# 数据写入

## 写入方

vlog的上游支持多个写入方，参考[官方文档](https://docs.victoriametrics.com/victorialogs/data-ingestion/?_gl=1*1ylx3xg*_gcl_au*MTEwMDUzNTk1MS4xNzg1NTczMjUz*_ga*MTE0ODc1MDU0LjE3NTk5OTMxNjI.*_ga_N9SVT8S3HK*czE3ODg0MDIxNzQkbzMkZzEkdDE3ODg0MDQyMzAkajMwJGwwJGgw)的枚举列表

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-7.png)

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-9.png)

使用 vector 的话，官方支持两种协议，分别是 HTTP-Json 和 Elasticsearch 的协议



## 存储

数据接收到了之后，比如集群内使用 vector 收集，收集完成之后写入到 vlog

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-8.png)

vlog会有一个 buffer 接受这些写来的日志数据，buffer 是多 shard 的，分片数就等于 CPU 的 Core 数，比如上面图中的 3 个SHARD，那么 CPU 理论上就是 3 core

Buffer 里面的数据每秒（或者 buffer 快写满了）就会写到内存里面的 PART 里面去

vlog 的存储的最小单位就是 part，看官方对 part 的定义如下

> a part is the self-contained, searchable bundle of logs that a buffer turns into when it flushes.

part 就是 vlog 存储的最小的可查询日志单位，而实际上 part 在 vlog 里面也分为三种，分别是

**In-memory parts：**&#x8FD9;是最新创建的日志，还在内存里面没有落盘到磁盘

**Small parts：**&#x8FD9;是为了持久化已经从内存落盘到磁盘的小文件

**Big parts：**&#x8FD9;是由上面的 small parts 聚合成的一整个大文件

这三者关系如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-6.png)

关键指标

> `vl_insert_flush_duration_seconds`: how long it takes to turn a buffered batch into an in-memory part.

`vl_insert_flush_duration_seconds` 就是用来看将缓存批次转到内存分区的耗时

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image.png)

也就是开源的这个指标，一般看的是 P99 的耗时



## 查询性能

Vlog 之所以查询性能如此优秀，主要来源其数据结构和列式存储

### 数据结构

数据落盘后，实际的存储的最小单位是一个块（也称为block），一个block会包括某个特定的stream的多行日志；这些各种的 block 的排列规则是先按照 stream，然后再按照时间戳来排列

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-1.png)

比如下面这些 block 的示意图；单个block的未压缩的数据量上限大概是 2MB 作用

每个block里面有一个东西叫 block header, 这个东西是记录 block 的元信息，比如

* 这个 block 属于哪个 stream

* 有多少行日志在这个block

* 以及这个block的最小和最大的日志时间戳范围

注意 block header 是旁路存储在 index.bin 这个文件里面的，这就使得vlog查询的时候可以直接扫描这个 index.bin 来确定哪些block需要读取，不需要触碰中间跳过的block的任何日志；极大的加速了查询速度；其实也类似于索引的一种机制

### 列式存储

列式存储是另一个使得vlog查询性能很高的特点

在每个 block 的内部，都使用列式存储来存储具体的 filed 字段

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-2.png)

列式存储存储的优点就是只扫描对应的列，不需要每个行的其他的冗余的列也参与查询扫描

列式存储的另外一个核心优势就是压缩，因为日志的场景某些filed的值都是比较固定的

* 比如 level 字段总是下面几种：infor，error 等

* status 字段总是数字

* timestamp 时间戳也是一个数字

所以 vlog 的每一列的数据相同的太多就可以执行压缩；甚至 msg 里面的内容也可以压缩，因为同一数据流中的日志看起来非常相似，因此同一列中的相邻值会重复很多次，可以进一步压缩数据量（参考 zstd 压缩算法：对两个相似的语句，可以压缩完全相同的部分来减少数据量）

Vlog 按列的这种相似性，可以将 TB 级别的原始日志压缩到 GB 级别





# 核心特性

vlog几个核心的优势特点就是：列式存储、流式聚簇



## 三大基础概念

Message（\_msg）：每条日志必须有 \_msg 字段（人类可读的事件描述），Web UI  默认展示它，查询时可省略字段名

Time（\_time）：未指定时间戳时用摄取时间

Stream（\_stream）：逻辑上的“桶”，同一流的日志一起写盘并压缩成块。查询时通过读块头即可跳过无关块，大幅提速





## **流式聚簇**

vlog 的存储单位是 stream = 一组相同字段值（labels）的日志集合，如 `{namespace="project-aidata", pod="spark-xxx-master-0"}` 的所有日志归为一条流；同一条流的数据在物理磁盘上连续存放；依次来利用局部性原理提升查询效率



## 结构化和非结构化日志

结构化日志和非结构化日志区别，这里直接以一个例子为例，我们看一个简单的代码

```go
package main

import (
    "os"

    "github.com/rs/zerolog"
    "k8s.io/klog"
)

const (
    RequestID = "1234567890"
    Host      = "192.168.1.1"
    Path      = "/api/v1/users"
)

func main() {
    klog.Infof("request forwarded requestID=%s host=%s path=%s", RequestID, Host, Path)

    logger := zerolog.New(os.Stdout).With().Timestamp().CallerWithSkipFrameCount(4).Logger()
    logger.Info().Str("request_id", RequestID).Str("host", Host).Str("path", Path).Msg("request forwarded")
}
```

结果输入如下

```bash
I0901 11:59:40.127980 1512794 main.go:17] request forwarded requestID=1234567890 host=192.168.1.1 path=/api/v1/users
{"level":"info","request_id":"1234567890","host":"192.168.1.1","path":"/api/v1/users","time":"2026-09-01T11:59:40Z","caller":"/usr/local/go/src/runtime/asm_amd64.s:1771","message":"request forwarded"}
```

第一个你会发现答应的就是简单的行式文本，第二个就是结构化的文本，是以 json 格式显示的

可能很多人会觉得日志打印能看就行，但是确实是这样，但是如果搜索在很长时间范围内，非结构化的文本的劣势立马就体现出来了，说清楚这个问题得先说明日志的存储原理

以 vlog 来说，如果你的日志是结构化的json文本，那存储在 vlog 里面大致是这样的

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-3.png)

\_msg 对应的 value 是一个 json 字段（json 天生就是k-v这种形式），而vlog的分词规则如下

参考文档这里非常关键的[一段话](https://docs.victoriametrics.com/victorialogs/logsql/#key-concepts)

> A token is considered a word if it contains only <u>UTF-8</u> -encoded Unicode letters, digits, and underscores.

理解就是 如果一个词元仅包含UTF-8编码的Unicode字母、数字和下划线，则该词元被视为一个单词；比如下面这个例子，字符串 `foo_bar (baz-123,"тест45")!` 会被切割为 foo\_bar, baz, 123, tect45

所以字词分割的标准就是遇到非字符，数字和下划线以外的所有字符都会触发子词的分割

所以这就告诉我们打日志的时候尽量使用 \_ 下划线，而不是 - 中划线，因为中划线会导致字词分割，从而影响查询效率



vlog 侧的自动索引，前提是这个字段真的作为独立 key 存在于摄入的文档里；而现在的问题是：\_msg 是一整块文本，比如上面截图的那个内容，你会发现 \_msg 对应的 value 就是一个巨大的 json 字段，这个 json 里面的 key 都不是独立的 key 存在于摄入的文档里面





## 日志高基数

正确理解日志的高基数，日志的高基数和指标的高基数完全不太一样，理解这个点需要先理解vlog里面的字段的区分

vlog 的字段分两位，处理方式完全不同

* Stream field（\_stream 里的字段）比如下面这种

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-4.png)

\_stream 的作用类似 "物理分区/索引前缀"：所有的 stream filed 相同的日志都会被归类到同一个 log stream，物理上是连续存储的；所以同一个 stream 内的日志应该是同一个实体产生的，比如上面的 同一个 ns 的同一个 pod 的同一个容器

* 普通字段（\_msg里的字段），比如下面这种

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/vlog/images/vlog-image-5.png)

这种普通字段不参与分 stream，但 vlog 会自动给它建立全文索引，支持按这种方式 trace\_id:="xxx" 这种方式高效检索；这层索引确实是 "值越唯一则定位就越精准"

那思考为什么 trace\_id 或者 request\_id 放到 stream 就是灾难，放到普通字段却没事呢?

首先要明白 stream 是日志存储元数据的一种组织方式，这种方式利用了局部性原理来快速检索日志；如果把 trace\_id 放到了 stream 里面，那么每一条日志几乎都有不同的 trace\_id，则系统认为来了一套 stream 元数据，新开一组 chunk 文件，结果就导致了 stream 数量保障增长，从而日志摄入变量，导致 CPU/内存/磁盘 IO 全面飙升

如果把 trace\_id 放到 \_msg 里面，那么就只会被 vlog 的全文索引收录，只建索引，查询会非常快

所以在日志层面高基数本身不是问题，而是把基数字段当成流的分组维度才是问题，所以总结如下

* 不会变(container/host/pod) 则适合做 stream field

* 每条都变(trace\_id/user\_id/ip) 则只能做普通字段，绝不能塞进 stream







# 参考文档

[How VictoriaLogs Stores Your Logs in a Columnar Layout](https://victoriametrics.com/blog/victorialogs-internals-columnar-storage-on-disk/?_gl=1*1ivugk3*_gcl_au*MTEwMDUzNTk1MS4xNzg1NTczMjUz*_ga*MTE0ODc1MDU0LjE3NTk5OTMxNjI.*_ga_N9SVT8S3HK*czE3ODc4ODU3MDYkbzExJGcxJHQxNzg3OTE0MzMxJGo2MCRsMCRoMA..)
