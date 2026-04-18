---
date : '2026-04-18T21:00:10+08:00'
draft : false
title : 'envoy-gateway 解析'
tags : ["gateway"]
categories: ["网关"]
---

# 快速上手

## CRD 安装

安装方式参考官网 https://gateway.envoyproxy.io/docs/tasks/quickstart/

```bash
➜  ~ k get svc -n envoy-gateway-system
NAME                                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                            AGE
backend                                  ClusterIP      10.104.46.205    <none>        3000/TCP                                           96s
envoy-envoy-gateway-system-eg-5391c79d   LoadBalancer   10.100.30.231    <pending>     80:32138/TCP                                       96s
envoy-gateway                            ClusterIP      10.107.140.234   <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP,9443/TCP   3m11s
```

为了方便测试，建议所有资源都安装到同一个ns下面即可；上面的 LoadBalancer 没有分配 IP 没有关系，检查对应的的endpoint都创建出来即可

```bash
➜  ~ k get ep -n envoy-gateway-system
NAME                                     ENDPOINTS                                                                   AGE
backend                                  192.168.90.229:3000                                                         2m38s
envoy-envoy-gateway-system-eg-5391c79d   192.168.90.206:10080                                                        2m38s
envoy-gateway                            192.168.90.228:18000,192.168.90.228:18001,192.168.90.228:9443 + 2 more...   4m13s
```

## 测试应用

测试应用是否能访问，访问上面的 svc/envoy-envoy-gateway-system-eg-5391c79d 的 ClusterIP:port，有下面的结果输出即可

```json
➜  ~ curl -H "Host: www.example.com" http://10.100.30.231:80/get
{
 "path": "/get",
 "host": "www.example.com",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "User-Agent": [
   "curl/7.81.0"
  ],
  "X-Envoy-External-Address": [
   "10.39.35.142"
  ],
  "X-Forwarded-For": [
   "10.39.35.142"
  ],
  "X-Forwarded-Proto": [
   "http"
  ],
  "X-Request-Id": [
   "78fc52f9-224f-46c7-8172-dfddb239bd4a"
  ]
 },
 "namespace": "envoy-gateway-system",
 "ingress": "",
 "service": "",
 "pod": "backend-55d64d8794-fz2vj"
}
```



# 权限设置

envoy-Gateway 控制面利用的 ServiceAccount 是 envoy-gateway，默认是在 envoy-gateway-system 这个 ns 下面

```bash
➜  ~ k get sa envoy-gateway -n envoy-gateway-system
NAME            SECRETS   AGE
envoy-gateway   0         24m
```

绑定了一个 ClusterRole 和 两个 Role

```bash
# 一个 ClusterRole
NAME                                 CREATED AT
eg-gateway-helm-envoy-gateway-role   2025-11-17T12:05:31Z

# 两个 Role
NAME                                   CREATED AT
eg-gateway-helm-infra-manager          2025-11-17T12:05:31Z
eg-gateway-helm-leader-election-role   2025-11-17T12:05:31Z
```



# 架构解析

## 访问架构

整个访问的架构如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image.png)

## 资源架构

资源架构解析如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-1.png)

## CRD

这里要区分两种资源

* k8s Gateway API Resources

  * GatewayClass：定义 Gateway 的一些通用配置

  * Gateway：定义哪些流量需要网关路由

  * Routes: HTTPRoute, GRPCRoute, TLSRoute, TCPRoute, UDPRoute：定义不同的路由规则

* Envoy Gateway API Resources

  * EnvoyProxy：管理数据面 EnvoyProxy 的应用和配置

  * EnvoyPatchPolicy, ClientTrafficPolicy, SecurityPolicy, BackendTrafficPolicy, EnvoyExtensionPolicy, BackendTLSPolicy: Envoy Gateway 特有的其他策略和配置

  * Backend：资源简化了到集群外部后端的路由，并允许通过 Unix 域套接字访问外部进程



## 数控分离

Envoy Gateway 是一个系统，由两部分组成

* 数据面：处理实际的网络流量

* 控制面：管理配置和数据面组件

对应到创建出来的 Deployment

```plain&#x20;text
➜  ~ k get deploy
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
backend                                  1/1     1            1           137d
envoy-envoy-gateway-system-eg-5391c79d   1/1     1            1           137d
envoy-gateway                            1/1     1            1           137d
```

envoy-gateway ：控制面，负责监听 Gateway API 资源、生成配置、下发给数据面

