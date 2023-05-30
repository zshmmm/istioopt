# 安装部署

本教程以 1.14 版本为例。

## 1. 下载
istio 部署包可以通过如下两种方式下载
1. 官方页面下载
istio 版本发布页面：https://github.com/istio/istio/releases
在版本发布页面下载 1.14.6 版本

```bash
mkdir -p /data/opt/k8s/istio/learn
cd /data/opt/k8s/istio/learn
wget https://github.com/istio/istio/releases/download/1.14.6/istio-1.14.6-linux-amd64.tar.gz
tar xzvf istio-1.14.6-linux-amd64.tar.gz
# 安装 istioctl
ln -s /data/opt/k8s/istio/learn/istio-1.14.6/bin/istioctl /usr/sbin/
```

2. 通过官方下载命令下载

```bash
# 当前方式需要一些科学上网，建议使用第一种方式
mkdir -p /data/opt/k8s/istio/learn
cd /data/opt/k8s/istio/learn
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.14.6 TARGET_ARCH=x86_64 sh -
# 安装 istioctl
ln -s /data/opt/k8s/istio/learn/istio-1.14.6/bin/istioctl /usr/sbin/
```

## 2. 安装部署
istio 官方建议使用 `istioctl install` 方式安装。

`istioctl install --set profile=<mode>` 安装配置档有几种，按照配置差异如下：

|配置档|default|demo|minimal|remote|empty|preview|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|核心组件||||||
|istio-egressgateway| |√| | | | |
|istio-ingressgateway|√|√| | | |√|
|istiod|√|√|√| | |√|

支持的配置模式可以使用如下命令获取

```bash
istioctl profile list
```

### 2.1 生成证书文件
默认情况下，Istio CA 会生成一个自签名的根证书和密钥，并使用它们来签署工作负载证书。如果只是测试使用使用默认证书也不会有问题。为了保护根 CA 密钥，或者多集群互信，在部署 istio 之前应该先生成根CA，并使用根 CA 向运行在每个集群上的 Istio CA 签发中间证书。Istio CA 可以使用管理员指定的证书和密钥来签署工作负载证书，并将管理员指定的根证书作为信任根分配给工作负载。
如果之前漏掉了本步骤，后续文章中会介绍如何为运行中的 istio 集群替换证书。
正常情况下应该使用公司根 CA 签发 istio 根证书，并由 istio 根证书签发 istio 集群证书，整体证书签发如下：

![istio CA 证书链][istio-CA-chain]

是一个三层的证书链，本文档为简化操作步骤，直接使用 istio 提供的证书工具进行证书生成，生成的证书是一个两层证书链（不包含最高的公司根CA）

```bash
cd istio-1.14.6
# 创建证书存储目录
mkdir -p certs

# 修改几个参数
# 证书有效期，修改 /tools/certs/common.mk 文件中 ROOTCA_DAYS 到 100 年，INTERMEDIATE_DAYS 到 50 年。
# 其他参数可自行修改，也可不修改

# 修改保留文件，生成证书过程的 .conf 文件不会保留，为了保留配置文件，修改 ../tools/certs/Makefile.selfsigned.mk 文件，如下两行
# .PRECIOUS: %/ca-key.pem %/ca-cert.pem %/cert-chain.pem %/cluster-ca.csr %/intermediate.conf
# .SECONDARY: root-cert.csr root-ca.conf

# 生成 istio root CA
cd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca
# 生成 root-ca.conf  root-cert.csr  root-cert.pem  root-key.pem 4个文件
# root-cert.pem：生成的根证书
# root-key.pem：生成的根密钥
# root-ca.conf：生成根证书的 openssl 配置
# root-cert.csr：为根证书生成的 CSR

# 查看 root-cert.pem 信息，可以看到证书相关信息
openssl x509 -in root-cert.pem -textopenssl x509 -in 

# 生成需要部署 istio 集群的集群证书，集群名字为参数前缀 <cluster-name>-cacerts
# 本集群名为： cluster1
make -f ../tools/certs/Makefile.selfsigned.mk cluster1-cacerts
# cluster1 集群证书生成在目录 cluster1 中，包含 ca-cert.pem  ca-key.pem  cert-chain.pem  cluster-ca.csr  intermediate.conf  root-cert.pem 几个文件
# ca-cert.pem：生成的中间证书
# ca-key.pem：生成的中间密钥
# cert-chain.pem：istiod 使用的生成的证书链
# root-cert.pem：根证书

# 创建 istio cacerts
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster1/ca-cert.pem \
      --from-file=cluster1/ca-key.pem \
      --from-file=cluster1/root-cert.pem \
      --from-file=cluster1/cert-chain.pem
```

