# MPLS 协议详解

## 概述
MPLS (Multiprotocol Label Switching, 多协议标签交换) 是一种数据转发技术，通过在数据包前插入短小的**标签**来指导转发，取代传统的逐跳 IP 路由查找。它工作在 **OSI L2 与 L3 之间**，常被称为 **"Layer 2.5"** 协议。

### 协议栈位置与关联

```
应用层    HTTP  DNS  DHCP  ...
传输层    TCP   UDP
            │    │
            ▼    ▼
网络层   ┌──────────────────┐
         │     IP (v4/v6)   │
         └────────┬─────────┘
                  │
Layer 2.5 ┌──────┴──────┐
          │    MPLS      │  ← 本文件
          │  标签交换层    │     在 IP 和链路层之间插入标签
          └──────┬───────┘
                 │
链路层          ▼
         Ethernet / PPP / 帧中继 / ATM
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 承载 IP 流量 | [IP](ip.md) | MPLS 最常见的用途是承载 IP 数据包 |
| 上层协议 | [tcpip/internet.md](../tcpip/internet.md) | IP 路由与 MPLS 标签交换的协作 |
| 下层承载 | [tcpip/link.md](../tcpip/link.md) | MPLS 标签栈插入在 L2 帧头和 L3 IP 头之间 |
| VPN 场景 | [案例2](../cases/02-vpn-remote-access.md) | IPSec VPN vs MPLS VPN 对比 |
| 数据中心 | [案例5](../cases/05-datacenter-east-west.md) | Overlay 网络与 MPLS 的关系 |
| OSI 层 | [L2-datalink](../osi/L2-datalink.md) / [L3-network](../osi/L3-network.md) | MPLS 跨越 L2/L3 边界 |
| TCP/IP 层 | [tcpip/internet.md](../tcpip/internet.md) | 工程上归入网际层讨论 |

---

## 为什么需要 MPLS

### 传统 IP 路由的问题

```
传统 IP 逐跳路由:
  每个路由器都要:
    1. 检查目的 IP
    2. 查最长前缀匹配 (Longest Prefix Match)
    3. 查找路由表
    → 慢，且每跳都做同样的工作

  路由器A → 路由器B → 路由器C → 路由器D
   查路由表    查路由表    查路由表    查路由表
   (重复劳动)  (重复劳动)  (重复劳动)  (重复劳动)
```

### MPLS 的解决思路

```
MPLS 标签交换:
  入口路由器 (LER):
    1. 查一次路由表
    2. 分配标签，压入标签栈
    → 后续路由器只看标签，不查 IP 路由表

  LER ──→ LSR ──→ LSR ──→ LER
  查路由表   换标签    换标签   弹出标签
  压标签     (快速)    (快速)   查路由表
           ↑                     ↑
        标签交换比 IP 查找快得多
```

### 核心优势

| 优势 | 说明 |
|------|------|
| **快速转发** | 标签查找（精确匹配）比 IP 最长前缀匹配快 |
| **流量工程** | 可以指定路径，不依赖 IGP 最短路径 |
| **VPN 隔离** | 用标签实现 L3VPN / L2VPN，无需加密 |
| **QoS 保障** | 标签中 EXP 字段支持优先级标记 |
| **快速重路由** | 链路故障时预计算备份路径，切换 <50ms |

---

## MPLS 标签结构

### 标签栈条目 (Label Stack Entry)

MPLS 在 L2 帧头和 L3 IP 头之间插入一个 4 字节的 **Shim Header**：

```
┌──────────┬──────────┬──────────────────────────────────────────────┐
│ Ethernet │ MPLS     │  IP Header                                   │
│ Header   │ Shim     │  + TCP/UDP + Application Data                │
│          │ Header   │                                              │
│ 14 bytes │ 4 bytes  │  (原来的完整 IP 包)                           │
└──────────┴──────────┴──────────────────────────────────────────────┘
```

### 4 字节标签结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Label (20 bits)              | EXP |S|  TTL  |
|                                                | (3) |1|  (8)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| **Label** | 20 bit | 标签值，0-1048575。实际转发依据此值 |
| **EXP** | 3 bit | 实验位，现用于 QoS/CoS（优先级 0-7） |
| **S** | 1 bit | 栈底标志。1=最后一个标签，0=后面还有标签 |
| **TTL** | 8 bit | 生存时间，类似 IP TTL，每跳减 1 |

### 特殊标签值

| 标签值 | 含义 | 说明 |
|--------|------|------|
| 0 | IPv4 Explicit NULL | 弹出标签后按 IPv4 转发 |
| 1 | Router Alert | 路由器需检查包内容（类似 IP 的 Router Alert） |
| 2 | IPv6 Explicit NULL | 弹出标签后按 IPv6 转发 |
| 3 | Implicit NULL | 倒数第二跳弹出（PHP），不出现在实际包中 |
| 4-15 | 保留 | |

---

## MPLS 网络架构

### 角色定义

```
                  MPLS 域 (MPLS Domain)
          ┌──────────────────────────────────┐
          │                                  │
   CE ──→ LER ──→ LSR ──→ LSR ──→ LER ──→ CE
  (用户)  (入口)   (中间)   (中间)   (出口)  (用户)
          │                                  │
          └──────────────────────────────────┘

