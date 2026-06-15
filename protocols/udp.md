# UDP 协议详解

## 概述
UDP (User Datagram Protocol, RFC 768) 是一个简单的、无连接的传输层协议，提供尽力交付的数据报服务。

### 协议栈位置与关联

```
应用层    DNS  DHCP  SNMP  RTP  QUIC/HTTP3 ...
            │    │     │    │    │
            ▼    ▼     ▼    ▼    ▼
传输层   ┌─────┐  ┌──────────┐
         │ UDP │  │   TCP    │
         └──┬──┘  └────┬─────┘
            │          │
网络层      ▼          ▼
         ┌──────────────────┐
         │     IP (v4/v6)   │
         └──────────────────┘
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 上层依赖者 | [DNS](dns.md) · [DHCP](dhcp.md) · [HTTP/3](http.md#http3-基于-quic) | 低延迟或广播场景的应用层协议 |
| 下层依赖 | [IP](ip.md) | UDP 数据报封装在 IP 数据报中 |
| 对比协议 | [TCP](tcp.md) | 面向连接的可靠替代 |
| QUIC 关系 | [HTTP - HTTP/3](http.md#http3-基于-quic) | QUIC 在 UDP 之上实现可靠传输+加密 |
| OSI 层 | [L4 传输层](../osi/L4-transport.md) | |
| TCP/IP 层 | [tcpip/transport.md](../tcpip/transport.md) | |

## 协议特性
- **无连接**：无需建立连接，直接发送
- **不可靠**：不保证交付、不保证有序、不重传
- **低开销**：头部仅 8 字节
- **面向报文**：保留应用层消息边界
- **支持多播/广播**：一对多通信

## UDP 数据报格式

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Data (Payload)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| Source Port | 16 bit | 源端口（可选，不回复时可置 0） |
| Destination Port | 16 bit | 目的端口 |
| Length | 16 bit | 总长度（头部+数据），最小 8 |
| Checksum | 16 bit | 校验和（IPv4 可选，IPv6 必须） |

## UDP vs TCP 对比

| 维度 | TCP | UDP |
|------|-----|-----|
| 连接 | 面向连接 | 无连接 |
| 可靠性 | 可靠（确认、重传） | 不可靠 |
| 有序性 | 保证 | 不保证 |
| 流量控制 | 有 | 无 |
| 拥塞控制 | 有 | 无 |
| 头部开销 | 20-60 字节 | 8 字节 |
| 传输单位 | 字节流 | 数据报 |
| 多播/广播 | 不支持 | 支持 |
| 速度 | 较慢 | 较快 |

## 使用 UDP 的协议

| 协议 | 端口 | 原因 |
|------|------|------|
| DNS | 53 | 小包、低延迟 |
| DHCP | 67/68 | 广播、无连接 |
| SNMP | 161 | 管理报文小 |
| NTP | 123 | 时间敏感 |
| TFTP | 69 | 简单文件传输 |
| RTP | 动态 | 实时音视频 |
| QUIC | 443 | HTTP/3 底层 |
| syslog | 514 | 日志传输 |

## UDP 编程模型

### Socket API
```
Server:
  socket()      → 创建 UDP socket
  bind()        → 绑定端口
  recvfrom()    → 接收数据报（阻塞）
  sendto()      → 发送数据报

Client:
  socket()      → 创建 UDP socket
  sendto()      → 发送数据报
  recvfrom()    → 接收响应
```

### Python 示例
```python
# Server
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 9999))
while True:
    data, addr = sock.recvfrom(1024)
    sock.sendto(data.upper(), addr)

# Client
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.sendto(b'hello', ('127.0.0.1', 9999))
data, addr = sock.recvfrom(1024)
```

## UDP 可靠性方案

由于 UDP 本身不可靠，需要在应用层实现可靠性：

### 方案 1：简单超时重传
```
发送数据 → 启动定时器 → 超时未收到 ACK → 重传
```

### 方案 2：序列号 + 去重
```
每个数据报添加序列号
接收方按序列号排序
重复序列号丢弃
```

### 方案 3：QUIC (HTTP/3)
```
基于 UDP 实现完整传输层：
├─ 连接建立（0-RTT / 1-RTT）
├─ 可靠性（重传 + 序列号）
├─ 拥塞控制（CUBIC / BBR）
├─ 多路复用（无队头阻塞）
└─ 加密（内置 TLS 1.3）
```

## UDP 数据报大小限制
- **理论最大**：65535 字节（Length 字段限制）
- **实际建议**：≤ 508 字节（保证不分片）
- **MTU 限制**：通常 ≤ 1472 字节（1500 - 20 IP头 - 8 UDP头）
- **超大报文**：需分片或应用层分段
