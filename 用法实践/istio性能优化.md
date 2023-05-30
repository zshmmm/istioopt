# 性能优化

## 1. 控制层面

### 1.1 xDS 优化

#### 1.1.1 xDS 推送优化

**1. 优化推送策略**

istiod xDS 推送相关的几个环境变量:
1. `PILOT_ENABLE_EDS_DEBOUNCE`：xDS 推送防抖中包含 EDS，防抖的时间由 `PILOT_DEBOUNCE_AFTER` 和 `PILOT_DEBOUNCE_MAX`， 该配置可能会导致 EDS 推送延迟，默认开启。
2. `PILOT_DEBOUNCE_AFTER`：防抖处理的延迟时间，在该推送间隔内如果没有任何改动，则推送 xDS，否则计时器重新开始计时。最大延迟时间由 `PILOT_DEBOUNCE_MAX` 配置决定。默认为 100ms 。
3. `PILOT_DEBOUNCE_MAX `：防抖推送等待的最长时间。默认为 10s 。
4. `PILOT_PUSH_THROTTLE`：并发推送数，在大集群中增加该配置会提高 xDS 的推送效率。

在大集群中，增加并发推送数，增加防抖推送间隔，降低推送等待最长时间，可以有效的缓解 xDS 推送压力，同时更及时的推送 xDS 到 istio-proxy。

**2. 通过 Sidecar 降低 xDS 推送大小**

istio 默认会将集群内所有发现的资源通过 xDS 下发给 istio-proxy，为减少不相干 xDS 的推送，可以通过 Sidecar 资源来精确配置工作负载的访问规则。间接上降低 xDS 到工作负载的下发量。

> **注意：新增的集群内的访问需要修改 Sidecar 来实现负载通信，否则请求将根据 `outboundTrafficPolicy` 的配置来确认是否转发流量到网格外**
>
> **如果配置为：`REGISTRY_ONLY`，则服务不可访问**
>
> **如果配置为：`ALLOW_ANY`，则服务可以访问，只不过是通过 k8s svc 的方式访问**

具体配置

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: grpc
spec:
  egress:
  - hosts:
    - ./*
    - istio-system/*
```

如上配置 istid 只推送 grpc 和 istio-system namespace 内负载的 xDS 到 grpc namespace 下的工作负载。减少 xDS 的推送量，提高推送效率。

思考：手动调整 Sidecar 遗漏引发问题，如何动态的根据请求自动调整 Sidecar 才是解决的根本方案（懒加载）。

**3. 通过 discoverySelectors 配置减少服务发现的数量**
istio 默认情况下会自动发现集群内所有服务，并生成相应的 xDS 信息，可以通过 istio 配置发现哪些 namespace 中的服务来减少 xDS 信息。
同样，不在网格内的服务根据 `outboundTrafficPolicy` 配置来决定是否转发流量。

在 istio configmap 中 `data.mesh.discoverySelectors` 路径中增加如下配置

```yaml
    - matchExpressions:
      - key: discovery/ignore
        operator: DoesNotExist
```

表示不自动发现包含 `discovery/ignore` 标签的 namespace 的服务。

### 1.2 istiod 资源配置

istiod 的资源配置没有一个通用的合理配置，需要根据实际使用情况来调整，随着网格资源的增加，istiod 需要提高资源配置。

### 1.3 istiod 负载均衡

istiod 与网格内 istio-proxy 使用 gRPC 长链接进行通信，引起 istiod 连接的 pod 数量不一致，导致 istiod 负载不均衡。定期断开连接，让 envoy 重新连接会缓解 istiod 不均衡的情况。
通过配置 istiod 启动参数 `keepaliveMaxServerConnectionAge` 来控制断开时间，该值默认为 `30m`，可以根据实际需求来调整。


思考：为什么不在 istio-proxy 中配置主动断开？

## 2. 数据层面

### 2.1 istio-proxy 优化

### 2.2 gateway 优化



## 4. 参考

1. [Better load balancing of Envoys across Pilot instances](https://github.com/istio/istio/issues/11181)
2. [pilot-discovery 环节变量](https://istio.io/latest/docs/reference/commands/pilot-discovery/#envvars)
3. [SideCar](https://istio.io/latest/docs/reference/config/networking/sidecar/)