CE  = Customer Edge        用户边缘设备（不感知 MPLS）
LER = Label Edge Router    标签边缘路由器（入口/出口）
LSR = Label Switch Router  标签交换路由器（中间节点）
```

### 转发过程

```
                        标签操作示意

[CE-A] ──IP包──→ [LER-1] ──标签包──→ [LSR-2] ──标签包──→ [LSR-3] ──标签包──→ [LER-4] ──IP包──→ [CE-B]
                  Push 100            Swap 100→200        Swap 200→300         Pop 300
                  (压入)              (交换)              (交换)              (弹出)
```

### 三种标签操作

| 操作 | 英文名 | 执行者 | 说明 |
|------|--------|--------|------|
| **压入** | Push | LER (入口) | 在 IP 包前插入 MPLS 标签 |
| **交换** | Swap | LSR (中间) | 替换标签值（入标签→出标签） |
| **弹出** | Pop | LER (出口) | 移除 MPLS 标签，恢复 IP 包 |

---

## MPLS 转发机制

### FEC (Forwarding Equivalence Class)

```
FEC = 转发等价类
  → 一组具有相同转发处理方式的数据包

常见 FEC 分类依据:
  - 目的 IP 前缀 (最常见)
  - 源 IP + 目的 IP
  - 应用端口号
  - DSCP/TOS 值

同一个 FEC 的所有包走相同路径 (LSP)
```

### LSP (Label Switched Path)

```
LSP = 标签交换路径
  → MPLS 网络中从入口到出口的单向路径

LSP 建立过程:
  1. 路由协议 (OSPF/IS-IS) 发现网络拓扑
  2. 标签分发协议 (LDP/RSVP-TE) 分配标签
  3. 每个 LSR 建立标签映射表
  4. 数据沿 LSP 转发

FEC: 10.0.0.0/8 的 LSP:
  LER-1 (Push:100) → LSR-2 (Swap:100→200) → LSR-3 (Swap:200→300) → LER-4 (Pop)
```

### 标签转发表

每个 LSR 维护一张标签转发表 (LFIB):

```
┌───────────┬───────────┬───────────┬───────────┐
│ 入标签     │ 出标签     │ 出接口     │ 下一跳     │
├───────────┼───────────┼───────────┼───────────┤
│ 100       │ 200       │ Eth0/1    │ 10.1.1.2  │
│ 200       │ 300       │ Eth0/2    │ 10.1.2.3  │
│ 300       │ Pop       │ Eth0/3    │ 10.1.3.4  │
│ 400       │ Swap→500  │ Eth0/1    │ 10.1.1.2  │
└───────────┴───────────┴───────────┴───────────┘

转发过程:
  收到标签=100 的包 → 换为 200 → 从 Eth0/1 发出
  收到标签=300 的包 → 弹出标签 → 从 Eth0/3 发出 (恢复 IP 转发)
