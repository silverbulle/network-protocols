# 案例 2：通过 VPN 访问公司内网资源

## 场景
员工在家通过 IPSec VPN 连接公司内网，访问内网 Git 服务器 `10.10.0.5`。

## 数据路径

```
[员工笔记本] (家庭网络 192.168.1.x)
 │
 │ ① VPN 隧道建立 (IPSec)
 │    Phase 1: IKE (UDP:500) — 身份认证 + 密钥协商
 │      Client (公网IP: 223.x.x.x) ↔ VPN Gateway (公司公网IP: 120.x.x.x)
 │    Phase 2: 建立 IPSec SA (ESP 隧道)
 │      ※ 此后所有内网流量被加密封装在 ESP 报文中
 │
 │ ② 访问 git clone git@10.10.0.5
 │    原始数据包:
 │      [TCP Header | SSH Data]
 │      Src: 10.20.0.100 (VPN 虚拟IP)  Dst: 10.10.0.5 (内网 Git)
 │
 │    IPSec 封装后 (隧道模式):
 │      [新 IP Header | ESP Header | 原 IP Header | TCP | SSH | ESP Trailer]
 │      Src: 223.x.x.x (员工公网IP)   Dst: 120.x.x.x (VPN 网关公网IP)
 │      ※ 外层 IP 用于公网路由，内层 IP 是真正的源/目的
 │
 │ ③ 数据路径
 │    笔记本 → 家庭路由器(NAT) → ISP → 互联网 → 公司防火墙
 │      → 公司 VPN 网关(解封 ESP) → 公司核心交换机
 │      → 内网 Git 服务器 10.10.0.5
 │
 │ ④ 响应返回 (反向路径)
 │    Git 服务器 → VPN 网关(ESP 封装) → 互联网 → 笔记本(解封)
 ▼
```

## 双层封装示意
```
                     ┌─── 公网可见 ──────────────────────────────┐
                     │                                           │
[外层 IP: 公网路由]   │  [ESP Header] [内层 IP: 10.20→10.10]     │
Src: 223.x.x.x      │  (加密)       [TCP | SSH | Git Data]      │
Dst: 120.x.x.x      │               (原始内网包，被加密保护)      │
                     │                                           │
                     └───────────────────────────────────────────┘

 中间路由器只看到外层 IP，不知道内部通信的是 10.20.0.100 ↔ 10.10.0.5
 VPN 网关解封后，取出内层包转发到内网
```

## 关键协议协作

| 阶段 | 协议 | 说明 |
|------|------|------|
| 密钥协商 | IKE (UDP:500/4500) | Diffie-Hellman 密钥交换 |
| 数据加密 | ESP (IP Protocol 50) | 封装安全载荷，加密+认证 |
| NAT 穿透 | UDP Encapsulation (4500) | ESP 包在 UDP 中穿越 NAT |
| 内网路由 | OSPF / 静态路由 | VPN 网关到 Git 服务器的内网路由 |

## 涉及的协议文档

| 协议 | 在本知识库中的位置 | 在此案例中的角色 |
|------|-------------------|-----------------|
| IP | [protocols/ip.md](../protocols/ip.md) | 双层 IP 封装（外层公网路由 + 内层虚拟网络） |
| UDP | [protocols/udp.md](../protocols/udp.md) | IKE 密钥协商 + NAT 穿透 |
| TCP | [protocols/tcp.md](../protocols/tcp.md) | SSH 连接 Git 的可靠传输 |
| ARP | [protocols/arp.md](../protocols/arp.md) | 家庭路由器到 ISP 的 MAC 解析 |
| IPsec 加密层级 | [protocols/encryption-layers.md L3](../protocols/encryption-layers.md#l3-网络层加密) | IPsec 传输/隧道模式、IKEv2 完整协商流程 |
