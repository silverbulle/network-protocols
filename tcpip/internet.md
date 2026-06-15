# TCP/IP - 网际层 (Internet Layer)

## 概述
网际层（也称互联网层）是 TCP/IP 模型的核心，负责将数据包从源主机跨网络路由到目的主机。

## IP 协议

### IPv4 关键机制

#### 地址结构
```
|←──── 网络号 ────→|←──── 主机号 ────→|
  32 bits total
```

#### 子网掩码与 CIDR
```
IP:      192.168.1.100
掩码:    255.255.255.0  (/24)
网络号:  192.168.1.0
主机号:  100
广播地址: 192.168.1.255
可用范围: 192.168.1.1 - 192.168.1.254
```

#### NAT (网络地址转换)
```
内网 192.168.1.x ──→ [NAT 路由器] ──→ 公网 IP
     私有地址          端口映射        公共地址

NAT 类型：
- 静态 NAT：一对一映射
- 动态 NAT：地址池映射
- PAT/NAPT：端口地址转换（最常见）
```

### IPv6 关键改进

| 特性 | IPv4 | IPv6 |
|------|------|------|
| 地址长度 | 32 bit | 128 bit |
| 地址数量 | ~43 亿 | 3.4×10³⁸ |
| 头部长度 | 20-60 字节 | 固定 40 字节 |
| 分片 | 路由器可分片 | 仅源端分片 |
| 校验和 | 头部有 | 无（依赖上层） |
| 自动配置 | DHCP | SLAAC + DHCPv6 |
| IPsec | 可选 | 内置支持 |

#### IPv6 地址表示
```
完整: 2001:0db8:0000:0000:0000:0000:0000:0001
压缩: 2001:db8::1

特殊地址:
::1          - 回环 (等同 127.0.0.1)
::           - 未指定 (等同 0.0.0.0)
fe80::/10    - 链路本地
fc00::/7     - 唯一本地 (私有)
ff00::/8     - 多播
```

## 路由协议

### IGP vs EGP

```
              ┌── AS 1 ─────────────────────┐
              │ IGP: OSPF / IS-IS / EIGRP   │
              │ 负责内部路由，关注收敛速度   │
              └──┬──────────────────────────┘
                 │ eBGP
              ┌──┴─ AS 2 ───────────────────┐
              │ iBGP 全互联 / Route Reflector│
              │ EGP: BGP                     │
              │ 负责 AS 间路由，关注策略控制 │
              └─────────────────────────────┘
```

| 类型 | 协议 | 算法 | 关注点 |
|------|------|------|--------|
| **IGP** | OSPF, IS-IS, RIP, EIGRP | SPF / BF / DUAL | 快速收敛、最优路径 |
| **EGP** | BGP | 路径向量 | 策略控制、可扩展性 |

### 内部网关协议 (IGP)

#### OSPF (Open Shortest Path First)
- **类型**：链路状态
- **算法**：Dijkstra SPF
- **度量**：代价（基于带宽）
- **区域**：支持层次化设计（Area 0 为骨干区域）
- **Hello 间隔**：10s（广播网）/ 30s（NBMA）
- **特点**：企业网最主流 IGP，VLSM/CIDR 支持好，认证完善

```
        Area 0 (骨干)
       /      |      \
   Area 1   Area 2   Area 3
   (ABR)    (ABR)    (ABR)
```

#### IS-IS (Intermediate System to Intermediate System)
- **类型**：链路状态（OSI 协议栈，直接跑在 L2 之上）
- **算法**：Dijkstra SPF
- **层次**：Level 1（区域内）/ Level 2（区域间）/ L1/L2
- **度量**：默认 10，wide metric 可扩展到 24 bit
- **特点**：**运营商骨干首选**、TLV 扩展灵活（SR-MPLS/SRv6/TE 都是加 TLV）、协议不依赖 IP 栈更安全
- **Hello 间隔**：10s（广播/点对点）

