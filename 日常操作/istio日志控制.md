# 日志控制

## 1. 全局日志控制

> 注意：istio 默认不开启日志，istio 开启日志会消耗CPU性能

### 1.1 通过 istio 全局配置调整

开启日志输出目录和日志格式（TEXT，JSON），增加如下配置
```diff
+ MeshConfig.accessLogFile
+ MeshConfig.accessLogEncoding
```

在 istio configmap 中加入配置
```yaml
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
```

### 1.2 通过 Telemetry 开启日志

- 开启日志，创建如下 Telemetry 资源，默认使用 envoy 的日志输出规则（以TXT格式输出到 /dev/stdout）

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: envoy
```

- 通过 Telemtry 调整日志输出格式为 JSON

在 istio 全局配中增加 extensionProviders 的日志定义配置，命名为 `jsonlog`，日志输出到 `/dev/null`， 日志输出格式为 `JSON`。
更多配置可以参考istio extensionProviders的相关配置。
```yaml
apiVersion: v1
data:
  mesh: |-
    extensionProviders:
    - name: jsonlog
      envoyFileAccessLog:
        path: /dev/stdout
        logFormat:
          labels: {}
```
在 Telemtry 中使用该 provider 配置
```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
  - providers:
    - name: jsonlog
```


** 参考文档 **
- [Global Mesh Options](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig)
- [Envoy Access Logs](https://istio.io/latest/docs/tasks/observability/logs/access-log/)