envoy-envoy-gateway-system-eg-5391c79d ：数据面（Envoy Proxy 实例），实际处理流量转发



查看 HTTPRoute 配置

```yaml
➜  ~ k get httproute backend -oyaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
...
spec:
  hostnames:
  - www.example.com
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: eg
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: backend
      port: 3000
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /

➜  ~ k get svc backend
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
backend   ClusterIP   10.104.46.205   <none>        3000/TCP   137d
```

可看到 hostname 为 www.example.com 的话会转发给后端的 backend 的 service



查看控制面的配置文件，是存储到一个 configmap 里面

```yaml
➜  ~ k get cm envoy-gateway-config  -oyaml
apiVersion: v1
data:
  envoy-gateway.yaml: |
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: EnvoyGateway
    extensionApis: {}
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    logging:             # 日志级别
      level:
        default: info
    provider:            # 这里指定provider的类型
      kubernetes:
        rateLimitDeployment:  # 指定限流的服务镜像
          container:
            image: docker.io/envoyproxy/ratelimit:99d85510
          patch:
            type: StrategicMerge        # 用策略合并方式 patch Deployment
            value:
              spec:
                template:
                  spec:
                    containers:
                    - imagePullPolicy: IfNotPresent
                      name: envoy-ratelimit
        shutdownManager:
          image: docker.io/envoyproxy/gateway:v1.6.0
      type: Kubernetes
kind: ConfigMap
```

注意这个里面的 EnvoyGateway 不是一个 CRD，而是控制面的配置文件，可以当做是控制面的启动配置

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-2.png)

我们查看 envoyProxy 的所有CRD配置的话，也是没有这个 EnvoyGateway 的 CRD 的



## EnvoyProxy

EnvoyProxy 主题要是用来连接 Gateway 和 GatewayClass 这两种资源的

使用 EnvoyProxy 可以允许用户自定义管理 Deployment 和 Service，比如设置pod的规格，注解以及镜像地址等

> 推荐先安装 EnvoyProxy，再安装 Gateway 和 GatewayClass，避免服务中断

创建如下一个 EnvoyProxy 的配置文件

```yaml
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        replicas: 2
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: custom-proxy-config
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
    namespace: default
```

之后将 Gateway 和 GatewayClass 的 spec 里面都加上 **parametersRef&#x20;**&#x7684;配置

再来看数据面的副本数已经变成了我们设置的两个副本

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-3.png)



## 限流

为什么需要限流？

1. 预防恶意的攻击，比如 DDos 攻击？

2. 避免系统过载对资源（比如数据库）造成压大太大

3. 根据用户权限创建 API 的限制

Envoy Gateway 支持两种类型的限流：

Local rate limiting：对**单个 envoy-proxy 实例**设置的限流

Global rate limiting：对**所有 envoy-proxy 实例**设置的限流，详细[参考文档](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting)

Local rate limiting 支持细粒度的限流，比如按照header，path 甚至请求的 HTTP-Method 来进行限流

**BackendTrafficPolicy 就是用来进行限流的一些配置**，比如下面这个就是按照 header 来进行限流

```yaml
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy 
metadata:
  name: policy-httproute
  namespace: envoy-gateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: backend
  rateLimit:
    local:
      rules:
      - clientSelectors:
        - headers:
          - name: x-user-id
            value: one
        limit:
          requests: 3
          unit: Hour
```

尝试访问多次就能得到如下报错，代表达到限流阈值

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-4.png)



## JWT认证

如果想在 envoy-gateway 里面集成 jwt 认证，所需步骤如下

1. 生成 RSA 密钥对

2. 将公钥抓换成 JWKS 格式

3. 私钥签发 JWT

4. 配置到 EnvoyGateway 里面

5. 测试访问

### JWKS 解析

JWKS 全称是 **JSON Web Key Sets**

主要包括下面几个字段

kty：全称 Key Type，代表密钥算法类型，如 RAS

use：全称 Public Key Use，代表用途，如 sig（签名）、enc（加密）

kid：全称 Key ID，密钥唯一标识

alg：全称 Algorithm，具体签名算法，如 RS256、ES256

n：全称 Modulus，RSA 公钥的模数（Base64url编码）

e：全称 Exponent，RAS 公钥的指数（Base64url编码）

key\_ops：全称 Key Operations，允许的操作列表，比 use 更细粒度，如 \["verify"]、\["sign"]



### **生成密钥对**

```bash
  # 生成私钥
  openssl genrsa -out private-key.pem 2048

  # 从私钥导出公钥
  openssl rsa -in private-key.pem -pubout -out public-key.pem
```