> 详细实现逻辑与 OSPF 对比见 [osi/L3-network.md 路由协议详解](../osi/L3-network.md#2-链路状态路由协议)

#### RIP (Routing Information Protocol)
- **类型**：距离向量
- **度量**：跳数（最大 15）
- **更新间隔**：30s
- **版本**：RIPv1（有类）/ RIPv2（无类，支持 VLSM）/ RIPng（IPv6）
- **现状**：遗留网络 / 教学用，收敛慢、规模受限，现代生产网很少使用

#### EIGRP (Cisco 私有)
- **类型**：高级距离向量（DUAL 算法）
- **度量**：带宽 + 延迟（可选负载、可靠性）
- **特点**：收敛快（有可行后继时亚秒级）、支持不等价负载均衡、仅 Cisco 设备

### 外部网关协议 (EGP)

#### BGP (Border Gateway Protocol)
- **类型**：路径向量
- **AS**：自治系统，每个 AS 有唯一编号 (ASN)
- **选路**：基于策略（Local Pref、AS Path、MED 等）
- **邻居**：手动配置，使用 TCP 179
- **类型**：eBGP（跨 AS）/ iBGP（同 AS，需全互联或 Route Reflector）

```
  AS 100 ←──── eBGP ────→ AS 200
   │                        │
  iBGP                    iBGP
   │                        │
  Router ←──── eBGP ────→ Router
```

### 路由协议对比总览

| 协议 | 类型 | 算法 | 度量 | 收敛 | 典型场景 |
|------|------|------|------|------|----------|
| **RIP** | IGP 距离向量 | Bellman-Ford | 跳数 | 分钟 | 教学/遗留 |
| **OSPF** | IGP 链路状态 | Dijkstra | Cost (BW) | 秒级 | 企业网 |
| **IS-IS** | IGP 链路状态 | Dijkstra | 默认 10 / wide | 秒级 | ISP 骨干 |
| **EIGRP** | IGP 混合 | DUAL | BW+Delay | 亚秒 | Cisco 网 |
| **BGP** | EGP 路径向量 | 策略决策 | 多属性 | 分钟 | 互联网/DC |

> 详细实现逻辑、协议角色定位、现代演进（SR/BGP-LS）见 [osi/L3-network.md 路由协议详解](../osi/L3-network.md#路由协议详解)

## ICMP (Internet Control Message Protocol)

### 消息类型
| 类型 | 代码 | 说明 |
|------|------|------|
| 0 | 0 | Echo Reply (ping 响应) |
| 3 | 0-15 | 目标不可达（网络/主机/端口/协议等） |
| 5 | 0 | 重定向 |
| 8 | 0 | Echo Request (ping 请求) |
| 11 | 0 | TTL 超时 (traceroute 利用此特性) |
| 12 | 0 | 参数问题 |

### traceroute 原理
```
发送 TTL=1 的包 → 第一跳路由器 TTL 减为 0 → 返回 ICMP TTL Exceeded
发送 TTL=2 的包 → 第二跳返回 ICMP TTL Exceeded
发送 TTL=3 的包 → 第三跳返回 ICMP TTL Exceeded
...直到目标
```

## ARP (Address Resolution Protocol)

### 工作流程
```
1. 主机 A 要发包给 192.168.1.100
2. 查 ARP 缓存，无记录
3. 广播 ARP Request: "Who has 192.168.1.100?"
4. 192.168.1.100 单播 ARP Reply: "I am at 00:1A:2B:3C:4D:5E"
5. 主机 A 缓存 IP → MAC 映射
```

### ARP 缓存
```
$ arp -a
Interface: 192.168.1.10
  Internet Address      Physical Address      Type
  192.168.1.1           00-1a-2b-3c-4d-5e     dynamic
  192.168.1.100         aa-bb-cc-dd-ee-ff     dynamic
```

### ARP 安全问题
- **ARP 欺骗/投毒**：攻击者发送虚假 ARP 响应，劫持流量
- **防御**：动态 ARP 检测 (DAI)、静态 ARP 绑定

## MPLS (多协议标签交换)

MPLS 工作在 IP 层与链路层之间（Layer 2.5），是运营商骨干网的核心转发技术。

### 核心概念
```
传统 IP 转发:  每跳查路由表 (最长前缀匹配)
MPLS 转发:     入口压标签 → 中间换标签 → 出口弹标签 (精确匹配，更快)

封装位置:
  [ETH Header] [MPLS Label: 4B] [IP Header] [TCP/UDP] [Data]
                 ↑ 插入在 L2 和 L3 之间
```

### 主要应用
| 应用 | 说明 |
|------|------|
| **MPLS VPN** | L3VPN (双层标签 + MP-BGP) / L2VPN (VPWS/VPLS) |
| **流量工程** | RSVP-TE 指定路径、预留带宽、快速重路由 |
| **快速转发** | 标签精确匹配比 IP 最长前缀匹配更高效 |

> 详见 [protocols/mpls.md](../protocols/mpls.md)

## QoS (服务质量)

IP 层的 QoS 主要通过 DSCP/ECN 字段实现，分为两种服务模型：

### DiffServ (区分服务)

```
DiffServ = 在 IP 头部标记优先级，路由器按标记做队列调度

  终端:  设置 DSCP = EF (语音)
          │
  路由器: 根据 DSCP 值分配不同队列
          ├── EF  → LLQ (低延迟队列，优先转发)
          ├── AF  → 保证带宽队列
          └── BE  → 尽力而为队列 (可能被丢弃)
```

| 模型 | 说明 | 可扩展性 |
|------|------|---------|
| **DiffServ** | 逐跳按 DSCP 标记做差异化处理 | 好（主流方案） |
| **IntServ** | 端到端资源预留 (RSVP) | 差（状态太多） |

> DSCP/ECN 字段详解见 [protocols/ip.md DSCP 与 ECN](../protocols/ip.md#dscp-与-ecn-原-tos-字节)
> QoS 跨层字段总览见 [README QoS 章节](../README.md#qos-在各层的体现)

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| OSI 对应 | [L3 网络层](../osi/L3-network.md) | 网际层 = OSI 网络层 |
| 上层 | [transport.md](transport.md) | 传输层段封装在 IP 数据报中 |
| 下层 | [link.md](link.md) | IP 包封装在链路层帧中，每跳需 ARP 解析 |
| IP 详解 | [protocols/ip.md](../protocols/ip.md) | IPv4/IPv6 报文格式、分片、地址体系 |
| ICMP | [protocols/icmp.md](../protocols/icmp.md) | 直接封装在 IP 中（Protocol=1），不经过 TCP/UDP |
| ARP | [protocols/arp.md](../protocols/arp.md) | L2/L3 桥梁，IP 发包前解析下一跳 MAC |
| MPLS | [protocols/mpls.md](../protocols/mpls.md) | Layer 2.5，在 IP 和链路层之间插入标签，运营商骨干核心 |
| IPsec | [protocols/encryption-layers.md L3](../protocols/encryption-layers.md#l3-网络层加密) | 网络层加密：ESP 传输模式/隧道模式、IKEv2、WireGuard |
| 路由协议详解 | [osi/L3-network.md 路由协议详解](../osi/L3-network.md#路由协议详解) | OSPF/IS-IS/RIP/EIGRP/BGP 实现逻辑、角色定位、协议对比 |
| 案例 | [案例2: VPN](../cases/02-vpn-remote-access.md) | IPSec 双层 IP 封装与隧道传输 |
