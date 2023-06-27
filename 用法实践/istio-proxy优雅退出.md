# istio-proxy 优雅退出

## 1. 背景

一些服务在退出的过程中，业务流程未处理完，引发业务日志报错，应用SLI指标降低（业务在未引入 istio 时可以实现优雅退出），本文主要介绍在 istio 场景下业务优雅退出需要关注的配置点。

几个相关参数：
- terminationDrainDuration 默认情况下 istio-proxy 在收到 SIGTERM 或 SIGINT 信号后，会通知 envoy 开始“排水”（停止新建连接，允许已存在连接继续通讯），并在 terminationDrainDuration 时间后退出。该配置默认为 5s，对处理耗时较长的业务有影响。
- terminationGracePeriodSeconds k8s 发送 SIGTERM 信号给容器内主进程以通知容器进程开始优雅停止，在 terminationGracePeriodSeconds 时间后如果尚未完全停止，则发送 SIGKILL 信号将其强制杀死。该配置默认为 30s。
- EXIT_ON_ZERO_ACTIVE_CONNECTIONS  istio 1.12 版本后开始支持 EXIT_ON_ZERO_ACTIVE_CONNECTIONS 环境变量，作用是等待 Sidecar “排水”完成，在响应时，通知客户端去关闭长连接（对于 HTTP1 响应 “Connection: close” 这个头，对于 HTTP2 响应 GOAWAY 这个帧）。

## 2. 配置

### 2.1 istio 版本 < 1.12

修改 `terminationDrainDuration` 配置保持与 `terminationGracePeriodSeconds` 时间一致或稍大几秒，保证在业务退出之前 istio-proxy 不退出。

**例子** `terminationGracePeriodSeconds` 保持默认配置 30s

#### 2.1.1 全局配置

```yaml
apiVersion: v1
kind: ConfigMap
data:
  mesh: |-
    defaultConfig:
      holdApplicationUntilProxyStarts: true
      terminationDrainDuration: 35
  meshNetworks: 'networks: {}'
```

#### 2.1.2 指定负载配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        proxy.istio.io/config: |
          holdApplicationUntilProxyStarts: true
          terminationDrainDuration: 35
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: "nginx"
        imagePullPolicy: IfNotPresent
```

### 2.2 istio 版本 >= 1.12

#### 2.2.1 全局配置

```yaml
apiVersion: v1
kind: ConfigMap
data:
  mesh: |-
    defaultConfig:
      holdApplicationUntilProxyStarts: true
      proxyMetadata: 
        EXIT_ON_ZERO_ACTIVE_CONNECTIONS: "true"
  meshNetworks: 'networks: {}'
```

#### 2.2.2 指定负载配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        proxy.istio.io/config: |
          holdApplicationUntilProxyStarts: true
          proxyMetadata:
            EXIT_ON_ZERO_ACTIVE_CONNECTIONS: 'true'  
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: "nginx"
        imagePullPolicy: IfNotPresent
```

**注意：istio-proxy 不会一直排水，最终 sidecar 在 `terminationGracePeriodSeconds` 时间后被 Kill 掉**

## 3. 思考

该配置是在应用已经正确实现“优雅退出”的前提下的配置，优雅退出在云原生环境中的重要性不容忽视，正确的实现优雅退出在更新、重启、缩扩容的情况下不会中断用户体验和业务流程。因此优雅退出，有助于提高系统的稳定性和可用性。

## 4. 参考

1. [terminationGracePeriodSeconds配置](https://istio.io/v1.14/docs/reference/config/istio.mesh.v1alpha1/)
2. [Pod Lifecycle配置](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
3. [EXIT_ON_ZERO_ACTIVE_CONNECTIONS配置](https://istio.io/v1.14/docs/reference/commands/pilot-agent/)
4. [proxyMetadata配置](https://istio.io/v1.14/docs/reference/config/istio.mesh.v1alpha1/)