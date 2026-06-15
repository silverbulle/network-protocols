# TCP/IP - 传输层 (Transport Layer)

## 概述
传输层提供端到端的数据传输服务，是进程间通信的桥梁。主要协议为 TCP 和 UDP。

## 端口号体系
- **范围**：0 - 65535（16 bit）
- **知名端口**：0 - 1023（系统保留）
- **注册端口**：1024 - 49151（IANA 注册）
- **动态端口**：49152 - 65535（临时使用）

## TCP (Transmission Control Protocol)

### 核心特性
1. **面向连接**：通信前需建立连接
2. **可靠传输**：序列号 + 确认 + 重传
3. **有序交付**：按序列号重组
4. **流量控制**：滑动窗口
5. **拥塞控制**：慢启动、拥塞避免、快重传、快恢复

### TCP 状态机
```
         CLOSED
           │
           │ 主动打开
           ▼
         SYN_SENT ──────→ 收到 SYN+ACK ──→ ESTABLISHED
           │                                    │
           │ 超时                               │ 主动关闭
           ▼                                    ▼
         CLOSED                            FIN_WAIT_1
                                               │
         被动打开                          收到 ACK
           │                                    │
           ▼                                    ▼
         LISTEN                            FIN_WAIT_2
           │                                    │
           │ 收到 SYN                      收到 FIN
           ▼                                    │
         SYN_RCVD ◄──── 收到 FIN ───────────────┘
           │               (同时关闭)
           │ 收到 ACK      │
           ▼               ▼
       ESTABLISHED     TIME_WAIT
           │               │
           │ 被动关闭    2MSL 超时
           ▼               │
        CLOSE_WAIT      CLOSED
           │
           │ 发送 FIN
           ▼
        LAST_ACK
           │
           │ 收到 ACK
           ▼
         CLOSED
```

### TCP 拥塞控制四阶段

#### 1. 慢启动 (Slow Start)
```
cwnd: 1 → 2 → 4 → 8 → 16 → ... (指数增长)
直到达到 ssthresh 或发生丢
```

#### 2. 拥塞避免 (Congestion Avoidance)
```
cwnd: 每个 RTT 增加 1 (线性增长)
直到发生丢包
```

#### 3. 快重传 (Fast Retransmit)
```
收到 3 个重复 ACK → 立即重传丢失的段
不等待超时
```

#### 4. 快恢复 (Fast Recovery)
```
ssthresh = cwnd / 2
cwnd = ssthresh + 3
收到新 ACK 后进入拥塞避免阶段
```

### ECN (显式拥塞通知)

ECN 是丢包之外的另一种拥塞信号机制：路由器在拥塞时标记 IP ECN=CE（不丢包），接收方通过 TCP ECE 标志通知发送方减速。详见 [TCP ECN](../protocols/tcp.md#ecn-显式拥塞通知) 和 [IP ECN](../protocols/ip.md#ecn-状态)。

### TCP 变体
| 变体 | 特点 | 适用场景 |
|------|------|----------|
| TCP Reno | 经典拥塞控制 | 通用 |
| TCP CUBIC | Linux 默认，基于三次函数 | 高带宽长肥管道 |
| TCP BBR | Google 开发，基于带宽和 RTT 测量 | 现代互联网 |
| TCP Fast Open | SYN 中携带数据 | 减少握手延迟 |
| MPTCP | 多路径 TCP | 移动设备 |

## UDP (User Datagram Protocol)

### 核心特性
1. **无连接**：无需建立连接
2. **不可靠**：不保证交付、不保证有序
3. **低开销**：头部仅 8 字节
4. **支持多播/广播**

### UDP 适用场景
- DNS 查询（小包、低延迟要求）
- 视频/音频流（容忍少量丢失）
- 游戏（低延迟优先）
- DHCP、SNMP
- QUIC/HTTP3 的底层传输

### UDP 可靠性补偿方案
应用层可自行实现：
- 超时重传
- 序列号排序
- 校验和验证

```
应用层
  ├─ 自定义可靠性层（如 QUIC、RUDP）
  └─ UDP
```

## 常见端口对照

| 端口 | 协议 | 服务 |
|------|------|------|
| 20/21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| 443 | TCP | HTTPS |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP 代理 |

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| OSI 对应 | [L4 传输层](../osi/L4-transport.md) | 传输层在两个模型中职责一致 |
| 上层 | [application.md](application.md) | 应用层协议选择 TCP 或 UDP |
| 下层 | [internet.md](internet.md) | TCP/UDP 段封装在 IP 数据报中 |
| TCP 详解 | [protocols/tcp.md](../protocols/tcp.md) | 报文格式、状态机、拥塞控制四阶段 |
| UDP 详解 | [protocols/udp.md](../protocols/udp.md) | 无连接传输、QUIC 的底层 |
| 案例 | [案例1: 负载均衡](../cases/01-remote-load-balancing.md) | TCP+TLS+HTTP 在公网上的完整路径 |