### **生成 JWKS 文件以及 JWT**

```go
func main() {
    // 1. 读取私钥
    privPEM, err := os.ReadFile("private-key.pem")
    if err != nil {
        log.Fatal(err)
    }
    block, _ := pem.Decode(privPEM)
    privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
    if err != nil {
        // 如果是 PKCS8 格式
        key, err2 := x509.ParsePKCS8PrivateKey(block.Bytes)
        if err2 != nil {
            log.Fatalf("parse private key failed: %v / %v", err, err2)
        }
        privateKey = key.(*rsa.PrivateKey)
    }

    // 2. 生成 JWKS（公钥）
    jwk := jose.JSONWebKey{
        Key:       &privateKey.PublicKey,
        KeyID:     "my-key-1",
        Algorithm: string(jose.RS256),
        Use:       "sig",
    }
    jwks := jose.JSONWebKeySet{Keys: []jose.JSONWebKey{jwk}}
    data, _ := json.MarshalIndent(jwks, "", "  ")
    fmt.Println("=== JWKS ===")
    fmt.Println(string(data))

    // 3. 签发 JWT
    signer, err := jose.NewSigner(
        jose.SigningKey{Algorithm: jose.RS256, Key: privateKey},
        (&jose.SignerOptions{}).WithHeader("kid", "my-key-1"),
    )
    if err != nil {
        log.Fatal(err)
    }

    exp := time.Now().AddDate(10, 0, 0).Unix()
    payload := fmt.Sprintf(`{"iss":"my-issuer","sub":"user1","exp":%d}`, exp)
    jws, err := signer.Sign([]byte(payload))
    if err != nil {
        log.Fatal(err)
    }

    token, _ := jws.CompactSerialize()
    fmt.Println("\n=== JWT Token ===")
    fmt.Println(token)
}
```

### **配置 SecurityPolicy**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-jwks
  namespace: envoy-gateway-system
data:
  jwks.json: |
    {
      "keys": [
        {
          "use": "sig",
          "kty": "RSA",
          "kid": "my-key-1",
          "alg": "RS256",
          "n": "uGiavFFe7wfkecNernkljptJIqARMeJkpwtNpESqYfvhjwf5r7iFkGx-n3Y_GU7G6Y-3Tw8PDx2bzZSfKM7gjbGD9cgvHvEfC0eUzg3AUsr2GG3hBmwxOcYIF6HKGn18m9tMNZh6Dh_7pWtdQLNhhoQqPnorrA_pfAOlivIcBH7t7zBQDz0Ol9cAMtNpT1zva8fVxCzFDKWMDLG4wyE8iGNVKsfKKNTSwejgP8U8AwDCYKjH--zPUC7HhB7W-YoojxdO3UBA5wLXZOgA8IN_ddKs28J6T9jR_YStkFQQG1QINm3A6AR90L7n4po2hxIfNrZWCuEGMfcdzMZr51Huuw",
          "e": "AQAB"
        }
      ]
    }
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: jwt-auth
  namespace: envoy-gateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: backend
  jwt:
    providers:
    - name: my-provider
      issuer: "my-issuer"
      localJWKS:
        type: ValueRef
        valueRef:
          kind: ConfigMap
          name: my-jwks
```

注意这里 SecurityPolicy 应用 Configmap 时不需要指定 key

这是因为 Envoy Gateway 对 JWKS 类型的 ConfigMap 有约定：它会自动去找 ConfigMap 中 key 为 `jwks` 的字段

### **测试访问**

直接访问报错

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-5.png)

加入 JWT 的 Token访问

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-6.png)





# 压力测试

其实非常好奇 envoy 的性能如何，查看官网的数据

Envoy-Gateway 的性能如下, [参考](https://gateway.envoyproxy.io/tools/benchmark-report-explorer/)

但是 Envoy-Proxy 的性能并未得到明确的信息, [参考](https://www.envoyproxy.io/docs/envoy/latest/faq/performance/how_fast_is_envoy)

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-7.png)

于是想实际压测一下到底这个组件能承受多少 QPS

## VM环境搭建

部署 victor-a-metrics-k8s-stack: https://docs.victoriametrics.com/helm/victoria-metrics-k8s-stack/

这里 vmsingle 运行不成功是因为 PVC 没有挂载到对应的 PV 里面，需要手动创建 PV

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-8.png)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vm-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: 
  hostPath:
    path: /data/vm
  volumeMode: Filesystem
```

