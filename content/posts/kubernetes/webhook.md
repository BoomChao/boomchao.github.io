---
date : '2025-09-23T11:17:13+0800'
draft : false
title : 'k8s-webhook'
tags : ["kubernetes"]
categories: ["kubernetes"]
---

# 定义

> Webhook Definition：In web development, a **webhook** is a method of augmenting or altering the behavior of a web page or web application with custom callbacks;

上面是维基百科的定义，可以明确 webhook 就是一个回调函数，只不过这个回调函数修改的是 Web 也就是 HTTP 的请求内容

k8s 里面的 webhook 基本就只有两个，一个是 [MutatingWebhook](https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/mutating-webhook-configuration-v1/) 另外一个是 [ValidatingWebhook](https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/validating-webhook-configuration-v1/)

**ValidatingWebhook** 顾名思义就是判断请求是否可准入

**MutatingWebhook** 顾名思义就是用来修改请求的参数，包括请求参数以及返回参数

MutatingAdmissionWebhook 和 ValidateAdmissionWebhook 各自作用位置如下

![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/kubernetes/webhook/webhook-api.PNG?raw=true)

具体流程如下

![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/kubernetes/webhook/hook.PNG?raw=true)

1.  先提交资源的请求（CREATE、UPDATE、PATCH等）给 API-Server
2.  通过API-Server的认证和授权
3.  经过 MutatingAdmission，经过其对应的 webhook-controller 进行处理
4.  经过资源对象校验认证
5.  经过 ValidatingAdmission，经过其对应的 webhook-controller 进行处理
6.  再写入ETCD

可见 webhook 其实是工作在 API-Server 这一层的

# 配置参数

上面的两个webhook其实是做的事情不一定，但是单独看配置，这两者几乎没有很大的区别

下面以一个具体的配置来进行解析

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "simple-kubernetes-webhook.acme.com"
webhooks:
  - name: "simple-kubernetes-webhook.acme.com"    
    namespaceSelector:                        # 这个代表webhook是对哪个ns下的资源生效
      matchLabels:
        admission-webhook: enabled
    rules:                        # 这个规则代表是具体对哪些resources生效
      - apiGroups: [""]           # 包括资源的版本，以及目标操作的类型
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "*"
    clientConfig:        # 这个表示webhook-server的配置
      service:           # 配置的server是通过service来进行访问的
        namespace: default    # service所在的ns
        name: simple-kubernetes-webhook    # service的名字
        path: /validate-pods  # 具体进行hook的路径
        port: 443             # 端口
      caBundle: |             # 鉴权用到的公钥
        LS0tLxxxxxx           
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 2        # 超时时间
```

创建好对应的配置之后，我们编写对应的代码

# 代码解析

代码仓库参考 <https://github.com/BoomChao/simple-kubernetes-webhook>

这里配置了对应的 validate 和 mutate 相关的准入操作行为

## Validate 配置

![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/kubernetes/webhook/code.png?raw=true)

这里配置如果pod的名称包含 offensive 则直接返回 error

## Mutate 配置

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/32d7bb75c4be45a69db6833832624716~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=5rTuhudEh5QAGvEKfbb3H4i0yBQ%3D)

这里配置了对应的污点的参数，任何一个pod创建出来其自身的pod参数都会带上对应的特定的污点计数

## 功能验证

部署下面的两个pod来校验上面的功能是否正常

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: offensive-pod
  namespace: apps
spec:
  containers:
    - args:
        - sleep
        - "3600"
      image: busybox
      name: lifespan-offensive
  restartPolicy: Always
```

该 pod 名称包含我们在 validate 里面的违禁词 offensive，所以此 pod 理论上无法创建出来

```bash
➜  pods git:(main) ✗ k apply -f bad-name.pod.yaml           
Error from server: error when creating "bad-name.pod.yaml": admission webhook "simple-kubernetes-webhook.acme.com" denied the request: pod name contains "offensive"
```

报错信息如上所示

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    acme.com/lifespan-requested: "3"
  name: lifespan-three
  namespace: apps
spec:
  containers:
    - args:
        - sleep
        - "3600"
      image: busybox
      name: lifespan-three
  restartPolicy: Always
