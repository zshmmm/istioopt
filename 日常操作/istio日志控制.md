# 日志控制
---

1. 全局日志控制

    通过 istio 全局配置调整，增加日志输出目录和日志格式（TEXT，JSON）
    ```yaml
    apiVersion: v1
    data:
      mesh: |-
        accessLogFile: /dev/stdout
        accessLogEncoding: JSON
    ```
