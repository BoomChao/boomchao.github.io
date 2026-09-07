---
date : '2026-09-07T20:00:00+08:00'
draft : false
title : '为什么 kubectl edit 改不动资源的 status'
tags : ["kubernetes", "CRD", "kubectl"]
categories: ["kubernetes"]
---

怎么修改资源的 status，今天在 k8s 发现一个问题，就是我集群有个 CRD 是这样的

```yaml
status:
  podStatuses:
    hostIP: 10.54.80.109
    nodeName: 10.54.80.109
    podIP: 10.54.96.128
    state: Running
  status: Suspended
  stopTime: "2026-09-03T07:18:59Z"
```

我想修改这个 status 下面的字段的 stopTime，来触发这个资源的垃圾回收，发现怎么修改都不生效，我使用的是 edit 操作

后面发现这个 CRD 其实定义了下面的这个 subresources 字段

```yaml
storage: true
subresources:
  status: {}
```

K8s 对启用 status 子资源的 CRD 有一条硬性规则

* 主资源端点（`PUT /apis/.../note/<name>`）的 update，apiserver 会静默丢弃 status 字段的改动——用请求前 etcd 里的旧 status 原样回填（prepareForUpdate 里直接 newObj.Status = oldObj.Status）结果就是不报错，但改了等于没改

* status 子端点（`.../note/<name>/status`）才能写 status

明白了上面这一点，现在改就好修改了；可以直接用 edit 加上 --subresource 参数就能编辑了

```yaml
--subresource='':
       If specified, edit will operate on the subresource of the requested object. Must be one of [status]. This flag is beta and may change in the future.
```