证书生成完成后回到 istio 按照包根目录。

### 2.2 生产配置档配置文件
在生产环境中，推荐使用 `minimal` 配置模式安装，然后修改相关配置。将配置进行版本化管理，任何调整都记录到配置文件。

```bash
# 创建保存部署文件目录
mkdir install/istio-1.14
cd install/istio-1.14
# 保存配置文件
istioctl profile dump minimal > istio-1.14.yaml
```

配置档文件是一个 `IstioOperator` CR，文件中包含6个组件(base, pilot, ingressGateways, egressGateways, cni, istiodRemote)，针对每一个组件的配置内容通过 `spec.components.<componet name>` 下的 API 中提供。所有的组件共享一个通用 API，用来修改 Kubernetes 特定的设置，它在 `spec.components.<component name>.k8s` 路径下，所有这些 k8s 的配置项设置均使用 Kubernetes API 相同的定义。
istio configmap 的配置在 `spec.meshConfig` 下的 API 中提供。
其他配置在 `spec.values.global` 路径下，比如 proxy、proxy_init 等的资源、镜像、HPA等配置。
一份调整(主要是根据往期文章优化相关方面的配置)过后的配置文件如下：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: false
      name: istio-egressgateway
    ingressGateways:
    - enabled: false
      name: istio-ingressgateway
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
      # 控制面 pilot 的一些配置环境变量
      k8s:
        env:
        - name: PILOT_ENABLE_QUIC_LISTENERS
          value: "true"
        - name: PILOT_DEBOUNCE_AFTER
          value: "500ms"
        - name: PILOT_DEBOUNCE_MAX
          value: "3s"
        - name: PILOT_REMOTE_CLUSTER_TIMEOUT
          value: "0"
        - name: PILOT_ENABLE_CROSS_CLUSTER_WORKLOAD_ENTRY
          value: "false"
        - name: PILOT_USE_ENDPOINT_SLICE
          value: "false"
        - name: PILOT_PUSH_THROTTLE
          value: "200"
        - name: PILOT_FILTER_GATEWAY_CLUSTER_CONFIG
          value: "true"
  hub: docker.io/istio # 可以替换为公司内镜像仓库，加速镜像下载（如果镜像仓库需要认证，则在 sepc.global.imagePullSecrets 中配置认证文件
  meshConfig:
    # istio  configmap 配置文件配置
    discoverySelectors:
      - matchExpressions:
        - key: discovery/ignore
          operator: DoesNotExist
    defaultConfig:
      concurrency: 4
      holdApplicationUntilProxyStarts: true
      proxyMetadata: 
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
        ISTIO_META_DNS_CAPTURE: "true"
        DNS_AGENT: ""
      proxyMetadata: {}
    enablePrometheusMerge: true
  profile: minimal
  tag: 1.14.6
  values:
    base:
      enableCRDTemplates: false
      validationURL: ""
    defaultRevision: ""
    gateways:
      istio-egressgateway:
        autoscaleEnabled: true
        env: {}
        name: istio-egressgateway
        secretVolumes:
        - mountPath: /etc/istio/egressgateway-certs
          name: egressgateway-certs
          secretName: istio-egressgateway-certs
        - mountPath: /etc/istio/egressgateway-ca-certs
          name: egressgateway-ca-certs
          secretName: istio-egressgateway-ca-certs
        type: ClusterIP
      istio-ingressgateway:
        autoscaleEnabled: true
        env: {}
        name: istio-ingressgateway
        secretVolumes:
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          secretName: istio-ingressgateway-certs
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          secretName: istio-ingressgateway-ca-certs
        type: LoadBalancer
    global:
      # meshID 和 network 配置
      meshID: mesh1
      network: network1
      configValidation: true
      defaultNodeSelector: {}
      defaultPodDisruptionBudget:
        enabled: true
      defaultResources:
        requests:
          cpu: 10m
      imagePullPolicy: ""
      imagePullSecrets: []
      istioNamespace: istio-system
      istiod:
        enableAnalysis: false
      jwtPolicy: third-party-jwt
      logAsJson: false
      logging:
        level: default:info
      meshNetworks: {}
      mountMtlsCerts: false
      # 多集群配置，集群名字
      multiCluster:
        clusterName: "cluster1"
        enabled: false
      network: ""
      omitSidecarInjectorConfigMap: false
      oneNamespace: false
      operatorManageWebhooks: false
      pilotCertProvider: istiod
      priorityClassName: ""
      proxy:
        autoInject: enabled
        clusterDomain: cluster.local
        componentLogLevel: misc:error
        enableCoreDump: false
        excludeIPRanges: ""
        excludeInboundPorts: ""
        # 网格排除端口- 为提升性能对常用 DB 和中间件端口的流量不经过 envoy 处理，直接 Passthrough
        excludeOutboundPorts: "2379,3306,6379"
        image: proxyv2
        includeIPRanges: '*'
        logLevel: warning
        privileged: false
        readinessFailureThreshold: 30
        readinessInitialDelaySeconds: 1
        readinessPeriodSeconds: 2
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 128Mi
        statusPort: 15020
        tracer: zipkin
      proxy_init:
        image: proxyv2
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 10m
            memory: 10Mi
      sds:
        token:
          aud: istio-ca
      sts:
        servicePort: 0
      tracer:
        datadog: {}
        lightstep: {}
        stackdriver: {}
        zipkin: {}
      useMCP: false
    istiodRemote:
      injectionURL: ""
    pilot:
      # 控制面 HPA
      autoscaleEnabled: true
      autoscaleMax: 5
      autoscaleMin: 2
      configMap: true
      cpu:
        targetAverageUtilization: 300
      enableProtocolSniffingForInbound: true
      enableProtocolSniffingForOutbound: true
      env: {}
      image: pilot
      keepaliveMaxServerConnectionAge: 30m
      nodeSelector: {}
      podLabels: {}
      replicaCount: 2
      traceSampling: 1
    telemetry:
      enabled: true
      v2:
        enabled: true
        metadataExchange:
          wasmEnabled: false
        prometheus:
          enabled: true
          wasmEnabled: false
        stackdriver:
          configOverride: {}
          enabled: false
          logging: false
          monitoring: false
          topology: false
