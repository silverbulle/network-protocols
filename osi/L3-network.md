# L3 - 网络层 (Network Layer)

## 职责
负责数据包的逻辑寻址、路由选择和转发，实现跨网络的主机到主机通信。

## 核心功能

### 1. 逻辑寻址
- IP 地址标识网络中的设备
- IPv4：32 位，如 192.168.1.1
- IPv6：128 位，如 2001:0db8::1
- 子网划分与 CIDR 表示法

### 2. 路由选择
- 确定数据包从源到目的的最佳路径
- 路由算法：距离向量、链路状态、路径向量

### 3. 数据包转发
- 根据路由表将包从输入接口转到输出接口
- TTL（生存时间）防止环路

### 4. 分片与重组
- 当数据包大于链路 MTU 时分片
- IPv4：路由器可分片；IPv6：仅源端可分片

## 主要协议

| 协议 | 说明 |
|------|------|
| IPv4 | 互联网协议 v4，32 位地址 |
| IPv6 | 互联网协议 v6，128 位地址 |
| ICMP | 控制消息协议（ping、traceroute） |
| IGMP | 组管理协议（多播） |
| OSPF | 开放最短路径优先（IGP，链路状态，企业网主流） |
| IS-IS | 中间系统到中间系统（IGP，链路状态，运营商骨干主流） |
| RIP | 路由信息协议（IGP，距离向量，遗留/教学用） |
| EIGRP | 增强型内部网关路由协议（IGP，Cisco 私有，DUAL 算法） |
| BGP | 边界网关协议（EGP，路径向量，互联网核心） |
| ARP | 地址解析协议（IP → MAC） |

## IPv4 数据报格式
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |    DSCP   |ECN|         Total Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |        Header Checksum        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if IHL > 5)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Data                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段说明**：
- **Version (4bit)**：4 = IPv4
- **IHL (4bit)**：头部长度（单位：4字节），最小 5（即 20 字节）
- **TTL (8bit)**：每经过一个路由器减 1，到 0 丢弃
- **Protocol (8bit)**：上层协议号（6=TCP, 17=UDP, 1=ICMP）
- **Flags (3bit)**：DF（不分片）、MF（更多分片）

## IPv6 数据报格式（简化）
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                       Source Address (128 bit)                |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                    Destination Address (128 bit)              |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## 路由协议详解

### 协议分类视角

路由协议可从多个维度分类：

| 维度 | 分类 |
|------|------|
| **作用范围** | IGP（内部网关协议）vs EGP（外部网关协议） |
| **算法类型** | 距离向量 / 链路状态 / 路径向量 / 混合 |
| **收敛机制** | 逐跳 Bellman-Ford / SPF / 路径向量 |
| **协议栈位置** | 运行在 UDP (RIP) / 直接跑在 IP 上 (OSPF, IS-IS) / 运行在 TCP (BGP, EIGRP) |

```
                   ┌─────────────── AS（自治系统）──────────────┐
                   │                                           │
  AS 100           │  IGP：内部路由（OSPF/IS-IS/RIP/EIGRP）    │          AS 200
                   │  目标：快速收敛、计算最短路径              │
  ┌────────┐  eBGP │              ┌──────────┐ iBGP            │ eBGP  ┌────────┐
  │ Router │◄─────►│  iBGP Router│◄────────►│Router           │◄─────►│ Router │
  └────────┘       │              └──────────┘                 │       └────────┘
                   │                                           │
                   │  EGP：BGP（AS 间路由）                    │
                   │  目标：策略控制、可扩展性                  │
                   └───────────────────────────────────────────┘
```

### 1. 距离向量路由协议

核心思想：每个路由器只知道自己到目的网络的"距离"和"方向（下一跳）"，不掌握全网拓扑。"路由即谣言"——你告诉邻居你能到哪，邻居再加一跳传播出去。

#### RIP (Routing Information Protocol)

| 项 | 说明 |
|----|------|
| **RFC** | 1058 (RIPv1, 1988) / 2453 (RIPv2, 1998) |
| **运行位置** | UDP 520 |
| **算法** | Bellman-Ford |
| **度量** | 跳数（hop count），最大 15 |
| **更新周期** | 30 秒（整表广播） |
| **版本** | RIPv1（有类，广播）/ RIPv2（无类 CIDR，组播 224.0.0.9）/ RIPng（IPv6，UDP 521） |

