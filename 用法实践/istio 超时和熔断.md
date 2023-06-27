# istio 超时和熔断

## 1. 背景

随着 kubernetes 集群规模不断增大，pod 数量不断增加。业务高峰期时容器大量水平扩容，低峰期时容器大量缩容。在这个过程中 xDS 频繁变化 istiod 推送压力增加，引发 xDS 下发延迟。请求短时间内无法到达新的 pod，或请求依然发送已经销毁的 pod ，两种情况都有可能降低业务可用性，尤其是后者。虽然 istio 有默认的连接超时(10s)和重试机制(3次)，超时时间过长，引发业务测超时从而影响业务稳定性。istio 提供连接超时和熔断的配置，进行合理的配置提早发现不肯用 endpoint 并熔断，可以有效降低 xDS 推送不及时引发的问题(仅仅是缓解，无法从根本上解决 xDS 推送不及时问题)。

## 2. 配置

连接超时和熔断的 DR 配置

```yaml
# finename connect-timeout-dr.yaml
kind: DestinationRule
metadata:
  name: connect-timeout
spec:
  host: recommendation.weather.svc.cluster.local # 目标 host
  trafficPolicy:
    connectionPool:
      tcp:
        connectTimeout: 100ms # 在局域网中，TCP 连接时长会远远小于 100 ms ，在 100ms 内没有建立连接则超时。
        tcpKeepalive:
          time: 7200s
          interval: 75s
    loadBalancer:
      simple: ROUND_ROBIN # 负载均衡模式
      localityLbSetting:
        enabled: false # 关闭区域感知(本次测试不涉及区域，有条件的可以打开)
    outlierDetection: # 熔断配置
      baseEjectionTime: 30s # endpoint 最短被剔除时间(保持剔除的时间等于最短剔除时间和 endpoint 被弹出次数的乘积)
      consecutive5xxErrors: 5 # endpoint 被剔除之前 5xx 的次数(连接超时、连接错误/失败和请求失败事件都属于 5xx)
      consecutiveGatewayErrors: 3 # endpoint 被剔除之前的网关错误数(当通过 HTTP 访问上游主机时，502、503 或 504 返回代码被视为网关错误)
      interval: 15s # 检查间隔
      maxEjectionPercent: 50 # 恐慌值，最多剔除百分比的不可用 endpoint
```

## 3. 验证

### 3.1 环境

在 weather namespace 下 名为 forecast 的应用访问 recommendation.weather.svc.cluster.local:3005 服务。通过故障注入截断 forecast 与 istio 15012 端口的通信(模拟 xDS 下发不及时的场景)

recommendation 应用有5个副本，调整副本数到3个，由于 forecast-v1 与 istiod 无法通信，获取不到最小的 xDS(无法更新 EDS)，即获取不到 recommendation 服务最新的 endpoints 。环境如下图所示

![验证环境][超时熔断测试环境]

```bash
# recommendation 副本数，5 个 pod
kubectl get pods -l app=recommendation

# NAME                                 READY   STATUS    RESTARTS   AGE
# recommendation-v1-66dd9f6595-494qj   2/2     Running   0          13s
# recommendation-v1-66dd9f6595-7txkf   2/2     Running   0          13s
# recommendation-v1-66dd9f6595-rl9xf   2/2     Running   0          13s
# recommendation-v1-66dd9f6595-tsc9m   2/2     Running   0          12d
# recommendation-v1-66dd9f6595-xp9x2   2/2     Running   0          13s


# forecast 对 recommendation.weather.svc.cluster.local 的 endpoint
istioctl pc endpoint forecast-v1-869876d4f4-4tp4m --cluster "outbound|3005|v1|recommendation.weather.svc.cluster.local"

# 5 个 endpoint，对应上面的 5 个 pod
# ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
# 10.42.1.39:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.1.40:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.1.41:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.2.13:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.2.6:3005      HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
```

### 3.2 验证

#### 3.2.1 使用 DR 前

观察 forecast 访问 recommendation 情况，调整 recommendation 副本数到 3 。

```bash
# 观察访问情况，在 forecast 中请求 recommendation
kubectl exec -it forecast-v1-869876d4f4-4tp4m -c forecast -- /bin/bash
for i in {1..10000}; do curl -s -w "Status: %{http_code}\n" 'http://recommendation.weather.svc.cluster.local:3005/activity?weather=rain&temp=01'; sleep 0.5;done

# 调整 recommendation 副本数到 3
kubectl scale deployment --replicas=3 recommendation-v1

# 查看 pod 数
kubectl get pods -l app=recommendation -o wide

# NAME                                 READY   STATUS    RESTARTS   AGE   IP           NODE             NOMINATED NODE   READINESS GATES
# recommendation-v1-66dd9f6595-7txkf   2/2     Running   0          31m   10.42.1.40   192.168.18.101   <none>           <none>
# recommendation-v1-66dd9f6595-tsc9m   2/2     Running   0          12d   10.42.2.6    192.168.18.102   <none>           <none>
# recommendation-v1-66dd9f6595-xp9x2   2/2     Running   0          31m   10.42.2.13   192.168.18.102   <none>           <none>

# forecast 对 recommendation.weather.svc.cluster.local 的 endpoint
istioctl pc endpoint forecast-v1-869876d4f4-4tp4m --cluster "outbound|3005|v1|recommendation.weather.svc.cluster.local"

# 由于 xDS 无法下发，对应的 endpoint 没有减少
# ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
# 10.42.1.39:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.1.40:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.1.41:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.2.13:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.2.6:3005      HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local

# 同时可以观察到请求会卡主，由于 istio 有重试，请求并不会失败，只是整体请求时间变长。
```

