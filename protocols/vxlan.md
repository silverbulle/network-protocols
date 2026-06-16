# VXLAN (Virtual eXtensible LAN) 详解

## 概述

| 项 | 说明 |
|----|------|
| **RFC** | 7348 (2014) |
| **全称** | Virtual eXtensible LAN |
| **类型** | L2 over L3 隧道封装（Overlay 技术） |
| **核心作用** | 在 L3 网络之上构建虚拟 L2 网络，把 VLAN 的 12 bit（4094）扩展到 24 bit（1600 万） |
| **典型场景** | 数据中心多租户、虚拟化、跨 Pod/跨机房 L2 延伸、云网络 |
| **封装** | 原始 L2 帧 + VXLAN 头 (8B) + 外层 UDP + 外层 IP + 外层以太网 |

**为什么需要 VXLAN？**

```
传统 VLAN 的问题:
  1. VLAN ID 仅 12 bit → 最多 4094 个 → 不够多租户
  2. STP 阻塞链路 → 链路利用率低
  3. MAC 表规模受限 → 大二层广播域扛不住
  4. L2 延伸跨机房困难 → STP/MSTP 限制

VXLAN 解决:
  1. VNI 24 bit → 1600 万个虚拟网络
  2. 底层用 L3 路由 (ECMP) → 链路充分利用
  3. VTEP 只在边缘学 MAC → 核心设备只看外层 IP
  4. 隧道穿越 L3 → 跨机房/跨 Pod 无压力
```

## 协议栈位置

```
┌────────────────────────────────────────────────────┐
│ 虚拟机 / 容器 (Guest OS)                            │
│   └─ 内层 Ethernet Frame (虚拟机的原始帧)          │
└──────────────────────┬─────────────────────────────┘
                       ▼
┌────────────────────────────────────────────────────┐
│ VXLAN 封装层 (VTEP 添加)                            │
│   ┌────────────┐                                    │
│   │ VXLAN 头 8B│                                    │  ← 新增 8 字节
│   └────────────┘                                    │
└──────────────────────┬─────────────────────────────┘
                       ▼
┌────────────────────────────────────────────────────┐
│ 外层传输 (Underlay)                                 │
│   外层 UDP (8B) + 外层 IP (20B) + 外层 ETH (14B)   │
└──────────────────────┬─────────────────────────────┘
                       ▼
┌────────────────────────────────────────────────────┐
│ 物理网络 (Underlay / Spine-Leaf)                    │
│   仅按外层 IP/UDP 路由 → 完全不知道内层是 L2 帧     │
└────────────────────────────────────────────────────┘
```

> VXLAN **不是严格的 L2 也不是 L3**，而是 **Overlay 在 L3 之上的 L2 虚拟层**。
> 物理网络（Underlay）只看到 UDP 流量，按 L3 路由。

## 报文格式详解

### 完整封装结构
```
┌──────────────┬──────────┬──────────┬────────┬──────────────┬────────────┬──────────────┬──────────┐
│ Outer ETH    │ Outer IP │ Outer UDP│ VXLAN  │ Inner ETH    │ Inner IP   │ Inner TCP/UDP│ Inner    │
│ 14B          │ 20B      │ 8B       │ 8B     │ 14B          │ 20B        │ 可变         │ Payload  │
└──────────────┴──────────┴──────────┴────────┴──────────────┴────────────┴──────────────┴──────────┘
└─ Underlay 可见 ────────────────┘  └─── VXLAN 隧道 ───┘  └─── Overlay (虚拟 L2) ───────────────────┘
                                    └────────────── 租户的原始帧 ─────────────────────────────────────┘
```

**总头部开销：14 + 20 + 8 + 8 + 14 = 64 字节**（不含 FCS），MTU 必须扩大（典型 1600-9000）

### 各字段详解

#### 外层以太网头 (14B)
| 字段 | 长度 | 说明 |
|------|------|------|
| Outer DMAC | 6B | 下一跳（下一台 Spine/Leaf 或 VTEP）的 MAC |
| Outer SMAC | 6B | 当前 VTEP 出接口的 MAC |
| EtherType | 2B | 0x0800 (IPv4) / 0x86DD (IPv6) |