**实现逻辑**：
```
初始化: 每个路由器 R 的路由表只包含直连网络，距离=0

循环 (每 30 秒):
  对每个邻居 N:
    1. 发送自己完整路由表（目的网络 + 距离）
    2. 收到 N 的路由表后，对每条记录 (D, cost):
       if cost + 1 < 16:
         新距离 = cost + 1
         if 新距离 < 已知距离:
           更新路由表：目的 D，下一跳 N，距离=新距离

触发更新: 链路变化时立即发送（不等 30s）

防环机制:
  - 最大跳数 16 = 不可达（水平分割的兜底）
  - 水平分割 (Split Horizon): 不从某接口发回从该接口学来的路由
  - 毒性反转 (Poison Reverse): 从某接口发回从该接口学来的路由时，距离标为 16
  - 抑制计时器 (Hold-down): 收到变差的路由，180s 内不接受更差的
  - 触发更新: 立即通告变化，不等周期
```

**收敛慢的原因**：坏消息传播慢（每次只跳一跳），全网收敛需分钟级。

#### EIGRP (Enhanced Interior Gateway Routing Protocol)

| 项 | 说明 |
|----|------|
| **厂商** | Cisco 私有（2013 年部分公开为 informational RFC 7868） |
| **运行位置** | IP 协议号 88（不跑在 TCP/UDP） |
| **算法** | DUAL（Diffusing Update Algorithm） |
| **度量** | 复合公式：带宽 + 延迟（默认），可扩展负载、可靠性 |
| **邻居发现** | Hello 组播 224.0.0.10 |

**实现逻辑**：
```
度量公式:
  metric = 256 × (K1·BW + K2·BW/(256-load) + K3·Delay)
           × (K5/(reliability + K4))
  默认 K1=K3=1, K2=K4=K5=0 → metric = 256 × (BW + Delay)

三张表:
  - 邻居表 (Neighbor Table): Hello 维护
  - 拓扑表 (Topology Table): 所有已知路径（含后继 + 可行后继）
  - 路由表 (Routing Table): 仅最优路径

DUAL 核心:
  - 后继 (Successor): 当前最优下一跳
  - 可行后继 (Feasible Successor, FS): 满足可行性条件的备份下一跳
    可行性条件: 邻居通告的距离 < 本路由器的 FD (Feasible Distance)
  - 主路径失效时:
    if 存在 FS: 直接切换到 FS（亚秒级收敛，无环路保证）
    else: 进入"活跃状态" (Active)，向所有邻居发 Query 重新计算
```

**特点**：收敛快（有备份路径时无需重算）、部分更新（只发变化）、支持不等价负载均衡（variance 命令）。

### 2. 链路状态路由协议

核心思想：每个路由器掌握全网拓扑（链路状态数据库 LSDB），各自独立运行 Dijkstra 算法计算最短路径树。"地图式路由"——每个人都有完整地图，自己算最短路线。

#### OSPF (Open Shortest Path First)

| 项 | 说明 |
|----|------|
| **RFC** | 2328 (OSPFv2, IPv4) / 5340 (OSPFv3, IPv6) |
| **运行位置** | 直接跑在 IP 上，协议号 89 |
| **算法** | Dijkstra SPF |
| **度量** | 代价 Cost（参考带宽 / 接口带宽，默认参考 100 Mbps） |
| **Hello 间隔** | 广播网 10s / NBMA 30s / 点对点 10s |
| **组播地址** | 224.0.0.5（所有 OSPF 路由器）/ 224.0.0.6（DR/BDR） |

**实现逻辑**：
```
1. 邻居发现与邻接建立
   Hello → 双向通信 → (广播/NBMA 网选举 DR/BDR) → 邻接
   状态机: Down → Init → 2-Way → ExStart → Exchange → Loading → Full

2. 链路状态通告 (LSA)
   类型 1: Router LSA      每个路由器发，描述自身链路
   类型 2: Network LSA     DR 发，描述广播网内所有路由器
   类型 3: Summary LSA     ABR 跨区域传播
   类型 4: ASBR Summary    指向 ASBR
   类型 5: External LSA    ASBR 发，描述外部路由
   类型 7: NSSA External   NSSA 区域内的外部路由

3. LSDB 同步
   泛洪 (flooding) + 序列号 + 校验和 + 老化时间 (3600s)
   每台路由器拥有完全一致的 LSDB

4. SPF 计算
   以自身为根，Dijkstra 构建最短路径树 → 路由表

5. 层次化区域设计
   Area 0 (骨干) 必须存在，所有非骨干区必须直连 Area 0
   ABR: 区域边界路由器；ASBR: 自治系统边界路由器
```

