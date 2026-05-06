---
date : '2026-05-06T21:00:10+08:00'
draft : false
title : 'traefik 解析'
tags : ["gateway", "云原生"]
categories: ["网关"]
---

# 快速入门

## 安装

更新helm

```go
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

下载最新的chart包并安装

```bash
h pull traefik/traefik --untar   
h install traefik ./traefik -n traefik -f traefik/values.yaml 
```



## 查看Dashboard

查看安装的 CRD

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image.png)

查看应用

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-1.png)

其实就一个deployment，没有其他模块

修改上面部署的 helm-chart 的 values.yaml 文件

```yaml
# values.yaml
ingressRoute:
  dashboard:
    enabled: true
    matchRule: Host(`dashboard.localhost`)
    entryPoints:
      - web
providers:
  kubernetesGateway:
    enabled: true
gateway:
  listeners:
    web:
      namespacePolicy:
        from: All
```

将默认的 dashboard 暴露出来，升级完成之后查看

```yaml
➜  ~ k get ingressroute traefik-dashboard -oyaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  annotations:
    meta.helm.sh/release-name: traefik
    meta.helm.sh/release-namespace: traefik
  creationTimestamp: "2026-04-17T07:03:46Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: traefik-traefik
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: traefik
    helm.sh/chart: traefik-39.0.7
  name: traefik-dashboard
  namespace: traefik
  resourceVersion: "45525861"
  uid: bb8a0425-09d8-4ec0-9149-0b18356b3d00
spec:
  entryPoints:
  - web
  routes:
  - kind: Rule
    match: Host(`dashboard.localhost`)
    services:
    - kind: TraefikService
      name: api@internal
```

已经暴露出了一个 dashboard 的 ingressroute，访问地址 http://dashboard.localhost/dashboard/

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-2.png)

便能看到类似这样的展示信息

## 部署应用

我们创建一个应用，涉及 deployment 以及 service 和 ingressroute

```yaml
# whoami.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: traefik
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
---
# whoami-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: traefik
spec:
  ports:
    - port: 80
  selector:
    app: whoami
---
# whoami-ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
  namespace: traefik
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`whoami.localhost`)
      kind: Rule
      services:
        - name: whoami
          port: 80
```

测试访问

```bash
➜  ~ k get svc
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.105.235.195   <pending>     80:31799/TCP,443:32037/TCP   25m
whoami    ClusterIP      10.96.42.170     <none>        80/TCP                       3m
```

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-3.png)

因为测试环境没有 LoadBalancer 的 IP，我们访问 clusterip 通过指定 Host 的方式能够得到预期的输出



## 部署方式对比

Traefik 支持两种部署方式

第一种：对接 k8s 的 Gateway 资源，利用 GatewayClass ➕ Gateway ➕ HTTPRoute &#x20;

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-4.png)

第二种：利用 Traefik 自带的 IngressRoute

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-diagram.png)

区别显而言之，用 IngressRoute 会更加方便，相当于一个 IngressRoute 替换了上面 Gateway 的三个CRD

但是缺点也很明显，用了 Gateway 的话流量控制的力度会更细，比如用 IngressRoute 无法做到按照 header 来做粘性会话，而用 envoy-gateway 的 [BackendTrafficPolicy](https://gateway.envoyproxy.io/docs/concepts/gateway_api_extensions/backend-traffic-policy/) 就能实现按照header的字段来做流量控制



# 路由策略

路由策略基本就包含两部分内容，分别是会话保持和负载均衡

整体链路如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-diagram-1.png)

## 会话保持

### 基本逻辑

基本步骤如下

1. 第一次客户端访问的时候在返回体上网关侧会带上cookie:value 相关的值返回

2. 第二次客户端只需要带上上次相同的cookie值就能命中上一次同样的后端实例，以此来实现粘性会话

Traefik 目前只支持基于 cookie 的会话保持策略，只需要在指定的 ingressRoute 里面开启即可

```yaml
services:
- name: whoami
  port: 80
  sticky:
    cookie:
      name: test-cookie
      httpOnly: false
      secure: false
