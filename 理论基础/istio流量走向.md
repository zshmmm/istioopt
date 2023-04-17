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

/usr/local/bin/pilot-agent 启动 /usr/local/bin/envoy

**pilot-agent 的主要工作**
1. 生成 envoy Bootstrap 配置文件
2. 证书相关（准备证书、申请证书等）
3. envoy 守护进程

**envoy的主要工作**
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


## 3. xDS

### 3.1 术语

![xDS术语][xDS术语]

### 3.2 概念

xDS 是控制平面（Istio）和数据平面之间的通讯协议。DS：discovery service，x 表示具体的服务发现
- LDS (Listener Discovery Service) 
- CDS (Cluster Discovery Service)
- EDS (Endpoint Discovery Service) 
- RDS (Route Discovery Service) 
- *ADS聚合发现服务*(本文不做介绍)
后续以请求的视角从LDS - CDS - EDS - RDS 的路径介绍。

### 3.3 查看配置

```bash
# 获取istio-proxy全部配置
kubectl exec -it -n zhuzhu14 <podname> -c istio-proxy -- curl 127.0.0.1:15000/config_dump
# 获取初始化静态配置
kubectl exec -it -n zhuzhu14 <podname> -c istio-proxy cat /etc/istio/proxy/envoy-rev0.json
```

### 3.4 LDS
Envoy通过listener来接收并处理发过来的请求，包括出和入的流量

```bash
#查看
istioctl pc l -n zhuzhu14 <podname> --port 15001 -o json
```

#### 3.4.1 VirtualOutboundListener
Envoy监听15001端口，Iptables将Envoy所在Pod的对外请求拦截后发向本地的15001端口，该监听器接收后并不进行业务处理，而是根据请求的目的端口分发给其他监听器处理（透明代理：**SO_ORIGINAL_DST** ）。

```bash
#查看
istioctl pc l -n zhuzhu14 <podname> --port 15001 -o json
```

#### 3.4.2 OutboundListener
Envoy为网格中的外部服务端口创建多个Listener，用于处理出向请求。

```bash
#查看 8080 outboundlistener 
istioctl pc l -n zhuzhu14 <podname> --port 8080 -o json
```
监听器的名字：0.0.0.0_8080，匹配发向任意IP的8080的请求。匹配到http协议后通过filterChains来处理，并根据路由配置来跳转（**filterChains中匹配路由规则 routeConfigName:8080**）。如果协议为非http，则通过defaultFilterChain转发到PassthroughCluster。

#### 3.4.3 VirtualInboundListener
Envoy监听15006端口，Iptables将Envoy所在Pod的对内请求拦截后发向本地的15006端口，接受请求后采用一系列filterChain对入向请求进行处理，而不是像VirtualOutboundListener一样分发给其它独立的Listener进行处理。

```bash
#查看
istioctl pc l -n <namespace> --port 15006 -o json
```
**filterChains中匹配路由规则 routeConfigName:inbound|8080||**

#### 3.4.4 OutboundTrafficPolicy
在Enovy的配置中找不到和请求目的地端口的listener，则将会根据Istio的outboundTrafficPolicy全局配置选项进行处理。[配置](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy)

- 如果outboundTrafficPolicy设置为ALLOW_ANY：Mesh允许发向任何外部服务的请求，不管该服务是否在Pilot的服务注册表中。在该策略下，istiod将会在下发给Enovy的VirtualOutbound Listener加入一个upstream cluster为PassthroughCluster的TCP proxy filter，找不到匹配端口listener的请求会被该TCP proxy filter处理，请求将会被发送到其IP头中的原始目的地地址。
- 如果outboundTrafficPolicy设置为REGISTRY_ONLY：只允许请求istiod服务注册表中存在的服务的对外请求。在该策略下，Pilot将会在下发给Enovy的VirtualOutbound Listener加入一个upstream cluster为BlackHoleCluster的TCP proxy filter，找不到匹配端口listener的请求会被该TCP proxy filter处理，由于BlackHoleCluster中没有配置upstteam host，请求实际上会被丢弃。


### 3.5 RDS
Router: Envoy的路由规则。Istio下发的路由规则中对每个端口设置了一个路由规则，根据host来对请求进行路由转发。

```bash
#查看
istioctl pc r -n zhuzhu14 <podname>
```

在前面 **[OutboundListener](https://github.com/zshmmm/istioopt/blob/main/%E7%90%86%E8%AE%BA%E5%9F%BA%E7%A1%80/istio%E6%B5%81%E9%87%8F%E8%B5%B0%E5%90%91.md#342-outboundlistener)** 章节中有一个8080的 **routeConfigName**。

```bash
#查看
istioctl pc r -n zhuzhu14 <podname> --name 8080
```
8080路由包含一个配置转发到 cluster: outbound|8080||server-svc.grpc.svc.cluster.local
![server-svc.grpc.svc.cluster.local][server-svc.grpc.svc.cluster.local]

在 **[VirtualInboundListener](https://github.com/zshmmm/istioopt/blob/main/%E7%90%86%E8%AE%BA%E5%9F%BA%E7%A1%80/istio%E6%B5%81%E9%87%8F%E8%B5%B0%E5%90%91.md#343-virtualinboundlistener)** 章节中有一个 **routeConfigName:inbound|8080||** 的配置，查看这个Route

```bash
istioctl pc r -n grpc server-7bc7fd5479-xvv99 --name 'inbound|8080||' -o json
```

![inbound8080][inbound8080]

可以看到Cluster为：**inbound|8080||**




[pod启动顺序]: ../images/pod启动顺序.jpeg
[istio-init-iptables]: ../images/istio-init-iptables.PNG
[istio-proxy启动过程]: ../images/istio-proxy启动过程.jpeg
[拓扑图]: ../images/拓扑图.jpg
[server监听端口]: ../images/server监听端口.png
[grpc-svc]: ../images/grpc-svc.png
[xDS术语]: ../images/xDS%E6%9C%AF%E8%AF%AD.jpg
[server-svc.grpc.svc.cluster.local]: ../images/server-svc.grpc.svc.cluster.local.png
[inbound8080]: ../images/inbound8080.png
