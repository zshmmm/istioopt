# Istio 流量走向 - 数据层面

## 1. POD启动顺序

![pod启动顺序][pod启动顺序]

### 1.1 Istio-init 准备iptables规则
具体iptables规则通过如下方式查看
```bash
kubectl logs -n <namespace> <podname> -c istio-init
nsenter -t <pid> -n iptables -nvL -t nat
```
![istio-init-iptables][istio-init-iptables]

### 1.2 Istio-proxy 准备证书/启动envoy

/usr/local/bin/pilot-agent  启动 /usr/local/bin/envoy
**pilot-agent 的主要工作**
1. 生成 envoy Bootstrap 配置文件
2. 证书相关（准备证书、申请证书等）
3. envoy 守护进程

**envoy**
1. 应用xDS
2. 处理流量
3. 监听端口
    - 15001: Envoy的Virtual Outbound监听器，iptable会将服务发出的出向流量导入该端口中由Envoy进行处理
    - 15006: Envoy的Virtual Inbound监听器，iptable会将发到的入向流量导入该端口中由Envoy进行处
    - 15000: Envoy管理端口，该端口绑定在本地环回地址上，只能在Pod内访问
    - 15090：指向127.0.0.1：15000/stats/prometheus, 用于对外提供Envoy的性能统计指标

### 1.3 istio-proxy启动过程
![istio-proxy启动过程][istio-proxy启动过程]


## 2. 环境
为减少干扰，采用如下简单环境
![拓扑图][拓扑图]

查看服务监听端口
![server监听端口][server监听端口]
K8S 服务
![grpc-svc][grpc-svc]













[pod启动顺序]: ../images/pod启动顺序.jpeg
[istio-init-iptables]: ../images/istio-init-iptables.PNG
[istio-proxy启动过程]: ../images/istio-proxy启动个遇.jpeg
[拓扑图]: ../images/拓扑图.jpg
[server监听端口]: ../images/server监听端口.png
[grpc-svc]: ../images/grpc-svc.png