```

比如我现在访问

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-5.png)

则会返回一个 Set-Cookie 的选项，下次请求直接指定这个 cookie 的 header 的选项，则能实现粘性会话

```yaml
curl -H 'Host: whoami.localhost' -b 'test-cookie=57532e7592b9a5ef' http://10.105.235.195:80/
```

多次访问会发现均会返回一个固定的访问地址，以此来实现粘性会话

### 实现原理

当给 IngressRoute 设置了开启 cookie 的粘性会话功能时，其实有个很有趣的问题：比如我现在网关是多实例，我第一次请求网关实例一处理，然后返回了一个cookie的值；那要是第二次请求到了网关实例二了，网关实例二怎么知道这个请求应该分发到对应的后端哪个实例呢？

就是网关侧有没有全局视野的存储，怎么做到分散的网关实例利用同一个值命中预期的实例呢？

看到代码实现如下，每个 LoadBalancer 其实都会存一个对应的 Value

```go
func (b *DefaultLoadBalancer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    if b.StickyCookie != nil {
        cookie, err := req.Cookie(b.StickyCookie.Name)

        if err != nil && !errors.Is(err, http.ErrNoCookie) {
            log.Warn().Err(err).Msg("Error while reading cookie")
        }

        if err == nil && cookie != nil {
            b.HandlersMu.RLock()
            // 这里会判断是否handlerMap里面是否有这个值
            handler, ok := b.HandlerMap[cookie.Value]    
            b.HandlersMu.RUnlock()

            if ok && handler != nil {
                b.HandlersMu.RLock()
                _, isHealthy := b.Status[handler.Name]
                b.HandlersMu.RUnlock()
                if isHealthy {
                    handler.ServeHTTP(w, req)
                    return
                }
            }
        }
    }
    .......
    if b.StickyCookie != nil {
        cookie := &http.Cookie{
            Name:     b.StickyCookie.Name,
            Value:    Hash(server.Name),    // 这里会将 server.Name 做哈希返回
            Path:     "/",
            HttpOnly: b.StickyCookie.HttpOnly,
            Secure:   b.StickyCookie.Secure,
            SameSite: convertSameSite(b.StickyCookie.SameSite),
            MaxAge:   b.StickyCookie.MaxAge,
        }
        http.SetCookie(w, cookie)
    }

    server.ServeHTTP(w, req)
}
```

哈希的代码如下

```go
func Hash(input string) string {
    hasher := fnv.New64()
    // We purposely ignore the error because the implementation always returns nil.
    _, _ = hasher.Write([]byte(input))

    return strconv.FormatUint(hasher.Sum64(), 16)
}
```

看到这里就比较明确了，**每个后端的 server 的 Name 通过哈希的方式存入到 traefik 的map里面，那只要 server 的名字不变，我任何网关实例同样哈希出来的值肯定就是一样的**

> 另外在 k8s 集群部署的方式，这个 server.Name 其实就是每个后端service对应的 endpoint





## 负载均衡

Traefik 的负载均衡主要分为服务层面的四种负载均衡，已经提供了更为高级的service来做灰度发布以及蓝绿部署等，[参考文档](https://doc.traefik.io/traefik/reference/routing-configuration/http/load-balancing/service/#advanced-service-types)

### 服务层面负载均衡

Traefik 默认支持下面四种负载均衡策略

#### WRR

wrr 是加权轮询的策略，将请求轮询发给后端的实例，代码实现如下

```go
var handler *namedHandler
for {
    // Pick handler with closest deadline.
    handler = heap.Pop(b).(*namedHandler)

    // curDeadline should be handler's deadline so that new added entry would have a fair competition environment with the old ones.
    b.curDeadline = handler.deadline
    handler.deadline += 1 / handler.weight

    heap.Push(b, handler)
    if _, ok := b.status[handler.name]; ok {
        if _, ok := b.fenced[handler.name]; !ok {
            // do not select a fenced handler.
            break
        }
    }
}
```

注意这里实现加权轮询的逻辑，是每个实例都会塞到到一个堆里面，比如我现在有两个实例

实例 A 的权重 weight 是 3，实例 B 的权重 weight 为 1

则初始的实例 A 的 deadline 为 1/3，实例 B 的 deadline 为 1/1，因为这个最小堆，默认每次都是选 deadline 最小的那个，所以请求模拟如下

第一次请求，选中 A，之后 A.deadline = 1/3 + 1/3 = 2/3, B.deadline = 1/1  = 1

第二次请求，选中 A，之后 A.deadline = 2/3 + 1/3 = 1, B.deadline = 1/1 = 1&#x20;

第三次请求，因为 A，B deadline 的值一样，则随机选一个，无论选中 A 或者 B，则下一次一定是另外一个

所以 3:1 的权重，这四个请求，一定会有 3 个请求命中 A，1 个请求命中 B



#### p2c&#x20;

p2c负载均衡则是随机选择两个后端 server，然后挑选其中一个链接数最少的 server

逻辑如下

```go
func (b *Balancer) nextServer() (*namedHandler, error) {
    .......
    n1, n2 := b.rand.Intn(len(healthy)), b.rand.Intn(len(healthy))
    b.randMu.Unlock()

    // 如果随机发现选中的实例一样则错开
    if n2 == n1 {
        n2 = (n2 + 1) % len(healthy)
    }
    
    h1, h2 := healthy[n1], healthy[n2]
    // Ensure h1 has fewer inflight requests than h2.
    if h2.inflight.Load() < h1.inflight.Load() {
        log.Debug().Msgf("Service selected by P2C: %s", h2.name)
        return h2, nil
    }
    .....
}
```

注意这里链接数的计算方式如下

```go
func (h *namedHandler) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
    h.inflight.Add(1)
    defer h.inflight.Add(-1)

    h.Handler.ServeHTTP(rw, req)
}
```

就是使用一个原子计数来计算有多少链接还存着



#### Hrw

Hrw 全称是 Highest Random Weight，使用 IP 来做一致性哈希以此进行会话保持

主要实现函数如下

```go
func (b *Balancer) nextServer(key string) (*namedHandler, error) {
    .....
    var handler *namedHandler
    score := 0.0
    for _, h := range healthy {
        s := getNodeScore(h, key)
        if s > score {
            handler = h
            score = s
        }
    }
    ....
    return handler, nil
}

