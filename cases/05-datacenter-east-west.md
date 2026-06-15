# 案例 5：同一数据中心内的东西向流量（微服务调用）

## 场景
K8s 集群中，前端 Pod (10.244.1.5) 调用后端微服务 Pod (10.244.2.8)，流量经过 Overlay 网络。

## 数据路径

```
[前端 Pod 10.244.1.5] (Node A: 192.168.1.10)
 │
 │ ① HTTP 请求
 │    GET http://user-service:8080/api/users/123
 │    DNS 解析 (CoreDNS) → 10.96.0.50 (ClusterIP)
 │    kube-proxy (iptables) DNAT → 10.244.2.8:8080 (后端 Pod)
 │
 │ ② Overlay 封装 (如 VXLAN / Calico)
 │    原始包:
 │      [TCP | HTTP Data]  Src: 10.244.1.5 → Dst: 10.244.2.8
 │
 │    VXLAN 封装后:
 │      [外层 ETH] [外层 IP: 192.168.1.10 → 192.168.1.20]
 │      [VXLAN Header] [内层 ETH] [内层 IP: 10.244.1.5 → 10.244.2.8]
 │      [TCP | HTTP Data]
 │      ※ Pod 网络是虚拟的，实际在宿主机物理网络上传输
 │
 │ ③ 物理网络传输
 │    Node A (192.168.1.10) → ToR 交换机 → Node B (192.168.1.20)
 │    ※ 交换机只看到外层 IP 和 MAC，不知道 Pod 网络
 │
 │ ④ Node B 解封
 │    去除 VXLAN 封装 → 取出原始包 → 转发到目标 Pod
 ▼
[后端 Pod 10.244.2.8]
```

## Overlay 双层封装

```
┌─── 物理网络 (Underlay) ──────────────────────────────────────┐
│                                                               │
│  外层 ETH  │  外层 IP        │ VXLAN │ 内层 ETH │ 内层 IP    │
│  Node MAC  │ 192.168.1.10   │Header │ Pod MAC  │ 10.244.1.5 │
│  → ToR MAC │ → 192.168.1.20 │       │          │ →10.244.2.8│
│            │                 │       │          │            │
│            │  交换机据此转发  │       │  Pod 网络据此路由      │
│            │  (Underlay)    │       │  (Overlay)           │
│                                                               │
│  [TCP Header | HTTP Request Data]                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

## 涉及的协议文档

| 协议 | 在本知识库中的位置 | 在此案例中的角色 |
|------|-------------------|-----------------|
| HTTP | [protocols/http.md](../protocols/http.md) | 微服务间 REST 调用 |
| DNS | [protocols/dns.md](../protocols/dns.md) | CoreDNS 解析 Service 名到 ClusterIP |
| TCP | [protocols/tcp.md](../protocols/tcp.md) | 微服务间可靠传输 |
| IP | [protocols/ip.md](../protocols/ip.md) | 双层 IP 封装（Overlay + Underlay） |
| ARP | [protocols/arp.md](../protocols/arp.md) | Underlay 网络中 Node 间 MAC 解析 |
| Ethernet | [tcpip/link.md](../tcpip/link.md) | 物理帧传输、交换机转发 |
