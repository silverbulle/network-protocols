# TCP 协议详解

## 概述
TCP (Transmission Control Protocol, RFC 793) 是互联网最核心的传输层协议，提供面向连接的、可靠的、有序的字节流传输服务。

### 协议栈位置与关联

```
应用层    HTTP  DNS  FTP  SMTP  SSH ...
            │    │    │    │    │
            ▼    │    ▼    ▼    ▼
传输层   ┌──────────┐  ┌─────┐
         │   TCP    │  │ UDP │
         └────┬─────┘  └──┬──┘
              │           │
网络层        ▼           ▼
         ┌──────────────────┐
         │     IP (v4/v6)   │
         └──────────────────┘
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 上层依赖者 | [HTTP](http.md) · [DNS](dns.md) · [DHCP](dhcp.md)(间接) | 需要可靠传输的应用层协议 |
| 下层依赖 | [IP](ip.md) | TCP 段封装在 IP 数据报中 |
| 对比协议 | [UDP](udp.md) | 无连接的轻量替代 |
| OSI 层 | [L4 传输层](../osi/L4-transport.md) | TCP/IP 传输层 = OSI 传输层 |
| TCP/IP 层 | [tcpip/transport.md](../tcpip/transport.md) | 传输层整体视角 |

## 协议特性
- **面向连接**：通信前通过三次握手建立连接
- **可靠传输**：序列号 + 确认号 + 超时重传
- **有序交付**：按序列号重组数据
- **全双工**：双方可同时收发
- **字节流**：无消息边界，应用需自行处理
- **流量控制**：接收窗口 (rwnd) 限制发送速率
- **拥塞控制**：拥塞窗口 (cwnd) 避免网络过载

## TCP 段格式详解

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Data  |     |U|A|P|R|S|F|                                    |
|Offset | Res |R|C|S|S|Y|I|            Window Size             |
|       |     |G|K|H|T|N|N|                                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 字段详解

| 字段 | 长度 | 说明 |
|------|------|------|
| Source Port | 16 bit | 源端口号 |
| Destination Port | 16 bit | 目的端口号 |
| Sequence Number | 32 bit | 本报文段第一个字节的序列号 |
| Acknowledgment Number | 32 bit | 期望收到的下一个字节序号 |
| Data Offset | 4 bit | 头部长度（单位：32bit 字） |
| Reserved | 6 bit | 保留位 |
| Flags | 6 bit | 控制标志位 |
| Window Size | 16 bit | 接收窗口大小 |
| Checksum | 16 bit | 校验和（含伪头部） |
| Urgent Pointer | 16 bit | 紧急指针 |
| Options | 可变 | 选项字段 |

### 控制标志位
| 标志 | 名称 | 说明 |
|------|------|------|
| URG | Urgent | 紧急指针有效 |
| ACK | Acknowledgment | 确认号有效 |
| PSH | Push | 请求立即交付给应用 |
| RST | Reset | 重置连接 |
| SYN | Synchronize | 同步序列号（建立连接） |
| FIN | Finish | 发送方结束发送 |

## 连接管理

### 三次握手 (Three-Way Handshake)
```
Client                              Server
  │                                    │
  │──── SYN (seq=x) ──────────────────→│  ISN 初始化
  │                                    │
  │←─── SYN+ACK (seq=y, ack=x+1) ─────│  服务器 ISN 初始化
  │                                    │
  │──── ACK (seq=x+1, ack=y+1) ───────→│  连接建立
  │                                    │
  │════ ESTABLISHED ═══════════════════│
```

**为什么需要三次？**
- 两次：无法确认客户端的接收能力（历史连接问题）
- 三次：双方都确认了 SYN 和 ACK

### 四次挥手 (Four-Way Handshake)
```
Client                              Server
  │                                    │
  │──── FIN (seq=u) ──────────────────→│  客户端主动关闭
  │                                    │
  │←─── ACK (ack=u+1) ────────────────│  服务器确认（半关闭）
  │                                    │  服务器可能还有数据
  │←─── FIN (seq=w) ──────────────────│  服务器关闭
  │                                    │
  │──── ACK (ack=w+1) ────────────────→│  客户端确认
  │                                    │
  │  [TIME_WAIT: 2×MSL]               │  防止旧报文干扰新连接
  │                                    │
  │════ CLOSED ════════════════════════│
```

### TIME_WAIT 的作用
1. 确保最后的 ACK 能到达（若丢失，对方会重传 FIN）
2. 让网络中该连接的旧报文消亡
3. 时长：2×MSL (Maximum Segment Lifetime)，通常 60s

## 可靠传输机制

### 序列号与确认号
- **序列号 (SEQ)**：本报文段第一个字节的编号
- **确认号 (ACK)**：期望收到的下一个字节编号（累积确认）

```
发送 3 个段:
SEQ=0,   LEN=100 → 字节 0-99
SEQ=100, LEN=200 → 字节 100-299
SEQ=300, LEN=150 → 字节 300-449