// getNodeScore calculates the score of the couple of src and handler name.
// 这里的 src 就是客户端的 IP
func getNodeScore(handler *namedHandler, src string) float64 {
    h := fnv.New64a()
    h.Write([]byte(src + handler.name))
    sum := h.Sum64()
    score := float64(sum) / math.Pow(2, 64)
    logScore := 1.0 / -math.Log(score)

    return logScore * handler.weight
}
```

这里 getNodeScore 实际上就实现了一套一致性哈希的方案

确定性：同一个 Client (IP/Key) 总是被映射到同一台后端，无需任何状态存储

均匀性：不同 client 按权重比例分布在各后

最小扰动：增/删一台后端时，只有约 1/N 比例的 key 会被重新映射，其余 client 仍命中原后端（相比"取模 hash"全量重洗，差别巨大）

这种实现不需要传统的一致性哈希一样需要构造大量的虚拟节点，HRW 更加均匀





#### Leasttime&#x20;

leasttime 选择响应时间最短且活动连接数最少的服务器路由

实现基本如下

```go
// Score = (avgResponseTime × (1 + inflightCount)) / weight.
func (b *Balancer) nextServer() (*namedHandler, error) {
    ....
    // Calculate scores and find minimum.
    minScore := math.MaxFloat64
    var candidates []*namedHandler

    for _, h := range healthy {
        avgRT := h.getAvgResponseTime()
        inflight := float64(h.inflightCount.Load())
        // 分数 = 响应延迟 * 链接数 / 权重
        score := (avgRT * (1 + inflight)) / h.weight

        if score < minScore {
            minScore = score
            candidates = []*namedHandler{h}
        } else if score == minScore {
            candidates = append(candidates, h)
        }
    }

    if len(candidates) == 1 {
        return candidates[0], nil
    }

    // Multiple servers with same score: use WRR (EDF) tie-breaking.
    selected := b.selectWRR(candidates)
    if selected == nil {
        return nil, errNoAvailableServer
    }

    return selected, nil
}