```

### 2.3 生成清单文件
在 istio 安装前，使用 `istioctl manifest generate --set profile=<mode>` 命令生成清单文件，生成的清单文件可用于检查具体安装了什么。

**注意：** 虽然生成的文件可以直接通过 `kubectl apply` 的方式部署，但是请不要这样做！！因为不能保障将相同的依赖顺序应用于资源， 并且也没有在 Istio 发行版中测试过。

```bash
# 根据配置档文件生成清单文件
istioctl manifest generate -f istio-1.14.yaml > istio-1.14-resource.yaml
```

### 2.4 安装 istio

```bash
# 安装 istio 
istioctl install -f istio-1.14.yaml

# 安装验证
istioctl verify-install -f istio-1.14-resource.yaml 

# 注入验证
# grpc namesp 开启注入标签 istio-injection=enabled
kubectl label ns grpc istio-injection=enabled

# 重启应用查看注入
kubectl rollout restart deployment -n grpc
# 验证，所有pod已经被注入
kubect get pods -n grpc

# NAME                           READY   STATUS        RESTARTS   AGE
# grpc-client-6b8d875b89-dmjf4   2/2     Running       0          18m
# server-64d78996b4-lttb8        2/2     Running       0          18m
```

### 2.5 安装 istio 网关
出站网关用途较少，本文不涉及出站网关的安装部署。
网关因为支持定义多个入站网关，所以它是一种特殊类型的组件。 在 IstioOperator API 中，网关被定义为列表类型，配置路径为：`spec.components.ingressGateways`。 default 配置档会安装一个名为 istio-ingressgateway 的入站网关。
查看默认网关的默认值：

```bash
istioctl profile dump --config-path components.ingressGateways
istioctl profile dump --config-path values.gateways.istio-ingressgateway
```

#### 2.5.1 创建网关
新网关可以通过添加新的网关列表条目来创建：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  tag: 1.14.6
  components:
    ingressGateways:
      - namespace: istio-system
        name: grpc-ingress-gateway
        enabled: true
        k8s:
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
          service:
            ports:
            - name: http2
              port: 80
              protocol: TCP
              targetPort: 8080
            - name: https
              port: 443
              protocol: TCP
              targetPort: 8443
  values:
    gateways:
      istio-ingressgateway:
        autoscaleEnabled: true
        # 网关关联 service 的类型
        type: NodePort
        # 开启 gateway 注入
        injectionTemplate: gateway
```