#### 外层 IP 头 (20B+)
| 字段 | 说明 |
|------|------|
| Outer SIP | **源 VTEP IP**（VTEP 的 Loopback 或 VTEP 接口 IP） |
| Outer DIP | **目的 VTEP IP**（对端 VTEP，由 BUM 或单播决定） |
| Protocol | 17 (UDP) |
| DSCP | 通常复制内层 DSCP（保持 QoS） |
| DF | 通常置 1（不分片） |
| TTL | 通常置 64 |

#### 外层 UDP 头 (8B)
| 字段 | 长度 | 说明 |
|------|------|------|
| Outer Src Port | 16b | **基于内层帧头部哈希生成**（用于 ECMP 负载均衡） |
| Outer Dst Port | 16b | **4789**（IANA 分配给 VXLAN 的官方端口） |
| Length | 16b | UDP 总长度 |
| Checksum | 16b | IPv4 可置 0，IPv6 强制 |

> **源端口随机化的妙处**：内层同一对 VM 的所有流量，外层 UDP 源端口不同，Spine 按 5-tuple 做 ECMP 时能分散到不同链路，避免单流瓶颈

#### VXLAN 头 (8B)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|R|R|R|R|I|R|R|R|            Reserved                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                VXLAN Network Identifier (VNI) |   Reserved    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| Flags (8b) | 1B | 第 4 位 **I 标志位 = 1 表示 VNI 有效**（其他位保留，必须为 0） |
| Reserved (24b) | 3B | 保留，发 0 收忽略 |
| **VNI** | **24 bit** | **虚拟网络标识，1600 万个租户隔离** |
| Reserved (8b) | 1B | 保留，发 0 收忽略 |

> VNI 类比 VLAN ID：VLAN 12 bit = 4094，VNI 24 bit = 16,777,216

#### 内层以太网帧
- 租户 / 虚拟机的原始帧（含内层 DMAC/SMAC/EtherType/Payload）
- VTEP 不修改内层帧（透传）

### 完整字节布局示例
```
假设 VM-A (10.0.0.1, MAC-A, VNI=100) → VM-B (10.0.0.2, MAC-B)
VTEP-A IP = 198.51.100.1, VTEP-B IP = 198.51.100.2

抓到的物理包 (从左到右):
┌─Outer ETH─┬─Outer IP───────────────┬─UDP──────┬─VXLAN─┬─Inner ETH──────┬─Inner IP──────┬─Data──┐
│DMAC=Spine │SIP=198.51.100.1        │S=哈希    │I=1    │DMAC=MAC-B      │SIP=10.0.0.1    │HTTP..│
│SMAC=VTEP-A│DIP=198.51.100.2        │D=4789    │VNI=100│SMAC=MAC-A      │DIP=10.0.0.2    │      │
│Type=0x0800│Proto=17, TTL=64, DF=1  │Len=..    │       │Type=0x0800     │Proto=6 (TCP)   │      │
└───────────┴─────────────────────────┴──────────┴───────┴────────────────┴────────────────┴──────┘
```

## 核心组件

### 三大概念

| 组件 | 英文 | 说明 |
|------|------|------|
| **VTEP** | VXLAN Tunnel End Point | VXLAN 隧道端点，负责封装/解封装 |
| **NVE** | Network Virtualization Edge | 网络虚拟化边缘设备（VTEP 所在设备） |
| **VNI** | VXLAN Network Identifier | 24 bit 租户标识，对应一个虚拟 L2 域 |

### VTEP 类型
```
硬件 VTEP:
  - 物理交换机（如 Leaf 节点）
  - 线速封装，性能高
  - 典型: Cisco Nexus 9K、Arista 7050、华为 CE12800

软件 VTEP:
  - Hypervisor 内核模块
  - 封装在 vSwitch 中完成
  - 典型: Linux kernel vxlan、VMware NSX、Open vSwitch (OVS)、Windows Hyper-V

混合型:
  - Host 跑软件 VTEP (OVS)
  - 接入物理 Leaf 做硬件 VTEP（Service Leaf 模式）
```

