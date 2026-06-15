# TCP/IP - 网络接口层 (Link Layer)

## 概述
网络接口层（也称链路层、网络接入层）整合了 OSI 的数据链路层和物理层，负责在物理网络上实际传输数据帧。

## 核心职责
1. **帧封装**：将 IP 数据报封装为适合物理网络的帧格式
2. **物理寻址**：使用 MAC 地址标识设备
3. **介质访问控制**：协调多设备共享传输介质
4. **物理信号传输**：将比特转换为物理信号

## 以太网 (Ethernet)

### 发展历史
| 标准 | 速率 | 介质 | 年代 |
|------|------|------|------|
| 10BASE-T | 10 Mbps | 双绞线 | 1990 |
| 100BASE-TX | 100 Mbps | 双绞线 | 1995 |
| 1000BASE-T | 1 Gbps | 双绞线 | 1999 |
| 10GBASE-T | 10 Gbps | 双绞线 | 2006 |
| 25GBASE-T | 25 Gbps | 双绞线 | 2016 |
| 100GBASE-SR4 | 100 Gbps | 光纤 | 2015 |
| 400GBASE-SR8 | 400 Gbps | 光纤 | 2018 |

### 以太网帧结构 (Ethernet II)
```
┌──────────┬───────────┬──────────┬──────────┬─────────────────┬─────┐
│ Preamble │ Dest MAC  │ Src MAC  │ EtherType│ Payload         │ FCS │
│ + SFD    │           │          │          │                 │     │
│ 8 bytes  │ 6 bytes   │ 6 bytes  │ 2 bytes  │ 46-1500 bytes   │ 4B  │
└──────────┴───────────┴──────────┴──────────┴─────────────────┴─────┘
                                                                      ↑
                                                              最小帧 64 字节
                                                              最大帧 1518 字节
```

### MTU 与 Jumbo Frame
- **标准 MTU**：1500 字节（以太网 payload）
- **Jumbo Frame**：9000 字节（减少头部开销，用于数据中心）
- **Path MTU Discovery**：通过 DF 位 + ICMP 发现路径最小 MTU

### 802.1Q VLAN 标签帧

在标准以太网帧中插入 4 字节 802.1Q 标签，同时携带 VLAN ID 和 QoS 优先级：

```
普通 Ethernet II 帧:
┌──────────┬───────────┬──────────┬─────────────────┬─────┐
│ Dst MAC  │ Src MAC   │EtherType │    Payload      │ FCS │
│  6 B     │  6 B      │ 0x0800   │                 │ 4 B │
└──────────┴───────────┴──────────┴─────────────────┴─────┘

802.1Q Tagged 帧 (多 4 字节):
┌──────────┬───────────┬──────────┬─────────────────┬──────────┬─────────────────┬─────┐
│ Dst MAC  │ Src MAC   │  TPID    │      TCI        │EtherType │    Payload      │ FCS │
│  6 B     │  6 B      │  2 B     │      2 B        │ 0x0800   │                 │ 4 B │
└──────────┴───────────┴──────────┴─────────────────┴──────────┴─────────────────┴─────┘
                                         │
                           ┌───────┬─────┬──────────┐
                           │  PCP  │ DEI │   VID    │
                           │ 3 bit │1 bit│  12 bit  │
                           │ 优先级│丢弃 │ VLAN ID  │
                           │ 0~7   │ eligible 0~4095│
                           └───────┴─────┴──────────┘
```

| 字段 | 说明 |
|------|------|
| **TPID** | Tag Protocol ID，固定 0x8100，标识这是一个 802.1Q 标签帧 |
| **PCP** | Priority Code Point (3 bit)，L2 层 QoS 优先级，0=最低 7=最高 |
| **DEI** | Drop Eligible Indicator (1 bit)，1=拥塞时可优先丢弃 |
| **VID** | VLAN ID (12 bit)，标识 VLAN 编号，有效范围 1~4094 |