ACK=450 表示期望收到字节 450（即前 450 字节已全部收到）
```

### 超时重传 (RTO)
```
RTT 测量:
  SampleRTT = 实际往返时间
  EstimatedRTT = 0.875 × EstimatedRTT + 0.125 × SampleRTT
  DevRTT = 0.75 × DevRTT + 0.25 × |SampleRTT - EstimatedRTT|
  RTO = EstimatedRTT + 4 × DevRTT

超时 → 重传 → RTO 翻倍（指数退避）
```

### 选择性确认 (SACK)
```
传统累积确认: ACK=100 (期望 100，但 200-299 可能已收到)
SACK: ACK=100, SACK=[200-299],[400-499]
      → 发送方知道只需重传 100-199 和 300-399
```

## 流量控制

### 滑动窗口
```
发送窗口 = min(rwnd, cwnd)

接收方通过 ACK 中的 Window Size 告知发送方：
  "我还有 rwnd 字节的缓冲区空间"

发送方维护:
  [已发送已确认] [已发送未确认] [可发送] [不可发送]
                  ↑              ↑        ↑
               LastByteAcked   LastByteSent LastByteRwnd
```

### 零窗口处理
- 接收方 rwnd=0 时，发送方停止发送
- 发送方定期发送探测报文（Zero Window Probe）
- 接收方窗口恢复时通知发送方

## 拥塞控制

### 核心变量
- **cwnd**：拥塞窗口，反映网络容量
- **ssthresh**：慢启动阈值

### 四个阶段

#### 1. 慢启动
```
初始 cwnd = 2-4 MSS (现代实现常用 10 MSS)
每收到一个 ACK: cwnd += 1 (每个 RTT 翻倍)
达到 ssthresh 或丢包 → 切换阶段
```

#### 2. 拥塞避免
```
每个 RTT: cwnd += 1 MSS (线性增长)
发生丢包 → 快重传/快恢复
```

#### 3. 快重传
```
收到 3 个重复 ACK → 立即重传丢失的段
不必等待超时
```

#### 4. 快恢复
```
ssthresh = cwnd / 2
cwnd = ssthresh + 3 (重复 ACK 数)
收到新 ACK → 进入拥塞避免
```

### ECN (显式拥塞通知)

ECN (Explicit Congestion Notification, RFC 3168) 通过 IP 层的 ECN 位 + TCP 的 CWR/ECE 标志位实现**不丢包的拥塞信号**：

```
传统拥塞通知:
  拥塞 → 路由器丢包 → TCP 超时/重传 → 减小窗口
  问题: 丢包 = 数据丢失 + 延迟增加

ECN 拥塞通知:
  拥塞 → 路由器标记 CE (不丢包) → TCP 通知减速 → 减小窗口
  优势: 不丢包，更快响应

ECN 流程:
  发送方 (ECT)          路由器                接收方
    │── 包 ECN=10 ─────→│                     │
    │                   │── 拥塞，标记 CE ───→│
    │                   │  ECN=11 (不丢包)    │
    │                   │                     │── ACK + ECE=1 ──→│
    │←── CWR=1 (确认) ──│                     │
    │   减小 cwnd       │                     │
```

| TCP 标志位 | 含义 |
|-----------|------|
| **ECE** (ECN-Echo) | 接收方通知发送方：收到了 CE 标记的包 |
| **CWR** (Congestion Window Reduced) | 发送方确认已减小拥塞窗口 |

> IP ECN 字段定义见 [ip.md ECN 状态](ip.md#ecn-状态)

| 选项 | 长度 | 说明 |
|------|------|------|
| MSS | 4B | 最大报文段长度 |
| Window Scale | 3B | 窗口缩放因子（最大 14） |
| SACK Permitted | 2B | 允许选择性确认 |
| SACK | 可变 | 选择性确认块 |
| Timestamps | 10B | 时间戳（RTT 测量 + PAWS） |
| NOP | 1B | 填充对齐 |

## TCP 性能优化

### Nagle 算法
- 合并小包，减少头部开销
- 规则：有未确认数据时，缓存后续小包
- `TCP_NODELAY` 可禁用（实时应用常用）

### Delayed ACK
- 延迟发送确认（通常 200ms）
- 可与 Nagle 算法冲突导致延迟

### TCP Fast Open
- 三次握手的 SYN 中携带数据
- 使用 TFO Cookie 验证客户端
- 减少一个 RTT 的连接建立延迟

## Wireshark 抓包示例
```
No.  Source         Dest          Protocol Info
1    10.0.0.1      10.0.0.2      TCP      50000→80 [SYN] Seq=0 Win=65535
2    10.0.0.2      10.0.0.1      TCP      80→50000 [SYN,ACK] Seq=0 Ack=1 Win=65535
3    10.0.0.1      10.0.0.2      TCP      50000→80 [ACK] Seq=1 Ack=1 Win=65535
4    10.0.0.1      10.0.0.2      TCP      50000→80 [PSH,ACK] Seq=1 Ack=1 Len=512
5    10.0.0.2      10.0.0.1      TCP      80→50000 [ACK] Seq=1 Ack=513 Win=65023
```