```

---

## 标签分发协议

### LDP (Label Distribution Protocol)

```
LDP = 基于 IGP 路由表自动分配标签
  → 简单、自动、无需手动配置
  → 但只能沿 IGP 最短路径建立 LSP

工作原理:
  1. LSR 之间建立 LDP 会话 (TCP 646)
  2. 每个 LSR 根据路由表为每个 FEC 分配标签
  3. 通过 LDP 消息交换标签绑定

LDP 消息类型:
  - Hello: UDP 646, 发现邻居
  - Initialization: TCP 646, 建立会话
  - Label Mapping: 标签绑定通告
  - Label Request: 请求标签
  - Label Withdraw: 撤回标签
  - Label Release: 释放标签
```

### RSVP-TE (Resource Reservation Protocol - Traffic Engineering)

```
RSVP-TE = 支持流量工程的标签分发
  → 可以指定路径、预留带宽
  → 比 LDP 复杂但更灵活

工作原理:
  1. 入口 LER 计算端到端路径 (CSPF)
  2. 发送 PATH 消息沿路径预留资源
  3. 出口 LER 回复 RESV 消息确认
  4. 沿途每个 LSR 分配标签并预留带宽

RSVP-TE 扩展:
  - 显式路由 (Explicit Route): 指定经过的节点
  - 带宽预留 (Bandwidth Reservation): 保证带宽
  - 优先级 (Setup/Hold Priority): 抢占机制
  - 快速重路由 (Fast Reroute): 链路保护
```

### LDP vs RSVP-TE

| 特性 | LDP | RSVP-TE |
|------|-----|---------|
| 路径选择 | 跟随 IGP 最短路径 | 可指定路径 (CSPF) |
| 带宽预留 | 不支持 | 支持 |
| 复杂度 | 低（自动） | 高（需规划） |
| 适用场景 | 普通 MPLS 转发 | 流量工程、MPLS-TE |
| 会话协议 | TCP 646 | 直接使用 IP (Protocol 46) |
| 状态维护 | 较少 | 较多（每 LSP 状态） |

---

## PHP (Penultimate Hop Popping)

```
PHP = 倒数第二跳弹出

不使用 PHP:
  LSR-2 ──标签300──→ LSR-3 ──标签300──→ LER-4
  (Swap)             (Swap)            (Pop → 查IP)
                     ↑                  ↑
                   多余操作          两次查找

使用 PHP (默认行为):
  LSR-2 ──标签300──→ LSR-3 ──IP包──→ LER-4
  (Swap)             (Pop)          (只查IP)
                     ↑
                  倒数第二跳就弹出
                  出口路由器只需一次查找

标签 3 (Implicit NULL):
  出口 LER 通告 Implicit NULL 给上游
  → 上游 LSR 知道自己是倒数第二跳，直接弹出
```

---

## MPLS VPN

MPLS VPN 是运营商网络最重要的应用之一，分为 L3VPN 和 L2VPN。

### L3VPN (Layer 3 VPN)

```
L3VPN = MPLS + BGP + VRF
  → 每个客户一个 VRF (Virtual Routing and Forwarding)
  → 使用双层标签: 外层(传输) + 内层(VPN标识)

架构:
  [CE-A] ──→ [PE-1] ──MPLS骨干──→ [PE-2] ──→ [CE-B]
             (VRF-A)                (VRF-A)
              ↑                       ↑
           同一 VPN 的 PE 通过 MP-BGP 交换路由
```

#### L3VPN 标签封装

```
原始 IP 包:
  Src: 192.168.1.10 (客户A)  Dst: 192.168.2.20 (客户A)

PE-1 封装后 (双层标签):
  ┌─────────┬─────────┬─────────────────────────────────────┐
  │ 外层标签 │ 内层标签 │ 原始 IP 包                           │
  │ (传输)   │ (VPN)   │ Src:192.168.1.10 Dst:192.168.2.20  │
  │ Label=100│Label=500│                                     │
  └─────────┴─────────┴─────────────────────────────────────┘
       ↑          ↑
    骨干网 P     PE-2 据此
    路由器据此    知道是哪个 VRF
    转发到 PE-2
