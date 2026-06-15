# 案例 1：用户访问异地负载均衡的 Web 服务

## 场景
北京用户访问 `https://www.example.com`，该站点使用 DNS 轮询实现北京/上海双机房负载均衡。

## 数据路径

```
[北京用户浏览器]
 │
 │ ① DNS 解析 www.example.com
 │    DNS Query (UDP:53) → 本地 DNS → 根服务器 → .com TLD → 权威 DNS
 │    DNS Reply: www.example.com → 39.156.10.1 (北京节点, TTL=30)
 │    ※ DNS 根据来源 IP 返回最近的节点 IP（智能 DNS / GSLB）
 │
 │ ② TCP 三次握手 → 39.156.10.1:443
 │    SYN → [家用路由器 NAT] → [运营商城域网] → [骨干网] → [北京机房 LB]
 │    SYN+ACK ← 北京机房负载均衡器 (LVS / Nginx)
 │    ACK →
 │    ※ 数据经过多次路由跳转，每跳都经过 ARP 解析下一跳 MAC
 │
 │ ③ TLS 1.3 握手
 │    ClientHello → LB → (LB 终止 TLS 或透传至后端)
 │    ← ServerHello + Certificate + Finished
 │    → Finished
 │    ※ TLS 工作在应用层与传输层之间（OSI 表示层概念）
 │
 │ ④ HTTP/2 GET 请求
 │    → LB 根据负载均衡算法选择后端服务器 10.0.1.50
 │    → LB 与后端建立内部 TCP 连接（或直接转发）
 │    → 后端 10.0.1.50 处理请求，返回 HTTP Response
 │    ← HTTP/2 Response (200 OK) 经 LB 回传
 │
 ▼
[浏览器渲染页面]
```

## 协议栈穿透
```
浏览器
 └→ HTTP/2 (应用层)
     └→ TLS 1.3 (加密层)
         └→ TCP (传输层，可靠+有序)
             └→ IP (网络层，公网路由 39.156.10.1)
                 └→ Ethernet/光纤 (链路层+物理层)
                     └→ 多次路由跳转（每跳 MAC 变化，IP 不变）
```

## 关键设备与协议交互

| 设备 | 作用 | 涉及的协议 |
|------|------|-----------|
| 家用路由器 | NAT 转换、DHCP 分配内网 IP | NAT, DHCP, ARP |
| 运营商 DNS | 递归解析、智能调度 | DNS (UDP) |
| 骨干路由器 | BGP 选路、长距离转发 | BGP, OSPF, IP |
| 负载均衡器 | 四层/七层流量分发 | TCP, HTTP, TLS |
| 后端服务器 | 业务处理 | HTTP/2, TCP |

## 涉及的协议文档

| 协议 | 在本知识库中的位置 | 在此案例中的角色 |
|------|-------------------|-----------------|
| DNS | [protocols/dns.md](../protocols/dns.md) | 智能解析，返回最近节点 IP |
| TCP | [protocols/tcp.md](../protocols/tcp.md) | 可靠传输，三次握手建连 |
| TLS | [protocols/http.md (HTTPS)](../protocols/http.md#https-http-over-tls) | 加密通信 |
| HTTP | [protocols/http.md](../protocols/http.md) | HTTP/2 多路复用请求 |
| IP | [protocols/ip.md](../protocols/ip.md) | 公网路由寻址 |
| ARP | [protocols/arp.md](../protocols/arp.md) | 每跳解析下一跳 MAC |
| BGP | [tcpip/internet.md (BGP)](../tcpip/internet.md#bgp-border-gateway-protocol) | 骨干网跨 AS 选路 |
