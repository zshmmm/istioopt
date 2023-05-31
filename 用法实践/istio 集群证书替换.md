# istio 集群证书替换

## 1. 背景

在Istio中，替换证书是出于以下两个方面的考虑：

第一，如果在Istio安装时没有创建相应的证书文件，则会默认使用自签名证书。在这种情况下，为了提高安全性，可以替换为由CA颁发的证书。

第二，如果在Istio中部署了多个网格集群，则这些网格之间需要互信的信任关系。为了达到此目的，需要对每个网格生成证书，并确保跨集群之间的证书互信。

因此，为了实现证书的统一管理和多集群网格之间的互信，我们需要重新生成集群证书并将其替换为默认的自签名证书。

## 2. 生成证书

生成证书相关文档可参考 [基础理论/istio安装部署/2.1 生成证书文件](https://istio-opt.gitbook.io/istioopt/li-lun-ji-chu/istio-an-zhuang-bu-shu#2.1-sheng-cheng-zheng-shu-wen-jian) 章节，这里不再复述。


## 3. 替换证书

### 3.1 关闭集群 mTLS

替换证书之前，应该优先关闭集群的 mTLS 连接隧道。istio 集群中使用 `PeerAuthentication` 资源控制集群内工作负载见的通信是否使用 TLS 认证加密。

```yaml
# mtls.yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "defalut mTLS"
spec:
  mtls:
    mode: DISABLE
```

```bash
# 关闭集群内 mTLS ，资源应用到 istio 命名空间会对整个集群生效
kubectl apply -f mtls.yaml -n istio-system
```

mode 有如下几种模式
- UNSET：从父级继承，如果有的话，否则同PERMISSIVE
- DISABLE：连接不适用 mTLS 隧道
- PERMISSIVE：连接可以是明文或mTLS隧道
- STRICT：连接必须是 mTLS 隧道（必须提供带有客户端证书的TLS）

更多关于 `PeerAuthentication` 资源信息请参考[官方文档](https://istio.io/latest/docs/reference/config/security/peer_authentication/)。

### 3.2 应用新证书

```bash
# 重启 istiod ，使用新证书 cacerts
kubectl rollout restart deployment -n istio-system istiod

# 等待 istiod 全部重启完成
```

### 3.2 开启 mTLS

如果只是部分服务注入了 Envoy sidecar 的情况。对于一个已注入 sidecar 的服务而言，一旦开启服务的双向 TLS 通信模式，那么没有注入 Envoy sidecar 的负载将无法与注入了 Envoy sidecar 的负载通信。为了解决这个问题，Istio 认证策略提供了一种 “PERMISSIVE” 模式。

在集群级别使用 `PERMISSIVE` 模式，调整 `defalut mTLS` 模式为 `PERMISSIVE`。此时集群已经使用新的证书作为 mTLS 隧道的加密和认证。


## 4. 参考

1. [TLS 配置](https://istio.io/v1.14/docs/tasks/security/authentication/mtls-migration/)
2. [istio 安全](https://istio.io/v1.14/docs/concepts/security/)