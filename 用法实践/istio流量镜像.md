# istio 流量镜像

Auther: @雄哥

## 1. 流量镜像原理

envoy 会将流量复制一份影子流量发到分支服务，和正常流量的区别是对于分支服务发送影子流量后不会处理其返回响应。envoy 通过对请求头的 host 值标记（镜像流量会在原流量的 host 上加上 `-shadow` 的后缀）来区分分支服务的影子流量和正常服务流量。

![istio流量镜像图][istio流量镜像图]

以上图为例，镜像流量的 host 是 http://myservice-test.mycompany.com，其将被修改为myservice-backend.company.com-shadow。（如果服务中有对请求头的host进行处理需要注意这点）

## 2. 跨集群流量镜像配置

基于网关层做流量镜像一般多是用于为预发布环境导入线上真实流量，所以多是跨集群中使用到。
这里以源集群 clusterA 和目标集群 clusterB 命名，主体请求在 clusterA，由 clusterA 网关将流量镜像复制到 clusterB，如下图：

![istio流量镜像拓扑]

### 2.1 流量镜像配置

#### 2.1.1 clusterA 配置

clusterA 作为流量请求入口，相应 vs 配置如下：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: clusterA
  namespace: A
spec:
  gateways:
  - istio-system/internal-gateway
  hosts:
  - clusterA.cn-bj.A.mycompany.com
  http:
  - mirror:
      host: clusterB.cn-bj.B-staging.mycompany.com  #转发到clusterB域名
      port:
        number: 8000
    mirrorPercent: 50  # 流量镜像比例（最高100）
    route:
    - destination:
        host: clusterA.A.svc.cluster.local
        port:
          number: 8000
```

**注意：当前镜像的 host 是一个外部域名，需要通过 ServiceEntry 资源将 host 发布到网格内**

> Destination indicates the network addressable service to which the request/connection will be sent after processing a routing rule. The destination.host should unambiguously refer to a service in the service registry. Istio’s service registry is composed of all the services found in the platform’s service registry (e.g., Kubernetes services, Consul services), as well as services declared through the ServiceEntry resource.

创建 ServiceEntry 资源

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: mirror-cluster-b
spec:
  hosts:
  - clusterB.cn-bj.B-staging.mycompany.com
  location: MESH_EXTERNAL  #表示该服务在网格外部
  ports:
  - number: 8000
    name: http80
    protocol: HTTP
  resolution: DNS
```

#### 2.1.2 clusterB 配置

gateway 配置

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: internal-gateway
  namespace: istio-system
spec:
  selector:
    istio: internal-gateway
  servers:
  - hosts:
    - clusterB.cn-bj.B-staging.mycompany.com
    port:
      name: http-svc
      number: 8000
      protocol: HTTP
```

通过查看网关日志， istio-ingressgateway 的 envoy 对目标请求做了转换，在设置 cluster B 的路由策略时应该设置为clusterB.cn-bj.B-staging.mycompany.com-shadow

```bash
 kubectl logs -f  -n istio-system istio-ingressgateway-internal-f5cbf5f84-nx45b --tail=10|grep shadow

 # 输入如下

 {"upstream_transport_failure_reason":null,
"bytes_received":12,
"downstream_remote_address":"10.214.95.235:44714",
"start_time":"2023-04-19T13:12:42.948Z",
"upstream_host":"10.214.70.154:8000",
"response_flags":"-",
"upstream_service_time":"26",
"upstream_local_address":"10.214.70.123:56438",
"upstream_cluster":"outbound|8000||clusterA.cn-bj.A.mycompany.com",
"authority":"clusterB.cn-bj.B-staging.mycompany.com-shadow:8000", # clusterB.cn-bj.B-staging.mycompany.com -> clusterB.cn-bj.B-staging.mycompany.com-shadow
"path":"/rcmd.music_channel.MusicSongKtv/GetKtvLikeSong",
"request_id":"96c01c84-e71c-4bf4-98c8-7e4c1d56c108",
"downstream_local_address":"10.214.70.123:8000",
"method":"POST","requested_server_name":null,
"response_code":200,
"bytes_sent":7848,
"x_forwarded_for":"10.214.151.211,10.214.136.131,10.214.95.235",
"protocol":"HTTP/2","duration":27,"route_name":null,"user_agent":"grpc-go/1.49.0-dev"}
```

VirtualService 配置

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: clusterB
  namespace: B
spec:
  gateways: 
  - istio-system/internal-gateway
  hosts:
  - clusterB.cn-bj.B-staging.mycompany.com-shadow
  http:
  - route:
    - destination:
        host: clusterB.B.svc.cluster.local
        port:
          number: 8000
```

## 3. 思考



## 4. 参考

1. [VirtualService Destination](https://istio.io/latest/docs/reference/config/networking/virtual-service/#Destination)
2. [istio Mirroring](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)


[istio流量镜像图]: /images/istio流量镜像图.jpeg
[istio流量镜像拓扑]: /images/istio%E6%B5%81%E9%87%8F%E9%95%9C%E5%83%8F%E6%B5%81%E9%87%8F%E6%8B%93%E6%89%91%E5%9B%BE.jpg