> PCP 与 IP DSCP 的映射关系详见 [README QoS 章节](../README.md#qos-在各层的体现)

## Wi-Fi (IEEE 802.11)

### 版本对比
| 标准 | 商用名 | 频段 | 最大速率 | MIMO | 关键技术 |
|------|--------|------|----------|------|----------|
| 802.11n | Wi-Fi 4 | 2.4/5 GHz | 600 Mbps | 4×4 | MIMO, 40MHz 信道 |
| 802.11ac | Wi-Fi 5 | 5 GHz | 6.9 Gbps | 8×8 | MU-MIMO, 160MHz |
| 802.11ax | Wi-Fi 6 | 2.4/5 GHz | 9.6 Gbps | 8×8 | OFDMA, BSS Coloring |
| 802.11ax | Wi-Fi 6E | 6 GHz | 9.6 Gbps | 8×8 | 6 GHz 频段 |
| 802.11be | Wi-Fi 7 | 2.4/5/6 GHz | 46 Gbps | 16×16 | MLO, 320MHz |

### Wi-Fi 安全协议
| 协议 | 加密 | 强度 | 状态 |
|------|------|------|------|
| WEP | RC4 | 极弱 | 已弃用 |
| WPA | TKIP | 弱 | 已弃用 |
| WPA2 | AES-CCMP | 强 | 当前主流 |
| WPA3 | AES-GCMP | 更强 | 新一代 |

### CSMA/CA 流程
```
1. 监听信道是否空闲
2. 如空闲：等待 DIFS 时间
3. 随机退避（Backoff Counter）
4. 计数器减至 0 → 发送
5. 如信道忙：冻结计数器，等待空闲后继续
6. 收到 ACK 表示成功，否则重传
```

## PPP (Point-to-Point Protocol)

### 用途
- 拨号上网（已过时）
- PPPoE（DSL 宽带接入）
- 串行链路（路由器间直连）

### PPP 帧格式
```
┌──────┬─────────┬──────────┬─────────┬─────────┐
│ Flag │ Address │ Control  │Protocol │ Payload │ FCS │
│ 0x7E │  0xFF   │  0x03    │ 2 bytes │ variable│     │
└──────┴─────────┴──────────┴─────────┴─────────┘
```

### PPP 协商过程
```
LCP (链路控制协议)
  ├─ 协商链路参数（MRU、认证方式等）
  ├─ 链路质量检测
  └─ 链路终止

认证 (可选)
  ├─ PAP (明文密码)
  └─ CHAP (挑战-响应，更安全)

NCP (网络控制协议)
  ├─ IPCP → 协商 IP 地址
  ├─ IPv6CP → 协商 IPv6
  └─ ...
```

## 二层交换

### 交换机转发方式
| 方式 | 说明 | 延迟 | 错误处理 |
|------|------|------|----------|
| 存储转发 | 收完整帧后查表转发 | 高 | 检查 FCS |
| 直通转发 | 读到目的 MAC 立即转发 | 低 | 不检查 |
| 碎片过滤 | 读到 64 字节后转发 | 中 | 过滤碎片 |

### STP (生成树协议)
- 防止二层环路导致的广播风暴
- 选举根桥 → 确定根端口 → 确定指定端口 → 阻塞冗余端口
- 版本：STP (802.1D) / RSTP (802.1w) / MSTP (802.1s)

## 网络设备对照

| 设备 | 工作层 | 功能 |
|------|--------|------|
| 集线器 (Hub) | L1 | 多端口中继器，共享冲突域 |
| 交换机 (Switch) | L2 | MAC 地址转发，隔离冲突域 |
| 三层交换机 | L2+L3 | 交换+路由 |
| 路由器 (Router) | L3 | IP 路由转发 |
| 防火墙 | L3-L7 | 访问控制与安全策略 |
| 负载均衡器 | L4-L7 | 流量分发 |

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| OSI 对应 | [L2 数据链路层](../osi/L2-datalink.md) · [L1 物理层](../osi/L1-physical.md) | TCP/IP 网络接口层 = OSI L1+L2 |
| 上层 | [internet.md](internet.md) | 网络层的 IP 包封装在以太网帧中 |
| Layer 2.5 | [MPLS](../protocols/mpls.md) | MPLS 标签插入在 L2 帧头和 L3 IP 头之间 |
| 相关协议 | [ARP](../protocols/arp.md) | 工作在 L2/L3 边界，为 IP 层解析 MAC 地址 |
| DHCP | [protocols/dhcp.md](../protocols/dhcp.md) | 设备通过 DHCP 获取 IP，DHCP 广播依赖链路层 |
| 链路层加密 | [protocols/encryption-layers.md L2](../protocols/encryption-layers.md#l2-链路层加密) | MACsec、WPA2/3 无线加密 |
| 案例 | [案例4: 从零连通](../cases/04-dhcp-and-arp.md) | 新设备 DHCP 获取 IP + ARP 冲突检测 + 首次路由 |
| 案例 | [案例5: 数据中心](../cases/05-datacenter-east-west.md) | Overlay 网络下 Underlay 帧的封装与传输 |