```

#### L3VPN 转发流程

```
① CE-A 发送 IP 包 → PE-1
② PE-1 查 VRF-A 路由表 → 匹配到 PE-2 的 Loopback
③ PE-1 压入内层标签 (VPN 标签，由 MP-BGP 分配)
④ PE-1 压入外层标签 (传输标签，由 LDP/RSVP-TE 分配)
⑤ P 路由器根据外层标签转发到 PE-2
⑥ PE-2 弹出外层标签 (PHP)
⑦ PE-2 根据内层标签确定 VRF → 查 VRF-A 路由表 → 转发到 CE-B
```

### L2VPN (Layer 2 VPN)

```
L2VPN = 在 MPLS 骨干上传输二层帧
  → 客户感觉 PE 之间是直接的二层链路

两种主要类型:

VPWS (Virtual Private Wire Service):
  点对点 L2 连接
  [CE-A] ──帧──→ [PE-1] ──MPLS──→ [PE-2] ──帧──→ [CE-B]
                 (伪线 PW)

VPLS (Virtual Private LAN Service):
  多点 L2 连接 (模拟 LAN)
  [CE-A] ──→ [PE-1] ─┐
  [CE-B] ──→ [PE-2] ─┼── MPLS 全网状伪线 ──→ 模拟一个交换机
  [CE-C] ──→ [PE-3] ─┘
```

### L3VPN vs L2VPN

| 特性 | L3VPN | L2VPN (VPWS/VPLS) |
|------|-------|-------|
| 层级 | L3 (IP) | L2 (帧) |
| PE 角色 | 参与客户路由 | 透明传输帧 |
| 路由 | PE 维护客户路由 | CE 自己处理路由 |
| 扩展性 | 好 (MP-BGP) | VPLS 全连接开销大 |
| 典型用途 | 企业 WAN、多站点互联 | 运营商互联、透明 LAN 延伸 |

---

## MPLS 流量工程 (MPLS-TE)

### 为什么需要 TE

```
无 TE 的问题:
  IGP 最短路径可能导致某些链路过载，其他链路空闲

  A ─── 10G ─── B ─── 10G ─── D
   \                              /
    └──────── 100G ──────────┘

  IGP 可能把 A→D 流量全走上面的 10G 链路 (跳数少)
  而下面的 100G 链路空闲

MPLS-TE 的解决:
  显式指定 A→D 走下面 100G 链路
  → 带宽利用率更高
  → 避免拥塞
```

### TE 组件

```
┌─────────────────────────────────────────────────────────┐
│                    MPLS-TE 组件                          │
│                                                          │
│  CSPF (Constrained SPF):                                 │
│    带约束的最短路径计算                                   │
│    约束: 带宽、跳数、亲和位、排除节点                      │
│                                                          │
│  RSVP-TE:                                                │
│    沿 CSPF 计算的路径建立 LSP                             │
│    预留带宽、分配标签                                     │
│                                                          │
│  TE 数据库 (TED):                                        │
│    存储链路属性: 带宽、TE metric、亲和位                  │
│    通过 OSPF-TE / IS-IS-TE 泛洪                          │
│                                                          │
│  Fast Reroute (FRR):                                     │
│    预计算备份路径                                         │
│    链路故障时 <50ms 切换                                  │
│    两种模式: Link Protection / Node Protection           │
└─────────────────────────────────────────────────────────┘
```

### Fast Reroute (快速重路由)

```
Link Protection:
  主路径: A ──→ B ──→ C ──→ D
  B-C 链路故障:
  备份:   B ──→ E ──→ C (预先建立)
  切换时间: <50ms (不依赖 RSVP-TE 信令)

Node Protection:
  主路径: A ──→ B ──→ C ──→ D
  C 节点故障:
  备份:   B ──→ E ──→ D (绕过 C)
  更健壮，但需要更多备份 LSP