**网络类型与 DR/BDR**：
```
广播多接入 (Broadcast): 以太网 → 选举 DR/BDR，减少邻接关系 (n(n-1)/2 → 2n-3)
NBMA: Frame Relay / ATM → 需手动配邻居，选举 DR
点对点 (P2P): PPP / HDLC → 不选举
点对多点 (P2MP): 特殊配置
```

#### IS-IS (Intermediate System to Intermediate System)

| 项 | 说明 |
|----|------|
| **标准** | ISO 10589（原生为 CLNP 设计）/ RFC 1195（集成 IS-IS，支持 IP）/ RFC 5308（IPv6） |
| **运行位置** | **直接跑在数据链路层之上**（协议号不适用，EtherType 不封装，用 LLC/SNAP） |
| **算法** | Dijkstra SPF |
| **度量** | 默认每条链路 cost=10（可配 0-63，wide metric 0-16777215） |
| **Hello 间隔** | 广播网 10s / 点对点 10s |
| **组播地址** | L1: 01:80:C2:00:00:14 / L2: 01:80:C2:00:00:15 |

**为什么 IS-IS 是运营商首选**：
- 协议直接跑在 L2 上，不依赖 IP 栈 → 对 IP 网络攻击免疫，安全性更好
- TLV 编码 → 扩展性极强（SR-MPLS、BGP-LS、TE 都是加 TLV 实现）
- 大网络下扩展性好（路由器数量可达数千，OSPF 一般 < 1000）
- 收敛快，SPF 计算高效

**实现逻辑**：
```
1. 术语对照
   OSPF Router ID    → IS-IS System ID (6 字节)
   OSPF Area         → IS-IS Area (变长 1-13 字节，地址前缀)
   OSPF LSA          → IS-IS LSP (Link State PDU)
   OSPF LSDB         → IS-IS LSDB
   OSPF DR           → IS-IS DIS (Designated IS)
   OSPF ABR          → IS-IS L1/L2 IS

2. 两级层次结构
   Level 1 (L1): 区域内路由（类似 OSPF 非骨干区）
   Level 2 (L2): 区域间路由（类似 OSPF 骨干区，但 L2 可连续跨多区）
   L1/L2:        同时参与 L1 和 L2（类似 OSPF ABR）

   L1 路由器只知道本区域拓扑
   去往区域外的路由默认指向最近的 L1/L2 路由器（默认路由）

3. PDU 类型
   IIH (IS-IS Hello): 邻居发现
   LSP (Link State PDU): 链路状态通告
   CSNP (Complete SNP): 完整 LSDB 摘要（周期性/DR 发）
   PSNP (Partial SNP): 部分 LSDB 摘要（请求/确认）

4. 邻接与 DIS
   广播网: 选举 DIS（类似 DR），优先级高者胜（可抢占）
   点对点: 直接邻接，用三次握手确保可靠
   DIS 发 CSNP 同步 LSDB（每 10s），其他路由器用 PSNP 请求缺失的 LSP

5. SPF 与路由计算
   每个 Level 各自运行 SPF
   L1 路由 + L1/L2 泄露的 L2 路由 = 最终路由表
```

**与 OSPF 的关键差异**：

| 维度 | OSPF | IS-IS |
|------|------|-------|
| 协议栈位置 | IP 协议号 89 | L2 之上（OSI 栈） |
| 层次结构 | Area 0 为中心，非骨干必须连 Area 0 | L1/L2 两级，L2 可跨区连续 |
| 区域边界 | 在路由器上 (ABR) | 在链路上（同一路由器可属多个 Level） |
| 默认度量 | 基于带宽 | 固定 cost=10（可配 wide metric） |
| LSA/LSP 格式 | 固定字段 | TLV 可变长 |
| 扩展新特性 | 需 RFC 扩展 Opaque LSA | 加 TLV 即可（如 SR-MPLS） |
| 典型应用 | 企业网、园区网 | 运营商骨干（SP 网络） |
| DR 抢占 | DR 不抢占 | DIS 可抢占 |
| 收敛速度 | 快 | 通常更快（LSA 压缩更紧凑） |