// getAvgResponseTime returns the average response time in milliseconds.
// Returns 0 if no samples have been collected yet (cold start).
func (s *namedHandler) getAvgResponseTime() float64 {
    ....
    return s.responseTimeSum / float64(s.sampleCount)
}
```

先选择每个实例的平均响应时间，然后打分；过滤出打分最低的，如果有相同的最低分的则，则再使用 WRR 轮询选择一个权重最高的





### 增强的服务类型

TraefikService 提供了比原生的 service 更为高级的实现；

注意 TraefikService 主要运行在 service 层面，而不是实例 server 层面

WRR 和 HRW 和上面的服务的 strategy 实现类似，不再赘述

#### Mirroring

mirroring 的核心用途是 复制请求到一个或多个“镜像后端”。Traefik 官方对它的定义就是：把发往某个 service 的请求镜像到其他 service

```go
func (m *Mirroring) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
    // 获取激活的Mirrors
    mirrors := m.getActiveMirrors()
    ....
    // 然后clone请求进行转发
    m.routinePool.GoCtx(func(_ context.Context) {
        for _, handler := range mirrors {
            // prepare request, update body from buffer
            r := rr.Clone(req.Context())
            ....
            handler.ServeHTTP(m.rw, r.WithContext(contextStopPropagation{ctx}))
        }
    })
}
```



#### FailOver

FialOver 用于请求失败的兜底措施，当主服务响应错误配置中定义的特定 HTTP 状态代码时，故障转移服务会将所有请求转发到备用服务

实现逻辑如下

```go
func (f *Failover) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    f.handlerStatusMu.RLock()
    handlerStatus := f.handlerStatus
    f.handlerStatusMu.RUnlock()

    if handlerStatus {
        if len(f.statusCode) == 0 {
            f.handler.ServeHTTP(w, req)

            return
        }
        ....
        f.handler.ServeHTTP(rw, rr.Clone(req.Context()))

        if !rw.needFallback {
            return
        }

        req = rr.Clone(req.Context())
    }

    ....
}
```

而兜底的 handler 设置如下

```go
// SetHandler sets the main http.Handler.
func (f *Failover) SetHandler(handler http.Handler) {
    f.handlerStatusMu.Lock()
    defer f.handlerStatusMu.Unlock()

    f.handler = handler
    f.handlerStatus = true
}
```



# 插件机制

Traefik 有两种类型的插件

* Middleware 插件 — 在请求链路中做处理（认证、限流、header 修改等）

* Provider 插件 — 提供路由配置来源（从自定义数据源发现服务和路由规则）

## Middleware

### JWT

Jwt 插件的话 Traefik 起始也有两种，一种就是自带的 JWT 的认证，[参考文档](https://doc.traefik.io/traefik/secure/secure-api-access-with-jwt/)；另外一种就是开源的插件，可以使用traefik 的 plugin 的功能加载进去使用



**内置 JWT**

Traefik 内置的 JWT 校验需要提供一个 Identity Provider（身份提供商），[参考](https://doc.traefik.io/traefik/secure/secure-api-access-with-jwt/)

建议之前使用开源的 JWT 插件，签发 JWT 的服务自己管理，校验 JWT 则直接使用开源的插件来进行



**开源 JWT**

配置使用插件，修改 traefik 的启动配置文件；这里我们 jwt 解析校验使用开源的[插件](https://github.com/agilezebra/jwt-middleware)

```yaml
# Traefik experimental features
experimental:
  .....
  plugins:
    jwt:
      moduleName: github.com/agilezebra/jwt-middleware
      version: v1.3.8
```

该插件的标注配置如下

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
    name: my-jwt-middleware
    namespace: my-namespace
spec:
    plugin:
        jwt-middleware:
            issuers:
                - https://auth.example.com       # 这里指定颁发的实体方
            require:
                aud: test.example.com            # 这里指定授信的主体
            secret: ThisIsAPresharedSecret
```

我们声明中间件来使用该 jwt 插件

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: secure-api
  namespace: traefik
spec:
  plugin:
    jwt:
      issuers:
        - my-issuer
      skipPrefetch: "true"
      secret: |
        -----BEGIN PUBLIC KEY-----
        MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuGiavFFe7wfkecNernkl
        jptJIqARMeJkpwtNpESqYfvhjwf5r7iFkGx+n3Y/GU7G6Y+3Tw8PDx2bzZSfKM7g
        jbGD9cgvHvEfC0eUzg3AUsr2GG3hBmwxOcYIF6HKGn18m9tMNZh6Dh/7pWtdQLNh
        hoQqPnorrA/pfAOlivIcBH7t7zBQDz0Ol9cAMtNpT1zva8fVxCzFDKWMDLG4wyE8
        iGNVKsfKKNTSwejgP8U8AwDCYKjH++zPUC7HhB7W+YoojxdO3UBA5wLXZOgA8IN/
        ddKs28J6T9jR/YStkFQQG1QINm3A6AR90L7n4po2hxIfNrZWCuEGMfcdzMZr51Hu
        uwIDAQAB
        -----END PUBLIC KEY-----
