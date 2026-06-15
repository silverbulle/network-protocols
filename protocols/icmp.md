# ICMP 协议详解

## 概述
ICMP (Internet Control Message Protocol, RFC 792) 用于在 IP 网络设备间传递控制和错误信息。它是 IP 协议的辅助协议，不用于数据传输。

### 协议栈位置与关联

```
应用层    ping (命令)    traceroute (命令)
              │                │
              ▼                ▼
         (不经过 TCP/UDP)
              │                │
网络层   ┌──────────────────────────┐
         │  IP (v4/v6)              │
         │    └─ ICMP ← 本文件      │
         │       Protocol = 1       │
         └──────────────────────────┘
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 直接依赖 | [IP](ip.md) | ICMP 直接封装在 IP 数据报中，不经过传输层 |
| 利用 IP TTL | [IP - TTL](ip.md#字段详解) | traceroute 利用 TTL 递减 + ICMP 超时报文 |
| Path MTU | [IP - 分片](ip.md#ip-分片) | ICMP "Fragmentation Needed" 用于 PMTUD |
| IPv6 NDP | [IP - IPv6](ip.md#ipv6) | ICMPv6 的 NDP 替代了 ARP 的功能 |
| OSI 层 | [L3 网络层](../osi/L3-network.md) | ICMP 属于网络层辅助协议 |
| TCP/IP 层 | [tcpip/internet.md](../tcpip/internet.md#icmp-internet-control-message-protocol) | |

## 协议位置
- 逻辑上属于网络层（与 IP 同层）
- 封装在 IP 数据报中（Protocol = 1）
- 不依赖 TCP 或 UDP

## ICMP 报文格式
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                    Type-specific fields                        |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                      Data (variable)                          |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## ICMP 消息类型

### 查询消息 (Query Messages)

| Type | Code | 名称 | 说明 |
|------|------|------|------|
| 0 | 0 | Echo Reply | ping 响应 |
| 8 | 0 | Echo Request | ping 请求 |
| 13 | 0 | Timestamp Request | 时间戳请求 |
| 14 | 0 | Timestamp Reply | 时间戳响应 |

### 差错消息 (Error Messages)

| Type | Code | 名称 | 说明 |
|------|------|------|------|
| 3 | 0 | Network Unreachable | 网络不可达 |
| 3 | 1 | Host Unreachable | 主机不可达 |
| 3 | 2 | Protocol Unreachable | 协议不可达 |
| 3 | 3 | Port Unreachable | 端口不可达 |
| 3 | 4 | Fragmentation Needed | 需要分片但 DF 位设置 |
| 5 | 0 | Redirect (Network) | 路由重定向 |
| 11 | 0 | TTL Exceeded in Transit | TTL 超时 |
| 11 | 1 | Fragment Reassembly Timeout | 分片重组超时 |
| 12 | 0 | IP Header Bad | IP 头部参数错误 |

## Ping 工具

### 工作原理
```
发送方: ICMP Echo Request (Type=8) → 目的主机
接收方: ICMP Echo Reply (Type=0) → 发送方

测量: Round-Trip Time (RTT) = 回复时间 - 发送时间
```

### 使用示例
```bash
$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=15.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=14.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=117 time=15.1 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 14.8/15.0/15.2/0.2 ms
```

## Traceroute 工具

### 工作原理
```
第1轮: 发送 TTL=1 的包 → 第1跳路由器 TTL→0 → 返回 ICMP Type 11
第2轮: 发送 TTL=2 的包 → 第2跳路由器 TTL→0 → 返回 ICMP Type 11
第3轮: 发送 TTL=3 的包 → 第3跳路由器 TTL→0 → 返回 ICMP Type 11
...
第N轮: 到达目标 → 返回 ICMP Type 0 (或 UDP Port Unreachable)
```

### 使用示例
```bash
$ traceroute 8.8.8.8
traceroute to 8.8.8.8, 30 hops max
 1  192.168.1.1       1.2 ms  1.1 ms  1.0 ms
 2  10.0.0.1          5.3 ms  5.2 ms  5.1 ms
 3  172.16.0.1       10.5 ms  10.3 ms  10.4 ms
 4  * * *                                 (不响应 ICMP)
 5  8.8.8.8          15.2 ms  14.9 ms  15.0 ms
```

## Path MTU Discovery

```
发送方设置 DF (Don't Fragment) 位
↓
路由器发现包 > MTU
↓
返回 ICMP Type 3 Code 4 (Fragmentation Needed)
↓
包含下一跳 MTU 值
↓
发送方减小包大小重试
```

## ICMPv6

### 与 ICMPv4 的区别
- Protocol 号 = 58（ICMPv4 = 1）
- 整合了 ARP 功能（NDP 协议）
- 类型空间重新定义

### NDP (Neighbor Discovery Protocol) 消息

| Type | 名称 | 对应 IPv4 |
|------|------|-----------|
| 133 | Router Solicitation | — |
| 134 | Router Advertisement | — |
| 135 | Neighbor Solicitation | ARP Request |
| 136 | Neighbor Advertisement | ARP Reply |
| 137 | Redirect | ICMP Redirect |

### SLAAC (无状态地址自动配置)
```
1. 主机发送 Router Solicitation (Type 133)
2. 路由器响应 Router Advertisement (Type 134)
   - 包含网络前缀
3. 主机用前缀 + 接口标识符生成 IPv6 地址
4. DAD (重复地址检测): 发送 Neighbor Solicitation 验证唯一性
```

## ICMP 安全考虑
- **ICMP Flood**：大量 Echo Request 导致 DoS
- **Smurf 攻击**：伪造源地址广播 ping
- **ICMP 重定向攻击**：劫持路由
- **防御**：限速、过滤、禁用不必要的 ICMP 类型