### 3. 路径向量路由协议

#### BGP (Border Gateway Protocol)

| 项 | 说明 |
|----|------|
| **RFC** | 4271 (BGP-4) |
| **运行位置** | TCP 179（唯一使用 TCP 的路由协议） |
| **算法** | 路径向量 + 策略决策 |
| **路由携带** | NLRI (前缀) + 路径属性 (AS_PATH, NEXT_HOP, LOCAL_PREF, MED, COMMUNITY 等) |
| **邻居类型** | eBGP（不同 AS）/ iBGP（同一 AS） |

**实现逻辑**：
```
1. 邻居建立（4 种消息）
   OPEN → 协商参数（ASN、Hold time、Capabilities）
   KEEPALIVE → 维持连接
   UPDATE → 发布/撤销路由
   NOTIFICATION → 错误通告并断开

2. 路径属性分类
   公认强制: ORIGIN, AS_PATH, NEXT_HOP
   公认自决: LOCAL_PREF, ATOMIC_AGGREGATE
   可选过渡: AGGREGATOR, COMMUNITY
   可选非过渡: MED, ORIGINATOR_ID

3. 选路决策（BGP Best Path Selection，按顺序）
   1. Weight (Cisco 私有，本地最高)
   2. Local Preference (越大越好，iBGP 传播)
   3. 本地发起（network 或聚合）
   4. AS_PATH 长度（越短越好）
   5. Origin 类型 (IGP < EGP < Incomplete)
   6. MED (越小越好，eBGP 学来)
   7. eBGP 优先于 iBGP
   8. IGP 到 NEXT_HOP 的度量最小
   9. 若配了 maximum-paths，做负载均衡
   10. 最老的 eBGP 路由
   11. Router ID 最小
   12. Cluster List 最短
   13. Neighbor IP 最小

4. iBGP 全互联问题
   iBGP 学到的路由不再传给 iBGP 对等体（防环）
   → N 台路由器需 N(N-1)/2 条 iBGP 会话（不可扩展）

   解决方案:
   - Route Reflector (RR): 中心反射器打破全互联
   - Confederation: 把大 AS 分成多个小 AS（内部用小 eBGP）

5. 路由反射器 (RR)
   Client → RR: RR 反射给其他 Client + non-Client
   non-Client → RR: RR 反射给所有 Client
   Originator_ID / Cluster_List 防环
```

### 4. 各协议的角色定位

| 场景 | 首选协议 | 原因 |
|------|----------|------|
| **运营商骨干 (ISP)** | IS-IS + BGP | IS-IS 扩展性好、L2 安全、TLV 易扩展；BGP 做 AS 间策略路由 |
| **企业园区网** | OSPF | 标准化、配置友好、区域设计灵活 |
| **数据中心 (DC)** | BGP (eBGP in DC) / IS-IS | 大规模下 BGP ECMP 友好、收敛稳定；IS-IS 用于 VXLAN/SR |
| **小型网络 (< 50 台)** | 静态路由 / OSPF stub / EIGRP | 简单优先 |
| **家庭/SOHO** | 静态 + 默认路由 | 无需动态路由 |
| **Cisco 纯设备环境** | EIGRP | 收敛快，但厂商锁定 |
| **互联网核心** | BGP | 唯一可扩展到全球规模的协议 |
| **SDN / 流量工程** | IS-IS-TE + PCE / BGP-LS | TLV 扩展天然适合 TE |
| **Segment Routing** | IS-IS + SR 扩展 | 主流 SR-MPLS/SRv6 控制平面 |

### 5. 协议对比总览