```

生成 jwt ，我们借助如下代码生成

```go
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
```

注意这里的公钥和私钥要匹配

测试不带 jwt 访问

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-6.png)

直接提示没有 token

带上 jwt 访问

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-7.png)

访问成功



### ForwardAuth

思考一个问题：上面我们使用 JWT 鉴权直接使用的是 Traefik 的开源的插件，如果网关现在需要内部的一些认证措施，还能用插件来解决吗？

答案肯定是不能，即使能，插件利用的是 Go 的解释器来运行，势必会对性能造成损耗的情形

那这里 forwardAuth 这个插件就派上用场了，基本流程如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-diagram-2.png)

```yaml
# Forward authentication to example.com
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: test-auth
spec:
  forwardAuth:
    address: https://example.com/auth
```

自己编写一个 auth 服务，访问指定路由的时候带上这个 test-auth 的插件则会先处理鉴权的逻辑

### RateLimit

限流是所有网关中逃都逃不过的一个重要组件，traefik 的限流是基于令牌桶实现

基本配置如下

这里注意下面这三个关键参数，**average & period & burst**

```bash
average 和 period 共同组成平均速率的表量
average = 100, period=1s 代表的是平均速率=100/1s=100req/s
burst 代表的就是通的容量
```

本地部署好 redis，使用中间件来连接 Redis 做限流，配置好限流的中间件参数如下

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ratelimit
  namespace: traefik
spec:
  rateLimit:
    average: 1
    period: 2s
    burst: 2
    redis:
      endpoints:
        - "rate-limit-redis.traefik.svc.cluster.local:6379"
      db: 0                # redis的数据库序号
      poolSize: 50         
      minIdleConns: 10
      maxActiveConns: 200
      readTimeout: 3s
      writeTimeout: 3s
      dialTimeout: 5s
```

故意将 average 和 burst 设置的低一点，尝试请求接口，多请求几次就会有报错出现

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/traefik/images/Traefik%20%E8%A7%A3%E6%9E%90-image-8.png)

**源码解析**

traefik 利用 Redis 实现分布式限流的基本方法也是使用 lua 脚本

```go
func (r *redisLimiter) Allow(ctx context.Context, source string) (*time.Duration, error) {
    ok, delay, err := r.evaluateScript(ctx, source)
    ...
}


func (r *redisLimiter) evaluateScript(ctx context.Context, key string) (bool, *time.Duration, error) {
    ....
    params := []any{
        float64(r.rate / 1000000),
        r.burst,
        r.ttl,
        time.Now().UnixMicro(),
        r.maxDelay.Microseconds(),
    }
    // 这里传递参数到下面的 Luna 脚本里面
    v, err := AllowTokenBucketScript.Run(ctx, r.client, []string{redisPrefix + key}, params...).Result()
    ....
}

var AllowTokenBucketScript = redis.NewScript(AllowTokenBucketRaw)
```

redisLimiter 暴露出一个 Allow 方法，间接调用 evaluateScript ，再调用一个令牌桶限流的脚本如下

```go
var AllowTokenBucketRaw = `
local key = KEYS[1]
local limit, burst, ttl, t, max_delay = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4]),
    tonumber(ARGV[5])

local bucket = {
    limit = limit,
    burst = burst,
    tokens = 0,
    last = 0
}

local rl_source = redis.call('hgetall', key)

if table.maxn(rl_source) == 4 then
    -- Get bucket state from redis
    bucket.last = tonumber(rl_source[2])
    bucket.tokens = tonumber(rl_source[4])
end

local last = bucket.last
if t < last then
    last = t
end

local elapsed = t - last
local delta = bucket.limit * elapsed        // 计算该补多少令牌，limit就是速率(average/period)
local tokens = bucket.tokens + delta        // 加到桶里
tokens = math.min(tokens, bucket.burst)     // 令牌数不能大于最大的数量
tokens = tokens - 1     // 每次请求来了消耗一个令牌

