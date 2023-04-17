# sidecar 启动顺序

## 1. 背景

一些服务在启动的过程中，日志打印连接数据库、中间件失败等信息，排查原因是业务启动时需要连接数据库、中间件获取一些数据或配置。由于 envoy 还没有就绪（envoy需要从控制面拉取配置，需要一些时间），导致业务的流量无法被处理，从而调用失败。[参考k8s issue](https://github.com/kubernetes/kubernetes/issues/65502)。

## 2. 解决方案

### 2.1 istio 版本 < 1.7

在启动应用程序之前通过envoy健康检查接口`localhost:15020/healthz/ready`判断envoy xDS是否配置完成（配置完成后返回200，否则返回503）。通过增加envoy检查步骤，在监控检查通过后再启动业务服务。

**例子：** 应用通过`start-app-cmd`启动，通过修改yaml，增加envoy健康检查逻辑确保应用在envoy在xDS配置完成后启动，具体如下：

```yaml
command: ["/bin/bash", "-c"]
args: ["while [[ \"$(curl -s -o /dev/null -w ''%{http_code}'' localhost:15020/healthz/ready)\" != '200' ]]; do echo Waiting for Sidecar;sleep 1; done; echo Sidecar available; start-app-cmd"]
```

不足：该方案可以规避依赖顺序的问题，但需要对应用容器的启动脚本进行修改，对 Envoy 的健康状态进行判断。更好的方案应该是应用对 Envoy sidecar 不感知。

### 2.2 istio 版本 >= 1.7

为避免出现如上问题，理想的启动执行流程如下：

1. Kubernetes 启动 envoy sidecar。
2. Kubernetes 执行 postStart hook[^1]，postStart hook 通过 envoy 健康检查接口判断其配置初始化状态，直到 envoy 启动完成。
3. Kubernetes 启动应用容器。

Istio 在 1.7 版本中增加该方案，增加 HoldApplicationUntilProxyStarts[^2]，配置开关(默认为false)，需要主动打开配置。

#### 2.2.1 全局配置方案

修改 istio 的 configmap 全局配置

```bash
kubectl -n istio-system edit cm istio
```

在 defaultConfig 下加入 `holdApplicationUntilProxyStarts: true`

```yaml
apiVersion: v1
kind: ConfigMap
data:
  mesh: |-
    defaultConfig:
      holdApplicationUntilProxyStarts: true
  meshNetworks: 'networks: {}'
```

若使用 IstioOperator，defaultConfig 修改 CR 字段 meshConfig:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: installed-state
spec:
  meshConfig:
    defaultConfig:
      holdApplicationUntilProxyStarts: true
```

开启之后，新建的 pod 负载会在 istio-proxy 容器启动后增加如下 `postStart hook` 配置

```yaml
lifecycle:
  postStart:
    exec:
      command:
      - pilot-agent
      - wait
```

#### 2.2.2 指定负载配置

在 istio 版本 >= 1.8 时，可以为指定的负责开启 `holdApplicationUntilProxyStarts` 功能，在相应负载中增加 `proxy.istio.io/config` 注解，并在注解中增加 `holdApplicationUntilProxyStarts: true` 配置

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
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: "nginx"
        imagePullPolicy: IfNotPresent
```

1. [ProxyConfig holdApplicationUntilProxyStarts 配置项](https://istio.io/v1.14/docs/reference/config/istio.mesh.v1alpha1/#ProxyConfig)

[^1]: [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