进入对应的 Grafana 看板

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-9.png)

其实已经内置好了很多看板供我们使用，但是这些都是集群自带的一些看板



## 部署 Envoy-Gateway 的看板

这里需要注意，Envoy 的控制面和数据面的看板官方还没有更新到 Grafana 的 dashboard 的网站上

所以这里按照官网的提示去这里下载

https://github.com/envoyproxy/gateway/tree/main/charts/gateway-addons-helm/dashboards

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-10.png)

直接复制导入到上一步我们部署的 Grafana 看板里面即可

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-11.png)

导入后可看到一个属于控制面，一个属于数据面

## 采集配置

看板配好了，接下来配数据源以及采集参数配置

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-12.png)

我们部署的两个组件，控制面和数据面分别如上图

控制面的访问接口是 **/metrics**，数据面的访问接口是 **/stats/prometheus**

我们配置 VMPodScrape 让 VM 自动采集这些指标

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
metadata:
  name: envoy-proxy-metrics-monitor
  namespace: envoy-gateway-system
spec:
  namespaceSelector:
    matchNames:
      - envoy-gateway-system
  podMetricsEndpoints:
    - interval: 15s
      path: /stats/prometheus
      port: metrics
  selector:
    matchLabels:
      app.kubernetes.io/component: proxy
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
metadata:
  name: envoy-gateway-metrics-monitor
  namespace: envoy-gateway-system
spec:
  namespaceSelector:
    matchNames:
      - envoy-gateway-system
  podMetricsEndpoints:
    - interval: 15s
      path: /metrics
      port: metrics
  selector:
    matchLabels:
      app.kubernetes.io/instance: eg
      control-plane: envoy-gateway

```

注意上面部署的ns需要是 envoy 对应的pod所在的ns

看到如下两个监控看板均存在数据即可

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-13.png)

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-14.png)

## 压测准备

压测工具我们就采用 hey 来进行，参考 https://github.com/rakyll/hey

直接一键安装

```bash
go install github.com/rakyll/hey@latest
```

发送请求如下

```bash
curl -H "Host: www.example.com" http://10.100.30.231:80/get
```

转换成压测请求

```bash
hey -n 100 -c 100 -m GET  -H "Content-Type: application/json"    http://10.100.30.231:80/get
```

-n 表示发送的请求数量

-c 表示请求的并发度

注意上面如果直接指定端口来测试，curl 虽然能通；但是 hey 可能会报错 404

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-15.png)

所有我们这里手动修改本机的 host 文件，让其解析到指定的 IP 上

```bash
10.100.30.231  www.example.com
```

然后使用下面的命令就能测通

```bash
hey -n 100 -c 10  http://www.example.com/get/
```

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-16.png)

## 压测数据

### 10w请求，并发数 10

```bash
hey -n 100000 -c 10  http://www.example.com/get/
```

hey输出结果如下

```yaml
Summary:
  Total:        7.7556 secs
  Slowest:        0.0154 secs
  Fastest:        0.0002 secs
  Average:        0.0008 secs
  Requests/sec:        12893.8529

  Total data:        53500000 bytes
  Size/request:        535 bytes

Response time histogram:
  0.000 [1]        |
  0.002 [96387]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.003 [3142]        |■
  0.005 [316]        |
  0.006 [81]        |
  0.008 [52]        |
  0.009 [10]        |
  0.011 [6]        |
  0.012 [0]        |
  0.014 [3]        |
  0.015 [2]        |


Latency distribution:
  10% in 0.0004 secs
  25% in 0.0005 secs
  50% in 0.0007 secs
  75% in 0.0009 secs
  90% in 0.0013 secs
  95% in 0.0016 secs
  99% in 0.0025 secs

Details (average, fastest, slowest):
  DNS+dialup:        0.0000 secs, 0.0002 secs, 0.0154 secs
  DNS-lookup:        0.0000 secs, 0.0000 secs, 0.0006 secs
  req write:        0.0000 secs, 0.0000 secs, 0.0048 secs
  resp wait:        0.0007 secs, 0.0002 secs, 0.0140 secs
  resp read:        0.0000 secs, 0.0000 secs, 0.0094 secs

Status code distribution:
  [200]        100000 responses
```

此时 QPS 大概是 1.67k

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-17.png)



### 40w请求，并发数 10

```bash
hey -n 400000 -c 10  http://www.example.com/get/
```

hey输出结果如下

```yaml
Summary:
  Total:        32.6999 secs
  Slowest:        0.0492 secs
  Fastest:        0.0002 secs
  Average:        0.0008 secs
  Requests/sec:        12232.4674

  Total data:        214000000 bytes
  Size/request:        535 bytes