```

---

## Segment Routing (SR-MPLS)

### 传统 MPLS vs Segment Routing

```
传统 MPLS:
  需要 LDP/RSVP-TE 逐跳建立标签映射
  → 信令复杂、状态多

Segment Routing:
  源节点编码完整路径为一段标签序列 (Segment List)
  → 无需 LDP/RSVP-TE 信令
  → 中间节点无状态

传统: 每个 LSP 需要信令维护
SR:   路径编码在包头中，中间节点无需状态
```

### SR 基本概念

```
Segment = 一条转发指令
  - Prefix SID: 转发到特定节点 (全局唯一)
  - Adjacency SID: 经过特定链路 (本地有效)

Segment List 示例:
  源 LER 编码路径: [Prefix-SID-B, Adj-SID-B-C, Prefix-SID-D]
  → 先到 B，再走 B-C 链路，最后到 D

SR-MPLS 数据面:
  仍然使用 MPLS 标签转发
  但控制面由 IGP 扩展 (OSPF/IS-IS SR 扩展) 取代 LDP/RSVP-TE
```

---

## MPLS 在实际网络中的位置

### 运营商网络

```
┌─────────────────────────────────────────────────────────────┐
│                    运营商 MPLS 骨干网                         │
│                                                              │
│  [PE-1] ──┐                           ┌── [PE-3]            │
│           │    ┌─── P ─── P ───┐      │                     │
│  [PE-2] ──┼───┤   (核心 P 路由) ├──┼── [PE-4]              │
│           │    └─── P ─── P ───┘      │                     │
│  [PE-5] ──┘                           └── [PE-6]            │
│                                                              │
│  PE = Provider Edge (面向客户)                                │
│  P  = Provider (核心转发，不感知客户路由)                      │
│                                                              │
│  PE 之间: MPLS LSP + MP-BGP                                  │
│  PE-CE: BGP / 静态路由 / OSPF                                 │
└─────────────────────────────────────────────────────────────┘
```

### MPLS 与其他隧道技术对比

| 特性 | MPLS | VXLAN | GRE/IPSec | SRv6 |
|------|------|-------|-----------|------|
| 层级 | L2.5 | L2 over UDP | L3 | L3 |
| 标签/标识 | 20-bit Label | 24-bit VNI | - | SRv6 SID |
| 典型场景 | 运营商 WAN | 数据中心 Overlay | 站点互联 | 下一代骨干网 |
| 信令 | LDP/RSVP-TE | 控制器/BGP | 静态 | IGP/BGP |
| 转发效率 | 高 | 中（封装开销） | 低 | 高 |
| 状态 | 有状态 | 少状态 | 无状态 | 无/少状态 |

---

## 安全考虑

### MPLS 安全特性

| 方面 | 说明 |
|------|------|
| **天然隔离** | MPLS VPN 通过标签隔离，不同 VPN 流量互不可见 |
| **无加密** | MPLS 本身不加密数据，安全性依赖物理网络隔离 |
| **控制面安全** | LDP 使用 TCP MD5 / GTSM 保护；RSVP-TE 使用 IPsec |
| **数据面安全** | 标签不可伪造（需物理接入 MPLS 域） |

### 风险

- **标签欺骗**：攻击者接入 MPLS 域后注入伪造标签
- **DoS**：大量 LSP 建立请求耗尽 LSR 资源
- **管理平面**：网管通道暴露可导致配置篡改

---

## MPLS 关键 RFC

| RFC | 标题 | 说明 |
|-----|------|------|
| RFC 3031 | MPLS Architecture | MPLS 架构定义 |
| RFC 3032 | MPLS Label Stack Encoding | 标签栈编码 |
| RFC 5036 | LDP Specification | LDP 协议规范 |
| RFC 3209 | RSVP-TE | RSVP-TE 扩展 |
| RFC 4364 | BGP/MPLS IP VPN | L3VPN (原 RFC 2547) |
| RFC 4762 | VPLS | L2VPN VPLS |
| RFC 8402 | Segment Routing Architecture | SR 架构 |
| RFC 8660 | Segment Routing MPLS Data Plane | SR-MPLS 数据面 |