### VTEP 内部结构
```
┌───────────────────────────────┐
│ VTEP                          │
│                               │
│  ┌────────────┐ ┌──────────┐  │
│  │ VNI→VRF/BD │ │ MAC 表   │  │   BD = Bridge Domain (类似 VLAN)
│  │ 映射表     │ │ VNI+MAC→ │  │
│  └────────────┘ │  VTEP IP │  │
│                 └──────────┘  │
│  ┌────────────┐ ┌──────────┐  │
│  │ VTEP IP    │ │ 封装/    │  │
│  │ (Loopback) │ │ 解封装   │  │
│  └────────────┘ └──────────┘  │
└───────────────────────────────┘
```

## 实现逻辑

### 1. 数据面转发流程

#### 单播转发（VM-A → VM-B，同 VNI）
```
VM-A (Leaf-1)                         VM-B (Leaf-2)
  │                                      ▲
  │ (1) 内层帧: DMAC=MAC-B              │ (6) 解封后转发给 VM-B
  ▼                                      │
┌────────┐                          ┌────────┐
│ VTEP-1 │                          │ VTEP-2 │
└────┬───┘                          └────┬───┘
     │ (2) 查 MAC 表:                    │
     │    MAC-B → VTEP-2 IP              │
     │                                   │
     │ (3) 封装:                         │
     │    外层 SIP=VTEP-1 IP             │
     │    外层 DIP=VTEP-2 IP             │
     │    UDP dst=4789, VNI=100          │
     │                                   │
     │──── Underlay L3 路由 ────────────→│
     │    (Spine 只看外层 IP，做 ECMP)   │ (4) 收到 UDP 4789
     │                                   │ (5) 检查 VNI=100
     │                                   │    解封出内层帧
```

#### VTEP MAC 学习（数据面 Flood-and-Learn）
```
初始状态: VTEP-1 的 MAC 表是空的（不知道 MAC-B 在哪）

Step 1: VM-A 发帧给 MAC-B (未知单播)
  → VTEP-1 找不到 MAC-B → 触发 BUM 泛洪
  → 封装多份 VXLAN，发给 VNI=100 的所有其他 VTEP (VTEP-2, VTEP-3, ...)

Step 2: VTEP-2 收到 BUM 帧
  → 解封后在内层源 MAC = MAC-B 的 VTEP 是 VTEP-1（从外层 SIP 学）
  → 学习: MAC-B (本地) 在本机，MAC-A → VTEP-1 IP

Step 3: VTEP-2 转发给 VM-B，VM-B 回复
  → VTEP-2 已知道 MAC-A → VTEP-1，单播封装
  → VTEP-1 收到，学习 MAC-B → VTEP-2 IP

Step 4: 后续流量单播直达，不再泛洪
```

### 2. BUM 流量处理

BUM = Broadcast / Unknown Unicast / Multicast

#### 方式 A：多播 Underlay（早期方案）
```
每个 VNI 映射到一个 Underlay 多播组（如 239.1.1.100）
VTEP 加入该多播组
BUM 帧外层 DIP = 多播地址
物理网络用 PIM 做多播分发

优点: 物理网络高效分发
缺点: 需要 Underlay 支持多播（配置复杂、Spine 状态多）
```

#### 方式 B：头端复制 HER（Head-End Replication，主流）
```
VTEP 维护"VNI 的所有远端 VTEP 列表"
BUM 帧复制 N 份，每份外层 DIP = 一个远端 VTEP IP
全部以单播发送

优点: Underlay 不需要多播（纯 L3 即可）
缺点: VTEP 出方向流量 N 倍放大（小网络可接受）

典型实现:
  VNI=100 有 10 个 VTEP
  VM-A 发 ARP 广播
  VTEP-1 复制 9 份（每份给一个其他 VTEP）
  每个 VTEP 收到后解封，本地泛洪给对应 VNI 的所有 VM
```

### 3. 控制面演进

| 控制面 | 机制 | 现状 |
|--------|------|------|
| **Flood-and-Learn** | 数据面泛洪学习 MAC（无控制面） | 早期，小网络 |
| **OpenFlow / OVSDB** | SDN 控制器下发流表 | OpenStack Neutron 早期 |
| **BGP EVPN (RFC 7432)** | BGP 分发 MAC/IP/VTEP 信息 | **主流（现代 DC）** |