Response time histogram:
  0.000 [1]        |
  0.005 [399453]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.010 [493]        |
  0.015 [39]        |
  0.020 [0]        |
  0.025 [1]        |
  0.030 [1]        |
  0.034 [1]        |
  0.039 [4]        |
  0.044 [4]        |
  0.049 [3]        |


Latency distribution:
  10% in 0.0004 secs
  25% in 0.0005 secs
  50% in 0.0007 secs
  75% in 0.0010 secs
  90% in 0.0013 secs
  95% in 0.0017 secs
  99% in 0.0026 secs

Details (average, fastest, slowest):
  DNS+dialup:        0.0000 secs, 0.0002 secs, 0.0492 secs
  DNS-lookup:        0.0000 secs, 0.0000 secs, 0.0011 secs
  req write:        0.0000 secs, 0.0000 secs, 0.0113 secs
  resp wait:        0.0008 secs, 0.0002 secs, 0.0491 secs
  resp read:        0.0000 secs, 0.0000 secs, 0.0072 secs

Status code distribution:
  [200]        400000 responses
```

此时 QPS 大概为 6k

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-18.png)




### 80w请求，并发数 100

```bash
hey -n 800000 -c 100  http://www.example.com/get/
```

hey输出如下

```yaml
Summary:
  Total:        44.8930 secs
  Slowest:        0.1391 secs
  Fastest:        0.0002 secs
  Average:        0.0056 secs
  Requests/sec:        17820.1622

  Total data:        428000000 bytes
  Size/request:        535 bytes

Response time histogram:
  0.000 [1]        |
  0.014 [765920]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.028 [32939]        |■■
  0.042 [987]        |
  0.056 [33]        |
  0.070 [23]        |
  0.084 [0]        |
  0.097 [0]        |
  0.111 [12]        |
  0.125 [40]        |
  0.139 [45]        |


Latency distribution:
  10% in 0.0011 secs
  25% in 0.0024 secs
  50% in 0.0048 secs
  75% in 0.0076 secs
  90% in 0.0109 secs
  95% in 0.0135 secs
  99% in 0.0195 secs

Details (average, fastest, slowest):
  DNS+dialup:        0.0000 secs, 0.0002 secs, 0.1391 secs
  DNS-lookup:        0.0000 secs, 0.0000 secs, 0.0614 secs
  req write:        0.0000 secs, 0.0000 secs, 0.0634 secs
  resp wait:        0.0055 secs, 0.0002 secs, 0.1385 secs
  resp read:        0.0000 secs, 0.0000 secs, 0.0559 secs

Status code distribution:
  [200]        800000 responses
```



![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-19.png)

此时 QPS 能达到 13k

### 150w 请求，并发数 100

```bash
hey -n 1500000 -c 100  http://www.example.com/get/
```

查看hey的输出

```yaml
Summary:
  Total:        82.2977 secs
  Slowest:        0.0670 secs
  Fastest:        0.0002 secs
  Average:        0.0082 secs
  Requests/sec:        18226.5195

  Total data:        802500000 bytes
  Size/request:        802 bytes

Response time histogram:
  0.000 [1]        |
  0.007 [704872]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.014 [249018]        |■■■■■■■■■■■■■■
  0.020 [38246]        |■■
  0.027 [6209]        |
  0.034 [1333]        |
  0.040 [236]        |
  0.047 [73]        |
  0.054 [11]        |
  0.060 [0]        |
  0.067 [1]        |


Latency distribution:
  10% in 0.0011 secs
  25% in 0.0023 secs
  50% in 0.0046 secs
  75% in 0.0075 secs
  90% in 0.0107 secs
  95% in 0.0133 secs
  99% in 0.0193 secs

Details (average, fastest, slowest):
  DNS+dialup:        0.0000 secs, 0.0002 secs, 0.0670 secs
  DNS-lookup:        0.0000 secs, 0.0000 secs, 0.0283 secs
  req write:        0.0000 secs, 0.0000 secs, 0.0287 secs
  resp wait:        0.0081 secs, 0.0002 secs, 0.0670 secs
  resp read:        0.0001 secs, 0.0000 secs, 0.0154 secs

Status code distribution:
  [200]        1000000 responses
```

可看到输出只有 100w，说明被限流

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gateway/images/Envoy-Gateway-image-20.png)

查看 QPS，能达到 18.3k

