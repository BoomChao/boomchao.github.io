---
date : '2026-10-20T19:15:00+08:00'
draft : false
title : '一起 Envoy 链接数倾斜问题'
tags : ["envoy", "云原生"]
categories: ["网关"]
---

今天线上发现一个问题，就是突然链接数上去了，又明显的倾斜现象，一看监控如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/envoy/images/Envoy-image-5.png)

看这个曲线的时间线，10:00 之前都好好的，你会发现突然 10:00 之后链接数出现大量的倾斜，这是明显的热点问题，请求链接全部都达到了一个实例上面

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/envoy/images/Envoy-image-4.png)

看 CPU 使用率，明显和上面的链接数的曲线趋势非常匹配

当时初步断定链接数是因，负载不均衡造成的 CPU利用率高这个果，然后尝试重启 envoy-proxy 的四个实例，发现重启还是一样的现象，基本可以断定这个链接数和 CPU 利用率高都是果，而不是因

因为看到 POD 的 CPU 利用率很高，但是整机并没有高负载，于是才想可能就是处理不过来，然后继续排查，查看envoy-proxy 的并发的数量

```bash
max by (cluster, namespace, pod) (envoy_server_concurrency{})
```

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/envoy/images/Envoy-image-2.png)

发现居然内部的 workers 数有这么多，而我这个机器恰好就是 128 核的，和这个数目完全对的上，于是猜想可能就是因为请求本身就增多的情况，而workers数太多在抢 POD 层面的8core的容量（envoy-proxy我们部署的时候只配置了 8Core）

于是找到这个 concurrency 的官方定义[如下](https://www.envoyproxy.io/docs/envoy/v1.33.2/operations/cli#cmdoption-concurrency)

> **--concurrency \<integer>**
>
> *(optional)* The number of [worker threads](https://www.envoyproxy.io/docs/envoy/v1.33.2/intro/arch_overview/intro/threading_model.html#arch-overview-threading) to run. If not specified defaults to the number of hardware threads on the machine. If set to zero, Envoy will still run one worker thread.

看到这个解释就能明朗了，这个 workers threads 现在就等于我这个容器跑的整机的 CPU 的 Cores 数，也就是容器没有限制住，根因是我们没有显示设定这个参数

上面有一个概念是叫 worker threads，这就引入了 envoy 的线程模型概念，envoy 线程模型特点如下

* Envoy 是单进程多线程架构

* 一个 Main Thread 负责 xDS 配置更新，统计刷新，管理接口等控制面工作，不直接处理请求流量

* 多个 Worker Thread 负责监听、过滤和转发流量，这个数量是由 --concurrency 来控制

* 另外还有 File Flush Thread 异步来写 access log，避免阻塞流量处理

另外需要注意的是，如果一个 worker thread 分配到了连接，则这个连接的整个生命周期都由这个 worker 来处理

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/envoy/images/Envoy-image-1.png)

官方的建议，worker threads 数量应该等于核数，单机部署就等于机器核数，容器部署就等于 Pod 的核数

加上这个缺省的配置参数之后

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/envoy/images/Envoy-image-3.png)

再次查看监控，立马恢复

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/envoy/images/Envoy-image.png)

立马所有的 envoy-proxy 的 pod 的连接数就均衡了



Q：其实改完之后问题确实是解决了，但是还是有点没太明白，理论上envoy-proxy是单进程多线程模型的话，一个进程只占一个core，我指定worker threads数为容器的 CPU 的cores总数怎么就解决了呢？不应该调不调 envoy-proxy 都只会占一个core嘛？

A：理解这个需要注意：Linux 调度的基本单位是线程，不是进程；一个Envoy 进程可能包含 8 个活跃 worker 线程 ，这是可以同时跑满 8 个 Core

原来的情况是：

```bash
Envoy：128 个 worker
Pod：8 Core 的 CPU quota
```

任意时刻最多只能获得大约 8 Core 的计算能力。128 个 worker 都有任务时，内核只能轮流运行它们：

```plaintext
worker 1～8 运行
→ 被抢占
→ worker 9～16 运行
→ 不断切换
→ Pod 用完当期 CPU quota 后，所有线程被暂停
```

这就会导致：

* 大量线程上下文切换；

* CPU cache不断失效；

* 128 个独立 event loop 的维护开销；

* CPU quota 更快耗尽，触发 CFS throttling；

* 连接绑定在 worker 上，worker 被暂停时，该连接也只能等待

而改成 `concurrency=8` 后，相当于：8 个流量 worker 均分到 8 Core

每个 worker 可以比较连续地在一个 Core 上执行，不需要 128个 worker反复争抢8个运行位置，cache locality 也更好，CPU quota 不再浪费在线程切换和调度开销上

需要注意，`concurrency=8` 不是限制 Envoy 只能处理8个请求。每个 worker 是非阻塞事件循环，一个 worker 可以同时管理成千上万个连接和请求：

```bash
8 worker ≠ 8 并发请求
8 worker = 8 个并行事件循环
```

因此减少 worker 没有明显降低网络并发能力，反而提升了单位 CPU 的处理效率