#### BGP EVPN 详解

```
EVPN = 用 BGP 做 VXLAN 的控制面

核心思想:
  把 VTEP 学到的 MAC/IP 绑定关系，通过 BGP 通告给其他 VTEP
  其他 VTEP 直接写入 MAC 表，无需数据面泛洪学习

EVPN Route Types:
  Type 1: Ethernet Auto-Discovery (EAD)    → 发现同一 ES 的多个 PE
  Type 2: MAC/IP Advertisement             → 通告 MAC-IP-VTEP 绑定（核心）
  Type 3: Inclusive Multicast              → 通告 BUM 复制列表（HER 用）
  Type 4: Ethernet Segment                 → 多归接入
  Type 5: IP Prefix                        → 通告 IP 前缀（跨子网用）

典型流程 (Type 2):
  VM-A 上线 → VTEP-1 学 MAC-A (本地)
  VTEP-1 通过 BGP EVPN 发 Type 2 路由:
     "MAC-A, IP-A → VTEP-1 IP, VNI=100"
  所有其他 VTEP (Spine-RR 反射) 收到并写 MAC 表
  后续 VM-X → VM-A 的帧: VTEP-X 直接知道 VTEP-1，单播封装

EVPN 优势:
  - 无 BUM 泛洪学习 → 降低广播风暴风险
  - MAC 移动检测更快
  - 支持跨子网路由（Type 5）
  - 支持多归接入（Type 1/4，Active-Active/Active-Standby）
  - ARP/ND 抑制（VTEP 本地应答，不转发广播）
```

### 4. 跨子网通信（VXLAN Routing）

```
同 VNI 内的 VM: L2 转发（VXLAN 隧道）
跨 VNI / 跨子网的 VM: 需要 L3 路由

两种实现:

方式 A: 集中式网关 (Centralized Gateway)
  ┌────┐   VXLAN   ┌─────────────┐   VXLAN   ┌────┐
  │VM-A│◄─────────►│ Border Leaf │◄─────────►│VM-C│
  │VNI │           │ (VXLAN GW)  │           │VNI │
  │=100│           │ VNI=100↔200 │           │=200│
  └────┘           └─────────────┘           └────┘
                        ▲
                    路由在 GW 上做
  缺点: GW 成瓶颈，流量三角路由

方式 B: 分布式网关 (Distributed Gateway, 主流)
  每个 Leaf 都是 VXLAN Gateway
  VM-A (VNI=100) → 默认网关 → 本地 VTEP 路由 → 封装到 VNI=200 → VM-C
  流量直达，无瓶颈
  典型实现: BGP EVPN Type 5 (IP Prefix Route)
```

## MTU 考量

```
VXLAN 增加 50 字节头部 (14+20+8+8 = 50, 不含内层 FCS):
  原始 MTU 1500 + 50 = 1550 (Underlay 最小 MTU)

推荐 Underlay MTU:
  - Jumbo Frame 9216 (Leaf/Spine 互联)
  - 至少 1600 (避免分片)

分片策略:
  - 内层帧过大时:
    (a) VTEP 分片外层 IP (DF=0) → 性能差
    (b) VTEP 发 ICMP Frag Needed 给 VM → 触发 PMTUD，VM 降 MTU (DF=1)
  - 现代方案: (b) 为主，强制内层 PMTUD
```

## VXLAN vs 同类技术对比

| 技术 | 封装 | 标识 | 控制面 | 端口 | 现状 |
|------|------|------|--------|------|------|
| **VLAN (802.1Q)** | L2 内嵌 4B | VID 12b | STP/MSTP | — | 小规模 LAN |
| **VXLAN** | L2 over UDP | VNI 24b | EVPN / F&L | UDP 4789 | **DC 主流** |
| **NVGRE (Microsoft)** | L2 over GRE | VSID 24b | 控制面弱 | GRE | 淘汰（Azure 已转 VXLAN） |
| **STT (VMware)** | L2 over 类 TCP | 64b context | — | 类 TCP | 淘汰 |
| **Geneve (VMware+其他)** | L2 over UDP | VNI 24b + TLV 扩展 | 灵活 | UDP 6081 | OVN/K8s CNI 用 |
| **GUE (Google)** | UDP 直装 IP | 无租户 ID | — | UDP | Google 内部 |
| **MPLS / VPLS** | L2.5 标签 | Label 20b | LDP/BGP | — | 运营商 WAN |
| **EVPN + MPLS** | L2 over MPLS | Label + ESI | BGP EVPN | — | 运营商 DCI |
| **SRv6** | IPv6 扩展头 | SID | IGP/BGP | — | 新兴 |