```

该 pod 的标签上的 `acme.com/lifespan-requested` 的值为3，则理论上创建成功之后会增加到 14

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/250787d50df1428a837dfcd4f154a13e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=AFupHWl3NgiuO0OjZ2gIaCTa9j4%3D)

查看pod上的污点事件发现确实是会增加到14为止

# 新增资源超卖功能

## 基本原理

实际工作中我们可能经常遇到这样的场景，修改node自身的一些信息，但是这些信息又是其他组件，比如 kubelet 自动上报上来的，如果这时候 kubectl 直接修改则改动之后又会被复原回去

这时候 webhook 的作用就来了，我们可以让请求到达我们的 hook-server，然后hook-server劫持到该请求后尝试修改再发送给api-server，前后对比简图如下

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/a3f9084b73a04e96bbab765a9a47b66f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=8Wg%2F6OwA6ixQxM19gKGC85b2fRI%3D)

1.  其他资源上报给 API-Server 的请求会被 webhook 劫持；这里劫持是按照资源的 GVR 以及对应所执行的操作（比如CREATE、UPDATE、PATCH）进行实现的
2.  Webhook 获取到请求之后，会将该请求分发给对应的 Service，Service 将该请求再分发给该Service底下具体的应用
3.  由 Webhook 应用按照业务逻辑来修改请求参数
4.  之后该请求再发送给 API-Server

所以我们实际需要编写代码的组件只需要编写下面的 deployment 即可，也就是 webhook-controller

我们来做一个实际的功能，修改 node 的 allocatable 信息来做资源的超卖

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/bbb53dc6861e478dbf45d7eaae4e7c5c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=g8sUj%2BazekqBTCNQLUmhQUcFk9o%3D)

如果能够尝试将 allocatable 的值改大，那么不久意味着这个node上可以调度上去更多的pod（当未达到pod数量的最高限制之前）

我们修改上面定义的 Mutate 的 webhook 的配置，来实现获取 kubelet 上报 node-status 信息的请求

配置的 yaml 如下，新增下面一个规则即可

```yaml
rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["UPDATE"]
    resources: ["nodes/status"]
```

> 注意：这里的 resources 填写的是 nodes/status，而不是 nodes

下面我们尝试修改 kubelet 上报的信息，来放大 node 上的可分配的配额信息，以此来实现资源超卖

## 修改步骤

具体步骤如下，修改不能只修改 Allocatable 而不修改 Capacity

1.  计算出原始的 Capacity - Allocatable 的大小，这部分空间就是 reserverd 使用

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/088028e20442480fa193bd2e2d02cdd0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=AhWI9Yuk3CKQ2NHhokKiXwzKJbY%3D)

如下面这个图所示

```bash
Capacity = Allocatable + kube-reserverd + system-reserverd + eviction-threshold
```

2.  计算超卖的量，比如在 Allocatable 的基础上增大一倍，计算出新的 Allocatable
3.  则 Capacity 的值就等于 Allocatable + reserved 的使用

总结如下

```bash
reserverd = orginalCapacity - originalAllocatable
newAllocatable = [....]
newCapacity = newAllocatable + reserverd
```

## 代码实现

代码实现如下 [代码仓库](https://github.com/BoomChao/simple-kubernetes-webhook)

```go
func (nc *nodeCap) Mutate(node *corev1.Node) (*corev1.Node, error) {
    nc.Logger.Info("node-cap mutator")

    originalCPUCapacity := node.Status.Capacity.Cpu()
    originalCPUAllocatable := node.Status.Allocatable.Cpu()

    nc.Logger.Info("Original CPU Capacity: ", originalCPUCapacity.String())
    nc.Logger.Info("Original CPU Allocatable: ", originalCPUAllocatable.String())

    reserved := originalCPUCapacity
    reserved.Sub(*originalCPUAllocatable)

    newCPUAllocatable := originalCPUAllocatable.Value() * 2
    newCPUCapacity := newCPUAllocatable + reserved.Value()

    nc.Logger.Info("New CPU Allocatable: ", newCPUAllocatable)
    nc.Logger.Info("New CPU Capacity: ", newCPUCapacity)

    node.Status.Allocatable[corev1.ResourceCPU] = *resource.NewQuantity(newCPUAllocatable, resource.DecimalSI)
    node.Status.Capacity[corev1.ResourceCPU] = *resource.NewQuantity(newCPUCapacity, resource.DecimalSI)

    return node, nil
}
```

重新构建镜像

```bash
docker build -t simple-kubernetes-webhook:latest . 
```

然后将之前的 simple-kubernetes-webhook pod 删除重建

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/d16da76629ab4cddadc9c88c60fdaa7b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=0TGg4ucTrR%2BWEqjlJGBzhNDDl5w%3D)

可看到 node 层面这里的 Allocatable 以及 Capacity 均已经被我们调整过来了

我们尝试部署一个大核心的pod，限制 CPU 的 limit 和 request 都是 32core，尝试是否能够部署成功

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              cpu: "32"        # 请求 32 核心
            limits:
              cpu: "32"        # 限制为 32 核心
```