local wait_duration = 0
if tokens < 0 then    // 桶里令牌不足
    wait_duration = (tokens * -1) / bucket.limit    // 补回这些令牌需要多少时间
    if wait_duration > max_delay then               // 超过时间直接拒绝
        tokens = tokens + 1                         // 抵扣的令牌归还   
        tokens = math.min(tokens, burst)
    end
end

redis.call('hset', key, 'last', t, 'tokens', tokens)
redis.call('expire', key, ttl)

return {tostring(true), tostring(wait_duration),tostring(tokens)}`
```

上面基本步骤如下

1. 每个 key 用一个 Redis Hash 存两个字段：last(上次更新时间) 和 tokens(上次更新后的剩余令牌数)；key的值类似这样：`rate:traefik-ratelimit@kubernetescrd:10.39.35.142`，存储字段如下

```go
127.0.0.1:6379> HGETALL rate:traefik-ratelimit@kubernetescrd:10.39.35.142
1) "last"
2) "1778038491850036"
3) "tokens"
4) "0.8289999000000002
```

* 请求来了先判断这个 key 是否存在历史状态，长度为 4 说明两个字段都在，否则说明是首次访问

* 计算这段时间该补多少令牌，计算完成加到桶里；每次消耗一个令牌

* 如果令牌数够，直接放行

* 如果令牌数不够；算出补这些令牌需要多少时间

  1. 如果这个等待时间仍然在客户端可接受的范围内，则让客户端等待（返回一个等待时间，上游调用会等待）

  2. 如果等待时间超过 max\_delay，则视为拒绝，把刚才-1 扣减的令牌归还，再保险地用 `burst` 截断防止超容



maxDelay 的计算逻辑如下

```go
if config.Average > 0 {
    rtl = float64(config.Average*int64(time.Second)) / float64(period)
    // maxDelay does not scale well for rates below 1,
    // so we just cap it to the corresponding value, i.e. 0.5s, in order to keep the effective rate predictable.
    // One alternative would be to switch to a no-reservation mode (Allow() method) whenever we are in such a low rate regime.
    if rtl < 1 {
        maxDelay = 500 * time.Millisecond
    } else {
        maxDelay = time.Second / (time.Duration(rtl) * 2)
    }
}
```

maxDelay 的最大值是 500ms



## Provider

Provider 也称为服务发现，作用是获取有关路由的相关信息，当 Traefik 检测到更改时，有动态更新路由的功能

目前 Provider 主要有下面四种

1. 基于标签：每个部署的容器都有相关的标签

2. 基于 kv 键值对：每个部署的容器都更新其相关的键值对信息

3. 基于注解：每个部署的容器都有相关的注解定义区分

4. 基于文件：使用文件来发现服务，顾名意思就是将配置信息（比如路由，中间件，后端服务等）全部配置在单个或者多个文件里面

目前路由配置来源主要来源以下三种，也是用的比较多的如下

1. File provider：YAML、TOML、JSON

2. KV stores：Consul、etcd、Redis、Zookeeper

3. Kubernetes CRD (IngressRoute)

其中 Kubernetes CRD 是最常用的一种，更接近云原生的使用方式





# 插件编写

Traefik 的插件本质是用 Yaegi 这个 Go 语言的解释器来执行 Go 代码，既然是解释器，不是编译器，那么执行必定性能有损耗，并且解释器不支持以下用法

1. 不能用 cgo

2. 不能用 unsafe

3. 大部分需要反射底层的库不能使用（比如 encoding/gob 等这些）

> 注：网关本身是需要很轻量，插件的逻辑不能重，推荐只进行一些header修改以及一些 JWT 的鉴权等这些旁路的逻辑构成插件，其他重逻辑都不推荐使用插件来进行

插件的编写也有严格的标准，一定要暴露下面这几个接口 [参考文档](https://github.com/traefik/plugindemo)

```go
// Package example a example plugin.
package example

import (
        "context"
        "net/http"
)

// Config the plugin configuration.
type Config struct {
        // ...
}

// CreateConfig creates the default plugin configuration.
func CreateConfig() *Config {
        return &Config{
                // ...
        }
}

// Example a plugin.
type Example struct {
        next     http.Handler
        name     string
        // ...
}

// New created a new plugin.
func New(ctx context.Context, next http.Handler, config *Config, name string) (http.Handler, error) {
        // ...
        return &Example{
                // ...
        }, nil
}

func (e *Example) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
        // ...
        e.next.ServeHTTP(rw, req)
}
```

