# L2 - 数据链路层 (Data Link Layer)

## 职责
在物理层提供的比特流传输基础上，实现可靠的帧传输、物理寻址、差错检测和介质访问控制。

## 核心功能

### 1. 帧封装
- 将网络层的数据包封装为帧
- 添加帧头（源/目的 MAC 地址）和帧尾（FCS 校验）

### 2. 物理寻址
- 使用 MAC 地址（48 位）标识网络设备
- 格式：`XX:XX:XX:XX:XX:XX`（如 `00:1A:2B:3C:4D:5E`）

### 3. 介质访问控制（MAC）
- 控制多个设备共享同一传输介质
- CSMA/CD（以太网）、CSMA/CA（Wi-Fi）、令牌环

### 4. 差错检测
- CRC（循环冗余校验）检测帧传输错误
- 注意：只检测不纠正（纠正是传输层的职责）
- 以太网使用 CRC-32（FCS 字段），校验错帧静默丢弃

> 详见 [protocols/error-control.md L2](../protocols/error-control.md#l2-链路层--帧级检错--可选重传)

### 5. 流量控制
- 控制发送速率，防止接收方过载
- 简单场景使用停等协议或滑动窗口

## 两个子层
IEEE 802 标准将数据链路层分为：
- **LLC（逻辑链路控制）子层**：提供与网络层的统一接口
- **MAC（介质访问控制）子层**：控制对物理介质的访问

## 以太网帧格式（Ethernet II）
```
+--------+--------+----------+--------+------+-----+
| Preamble| Dest MAC | Src MAC | Type | Data | FCS |
| 8 bytes | 6 bytes | 6 bytes | 2B  |46-1500| 4B  |
+--------+--------+----------+--------+------+-----+
```

**关键字段说明**：
- **前导码 (Preamble)**：8 字节，用于时钟同步（7字节前导 + 1字节 SFD）
- **目的 MAC**：6 字节，目标设备的物理地址
- **源 MAC**：6 字节，发送设备的物理地址
- **类型 (EtherType)**：2 字节，标识上层协议（0x0800=IPv4, 0x0806=ARP, 0x86DD=IPv6）
- **数据 (Payload)**：46-1500 字节
- **FCS（帧校验序列）**：4 字节 CRC-32 校验

## CSMA/CD 工作原理（有线以太网）

> CD = Collision **Detection**（冲突检测）

```
1. 载波监听：发送前先检测信道是否空闲
2. 多路访问：多个设备可接入同一信道
3. 冲突检测：发送过程中持续监听是否发生冲突
4. 冲突处理：
   - 检测到冲突 → 发送 Jam 信号
   - 随机退避（二进制指数退避算法）
   - 重传
```

> 注：现代交换式以太网（全双工）已不再需要 CSMA/CD，冲突域被交换机隔离。

## CSMA/CA 工作原理（Wi-Fi / 802.11）

> CA = Collision **Avoidance**（冲突避免）

无线环境无法在发送的同时监听冲突（半双工），因此 Wi-Fi 用"避免"替代"检测"：

```
1. 载波监听：检测信道是否空闲（物理层 + 虚拟载波监听 NAV）
2. 空闲等待：信道空闲后等待 DIFS 时间
3. 随机退避：从竞争窗口中随机选一个退避时隙（Backoff Counter）
4. 计数发送：计数器减至 0 → 发送数据帧
5. 等待 ACK：接收方正确收到后回复 ACK
6. 冲突处理：未收到 ACK → 窗口翻倍 → 重新退避 → 重传
```

### 关键机制详解

```
虚拟载波监听 (NAV, Network Allocation Vector):
  每个帧头包含 Duration 字段 → 告知其他设备"信道将被占用多久"
  其他设备据此设置 NAV 计时器 → 计时器归零前不发送
  → 即使物理层没检测到信号，也认为信道忙

RTS/CTS 握手机制 (解决隐藏终端问题):
  A ──→ AP ←── B     (A 和 B 互相不可见，但都能到 AP)

  A 先发 RTS (Request to Send) → AP
  AP 回 CTS (Clear to Send)   → 所有能听到 AP 的设备
  B 收到 CTS → 知道信道被占用 → 等待

  隐藏终端: A 和 B 互相听不到，如果同时发就会在 AP 处冲突
  RTS/CTS 让"不可见"的设备也能知道信道状态
```

### CSMA/CD vs CSMA/CA 对比

| 特性 | CSMA/CD (有线) | CSMA/CA (无线) |
|------|---------------|---------------|
| 冲突策略 | 检测后处理 | 发送前避免 |
| 原因 | 有线可边发边听 | 无线半双工，无法边发边听 |
| 信道空闲后 | 直接发送 | 等待 DIFS + 随机退避 |
| 确认机制 | 无（冲突即失败） | 必须收到 ACK |
| 隐藏终端 | 不存在（有线共享介质） | 存在，用 RTS/CTS 解决 |
| 退避窗口 | 二进制指数退避 | 竞争窗口（CWmin → CWmax） |
| 现代状态 | 全双工交换网已弃用 | Wi-Fi 仍在使用 |

## 交换机工作原理
1. **学习**：记录源 MAC 地址与端口映射到 MAC 表
2. **转发**：根据目的 MAC 地址查表转发
3. **泛洪**：未知目的地址时向所有端口广播
4. **过滤**：同一端口收发不转发

## VLAN（虚拟局域网）
- 在交换机上逻辑划分广播域
- IEEE 802.1Q 标签格式（在普通以太网帧中插入 4 字节）：
```
普通帧:
| Dest MAC | Src MAC | EtherType (0x0800) | Data | FCS |

802.1Q 带标签帧:
| Dest MAC | Src MAC | 0x8100 |       TCI        | EtherType | Data | FCS |
                                     ↑ 4 字节
```

### TCI (Tag Control Information) 完整拆解

```
┌───────┬─────┬──────────────┐
│  PCP  │ DEI │     VID      │
│ 3 bit │1 bit│   12 bit     │
├───────┼─────┼──────────────┤
│优先级 │丢弃 │ VLAN ID      │
│ 0~7   │ eligible │ 0~4095  │
└───────┴─────┴──────────────┘
```

| 字段 | 位 | 说明 |
|------|----|------|
| **PCP** | 3 | Priority Code Point，L2 优先级（0=最低，7=最高），用于 QoS |
| **DEI** | 1 | Drop Eligible Indicator，1=拥塞时可优先丢弃 |
| **VID** | 12 | VLAN ID，0 和 4095 保留，可用范围 1~4094 |

#### PCP 优先级与典型用途

| PCP | 优先级 | 用途 | 对应 DSCP |
|-----|--------|------|-----------|
| 7 | 最高 | 网络控制 | CS7 (56) |
| 6 | 很高 | 网间控制 | CS6 (48) |
| 5 | 高 | 语音 (VoIP) | EF (46) |
| 4 | 中高 | 视频会议 | AF41 (34) |
| 3 | 中 | 流媒体 | AF31 (26) |
| 2 | 中低 | 重要数据 | AF21 (18) |
| 1 | 低 | 背景流量 | AF11 (10) |
| 0 | 最低 | Best Effort | BE (0) |

## 常见数据链路层协议

| 协议 | 类型 | 典型场景 | 状态 |
|------|------|----------|------|
| **Ethernet (IEEE 802.3)** | 多路访问 LAN | 有线局域网 | 主流 |
| **Wi-Fi (IEEE 802.11)** | 多路访问无线 LAN | 无线局域网 | 主流 |
| **PPP (RFC 1661)** | 点对点 | 拨号、串口、宽带接入 | 仍用（PPPoE） |
| **PPPoE (RFC 2516)** | PPP over Ethernet | 家庭宽带 | 主流 |
| **HDLC (ISO 13239)** | 点对点 / 多点 | 租用线路、Cisco 路由器串口 | 遗留 |
| **Frame Relay** | 面向连接 WAN | 企业广域网 | 已淘汰，被 MPLS/MPLS VPN 取代 |
| **ATM (异步传输模式)** | 面向连接 WAN | 运营商骨干、B-ISDN | 已淘汰 |
| **STP/RSTP/MSTP** | LAN 控制 | 防环 | 主流（RSTP/MSTP） |
| **LACP (802.3ad/ax)** | LAN 聚合 | 链路聚合 | 主流 |
| **LLDP (802.1AB)** | LAN 发现 | 邻居发现 | 主流 |
| **802.1X / EAP** | LAN 接入控制 | 端口认证 | 主流 |
| **L2TP** | 隧道 | VPN（常叠加 IPsec） | 遗留 |

### 1. PPP (Point-to-Point Protocol)

| 项 | 说明 |
|----|------|
| **RFC** | 1661 (PPP) / 1332 (IPCP) / 1334 (PAP) / 1994 (CHAP) |
| **介质** | 串口、拨号调制解调器、ISDN、SONET/SDH |
| **帧格式** | HDLC 变体（字节导向，标志 0x7E） |

#### PPP 帧格式
```
┌──────┬─────────┬─────────┬──────────┬──────────┬───────┬──────┐
│ Flag │ Address │ Control │ Protocol │ Payload  │  FCS  │ Flag │
│ 0x7E │ 0xFF    │ 0x03    │ 2B       │ 可变     │ 2/4B  │ 0x7E │
│ 1B   │ (广播)  │ (UI)    │          │          │       │      │
└──────┴─────────┴─────────┴──────────┴──────────┴───────┴──────┘
                   ↑
        Protocol 字段:
          0x0021 = IPv4
          0x0057 = IPv6
          0xC021 = LCP (链路控制协议)
          0x8021 = IPCP (IP 控制协议)
          0xC223 = CHAP
```

#### PPP 工作阶段
```
阶段 1: 链路建立 (LCP Open)
  ├── 协商 MRU（最大接收单元，默认 1500）
  ├── 协商认证方式（PAP / CHAP / EAP）
  ├── 协商压缩、多链路捆绑等选项
  └── LCP Echo Request 保活

阶段 2: 认证 (可选但通常启用)
  ├── PAP: 用户名+密码明文传输（不安全）
  ├── CHAP: 三次握手 + MD5 哈希（密码不上网）
  │   服务器 → Challenge (随机数)
  │   客户端 → Response (MD5(ID+密码+Challenge))
  │   服务器 → Success/Failure
  └── EAP: 可扩展框架（配合 802.1X/RADIUS）

阶段 3: 网络层协议 (NCP)
  ├── IPCP: 协商 IP 地址、DNS（拨号上网时分配 IP）
  ├── IPv6CP: IPv6 接口标识符协商
  └── 其他 NCP（IPXCP 等遗留协议）

阶段 4: 数据传输
  └── 网络层包封装在 PPP 帧中传输

阶段 5: 链路终止
  └── LCP Terminate
```

#### PPP 变体
| 变体 | 封装 | 应用 |
|------|------|------|
| **PPP over HDLC** | 字节流 + 0x7E 标志 | 传统串口/ISDN |
| **PPPoE (RFC 2516)** | PPP over Ethernet | 家庭宽带（光猫→路由器） |
| **PPPoA** | PPP over ATM | ADSL 早期 |
| **L2TP** | PPP 隧道到远端 | VPN（常叠加 IPsec） |

#### PPPoE（宽带接入主流）
```
PPPoE 帧格式（在以太网帧中）:
┌────────┬────────┬────────────┬───────────────────────────┐
│ DMAC   │ SMAC   │ EtherType  │ PPPoE 头 + PPP 帧         │
│ 6B     │ 6B     │ 0x8864(S)  │ Ver|Type|Code|SID|Len|...│
│        │        │ 0x8863(D)  │                           │
└────────┴────────┴────────────┴───────────────────────────┘
                          ↑
                    0x8864: PPPoE Session (数据传输)
                    0x8863: PPPoE Discovery (发现阶段)

PPPoE 发现阶段:
  1. PADI (Initiation, 广播): 客户端 → "寻找接入集中器"
  2. PADO (Offer, 单播):    BRAS  → "我能服务"
  3. PADR (Request, 单播):  客户端 → "选择你"
  4. PADS (Session-Confirm):BRAS  → "会话 ID = X"
  5. PADT (Terminate):      任一  → 终止会话

  建立后，PPP 帧在以太网上传输，用户名/密码在 BRAS 上认证
  BRAS 通过 RADIUS 与 AAA 服务器交互
```

### 2. HDLC (High-Level Data Link Control)

| 项 | 说明 |
|----|------|
| **标准** | ISO 13239 / ITU-T X.25 LAPB |
| **类型** | 比特导向协议（帧由比特边界界定） |
| **典型场景** | 路由器串口直连、租用线路（T1/E1）、Cisco 默认封装 |

#### HDLC 帧格式
```
┌──────┬─────────┬──────────┬──────────┬───────┬──────┐
│ Flag │ Address │ Control  │ Payload  │  FCS  │ Flag │
│ 0x7E │ 8 bit   │ 8/16 bit │ 可变     │ 2/4B  │ 0x7E │
└──────┴─────────┴──────────┴──────────┴───────┴──────┘

控制字段 (U/I/S 帧):
  I 帧 (Information):  数据传输，含 N(S) 和 N(R) 序列号
  S 帧 (Supervisory):  流量控制（RR/RNR/REJ/SREJ）
  U 帧 (Unnumbered):   链路管理（SABM/UA/DISC/DM）
```

#### HDLC 三种工作模式
| 模式 | 说明 |
|------|------|
| **NRM (正常响应)** | 主从，从站需主站允许才能发（遗留） |
| **ABM (异步平衡)** | 平等，两端都可主动发（现代主流） |
| **ARM (异步响应)** | 从站可主动发，但主站控链路（少见） |

#### Cisco 私有 HDLC 扩展
```
标准 HDLC 没有协议字段，无法多协议复用
Cisco HDLC 在 Address 后加了 2B Protocol 字段:
┌──────┬─────────┬──────────┬──────────┬───────┬──────┐
│ Flag │ Address │ Protocol │ Payload  │  FCS  │ Flag │
│ 0x7E │ 0x0F/0x8F│ 2B      │          │       │ 0x7E │
└──────┴─────────┴──────────┴──────────┴───────┴──────┘
                  ↑
             0x0800 = IPv4
             0x86DD = IPv6

  现代路由器串口默认 Cisco HDLC
  跨厂商互联需改用 PPP（标准）
```

### 3. Frame Relay (帧中继)

| 项 | 说明 |
|----|------|
| **标准** | ITU-T I.122 / ANSI T1.618 |
| **类型** | 面向连接的 L2 分组交换（虚电路） |
| **历史地位** | 90 年代企业 WAN 主流，后被 MPLS/MPLS VPN 取代 |
| **现状** | 基本淘汰，部分遗留网络仍用 |

#### Frame Relay 帧格式
```
┌──────┬─────────────────┬──────────┬───────┬──────┐
│ Flag │ Address (2-4B)  │ Payload  │  FCS  │ Flag │
│ 0x7E │ DLCI+FECN+BECN+DE │      │       │ 0x7E │
└──────┴─────────────────┴──────────┴───────┴──────┘

地址字段详解:
┌────────┬────────┬─────────┬────────┬────────┬──────┐
│ DLCI   │ C/R    │ EA      │ FECN   │ BECN   │ DE   │
│ 6/8/10b│ 1b     │ 1b      │ 1b     │ 1b     │ 1b   │
└────────┴────────┴─────────┴────────┴────────┴──────┘

  DLCI (Data Link Connection Identifier): 本地虚电路标识（类似 VLAN ID）
  FECN (Forward Explicit Congestion Notification): 正向拥塞通知
  BECN (Backward Explicit Congestion Notification): 反向拥塞通知
  DE (Discard Eligible): 1=拥塞时可优先丢弃（类似 802.1Q DEI）
  EA (Extended Address): 1=本字节是地址字段的最后一字节
```

#### Frame Relay 网络结构
```
┌──────┐    PVC     ┌──────────────────┐    PVC     ┌──────┐
│ 站点 │◄──────────►│  Frame Relay     │◄──────────►│ 站点 │
│ CE   │   DLCI=100 │  Switch (运营商) │   DLCI=200 │ CE   │
└──────┘            │  永久虚电路 PVC  │            └──────┘
                    │  或交换虚电路 SVC│
                    └──────────────────┘

  与 MPLS 的对比:
    Frame Relay: L2 虚电路 (DLCI)
    MPLS:        L2.5 标签交换 (Label)
    → MPLS 更灵活、支持 VPN、流量工程，最终取代 Frame Relay
```

### 4. ATM (异步传输模式, Asynchronous Transfer Mode)

| 项 | 说明 |
|----|------|
| **标准** | ITU-T I.361 |
| **单元大小** | 固定 **53 字节**（5B 头 + 48B 载荷） |
| **历史地位** | 90 年代运营商骨干、B-ISDN |
| **现状** | 基本淘汰，被 MPLS + 以太网取代 |

#### ATM 信元结构
```
┌──────────────────┬───────────────────────────────────────┐
│ Header (5 bytes) │ Payload (48 bytes)                    │
├──────────────────┤                                       │
│ GFC(4) VPI(8)    │                                       │
│ VCI(16) PT(3)    │    48 字节载荷                        │
│ CLP(1) HEC(8)    │                                       │
└──────────────────┴───────────────────────────────────────┘

  VPI (Virtual Path Identifier): 虚通道标识
  VCI (Virtual Channel Identifier): 虚信道标识
  PT (Payload Type): 载荷类型（用户/管理/OAM）
  CLP (Cell Loss Priority): 1=可丢弃（类似 IP Precedence 低优先级）
  HEC (Header Error Control): 头部 CRC-8（可单字节纠错）

  设计初衷: 固定长度 → 硬件交换快、时延可预测（适合语音+视频）
  失败原因:
    - 53 字节太小，包头开销大 (~10%)
    - 需要 ATM 适配层 (AAL) 才能装 IP 包（AAL5 SAR 复杂）
    - 以太网速率指数增长，ATM 跟不上
```

### 5. STP / RSTP / MSTP (生成树协议)

| 项 | 说明 |
|----|------|
| **标准** | STP (802.1D, 1990) / RSTP (802.1w, 2001) / MSTP (802.1s, 2002) |
| **作用** | 在二层冗余拓扑中**自动阻塞冗余链路**，防止广播风暴和 MAC 表震荡 |
| **工作原理** | 选举根桥 → 计算最短路径 → 阻塞非最短端口 |

#### 为什么需要 STP？
```
冗余拓扑下 L2 无 TTL，广播包会死循环:

  ┌─────┐        ┌─────┐
  │ SW1 │◄──────►│ SW2 │
  └──┬──┘        └──┬──┘
     │              │
     └─────►◄───────┘

  广播帧: A→B→A→B→... (广播风暴，秒级瘫痪全网)
  单播帧: MAC 表不断翻转，帧重复交付
  STP 自动阻塞一条链路，打破环路
```

#### STP 选举过程
```
1. 选举根桥 (Root Bridge)
   - 比较 Bridge ID = Bridge Priority (2B) + MAC (6B)
   - Priority 默认 32768，越小越优
   - 全网一个根桥

2. 每个非根桥选举根端口 (Root Port, RP)
   - 到根桥开销最小的端口
   - 开销依据链路带宽（10M=100，1G=4，10G=2）

3. 每个网段选举指定端口 (Designated Port, DP)
   - 离根桥最近的端口，负责转发 BPDU

4. 其余端口 → Blocking (阻塞)

  端口状态:
    Disabled → Blocking → Listening → Learning → Forwarding
    STP: 30-50 秒才能到 Forwarding（慢！）
```

#### RSTP (快速生成树)
| 改进 | 说明 |
|------|------|
| **收敛速度** | 秒级（STP 是 30-50 秒） |
| **端口角色** | Root/Designated/Alternate/Backup（替代 Blocking） |
| **端口状态** | Discarding/Learning/Forwarding（简化） |
| **Proposal/Agreement** | 点对点链路快速握手，直接进 Forwarding |
| **BPDU** | 每个 Hello Time (2s) 发送，丢失 3 个即失效（STP 要等 MaxAge 20s） |

#### MSTP (多实例生成树)
```
STP/RSTP 问题: 全网一个生成树 → 链路利用率低

  ┌─────┐        ┌─────┐
  │ SW1 │◄──────►│ SW2 │  VLAN 10 走左链路
  └──┬──┘ 阻塞   └──┬──┘  VLAN 20 走右链路
     │    ↑         │    → 需要多个生成树实例
     └──────────────┘

MSTP 解决方案:
  - 把多个 VLAN 映射到一个 MST Instance (MSTI)
  - 每个 MSTI 独立运行生成树
  - MSTI 0 = IST (内部生成树)，必须存在

配置要素:
  - Region Name + Revision + VLAN-Instance 映射
  - 同 Region 的交换机配置必须完全一致
```

#### STP 家族对比
| 特性 | STP (802.1D) | RSTP (802.1w) | MSTP (802.1s) |
|------|--------------|---------------|---------------|
| 收敛 | 30-50 s | 1-3 s | 1-3 s |
| 实例数 | 1 | 1 | 多 (每 VLAN 或每组合) |
| 链路利用 | 低 | 低 | 高 |
| 兼容性 | — | 兼容 STP | 兼容 RSTP |
| 部署 | 遗留 | 主流（默认） | 大型企业 |

#### STP 安全加固
| 特性 | 作用 |
|------|------|
| **BPDU Guard** | 边缘端口收到 BPDU 立即 error-disable（防非法交换机） |
| **Root Guard** | 防止外部桥抢占根桥 |
| **Loop Guard** | 防止单向链路导致环路 |
| **PortFast** | 边缘端口直接进 Forwarding（跳过 Listening/Learning） |

### 6. LACP (Link Aggregation Control Protocol)

| 项 | 说明 |
|----|------|
| **标准** | 802.3ad（原）→ 802.1AX-2008（现行） |
| **作用** | 多条物理链路聚合成一条逻辑链路，**提高带宽 + 提供冗余** |

```
LACP 聚合:
  ┌──────┐  4 × 1G  ┌──────┐
  │ SW1  │═══════════│ SW2  │  → 聚合后 4G 逻辑链路
  └──────┘  LACP     └──────┘
            Eth-Trunk / Port-Channel

  模式:
    - 静态 LAG (无协议，两端手工配)
    - 动态 LACP (LACPDU 协商，Active/Passive)
    - PAgP (Cisco 私有，PAgP desirable/auto)

  负载均衡算法:
    - 源 MAC / 目的 MAC
    - 源 IP / 目的 IP (L3)
    - 源+目的 IP+Port (L4)
    - 同一条流的包必须走同一物理链路（避免乱序）
```

### 7. LLDP (Link Layer Discovery Protocol)

| 项 | 说明 |
|----|------|
| **标准** | IEEE 802.1AB |
| **作用** | L2 邻居发现（类似 Cisco CDP，但跨厂商） |
| **组播** | 01:80:c2:00:00:0e（链路本地，不被转发） |

```
LLDP 工作:
  - 每台设备周期性发送 LLDPDU（默认 30s）
  - 包含: 系统名、端口 ID、能力（桥/路由/电话）、VLAN、PoE 等
  - 邻居保存信息到 MIB（默认 TTL 120s）
  - 网管工具/SDN 控制器通过 LLDP 发现拓扑

  LLDP-MED (Media Endpoint Discovery):
    扩展用于 VoIP 电话:
    - 自动配置 VLAN
    - 协商 PoE 功率
    - 通告 QoS DSCP/PCP 值

  同类协议:
    - CDP (Cisco Discovery Protocol): Cisco 私有
    - FDP (Foundry): 遗留
    - EDP (Extreme): 遗留
```

### 8. 802.1X 端口访问控制

| 项 | 说明 |
|----|------|
| **标准** | IEEE 802.1X-2020 |
| **作用** | 基于端口的网络接入控制（NAC），未认证设备无法访问网络 |
| **配套** | EAP (Extensible Authentication Protocol) + RADIUS |

```
802.1X 三角色:
┌───────────┐   EAPoL    ┌───────────────┐   RADIUS   ┌────────────┐
│ Supplicant│◄──────────►│ Authenticator │◄──────────►│ Auth Server│
│ (客户端)  │            │ (交换机/AP)   │            │ (RADIUS)   │
└───────────┘            └───────────────┘            └────────────┘

  端口状态:
    Unauth → 仅允许 EAPoL 帧通过（其他流量丢弃）
    Authed → 端口开放，允许所有流量

  EAP 方法:
    EAP-MD5:   密码哈希（不安全，易字典攻击）
    EAP-TLS:   双向证书（最安全）
    EAP-TTLS:  服务器证书 + 内部密码
    PEAP:      微软主推，服务器证书 + MS-CHAPv2

  认证后联动:
    - 动态分配 VLAN (RADIUS 下发 Tunnel-Private-Group-ID)
    - 下发 ACL (RADIUS Filter-ID)
    - 协商 MACsec 密钥 (MKA)
```

### 9. L2TP (Layer 2 Tunneling Protocol)

| 项 | 说明 |
|----|------|
| **RFC** | 2661 (L2TPv2) / 3931 (L2TPv3) |
| **运行** | UDP 1701 |
| **用途** | 在 IP 网络上隧道传输 L2 帧（PPP/以太网/HDLC） |

```
L2TP 结构:
  LAC (L2TP Access Concentrator) ←──隧道──→ LNS (L2TP Network Server)

  L2TPv2 (主要用于 VPN):
    [外层 IP][UDP 1701][L2TP 头][PPP 帧]
    本身不加密 → 通常叠加 IPsec:
    [外层 IP][ESP][内层 IP][UDP][L2TP][PPP]

  L2TPv3 (伪线 Pseudowire):
    隧道化任意 L2 协议（以太网/HDLC/Frame Relay/ATM）
    [外层 IP][L2TPv3 头(无 UDP)][原始 L2 帧]
    用于运营商 L2VPN（被 EVPN 逐步取代）
```

---

## L2 协议对比总览

| 协议 | 类型 | 拓扑 | 可靠 | 现状 | 典型场景 |
|------|------|------|------|------|----------|
| Ethernet | 多路访问 | 广播 LAN | ❌ | 主流 | 有线局域网 |
| Wi-Fi | 多路访问 | 无线 LAN | ✅ ACK | 主流 | 无线接入 |
| PPP | 点对点 | P2P | ❌ | 主流 | 宽带（PPPoE）、串口 |
| HDLC | 点对点/多点 | P2P/多点 | ❌ | 遗留 | 租用线路、Cisco 默认 |
| Frame Relay | 虚电路 | 多点 | ❌ | 淘汰 | 企业 WAN（已转 MPLS） |
| ATM | 虚电路 | 多点 | ❌ | 淘汰 | 运营商骨干（已转 MPLS/以太） |
| STP/RSTP/MSTP | 控制协议 | LAN | — | 主流 | 防环、冗余拓扑 |
| LACP | 聚合协议 | P2P | — | 主流 | 链路捆绑、带宽聚合 |
| LLDP | 发现协议 | P2P | — | 主流 | 邻居发现、拓扑收集 |
| 802.1X | 接入控制 | P2P | — | 主流 | 端口认证、NAC |
| L2TP | 隧道 | P2P | ❌ | 遗留 | VPN (L2TP/IPsec)、L2VPN |

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| TCP/IP 对应 | [tcpip/link.md](../tcpip/link.md) | L1+L2 合并为 TCP/IP 网络接口层 |
| 上层 | [L3 网络层](L3-network.md) | 网络层包封装在链路层帧中 |
| Layer 2.5 | [MPLS](../protocols/mpls.md) | 标签交换，插入在 L2 帧头和 L3 IP 头之间 |
| 下层 | [L1 物理层](L1-physical.md) | 帧转换为比特流在物理介质上传输 |
| 相关协议 | [ARP](../protocols/arp.md) | ARP 工作在 L2/L3 边界，为 IP 解析 MAC 地址 |
| 链路层加密 | [protocols/encryption-layers.md L2](../protocols/encryption-layers.md#l2-链路层加密) | MACsec (802.1AE)、WPA3 无线加密 |
| 链路层差错 | [protocols/error-control.md L2](../protocols/error-control.md#l2-链路层--帧级检错--可选重传) | 以太网 CRC-32 只检不重传，Wi-Fi 有 ACK 重传 |
| 以太网详解 | [tcpip/link.md 以太网部分](../tcpip/link.md#以太网-ethernet) | 帧格式、MTU、交换机原理 |
| PPP/PPPoE | [PPP 章节](#1-ppp-point-to-point-protocol) | 点对点协议、宽带接入主流 |
| STP/RSTP/MSTP | [STP 章节](#5-stp--rstp--mstp-生成树协议) | 防环协议家族，RSTP/MSTP 主流 |
| LACP / LLDP / 802.1X | [L2 协议章节](#6-lacp-link-aggregation-control-protocol) | 链路聚合、邻居发现、端口认证 |
| 遗留 WAN 协议 | [Frame Relay](#3-frame-relay-帧中继) / [ATM](#4-atm-异步传输模式) | 已被 MPLS 和以太网取代 |