可看到也能部署成功

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/3206f80108ba4354863fbafe03cf23de~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=DT3v8VAcsCx5fsWTKH4gL%2FZE0w4%3D)

我们删除掉 webhook 的配置，重启之前已经部署好的 nginx-pod

```bash
k delete -f mutating.node.config.yaml
k delete pod nginx-deployment-74c556b6bc-vhkfv
```

查看pod重建之后的状态

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/4461b7f26b794bd7b49f8a4cfc2fd9a3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=DWjZatmSxU8gu7qUGKjtXclhXXo%3D)

可发现如果不经超卖，则无法部署上去我们的大核心的pod

# Q\&A

## 怎么查看CVG?

参考这篇[文章](https://stackoverflow.com/questions/49396607/where-can-i-get-a-list-of-kubernetes-api-resources-and-subresources)

k8s暴露出的API一共只有如下几个，我们可以执行 k proxy 然后访问就能看到所有的 api 路径

```bash
➜  ~ curl 127.0.0.1:8001
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/crd.projectcalico.org",
    "/apis/crd.projectcalico.org/v1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1",
    "/apis/events.k8s.io/v1beta1",
    "/apis/flowcontrol.apiserver.k8s.io",
    "/apis/flowcontrol.apiserver.k8s.io/v1beta1",
    "/apis/flowcontrol.apiserver.k8s.io/v1beta2",
    "/apis/metrics.k8s.io",
    "/apis/metrics.k8s.io/v1beta1",
    "/apis/monitoring.coreos.com",
    "/apis/monitoring.coreos.com/v1",
    "/apis/monitoring.coreos.com/v1alpha1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/stable.example.com",
    "/apis/stable.example.com/v1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/aggregator-reload-proxy-client-cert",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/priority-and-fairness-config-consumer",
    "/healthz/poststarthook/priority-and-fairness-config-producer",
    "/healthz/poststarthook/priority-and-fairness-filter",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/aggregator-reload-proxy-client-cert",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/priority-and-fairness-config-consumer",
    "/livez/poststarthook/priority-and-fairness-config-producer",
    "/livez/poststarthook/priority-and-fairness-filter",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/openid/v1/jwks",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/informer-sync",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/aggregator-reload-proxy-client-cert",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/priority-and-fairness-config-consumer",
    "/readyz/poststarthook/priority-and-fairness-config-producer",
    "/readyz/poststarthook/priority-and-fairness-filter",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}%
```

而 node 的资源路径是在 `/api/v1` 路径下

执行 `k get --raw='/api/v1'` 就能看到很多信息

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/092374c2b41c49598e355ba4fc72817f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5LiN5pWP5oSf55qE5bCP5pyd5ZCM5a2m:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNjE4MzU2ODEyODEwMjY5In0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1759202127&x-orig-sign=r9CGVdITZ6IYpO1zCYyAYpVoYwY%3D)

过滤出 node 相关的信息

```yml
  - name: nodes
    singularName: ""
    namespaced: false
    kind: Node
    verbs:
      - create
      - delete
      - deletecollection
      - get
      - list
      - patch
      - update
      - watch
    shortNames:
      - no
    storageVersionHash: XwShjMxG9Fs=
  - name: nodes/proxy
    singularName: ""
    namespaced: false
    kind: NodeProxyOptions
    verbs:
      - create
      - delete
      - get
      - patch
      - update
  - name: nodes/status
    singularName: ""
    namespaced: false
    kind: Node
    verbs:
      - get
      - patch
      - update
```

可看到一共有三种资源，nodes，nodes/proxy，以及 nodes/status 这几种资源

# 参考资料

k8s关于webhook的介绍 [MutatingWebhookConfiguration](https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/mutating-webhook-configuration-v1/) [ValidatingWebhookConfiguration](https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/validating-webhook-configuration-v1/)

node 的 Allocatable 和 Capacity 之间的关系

<https://hwchiu.medium.com/introduction-to-kubernetes-resources-capacity-and-allocatable-4dc1bfbd1caf>

<https://mycloudjourney.medium.com/what-is-mutatingwebhook-in-kubernetes-a62f79598ecb>

