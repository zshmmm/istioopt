# istio-proxy 资源配置

## 1. 背景

istio-proxy 资源配置不合理会引起HPA异常，配置过小导致应用频繁缩扩容（POD的变化会引起xDS的推送，大量POD变化引发大量xDS推送，进而引起istiod性能问题），配置过大HPA扩容失效，会触发业务容器 CPU 节流影响系统可用性。

istio-proxy 默认的资源配置如下，默认配置会引发一些问题：
> **经验前提：在开启日志的情况下，正常的 http/grpc istio-proxy 的 CPU 使用与业务容器 CPU 使用比例接近 1:1。**

```yaml
resources:
  limits:
    cpu: "2"
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 128Mi
```

假设应用的资源配置为 `requests.cpu = 1000m`，HPA 配置 CPU 达到 50% 扩容。应用 CPU 使用为 300m，istio-proxy CPU 使用为 300m。此时POD的整体CPU使用率为 `(300+300)/1100*100` 54.5% 达到了 HPA 扩容阈值，业务开始不必要的扩容。反之 istio-proxy CPU 配置过大达不到 HPA 扩容阈值，业务无法扩容。

## 2. 资源配置最佳实践

资源配置需要综合考虑业务资源配置、sidecar 资源配置、HPA 阈值等多方面配置。

### 2.1 istio-proxy sidecar 最佳实践

最优的配置需要满足如下规则

1. istio-proxy 容器的 CPU request / 应用容器的 CPU request = istio-proxy CPU 实际使用 / 应用容器的 CPU 实际使用
2. istio-proxy 容器的 memory request = 7天 istio-proxy memory 最大值向上取整（128/256/512/1024等）

**注：线上应用的内存利用率会相对平稳，HPA 多以 CPU 利用率作为缩扩容阈值，如果通过内存做 HPA 缩扩容阈值，那么 istio-proxy 容器的 memory request 需要满足如下规(本文不涉及根据内存作为 HPA 的阈值)**

1. istio-proxy 容器的 memory request / 应用容器的 memory request = istio-proxy memory 实际使用 / 应用容器的 memory 实际使用

### 2.2 HPA 配置实践（基于CPU）

HPA CPU 配置最佳实践

|资源档位|request(app + istio-proxy)|HPA阈值|
|:---:|---:|---:|
|一档|[0.2,0.6)|<=200%|
|二挡|[0.6,1.5)|<=150%|
|三档|[1.5,3)|<=100%|
|四挡|[3,5)|<=80%|
|五档|[5,)|50% - 70%|


### 2.3 limit 配置实践（基于CPU）

HPA limit 配置最佳实践

|资源档位|request(app + istio-proxy)|limit 配置|
|:---:|---:|---:|
|一档|[0.2,0.6)|request * 4|
|二挡|[0.6,1.5)|request * 3|
|三档|[1.5,3)|request * 2|
|四挡|[3,5)|request * 1.5|
|五档|[5,)|request * 1 - request * 1.5|

### 2.4 整体原则

1. 有istio注入的，request * HPA阈值不得小于500m
2. request*HPA 阈值至少比 limit 小 1000m
3. 资源需求较大 >= 4C 的情况下，limit 配置等于 request，防止节点超分过多引起 CPU 节流
4. 如果单个Pod request太大，如果Pod无状态，在可能的情况下通过扩大Pod副本来降低单个实例的request。 

### 2.5 资源指标

1. 容器 CPU 使用指标 `container_cpu_usage_seconds_total`
    计算应用容器与 sidecar 的 CPU 使用比：`(sum(rate(container_cpu_usage_seconds_total{cluster=~"$cluster",pod=~"^$app-[a-z0-9]+-[a-z0-9]+$",image!="",container="service"}[1m])) by (cluster)) / (sum(rate(container_cpu_usage_seconds_total{cluster=~"$cluster",pod=~"^$app-[a-z0-9]+-[a-z0-9]+$",image!="",container="istio-proxy"}[1m])) by (cluster))`
2. 容器 memory 使用指标 `container_memory_usage_bytes`
    最后7天最大内存：`max(max_over_time(sum(container_memory_usage_bytes{cluster=~"$cluster",pod=~"^$app-[a-z0-9]+-[a-z0-9]+$", container=~"istio-proxy|service"}) by (cluster, pod, container)[7d:])) by (cluster, container)`
3. POD 请求数 `istio_requests_total`
    单核 istio-proxy CPU 能够处理的 QPS：`sum(rate(istio_requests_total{reporter="destination",cluster=~"$cluster",app="$app"}[1m])) / sum(rate(container_cpu_usage_seconds_total{cluster=~"$cluster",pod=~"^$app-[a-z0-9]+-[a-z0-9]+$",image!="",container=~"istio-proxy"}[1m]))`

## 3. 具体配置

istio-proxy 的资源配置在注解中：

```yaml
    sidecar.istio.io/proxyCPU: "0.3"
    sidecar.istio.io/proxyCPULimit: "1"
    sidecar.istio.io/proxyMemory: 512Mi
    sidecar.istio.io/proxyMemoryLimit: 1Gi
```

## 4. 思考

istio 的复杂性，引入后必然在稳定性上带来更多的问题，同时也会有额外的性能开销和资源占用（至少额外的30%）。

当想引入 istio 时，需要重新思考如下几个问题：
1. 当前是引入 istio 的时期么？
2. 哪些特性是必须由 istio 来提供么？
3. istio 引入带来的优势大于资源的消耗么？
4. 有专业的团队进行支撑么？

## 5. 参考

1. [负载注解配置](https://istio.io/latest/docs/reference/config/annotations/)
2. [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)