### VXLAN vs VLAN
| 维度 | VLAN | VXLAN |
|------|------|-------|
| 租户数 | 4094 | 16M |
| 扩展范围 | L2 物理范围 | 跨 L3，全球 |
| Underlay | L2 (STP) | L3 (ECMP, 高利用) |
| MAC 表 | 全网设备学 | VTEP 只学边缘 |
| 多路径 | STP 阻塞 | Underlay ECMP |
| 部署 | 简单 | 复杂（需 VTEP + 控制面） |

## 典型部署场景

### 场景 1：数据中心多租户
```
租户 A (VNI=100): VM-A1, VM-A2, VM-A3 (跨多个 Leaf)
租户 B (VNI=200): VM-B1, VM-B2

         ┌─────────┐     ┌─────────┐
         │ Spine-1 │     │ Spine-2 │   Underlay (纯 L3, ECMP)
         └────┬────┘     └────┬────┘
              │                │
     ┌────────┼────────────────┼────────┐
     │        │                │        │
┌────┴────┐ ┌─┴──────┐  ┌─────┴─┐ ┌────┴──┐
│ Leaf-1  │ │ Leaf-2 │  │Leaf-3 │ │Leaf-4 │   Leaf (VTEP)
│(VTEP-1) │ │(VTEP-2)│  │(VTEP-3│ │(VTEP-4│
└────┬────┘ └───┬────┘  └───┬───┘ └───┬───┘
     │          │           │         │
    VM-A1      VM-A2       VM-B1     VM-B2
    VNI=100    VNI=100     VNI=200   VNI=200

  租户 A 的 VM 跨 Leaf-1/2 通过 VXLAN 隧道直连（同 VNI）
  租户 B 的流量完全隔离（不同 VNI）
  跨租户通信需通过 L3 网关（分布式或集中式）
```

### 场景 2：虚拟化 Host 直连（Host VTEP）
```
┌─────────────── Host-1 ─────────────┐   ┌────── Host-2 ──────┐
│ Hypervisor (KVM/ESXi)              │   │ Hypervisor         │
│ ┌──────────┐  ┌──────────┐         │   │ ┌──────────┐       │
│ │  VM-A    │  │  VM-B    │         │   │ │  VM-C    │       │
│ └────┬─────┘  └────┬─────┘         │   │ └────┬─────┘       │
│      │              │               │   │      │             │
│ ┌────▼──────────────▼────────────┐  │   │ ┌────▼────────────┐│
│ │ OVS (软件 VTEP)                │  │   │ │ OVS (VTEP)      ││
│ │ VNI=100, VNI=200               │  │   │ │ VNI=100         ││
│ └──────────────┬─────────────────┘  │   │ └──────┬──────────┘│
└────────────────┼───────────────────┘   └────────┼──────────┘
                 │                                │
              ToR Switch (纯 L3, 不感知 VXLAN)
                 │                                │
                 └──── Underlay IP ───────────────┘

  优点: VTEP 在 Host 内，物理网络只需纯 L3
  缺点: Host CPU 封装开销（可用 SR-IOV 卸载）
```

### 场景 3：跨数据中心 L2 延伸（DCI）
```
DC-West                         DC-East
┌──────────┐                    ┌──────────┐
│ Leaf     │                    │ Leaf     │
│ (VTEP)   │◄── DCI 链路 ─────►│ (VTEP)   │
│ VNI=100  │   VXLAN over WAN  │ VNI=100  │
└──────────┘                    └──────────┘
     ▲                               ▲
    VM-A1                           VM-A2

  VM-A1 和 VM-A2 跨两个物理 DC，但在同一 VNI (虚拟 L2)
  典型用于: 虚拟机热迁移 (vMotion/Live Migration)、双活 DC

  注意: 跨 DC 的 VXLAN 必须配合:
    - 独立的控制面 (BGP EVPN 跨 DC peer)
    - 优化的 BUM 处理（避免跨 DC 泛洪）
    - 多路径 (Anycast Gateway)
```