创建网关
```bash
# 创建之前先生成资源清单
istioctl manifest generate  -f istio-1.14.yaml -f istio-ingress-gateway.yaml > istio-1.14-resource.yaml

# 创建网关
istioctl install -f istio-1.14.yaml -f istio-ingress-gateway.yaml

# 验证安装
istioctl verify-install -f istio-1.14-resource.yaml
```

内置的网关就像其他组件一样的可以被定制。但是 `spec.values.gateways.istio-ingressgateway/egressgateway` 被所有的入站网关共享。如果每个网关的配置需要定制，需要使用一个独立 IstioOperator CR 来生成用户网关的清单，并和 Istio 主安装清单隔离。

#### 2.5.2 创建独立的 IstioOperator CR 网关
因为集群已经安装了 istio 的组件，如果仅仅部署网关时，使用 `empty` 的 profile 来创建独立网关。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: empty
  tag: 1.14.6
  components:
    ingressGateways:
      - namespace: istio-system
        name: grpc-ingress-gateway-user
        enabled: true
        k8s:
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
          service:
            ports:
            - name: http2
              port: 80
              protocol: TCP
              targetPort: 8080
            - name: https
              port: 443
              protocol: TCP
              targetPort: 8443
  values:
    gateways:
      istio-ingressgateway:
        autoscaleEnabled: true
        type: NodePort
        # 开启 gateway 注入
        injectionTemplate: gateway
```

```bash
# 部署
istioctl install -f istio-1.14.yaml -f istio-ingress-gateway.yaml -f istio-ingress-gateway-empty.yaml 
```


### 2.6 配置档文件管理最佳实践

关于配置档文件管理的最佳实践：
1. 使用 `minimal` 配置模式生成基础配置文件
2. 调整基础配置文件中相关配置
3. 不同的环境、集群差异化配置单独保存
4. 安装时不安装任何网关
5. 网关单独部署，网关配置文件保存在相应环境/集群的目录中
6. 如网关需要独立配置，使用 `empty profile` 来创建
7. 使用版本管理工具来管理配置文件
8. 任何集群内配置调整，都应该同步修改配置文件

目录结构如下

```Scheme
.
├── istiod-1.14.yaml # 默认配置（基础配置、开关、镜像、版本等）
├── {环境名}
    ├── istiod-overlay.yaml # 环境配置覆盖（环境配置：资源、MeshID、Network等）
    ├── {集群名}
        ├── gateways # 网关配置文件
        │   ├── ingressgateway.yaml # 默认网关(可以不创建)
        │   └── ingressgateway-{business}.yaml # 业务网关
        └── istiod-overlay.yaml # 集群配置覆盖（集群配置：资源、集群名字等）
```

## 3. 多集群部署模式
各个集群部署完成后（根证书一致），多集部署可以参考官网。https://istio.io/v1.14/docs/setup/install/multicluster/
在网络条件（pod网络与物理网络在一个平面，并且多个集群网络也互通）允许的情况下，建议使用“多主架构”的部署模型。

**注意：多主架构的部署模式会加大 istiod xDS 的推送压力，适当的通过 Sidecar 做隔离，降低 xDS 推送量**


## 4. 参考

[通过 istioctl 安装部署 istio](https://istio.io/v1.14/docs/setup/install/istioctl/)

[IstioOperator Options](https://istio.io/v1.14/zh/docs/reference/config/istio.operator.v1alpha1/)

[istio 证书](https://istio.io/v1.14/docs/tasks/security/cert-management/plugin-ca-cert/)


[istio-CA-chain]: /images/istio证书链.png
