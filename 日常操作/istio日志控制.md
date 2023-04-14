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

