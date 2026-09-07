---
date : '2026-09-07T20:00:00+08:00'
draft : false
title : '为什么 kubectl edit 改不动资源的 status'
tags : ["kubernetes", "CRD", "kubectl"]
categories: ["kubernetes"]
---

# 从一个改不生效的问题说起

今天在 K8s 上发现一个问题，我集群里有个 CRD 是这样的：

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

我想修改这个 `status` 下面的 `stopTime` 字段，来触发这个资源的垃圾回收。结果发现**怎么修改都不生效**——我用的是 `kubectl edit`，改完保存退出，再 get 回来一看，status 还是老样子，而且整个过程不报任何错误。

# 原因：status 子资源

后面发现这个 CRD 其实定义了 `subresources` 字段：

```yaml
storage: true
subresources:
  status: {}
```

K8s 对启用了 status 子资源的 CRD 有一条硬性规则：

- **主资源端点**（`PUT /apis/.../note/<name>`）的 update，apiserver 会**静默丢弃 status 字段的改动**——用请求前 etcd 里的旧 status 原样回填（`prepareForUpdate` 里直接 `newObj.Status = oldObj.Status`）。结果就是不报错，但改了等于没改。
- **status 子端点**（`.../note/<name>/status`）才能写 status。

所以 `kubectl edit` 默认走的是主资源端点，我改的 `stopTime` 被 apiserver 悄悄丢弃了，这就是"改了不生效也不报错"的根因。

# 解决办法

明白了上面这一点，改起来就简单了：直接给 `edit` 加上 `--subresource` 参数，让它走 status 子端点：

```bash
kubectl edit <资源类型> <资源名> --subresource=status
```

这个参数在 kubectl help 里的说明如下：

```
--subresource='':
      If specified, edit will operate on the subresource of the requested object. Must be one of [status]. This flag is beta and may change in the future.
```

加上之后再去编辑，`stopTime` 就能真正写进去了。

# 总结

- CRD 一旦启用 `subresources.status`，主资源端点会静默丢弃 status 的改动，改了不报错也不生效；
- 想改 status 必须走 status 子端点：`kubectl edit <type> <name> --subresource=status`；
- 同理，`kubectl apply` / `kubectl patch` 想动 status，也得加 `--subresource=status`。