#### 3.2.2 应用 DR

```bash
kubectl apply -f connect-timeout-dr.yaml

# forecast 对 recommendation.weather.svc.cluster.local 的 endpoint
istioctl pc endpoint forecast-v1-869876d4f4-4tp4m --cluster "outbound|3005|v1|recommendation.weather.svc.cluster.local"

# ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
# 10.42.1.39:3005     HEALTHY     FAILED            outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.1.40:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.1.41:3005     HEALTHY     FAILED            outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.2.13:3005     HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
# 10.42.2.6:3005      HEALTHY     OK                outbound|3005|v1|recommendation.weather.svc.cluster.local
```
可以看到不存在的 pod 的 endpoint 被熔断掉了。同时请求也会变快，不会被卡主，因为 tcp 层面连接超时只有 100ms ，加快了故障感知熔断。

### 3.3 应用默认配置

上面的配置是针对单个应用的，集群中存在大量的应用，如何为存量的应用配置默认超时和熔断，在应用配置时又可以以应用的配置为主呢？ istiod 有一个环境变量 `PILOT_ENABLE_DESTINATION_RULE_INHERITANCE` 来控制 DR role 的继承。该变量默认是关闭的，需要先开启。
开启后在 istio 的 rootNamespace 中创建一个 host 为空的 DR 资源（注意：必须先开启 DR Role 功能，否则无法创建 host 为空的 DR）。

```yaml
# file-name connect-timeout-global.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: connect-timeout-global
  namespace: istio-system
spec:
  trafficPolicy:
    connectionPool:
      tcp:
        connectTimeout: 200ms # 全局配置 200 ms 超时
        tcpKeepalive:
          time: 7200s
          interval: 75s
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: false
    outlierDetection:
      baseEjectionTime: 30s
      consecutive5xxErrors: 5
      consecutiveGatewayErrors: 3
      interval: 15s
      maxEjectionPercent: 50
```

相应的配置会继承后合并，通过应用的 RDS 来查看应用情况。

```bash
# 查看有 DR 熔断配置的应用
istioctl pc cluster forecast-v1-84fd5dc7cc-278l2 --fqdn recommendation.weather.svc.cluster.local --direction outbound -o json | jq .[0].outlierDetection

# {
#     "consecutive5xx": 5,
#     "interval": "15s",
#     "baseEjectionTime": "30s",
#     "maxEjectionPercent": 50,
#     "enforcingConsecutive5xx": 100,
#     "enforcingSuccessRate": 0,
#     "consecutiveGatewayFailure": 3,
#     "enforcingConsecutiveGatewayFailure": 100
# }


istioctl pc cluster forecast-v1-84fd5dc7cc-278l2 --fqdn recommendation.weather.svc.cluster.local --direction outbound -o json | jq .[0].connectTimeout

# "0.100s"

# 可以看到并没有影响到应用的配置

# 查看没有 DR 熔断配置的应用
istioctl pc cluster -n grpc grpc-client-b485bcdf7-rx5pj --fqdn server-svc.grpc.svc.cluster.local --direction outbound -o json | jq .[0].outlierDetection | jq .[0].outlierDetection

#{
#    "consecutive5xx": 5,
#    "interval": "15s",
#    "baseEjectionTime": "30s",
#    "maxEjectionPercent": 50,
#    "enforcingConsecutive5xx": 100,
#    "enforcingSuccessRate": 0,
#    "consecutiveGatewayFailure": 3,
#    "enforcingConsecutiveGatewayFailure": 100
#}

istioctl pc cluster -n grpc grpc-client-b485bcdf7-rx5pj --fqdn server-svc.grpc.svc.cluster.local --direction outbound -o json | jq .[0].outlierDetection | jq .[0].connectTimeout

# "0.200s"

# 有了继承的熔断配置
```

## 4. 思考

1. 如果将 recommendation 副本数降低到 1 个或者 2 个时会发生什么？不可用 endpoint 会被踢掉么？答案是否定的，被踢掉多少和 "恐慌值" (maxEjectionPercent: 50 # 恐慌值，最多剔除百分比的不可用 endpoint) 有关，最大不能超过踢掉恐慌值的比例。

2. 合理的连接超时策略和熔断策略能够提高业务的稳定性，这也是 istio 能够带来的好处之一。

3. 本配置并不能从根本上解决 xDS 推送延迟问题，只能缓解带来的影响（xDS 推送一定会有延迟，合理的超时和熔断可以从根本上缓解带来的问题）。

4. 连接超时可以通过 DR 默认配置，请求超时是 VS 来控制的，配置默认的请求超时也可以触发熔断。由于当前无法配置全局的 VS，istiod 提供了一个默认的环境变量 `ISTIO_DEFAULT_REQUEST_TIMEOUT` 来控制请求超时，默认为 0s (永不超时)。配置一个合理的请求超时策略同样能够提高业务稳定性，这同样也是 istio 能够带来的好处之一。


## 5. 参考

1. [熔断](https://istio.io/v1.14/docs/reference/config/networking/destination-rule/#OutlierDetection)
2. [pilot-discovery 配置](https://istio.io/v1.14/docs/reference/commands/pilot-discovery/)


[超时熔断测试环境]: /images/istio熔断环境.png