## 配置示例（Linux + iproute2）

```bash
# 创建 VXLAN 接口 (VNI=100, VTEP IP 198.51.100.1)
ip link add vxlan100 type vxlan id 100 \
    local 198.51.100.1 \
    dstport 4789 \
    dev eth0

# 把 VXLAN 接口加入 Linux 网桥
ip link add br100 type bridge
ip link set vxlan100 master br100
ip link set veth-vm-a master br100   # VM 的虚拟网卡

# 启动接口
ip link set vxlan100 up
ip link set br100 up

# 静态配置远端 VTEP (无控制面时)
bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 198.51.100.2
bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 198.51.100.3

# 或使用 BGP EVPN (配合 FRR / GoBGP)
# FRR 配置示例 (router bgp 65000 + address-family l2vpn evpn)
```

## 抓包示例（tcpdump / Wireshark）

```
tcpdump -i eth0 udp port 4789 -vv

21:30:15.123456 IP 198.51.100.1.51234 > 198.51.100.2.4789: VXLAN, flags [I], vni 100
  MAC-B > MAC-A, ethertype IPv4, length 100:
    10.0.0.2.80 > 10.0.0.1.54321: Flags [.], ack 513, win 65023

Wireshark 解码:
  Outer Ethernet
  Outer IP: 198.51.100.1 → 198.51.100.2
  Outer UDP: src 51234, dst 4789
  VXLAN:
    Flags: 0x08 (I=1, VNI valid)
    VNI: 100
  Inner Ethernet
  Inner IP: 10.0.0.2 → 10.0.0.1
  Inner TCP: 80 → 54321
```

## 常见问题与排障

| 现象 | 可能原因 | 排障 |
|------|---------|------|
| VM-A ping 不通 VM-B（同 VNI） | VTEP 间 Underlay 不通 | `ping <VTEP-B-IP>` from VTEP-A |
| 同上 | VNI 不匹配 | 检查两端 VNI 配置 |
| 同上 | 对端 VTEP 没学到本地 MAC | `bridge fdb show dev vxlan100` |
| 大文件传输卡死 | MTU 不匹配（Underlay < 1550） | `ping -M do -s 1472 <dst>` 测 PMTU |
| 流量走单路径（ECMP 不生效） | UDP 源端口不随机 | 检查 hash 配置 |
| BUM 流量放大严重 | HER 列表过大 | 转用多播 Underlay 或优化 EVPN |
| MAC 漂移告警 | VM 迁移后旧 VTEP 未清理 | EVPN MAC Mobility 序列号机制 |

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| 控制面 | [L3 路由协议 BGP](../osi/L3-network.md#4-路径向量路由协议) | BGP EVPN 是 VXLAN 的现代控制面 |
| Underlay 路由 | [tcpip/internet.md 路由协议](../tcpip/internet.md#路由协议) | Underlay 用 OSPF/IS-IS/BGP |
| 同类 L2.5 | [protocols/mpls.md](mpls.md) | MPLS VPLS 也做 L2VPN，运营商场景 |
| 以太网基础 | [tcpip/link.md](../tcpip/link.md) | 内层/外层都是以太网帧 |
| UDP 封装 | [protocols/udp.md](udp.md) | VXLAN 跑在 UDP 4789 |
| L2 协议 | [osi/L2-datalink.md](../osi/L2-datalink.md) | VLAN/STP/LACP 等传统 L2 |
| QoS 传递 | [protocols/ip.md DSCP](ip.md#dscp-与-ecn-原-tos-字节) | VXLAN 外层 DSCP 复制内层（保持 QoS） |
| 案例 | [案例5: 数据中心](../cases/05-datacenter-east-west.md) | VXLAN 在 DC 东西向流量中的应用 |