| 特性 | RIP | OSPF | IS-IS | EIGRP | BGP |
|------|-----|------|-------|-------|-----|
| **算法** | Bellman-Ford | Dijkstra | Dijkstra | DUAL | 路径向量 |
| **类型** | 距离向量 | 链路状态 | 链路状态 | 混合 | 路径向量 |
| **范围** | IGP | IGP | IGP | IGP | EGP |
| **收敛** | 慢（分钟） | 秒级 | 秒级 | 亚秒 | 慢（分钟） |
| **规模** | 小（≤15 跳） | 中（~1000） | 大（数千） | 中 | 全球 |
| **协议栈** | UDP 520 | IP 89 | L2 之上 | IP 88 | TCP 179 |
| **度量** | 跳数 | Cost (BW) | 默认 10 | BW+Delay | 策略属性 |
| **VLSM/CIDR** | v2 支持 | 支持 | 支持 | 支持 | 支持 |
| **层次化** | 无 | Area 0 中心 | L1/L2 两级 | 无 | AS / Confederation |
| **认证** | v2 有 | MD5/SHA | TLV 内置 | MD5 | MD5/TCP-AO |
| **典型用途** | 教学/遗留 | 企业 | ISP 骨干 | Cisco 网 | 互联网 |

### 6. 现代演进

| 技术 | 说明 | 关联协议 |
|------|------|----------|
| **SR-MPLS** | Segment Routing，用源路由标签栈替代 LDP/RSVP-TE | IS-IS + SR 扩展 |
| **SRv6** | Segment Routing over IPv6，用 IPv6 扩展头实现 | IS-IS / OSPFv3 |
| **BGP-LS** | BGP 收集全网 TE 拓扑，供控制器/SDN 使用 | BGP |
| **BGP EVPN** | 数据中心 VXLAN 的控制平面 | BGP |
| **Babel** | 现代距离向量 IGP，适合 mesh/ad-hoc | RFC 8966 |
| **OpenFabric** | 基于 IS-IS 的数据中心路由 | 开源 |

> 详细协议报文格式见 [tcpip/internet.md 路由协议](../tcpip/internet.md#路由协议)
> OSPF、BGP、EIGRP 配置视角见 [tcpip/internet.md](../tcpip/internet.md#路由协议)

## IP 地址分类与子网

### IPv4 地址分类（传统）
| 类别 | 范围 | 默认掩码 | 网络/主机位 |
|------|------|----------|-------------|
| A | 1.0.0.0 - 126.255.255.255 | 255.0.0.0 | 8/24 |
| B | 128.0.0.0 - 191.255.255.255 | 255.255.0.0 | 16/16 |
| C | 192.0.0.0 - 223.255.255.255 | 255.255.255.0 | 24/8 |

### 私有地址
| 范围 | CIDR | 用途 |
|------|------|------|
| 10.0.0.0 - 10.255.255.255 | 10.0.0.0/8 | 大型企业内网 |
| 172.16.0.0 - 172.31.255.255 | 172.16.0.0/12 | 中型网络 |
| 192.168.0.0 - 192.168.255.255 | 192.168.0.0/16 | 小型网络/家庭 |

> 详见 [protocols/ip.md](../protocols/ip.md)、[protocols/icmp.md](../protocols/icmp.md)、[protocols/arp.md](../protocols/arp.md)

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| TCP/IP 对应 | [tcpip/internet.md](../tcpip/internet.md) | 网络层 = TCP/IP 网际层 |
| 上层 | [L4 传输层](L4-transport.md) | 传输层段封装在 IP 数据报中 |
| 下层 | [L2 数据链路层](L2-datalink.md) | IP 包封装在链路层帧中传输 |
| 核心协议 | [IP](../protocols/ip.md) · [ICMP](../protocols/icmp.md) · [ARP](../protocols/arp.md) | 网络层三大协议 |
| Layer 2.5 | [MPLS](../protocols/mpls.md) | 标签交换，工作在 L2/L3 之间，运营商骨干核心 |
| 路由协议 | [tcpip/internet.md 路由部分](../tcpip/internet.md#路由协议) | IGP/EGP 对比与配置视角 |
| IS-IS 详解 | [IS-IS 章节](#is-is-intermediate-system-to-intermediate-system) | 运营商骨干首选，L2 之上的链路状态协议 |
| 现代演进 | [现代演进章节](#6-现代演进) | SR-MPLS、SRv6、BGP-LS、BGP EVPN |
| 网络层加密 | [protocols/encryption-layers.md L3](../protocols/encryption-layers.md#l3-网络层加密) | IPsec 传输/隧道模式、WireGuard、IKEv2 协商流程 |
| 网络层差错 | [protocols/error-control.md L3](../protocols/error-control.md#l3-网络层--仅校验头部--icmp-错误报告) | IPv4 头部校验和、ICMP 错误报告、IPv6 移除头部校验 |
