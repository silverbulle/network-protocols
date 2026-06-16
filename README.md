# 001 - OSI 与 TCP/IP 协议知识库

## 项目目标
系统性汇总 OSI 七层模型及 TCP/IP 四层模型的知识文档。重点不在于罗列各层定义，而是揭示协议栈**从抽象到实现、从理论到工程**的完整关联链路。

---

## 一、两个模型的对应关系

OSI 是理论参考框架，TCP/IP 是工程实现。互联网基于 TCP/IP，但 OSI 的分层思维有助于理解每层的职责边界。

```
  OSI 七层                   TCP/IP 四层              文档索引
 ┌──────────────┐
 │ L7 应用层     │─────┐
 ├──────────────┤     │                            ┌────────────────────┐
 │ L6 表示层     │─────┼──→  应用层 (Application) ──→│ tcpip/application  │
 ├──────────────┤     │                            │ protocols/http     │
 │ L5 会话层     │─────┘                            │ protocols/dns      │
 ├──────────────┤                                  │ protocols/dhcp     │
 │ L4 传输层     │────────→  传输层 (Transport) ────→│ tcpip/transport    │
 ├──────────────┤                                  │ protocols/tcp      │
 │ L3 网络层     │─────┐                            │ protocols/udp      │
 ├──────────────┤     │                            │ tcpip/internet     │
 │              │     ├─→  网际层 (Internet) ──────→│ protocols/ip       │
 │ Layer 2.5    │     │   MPLS 标签交换 ──────────→│ protocols/icmp     │
 │ (Overlay)    │     │   VXLAN L2-over-L3 ───────→│ protocols/arp      │
 ├──────────────┤     │   (L2/L3 之间)             │ protocols/mpls     │
 │ L2 数据链路层 │─────┤                            │ protocols/vxlan    │
 ├──────────────┤     └──→  网络接口层 (Link) ─────→│ tcpip/link         │
 │ L1 物理层     │─────┘                            └────────────────────┘
 └──────────────┘

  osi/00-overview          tcpip/00-overview
  osi/L7 ~ osi/L1          tcpip/application ~ link
```

**阅读建议**：先看 `osi/00-overview` 理解分层思想，再看 `tcpip/00-overview` 理解实际工程如何简化合并。

---

## 二、数据从应用到线路：全链路封装

一个 HTTP 请求从浏览器发出，经历逐层封装，最终变成电缆上的电信号。反向过程则是逐层解封装。这是理解所有协议关联的核心线索。

```
 浏览器输入 http://example.com
 │
 ▼
┌─ 应用层 ─────────────────────────────────────────────┐
│ HTTP GET 请求报文 (protocols/http)                    │
│ "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"        │
│                                                       │
│ DNS 解析域名 → IP (protocols/dns)                     │
│   DNS 本身也是应用层协议，使用 UDP 53 端口             │
└──────────────────────────┬────────────────────────────┘
                           ▼
┌─ 传输层 ─────────────────────────────────────────────┐
│ TCP 段 (protocols/tcp)                                │
│ [Src Port | Dst Port=80 | Seq | ACK | Flags | Data]  │
│                                                       │
│ 若使用 HTTP/3 → 底层是 QUIC → 基于 UDP (protocols/udp)│
└──────────────────────────┬────────────────────────────┘
                           ▼
┌─ 网络层 ─────────────────────────────────────────────┐
│ IP 数据报 (protocols/ip)                              │
│ [IP Header | TCP Header | HTTP Data]                  │
│                                                       │
│ 路由选择：查路由表决定下一跳                            │
│ ARP 解析：下一跳 IP → MAC 地址 (protocols/arp)        │
│   ARP 工作在 L2/L3 之间，为网络层提供地址解析能力       │
└──────────────────────────┬────────────────────────────┘
                           ▼
┌─ 数据链路层 ─────────────────────────────────────────┐
│ 以太网帧 (tcpip/link)                                 │
│ [ETH Header | IP Header | TCP Header | Data | FCS]   │
│                                                       │
│ MAC 地址寻址，CRC 校验                                 │
│ 交换机根据 MAC 表转发 (osi/L2)                        │
└──────────────────────────┬────────────────────────────┘
                           ▼
┌─ 物理层 ─────────────────────────────────────────────┐
│ 比特流 (osi/L1)                                       │
│ 电信号 / 光信号 / 无线电波                              │
└──────────────────────────────────────────────────────┘
```

---

## 三、报文全景：逐层字段解剖

一个数据包从应用到线路，每经过一层就被添加该层的头部（有时还有尾部）。下面以字节级精度展示完整报文的字段归属，让你一眼看清**哪些字段属于哪层**。

### 报文 A：以太网 + IPv4 + TCP + HTTP（典型 Web 请求）

> 场景：浏览器发送 `GET /index.html` 到服务器，已经过 TCP 三次握手。

```
                ┌─── 前导码 (不属于任何层，物理层同步用) ───┐
                │  Preamble + SFD: 7+1 bytes               │
                └───────────────────────────────────────────┘

 ┌── L2 数据链路层 (14 bytes) ──────────────────────────────────────────┐
 │                                                                      │
 │  Dst MAC          Src MAC          EtherType                         │
 │  6 bytes          6 bytes          2 bytes                           │
 │  ┌──────────┐    ┌──────────┐    ┌──────┐                            │
 │  │AA:BB:...:01│    │CC:DD:...:02│    │0x0800│  ← 0x0800 = IPv4         │
 │  └──────────┘    └──────────┘    └──┬───┘                            │
 │                                      │                               │
 │  ┌───────────────────────────────────┼────────────────────────────┐  │
 │  │                                   ▼                            │  │
 │  │  ┌── L3 网络层: IPv4 (20 bytes，无选项) ────────────────────┐  │  │
 │  │  │                                                          │  │  │
 │  │  │  Ver│IHL│  DSCP/ECN  │     Total Length                  │  │  │
 │  │  │  4=IPv4│5=20B│  6 bits   │     2 bytes (整个IP包长度)     │  │  │
 │  │  │                                                           │  │  │
 │  │  │  Identification    │Flags│  Fragment Offset                │  │  │
 │  │  │  2 bytes           │ 3b  │  13 bits                       │  │  │
 │  │  │                                                           │  │  │
 │  │  │  TTL   │ Protocol  │     Header Checksum                  │  │  │
 │  │  │  1 byte│ 6 = TCP   │     2 bytes                          │  │  │
 │  │  │                                                           │  │  │
 │  │  │  Source IP Address (4 bytes)                               │  │  │
 │  │  │  192.168.1.100                                             │  │  │
 │  │  │                                                           │  │  │
 │  │  │  Destination IP Address (4 bytes)                          │  │  │
 │  │  │  93.184.216.34                                             │  │  │
 │  │  │                                                           │  │  │
 │  │  │  ┌────────────────────────────────────────────────────┐  │  │  │
 │  │  │  │  ┌── L4 传输层: TCP (20 bytes，无选项) ────────┐  │  │  │  │
 │  │  │  │  │                                              │  │  │  │  │
 │  │  │  │  │  Src Port          Dst Port                  │  │  │  │  │
 │  │  │  │  │  2 bytes           2 bytes                   │  │  │  │  │
 │  │  │  │  │  ┌─────┐          ┌─────┐                   │  │  │  │  │
 │  │  │  │  │  │54321│          │ 80  │  ← HTTP 端口       │  │  │  │  │
 │  │  │  │  │  └─────┘          └─────┘                   │  │  │  │  │
 │  │  │  │  │                                              │  │  │  │  │
 │  │  │  │  │  Sequence Number (4 bytes)                   │  │  │  │  │
 │  │  │  │  │  Acknowledgment Number (4 bytes)             │  │  │  │  │
 │  │  │  │  │                                              │  │  │  │  │
 │  │  │  │  │  Offset│Res│    Flags     │  Window          │  │  │  │  │
 │  │  │  │  │  4 bits│3b │ ACK PSH ...  │  2 bytes         │  │  │  │  │
 │  │  │  │  │                                              │  │  │  │  │
 │  │  │  │  │  Checksum (2B)  │  Urgent Pointer (2B)      │  │  │  │  │
 │  │  │  │  │                                              │  │  │  │  │
 │  │  │  │  │  ┌──────────────────────────────────────┐  │  │  │  │  │
 │  │  │  │  │  │  L5~L7 应用层: HTTP 请求              │  │  │  │  │  │
 │  │  │  │  │  │                                      │  │  │  │  │  │
 │  │  │  │  │  │  GET /index.html HTTP/1.1\r\n        │  │  │  │  │  │
 │  │  │  │  │  │  Host: www.example.com\r\n           │  │  │  │  │  │
 │  │  │  │  │  │  User-Agent: Mozilla/5.0\r\n         │  │  │  │  │  │
 │  │  │  │  │  │  Accept: text/html\r\n               │  │  │  │  │  │
 │  │  │  │  │  │  \r\n                                │  │  │  │  │  │
 │  │  │  │  │  │  (可变长度)                           │  │  │  │  │  │
 │  │  │  │  │  └──────────────────────────────────────┘  │  │  │  │  │
 │  │  │  │  └──────────────────────────────────────────────┘  │  │  │  │
 │  │  │  └────────────────────────────────────────────────────┘  │  │  │
 │  └─────────────────────────────────────────────────────────────┘  │  │
 │                                                                    │
 │  FCS (Frame Check Sequence): 4 bytes ← L2 尾部，CRC-32 校验       │
 └────────────────────────────────────────────────────────────────────┘
```

**各层头部大小汇总**：

| 层 | 协议 | 头部大小 | 关键字段 |
|----|------|---------|---------|
| L1 | 前导码 | 8 B | 同步时钟（不属帧） |
| L2 | Ethernet II | 14 B + 4 B (FCS) | MAC 地址、EtherType |
| L3 | IPv4 | 20~60 B | 源/目的 IP、TTL、Protocol |
| L4 | TCP | 20~60 B | 端口、序列号、标志位 |
| L7 | HTTP | 可变 | 方法、URI、头部、正文 |
| **总计** | | **≥78 B** | 最小帧 64 B（含 FCS） |

---

### 报文 B：以太网 + IPv4 + UDP + DNS（典型 DNS 查询）

> 场景：客户端查询 `www.example.com` 的 A 记录。

```
 ┌── L2: Ethernet (14 B) ───────────────────────────────────────────────┐
 │  Dst MAC (6B) │ Src MAC (6B) │ EtherType=0x0800 (2B)                │
 │                                                                       │
 │  ┌── L3: IPv4 (20 B) ──────────────────────────────────────────────┐ │
 │  │  Ver=4│IHL=5│DSCP│Len=45│ID│Flags│Frag│TTL=64│Proto=17│Chksum  │ │
 │  │                                              ↑                   │ │
 │  │                                         17 = UDP                 │ │
 │  │  Src IP: 192.168.1.100  │  Dst IP: 8.8.8.8                      │ │
 │  │                                                                  │ │
 │  │  ┌── L4: UDP (8 B) ─────────────────────────────────────────┐  │ │
 │  │  │  Src Port: 53421  │  Dst Port: 53   (DNS)                │  │ │
 │  │  │  Length: 25       │  Checksum                             │  │ │
 │  │  │                                                           │  │ │
 │  │  │  ┌── L7: DNS 查询 (12B 头 + 可变数据) ────────────────┐  │  │ │
 │  │  │  │  Transaction ID: 0xABCD  (2B)                      │  │  │ │
 │  │  │  │  Flags: 0x0100 (标准查询)  (2B)                     │  │  │ │
 │  │  │  │  Questions: 1     (2B)                              │  │  │ │
 │  │  │  │  Answer RRs: 0    (2B)                              │  │  │ │
 │  │  │  │  Authority RRs: 0 (2B)                              │  │  │ │
 │  │  │  │  Additional RRs: 0(2B)                              │  │  │ │
 │  │  │  │                                                     │  │  │ │
 │  │  │  │  Queries:                                           │  │  │ │
 │  │  │  │    3 www 7 example 3 com 0  (QNAME 编码)            │  │  │ │
 │  │  │  │    Type: A (1)  │  Class: IN (1)                    │  │  │ │
 │  │  │  └────────────────────────────────────────────────────┘  │  │ │
 │  │  └───────────────────────────────────────────────────────────┘  │ │
 │  └─────────────────────────────────────────────────────────────────┘ │
 │  FCS (4B)                                                            │
 └──────────────────────────────────────────────────────────────────────┘
```

**UDP vs TCP 头部对比**：

```
TCP (20+ bytes):                UDP (8 bytes):
┌────────┬────────┐             ┌────────┬────────┐
│Src Port│Dst Port│             │Src Port│Dst Port│
├────────┴────────┤             ├────────┴────────┤
│ Sequence Number │             │    Length       │
├─────────────────┤             ├─────────────────┤
│ Ack Number      │             │    Checksum     │
├────┬───┬────────┤             └─────────────────┘
│Off │Res│ Flags  │              ↑ 没有序列号、ACK、
├────┴───┴────────┤              ↑ 窗口、校验和可选
│    Window       │              → 适合小包、快速查询
├────────┬────────┤
│Checksum│Urgent  │
├────────┴────────┤
│   Options (可选) │
└─────────────────┘
```

---

### 报文 C：以太网 + MPLS + IPv4 + TCP（运营商骨干转发）

> 场景：MPLS 骨干网中 LSR 之间转发的数据包，带单层标签。

```
 ┌── L2: Ethernet (14 B) ──────────────────────────────────────────────┐
 │  Dst MAC (6B) │ Src MAC (6B) │ EtherType=0x8847 (MPLS unicast)     │
 │                                                   ↑                  │
 │                                          不是 0x0800！              │
 │                                          MPLS 直接承载在 L2 上      │
 │                                                                      │
 │  ┌── Layer 2.5: MPLS Shim Header (4 B) ─────────────────────────┐  │
 │  │                                                               │  │
 │  │  Label (20 bits)    │ EXP │S│ TTL                             │  │
 │  │  ┌──────────────┐   │ 3b  │1│ 8b                              │  │
 │  │  │   100200     │   │ 000 │1│ 63                               │  │
 │  │  │  (标签值)     │   │(QoS)│↑│(每跳-1)                         │  │
 │  │  └──────────────┘   │     │S=1: 栈底，后面无更多标签           │  │
 │  │                                                               │  │
 │  │  ┌── L3: IPv4 (20 B) ─────────────────────────────────────┐  │  │
 │  │  │  Ver│IHL│DSCP│Len│ID│Flags│Frag│TTL│Proto=6│Chksum     │  │  │
 │  │  │  Src IP: 10.20.0.100  │  Dst IP: 10.10.0.5             │  │  │
 │  │  │                                                         │  │  │
 │  │  │  ┌── L4: TCP (20 B) ───────────────────────────────┐  │  │  │
 │  │  │  │  Src Port │ Dst Port │ Seq │ ACK │ Flags │ Win   │  │  │  │
 │  │  │  │  ┌── L7: Application Data ──────────────────┐   │  │  │  │
 │  │  │  │  │  SSH / HTTP / 其他                        │   │  │  │  │
 │  │  │  │  └──────────────────────────────────────────┘   │  │  │  │
 │  │  │  └─────────────────────────────────────────────────┘  │  │  │
 │  │  └────────────────────────────────────────────────────────┘  │  │
 │  └───────────────────────────────────────────────────────────────┘  │
 │  FCS (4B)                                                            │
 └──────────────────────────────────────────────────────────────────────┘
```

**关键区别**：EtherType = `0x8847`（MPLS）而非 `0x0800`（IPv4），LSR 只看 MPLS 标签转发，不查 IP 路由表。

---

### 报文 D：以太网 + IPv4 + TCP + TLS + HTTP/2（HTTPS 加密请求）

> 场景：加密的 HTTPS 通信，TLS 在 TCP 和应用层之间。

```
 ┌── L2: Ethernet ─────────────────────────────────────────────────────┐
 │  Dst MAC │ Src MAC │ EtherType=0x0800                                │
 │                                                                      │
 │  ┌── L3: IPv4 ─────────────────────────────────────────────────────┐│
 │  │  ...│Proto=6 (TCP)│...│Src IP│Dst IP                            ││
 │  │                                                                 ││
 │  │  ┌── L4: TCP ─────────────────────────────────────────────────┐││
 │  │  │  Src Port │ Dst Port=443 │ Seq │ ACK │ Flags │ Win │ Chksum│││
 │  │  │                                                            │││
 │  │  │  ┌── TLS 1.3 Record Layer (5 B 头 + 加密载荷) ──────────┐│││
 │  │  │  │                                                       ││││
 │  │  │  │  Content Type: 0x17 (Application Data)  1B           ││││
 │  │  │  │  Version: 0x0303 (TLS 1.2 兼容)         2B           ││││
 │  │  │  │  Length: ...                              2B           ││││
 │  │  │  │                                                       ││││
 │  │  │  │  ┌── 加密区域 (密文，不可见) ───────────────────┐    ││││
 │  │  │  │  │                                              │    ││││
 │  │  │  │  │  实际内容: HTTP/2 Frame                      │    ││││
 │  │  │  │  │  ┌────────────────────────────────────────┐ │    ││││
 │  │  │  │  │  │  Length(3B)│Type(1B)│Flags(1B)│StreamID│ │    ││││
 │  │  │  │  │  │  HEADERS frame:                         │ │    ││││
 │  │  │  │  │  │    :method: GET                         │ │    ││││
 │  │  │  │  │  │    :path: /api/data                     │ │    ││││
 │  │  │  │  │  │    :authority: api.example.com          │ │    ││││
 │  │  │  │  │  │  (HPACK 头部压缩后)                     │ │    ││││
 │  │  │  │  │  └────────────────────────────────────────┘ │    ││││
 │  │  │  │  │                                              │    ││││
 │  │  │  │  │  + TLS 认证标签 (AEAD, 16B)                  │    ││││
 │  │  │  │  └──────────────────────────────────────────────┘    ││││
 │  │  │  └───────────────────────────────────────────────────────┘│││
 │  │  └───────────────────────────────────────────────────────────┘││
 │  └────────────────────────────────────────────────────────────────┘│
 │  FCS                                                                │
 └─────────────────────────────────────────────────────────────────────┘
```

**HTTPS 分层总结**：

```
层       │ 明文可见？ │ 中间设备能看到什么？
─────────┼────────────┼──────────────────────────────
L2       │ ✓          │ MAC 地址 (每跳变化)
L3       │ ✓          │ 源/目的 IP (NAT 可能改变源 IP)
L4       │ ✓          │ 源/目的端口 (通常 443)
TLS      │ ✗ (加密)   │ 只能看到 Content Type 和长度
HTTP/2   │ ✗ (加密)   │ 完全不可见 (除非 TLS 终止)
```

---

### 四种报文的层级对比总览

```
                报文 A         报文 B         报文 C         报文 D
               HTTP 明文      DNS 查询      MPLS 转发      HTTPS 加密
             ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  L2 以太网   │ Ethernet │  │ Ethernet │  │ Ethernet │  │ Ethernet │
             │  14 B    │  │  14 B    │  │  14 B    │  │  14 B    │
             ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤
  Layer 2.5  │    -     │  │    -     │  │ MPLS 4B  │  │    -     │
             ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤
  L3 网络层   │  IPv4    │  │  IPv4    │  │  IPv4    │  │  IPv4    │
             │  20 B    │  │  20 B    │  │  20 B    │  │  20 B    │
             ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤
  L4 传输层   │  TCP     │  │  UDP     │  │  TCP     │  │  TCP     │
             │  20+ B   │  │  8 B     │  │  20+ B   │  │  20+ B   │
             ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤
             │          │  │          │  │          │  │ TLS 5B   │
             │          │  │          │  │          │  │ +密文    │
             ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤
  L7 应用层   │  HTTP    │  │  DNS     │  │   App    │  │ HTTP/2   │
             │  可变    │  │  可变    │  │   数据   │  │ (加密)   │
             └──────────┘  └──────────┘  └──────────┘  └──────────┘
  EtherType  │  0x0800  │  │  0x0800  │  │  0x8847  │  │  0x0800  │
  IP Proto   │  6(TCP)  │  │ 17(UDP)  │  │  6(TCP)  │  │  6(TCP)  │
  最小总长   │  ≥78 B   │  ≥42+数据  │  ≥82 B     │  ≥83+密文  │
```

### QoS 在各层的体现

数据包经过的每一层都有自己的"优先级/服务质量"字段，它们之间互相映射，共同实现端到端的服务质量保障。

#### 同一含义，不同层名

```
                    QoS 标记在协议栈中的位置

  L2 数据链路层          802.1Q VLAN 标签中的 PCP (3 bit)
  ────────────────      ┌────────────┐
                        │ PCP │DEI│VID│   Priority Code Point
                        │ 3b  │1b │12b│   值 0~7，越高越优先
                        └────────────┘

  Layer 2.5              MPLS Shim Header 中的 EXP (3 bit)
  ────────────────      ┌────────────┐
                        │ Label │EXP│S│TTL│  Experimental bits
                        │ 20b   │3b │1│8b │  值 0~7，映射自 DSCP
                        └────────────┘

  L3 网络层              IPv4 头部中的 DSCP (6 bit) + ECN (2 bit)
  ────────────────      ┌────────────────┐
                        │  DSCP   │ ECN  │   Differentiated Services
                        │  6 bit  │ 2 bit│   Code Point + 拥塞通知
                        │  值 0~63│ 0~3  │
                        └────────────────┘

                        IPv6 的 Traffic Class (8 bit) 等同于 DSCP+ECN
```

#### 802.1Q VLAN 标签完整结构

普通以太网帧只看到 EtherType，但插入 802.1Q 标签后，多了优先级和 VLAN 信息：

```
普通帧:
  ┌──────────┬──────────┬──────────┬─────────────────┬─────┐
  │ Dst MAC  │ Src MAC  │EtherType │    Payload      │ FCS │
  │   6 B    │   6 B    │ 0x0800   │                 │ 4 B │
  └──────────┴──────────┴──────────┴─────────────────┴─────┘

802.1Q 带标签帧 (插入 4 字节):
  ┌──────────┬──────────┬──────────┬──────────────────────┬──────────┬─────────────────┬─────┐
  │ Dst MAC  │ Src MAC  │  TPID    │        TCI           │EtherType │    Payload      │ FCS │
  │   6 B    │   6 B    │ 0x8100   │                      │ 0x0800   │                 │ 4 B │
  └──────────┴──────────┴──────────┴──────────────────────┴──────────┴─────────────────┴─────┘
                                          │
                              TCI (Tag Control Information) 拆解:
                              ┌───────┬─────┬──────────────┐
                              │  PCP  │ DEI │     VID      │
                              │ 3 bit │1 bit│   12 bit     │
                              ├───────┼─────┼──────────────┤
                              │优先级 │丢弃 │ VLAN ID      │
                              │ 0~7   │ eligible │ 0~4095  │
                              └───────┴─────┴──────────────┘
```

#### PCP 优先级对照表

| PCP 值 | 优先级 | 典型用途 | 对应 DSCP |
|--------|--------|---------|-----------|
| 7 | 最高 | 网络控制（STP、OSPF） | CS7 (56) |
| 6 | 很高 | 网间控制（路由协议） | CS6 (48) |
| 5 | 高 | 语音 (VoIP) | EF (46) |
| 4 | 中高 | 视频会议 | AF41 (34) |
| 3 | 中 | 流媒体 | AF31 (26) |
| 2 | 中低 | 重要数据 | AF21 (18) |
| 1 | 低 | 背景流量 | AF11 (10) |
| 0 | 最低 | Best Effort（默认） | BE (0) |

#### DSCP 常用值 (PHB)

| PHB 名称 | DSCP 值 | 二进制 | 用途 |
|----------|---------|--------|------|
| **BE** | 0 | 000000 | Best Effort，默认无保障 |
| **EF** | 46 | 101110 | Expedited Forwarding，语音/实时，低延迟低抖动 |
| **AF41** | 34 | 100010 | 高优先视频 |
| **AF31** | 26 | 011010 | 流媒体 |
| **AF21** | 18 | 010010 | 重要业务数据 |
| **AF11** | 10 | 001010 | 低优先业务 |
| **CS6** | 48 | 110000 | 网络控制（路由协议） |
| **CS7** | 56 | 111000 | 网络最高优先（保留） |

> AF 命名规则：AF`xy`，x=类别(1~4)，y=丢弃优先(1~3)。x 越大越优先，y 越大越容易在拥塞时被丢。

#### ECN 拥塞通知

ECN (Explicit Congestion Notification) 用 IP 头的 2 bit + TCP 的 CWR/ECE 标志位实现**无丢包的拥塞通知**：

```
IP ECN (2 bit):
  00 = Not-ECT    不支持 ECN（默认）
  01 = ECT(1)     ECN 兼容传输
  10 = ECT(0)     ECN 兼容传输
  11 = CE         拥塞已发生（路由器标记）

ECN 工作流程:
  发送方                  路由器(拥塞)               接收方
    │                        │                        │
    │── ECT(0) 标记的包 ────→│                        │
    │                        │── 标记 CE ────────────→│
    │                        │  (不丢包，只标记)       │
    │                        │                        │── ACK + ECE ──→│
    │←── CWR (拥塞窗口减小) ──│                        │
    │                        │                        │
    ※ 传统方式: 拥塞→丢包→超时→重传 (慢)
    ※ ECN 方式: 拥塞→标记→通知→减速 (快，不丢包)
```

#### 跨层 QoS 标记传递

一个语音包从终端到运营商骨干，QoS 标记如何在各层间传递：

```
终端发出:
  IP DSCP = EF (46)     ← 应用层设置
    │
交换机 (802.1Q):
  根据 DSCP→PCP 映射表:
  VLAN PCP = 5 (语音)   ← 交换机自动映射
    │
MPLS 入口 (LER):
  根据 DSCP→EXP 映射表:
  MPLS EXP = 5           ← LER 自动映射
    │
MPLS 中间 (LSR):
  根据 EXP 值做队列调度:
  EXP=5 → 高优先队列 (LLQ)  ← LSR 按 EXP 转发
    │
MPLS 出口 (LER):
  Uniform 模式: EXP→DSCP 回写 → IP DSCP 仍为 EF
  Pipe 模式:    不回写 → IP DSCP 保持原始值
```

---

## 四、协议间的依赖与协作关系

协议不是孤立存在的，上层依赖下层提供的服务，同层协议之间也互相配合。

```
                    ┌─────────────────────────────────┐
                    │          应用层协议              │
                    │  HTTP  DNS  DHCP  FTP  SMTP SSH  │
                    └────┬────┬────┬────┬────┬───┬────┘
                         │    │    │    │    │   │
              ┌──────────┘    │    │    │    │   └──────────┐
              ▼               ▼    ▼    ▼    ▼              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                      传输层                                   │
  │                                                               │
  │   TCP ◄────────── HTTP, FTP, SMTP, SSH, IMAP                 │
  │    │    (需要可靠传输的应用)                                    │
  │    │                                                          │
  │   UDP ◄────────── DNS, DHCP, SNMP, RTP, QUIC/HTTP3           │
  │         (低延迟或广播场景)                                      │
  │                                                               │
  │   ICMP ◄─── 不基于 TCP/UDP，直接封装在 IP 数据报中              │
  │            Protocol=1，属于网络层的辅助协议                      │
  └──────┬──────────────────────────────────────┬─────────────────┘
         │                                      │
         ▼                                      ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                      网络层                                   │
  │                                                               │
  │   IP (v4/v6) ─── 为所有上层协议提供寻址和路由                   │
  │    │                                                          │
  │    ├── ICMP ─── 报告 IP 层的错误和控制信息                      │
  │    │           ping 用 Echo, traceroute 用 TTL Exceeded        │
  │    │                                                          │
  │    ├── IGMP ─── 管理多播组成员                                  │
  │    │                                                          │
  │    └── ARP ──── 为 IP 层解析下一跳的 MAC 地址                   │
  │              工作在 L2/L3 边界，用广播查询、单播应答              │
  └──────┬────────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │               Layer 2.5: MPLS 标签交换                        │
  │                                                               │
  │   MPLS ──── 在 IP 和链路层之间插入标签，指导快速转发            │
  │    │        LSP 路径由 LDP/RSVP-TE 建立                       │
  │    │        用于运营商骨干、MPLS VPN、流量工程                  │
  └────┬──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    网络接口层                                  │
  │                                                               │
  │   Ethernet (有线) ◄─── 帧格式、MAC 寻址、CSMA/CD              │
  │   Wi-Fi 802.11    ◄─── CSMA/CA、关联/认证过程                 │
  │   PPP             ◄─── 点对点链路、PPPoE 宽带接入              │
  │                                                               │
  │   交换机 / 网桥 ─── L2 转发、VLAN、STP 防环                   │
  └──────────────────────────────────────────────────────────────┘
```

### 关键关联说明

| 关系 | 说明 | 相关文档 |
|------|------|----------|
| **HTTP → TCP → IP** | HTTP 请求经 TCP 可靠传输，TCP 段封装在 IP 数据报中 | http → tcp → ip |
| **DNS → UDP → IP** | DNS 查询用 UDP（小包快速），响应超 512 字节则转 TCP | dns → udp → ip |
| **DHCP → UDP → IP** | DHCP 使用 UDP 广播（67/68），因为此时客户端还没有 IP | dhcp → udp |
| **ICMP → IP** | ICMP 直接封装在 IP 中（Protocol=1），不经过 TCP/UDP | icmp → ip |
| **ARP ↔ IP** | IP 发包前需知道下一跳 MAC，ARP 提供 IP→MAC 解析 | arp → ip |
| **traceroute → ICMP + IP TTL** | 利用 IP 的 TTL 字段递减机制 + ICMP 超时报文实现路径探测 | icmp → ip |
| **ping → ICMP Echo** | 直接使用 ICMP 的 Echo Request/Reply，不经过传输层 | icmp |
| **HTTP/3 → QUIC → UDP → IP** | HTTP/3 基于 QUIC，QUIC 在 UDP 之上实现可靠传输+加密 | http → udp → ip |
| **TLS → HTTP/HTTPS** | TLS 在传输层和应用层之间提供加密（OSI 表示层概念） | http, L6-presentation |
| **VLAN 标签 → 以太网帧** | 802.1Q 在以太网帧中插入 4 字节 VLAN 标签 | link, L2-datalink |
| **MPLS → IP + 链路层** | MPLS 标签栈插入在 IP 头和 L2 帧头之间（Layer 2.5） | mpls → ip, link |
| **VXLAN → L2 over UDP** | VXLAN 把 L2 帧封装在 UDP 4789 中，跨 L3 构建虚拟 L2（Overlay） | [vxlan](protocols/vxlan.md) |
| **VXLAN + BGP EVPN** | 现代数据中心 VXLAN 用 BGP EVPN 做控制面，分发 MAC/IP 绑定 | vxlan, L3 BGP |
| **MPLS VPN → BGP + VRF** | L3VPN 用双层标签 + MP-BGP 实现多租户隔离 | mpls |
| **MPLS-TE → RSVP-TE** | 流量工程通过 RSVP-TE 预留带宽、指定路径 | mpls |
| **IPsec → IP** | IPsec 在 IP 层插入 ESP/AH，提供逐跳或隧道加密（站点 VPN） | ip, [加密层级](protocols/encryption-layers.md) |
| **TLS → TCP + 应用** | TLS 1.3 在 L4/L7 之间提供端到端加密（HTTPS/SMTPS 等） | tcp, [加密层级](protocols/encryption-layers.md) |
| **MACsec → 以太网帧** | 802.1AE 在 L2 逐跳加密整个以太网帧 | L2-datalink, [加密层级](protocols/encryption-layers.md) |
| **差错控制贯穿全栈** | L1 FEC 纠错 → L2 CRC 检错 → L3 头部校验 → L4 TCP 可靠 → L7 业务重试 | [error-control](protocols/error-control.md) |

### 加密在各层的体现

加密可以部署在从 L1 到 L7 的每一层，越底层保护范围越大，越上层部署越灵活。

```
L7 应用层    PGP/Signal/ECH             ── 端到端，保护应用数据
L4 传输层    TLS 1.3 / QUIC             ── 端到端，保护字节流
L3 网络层    IPsec ESP · WireGuard      ── 逐跳或隧道，保护 IP 包
L2 链路层    MACsec · WPA3              ── 单跳，保护以太网帧/无线空口
L1 物理层    QKD · 线路加密器           ── 物理信号级
```

| 场景 | 推荐加密 |
|------|----------|
| HTTPS / Web 应用 | TLS 1.3 (L4) |
| 企业站点 VPN | IPsec 隧道模式 (L3) |
| 远程办公 VPN | WireGuard / IPsec (L3) |
| Wi-Fi 空口保护 | WPA3 (L2) |
| 数据中心 DCI | MACsec + IPsec (L2+L3) |
| 邮件端到端 | PGP/S/MIME (L7) + TLS (L4) |

> 详见 [protocols/encryption-layers.md](protocols/encryption-layers.md)：各层加密协议对比、IPsec VPN 完整工作流、WireGuard 解析、MACsec/WPA3 机制

### 差错控制在各层的体现

差错控制遵循**端到端原则**：中间层（L1-L3）尽力而为，真正的可靠性由端点（L4+）保证。

```
L7 应用层    HTTP 重试 · DNS 重传 · SHA/MD5 文件校验       ── 业务级校验
L4 传输层    TCP: 校验和+SEQ+ACK+SACK+RTO+FastRetransmit ── 端到端可靠
             UDP: 可选校验和, 不重传
L3 网络层    IPv4 头部校验和 · ICMP 错误报告              ── 只检头部
L2 链路层    Ethernet CRC-32 (只检不重传) · Wi-Fi ACK 重传 ── 单跳检错
L1 物理层    FEC (RS-FEC/KP4/LDPC) · 交织               ── 在线纠错
```

| 层级 | 检错 | 纠错/重传 | 失败行为 |
|------|------|----------|----------|
| L1 | FEC 编码冗余 | **在线纠错 (ns 级)** | 超出纠错能力 → 丢包 |
| L2 有线 | CRC-32 | ❌ 不重传 | 静默丢弃 |
| L2 无线 | CRC-32 | ✅ ACK + 重传 | 超时重传 |
| L3 | IPv4 头部校验 | ❌ 不重传 | ICMP 报告 + 丢包 |
| L4 TCP | 16 bit 校验 | ✅ RTO + 快速重传 | 重传 + 拥塞控制 |
| L4 UDP | 可选校验 | ❌ 不重传 | 静默丢弃 |
| L7 | 哈希/签名/状态码 | ✅ 应用重试 | 取决于应用 |

> 详见 [protocols/error-control.md](protocols/error-control.md)：各层差错控制机制对比、端到端原则、FEC/ARQ/AEAD 原理

---

## 五、目录结构

```
001-network-protocols/
├── README.md                  ← 本文件（全局关联地图）
│
├── osi/                       ← OSI 七层模型（理论视角）
│   ├── 00-overview.md         ← 七层总览 + 封装过程 + 与 TCP/IP 对应
│   ├── L7-application.md      ← 应用层 ──┐
│   ├── L6-presentation.md     ← 表示层    ├──→ tcpip/application.md
│   ├── L5-session.md          ← 会话层 ──┘
│   ├── L4-transport.md        ← 传输层 ──────→ tcpip/transport.md
│   ├── L3-network.md          ← 网络层 ──────→ tcpip/internet.md
│   ├── L2-datalink.md         ← 数据链路层 ─┐
│   └── L1-physical.md         ← 物理层 ─────┴→ tcpip/link.md
│
├── tcpip/                     ← TCP/IP 四层模型（工程视角）
│   ├── 00-overview.md         ← 四层总览 + 与 OSI 对应 + 设计原则
│   ├── application.md         ← 应用层协议族
│   ├── transport.md           ← 传输层（TCP/UDP 对比、拥塞控制）
│   ├── internet.md            ← 网际层（路由、NAT、ICMP、ARP）
│   └── link.md                ← 网络接口层（以太网、Wi-Fi、PPP）
│
├── protocols/                 ← 协议详解（纵向贯穿多层）
│   ├── tcp.md                 ← TCP ─── 传输层核心，可靠+有序+拥塞控制
│   ├── udp.md                 ← UDP ─── 传输层轻量替代，DNS/QUIC 的底层
│   ├── ip.md                  ← IP ─── 网络层核心，寻址+路由+分片
│   ├── icmp.md                ← ICMP ── 网络层辅助，直接封装在 IP 中
│   ├── arp.md                 ← ARP ─── L2/L3 桥梁，IP→MAC 解析
│   ├── dns.md                 ← DNS ─── 应用层，但使用 UDP/TCP 传输
│   ├── http.md                ← HTTP ── 应用层，经 TCP/TLS 或 QUIC 传输
│   ├── dhcp.md                ← DHCP ── 应用层，用 UDP 广播分配 IP
│   ├── mpls.md                ← MPLS ── Layer 2.5，标签交换，运营商骨干核心
│   ├── vxlan.md               ← VXLAN ── L2 over L3 Overlay，数据中心多租户主流
│   ├── encryption-layers.md   ← 加密 ── 各层加密协议对比，IPsec/TLS/MACsec/WireGuard
│   └── error-control.md       ← 差错 ── 各层差错控制对比，FEC/CRC/TCP 可靠传输
│
└── references/                ← 参考资料
    └── rfc-index.md           ← RFC 文档索引
```

---

## 六、推荐阅读路径

### 路径 A：从理论到实践（自顶向下）
```
osi/00-overview → osi/L7 → ... → osi/L1
  → tcpip/00-overview → tcpip/application → ... → tcpip/link
```

### 路径 B：追踪一个请求的完整旅程
```
protocols/http (应用层构造请求)
  → protocols/dns (解析域名)
  → protocols/tcp (建立连接、可靠传输)
  → protocols/ip (寻址路由)
  → protocols/arp (解析 MAC)
  → tcpip/link (帧封装与物理传输)
```

### 路径 C：按协议族深入
```
TCP 族: protocols/tcp → tcpip/transport → protocols/http → protocols/dns
IP 族:  protocols/ip → protocols/icmp → protocols/arp → tcpip/internet
接入族: tcpip/link → protocols/dhcp → osi/L1 → osi/L2
```

### 路径 D：按问题驱动
```
"ping 是怎么工作的？"    → protocols/icmp → protocols/ip → protocols/arp
"网页为什么能打开？"     → protocols/http → protocols/dns → protocols/tcp
"设备怎么自动获取 IP？"  → protocols/dhcp → protocols/arp → tcpip/link
"Wi-Fi 和有线网有什么区别？" → tcpip/link → osi/L1 → osi/L2
```

---

## 七、经典实战案例：数据路径剖析

以下案例追踪真实场景下的数据流转路径，标注每跳涉及的协议和网络设备，展示协议栈在实际工程中如何协同工作。

---

### 案例 1：用户访问异地负载均衡的 Web 服务

**场景**：北京用户访问 `https://www.example.com`，该站点使用 DNS 轮询实现北京/上海双机房负载均衡。

```
[北京用户浏览器]
 │
 │ ① DNS 解析 www.example.com
 │    DNS Query (UDP:53) → 本地 DNS → 根服务器 → .com TLD → 权威 DNS
 │    DNS Reply: www.example.com → 39.156.10.1 (北京节点, TTL=30)
 │    ※ DNS 根据来源 IP 返回最近的节点 IP（智能 DNS / GSLB）
 │
 │ ② TCP 三次握手 → 39.156.10.1:443
 │    SYN → [家用路由器 NAT] → [运营商城域网] → [骨干网] → [北京机房 LB]
 │    SYN+ACK ← 北京机房负载均衡器 (LVS / Nginx)
 │    ACK →
 │    ※ 数据经过多次路由跳转，每跳都经过 ARP 解析下一跳 MAC
 │
 │ ③ TLS 1.3 握手
 │    ClientHello → LB → (LB 终止 TLS 或透传至后端)
 │    ← ServerHello + Certificate + Finished
 │    → Finished
 │    ※ TLS 工作在应用层与传输层之间（OSI 表示层概念）
 │
 │ ④ HTTP/2 GET 请求
 │    → LB 根据负载均衡算法选择后端服务器 10.0.1.50
 │    → LB 与后端建立内部 TCP 连接（或直接转发）
 │    → 后端 10.0.1.50 处理请求，返回 HTTP Response
 │    ← HTTP/2 Response (200 OK) 经 LB 回传
 │
 ▼
[浏览器渲染页面]
```

**协议栈穿透**：
```
浏览器
 └→ HTTP/2 (应用层)
     └→ TLS 1.3 (加密层)
         └→ TCP (传输层，可靠+有序)
             └→ IP (网络层，公网路由 39.156.10.1)
                 └→ Ethernet/光纤 (链路层+物理层)
                     └→ 多次路由跳转（每跳 MAC 变化，IP 不变）
```

**关键设备与协议交互**：
| 设备 | 作用 | 涉及的协议 |
|------|------|-----------|
| 家用路由器 | NAT 转换、DHCP 分配内网 IP | NAT, DHCP, ARP |
| 运营商 DNS | 递归解析、智能调度 | DNS (UDP) |
| 骨干路由器 | BGP 选路、长距离转发 | BGP, OSPF, IP |
| 负载均衡器 | 四层/七层流量分发 | TCP, HTTP, TLS |
| 后端服务器 | 业务处理 | HTTP/2, TCP |

---

### 案例 2：通过 VPN 访问公司内网资源

**场景**：员工在家通过 IPSec VPN 连接公司内网，访问内网 Git 服务器 `10.10.0.5`。

```
[员工笔记本] (家庭网络 192.168.1.x)
 │
 │ ① VPN 隧道建立 (IPSec)
 │    Phase 1: IKE (UDP:500) — 身份认证 + 密钥协商
 │      Client (公网IP: 223.x.x.x) ↔ VPN Gateway (公司公网IP: 120.x.x.x)
 │    Phase 2: 建立 IPSec SA (ESP 隧道)
 │      ※ 此后所有内网流量被加密封装在 ESP 报文中
 │
 │ ② 访问 git clone git@10.10.0.5
 │    原始数据包:
 │      [TCP Header | SSH Data]
 │      Src: 10.20.0.100 (VPN 虚拟IP)  Dst: 10.10.0.5 (内网 Git)
 │
 │    IPSec 封装后 (隧道模式):
 │      [新 IP Header | ESP Header | 原 IP Header | TCP | SSH | ESP Trailer]
 │      Src: 223.x.x.x (员工公网IP)   Dst: 120.x.x.x (VPN 网关公网IP)
 │      ※ 外层 IP 用于公网路由，内层 IP 是真正的源/目的
 │
 │ ③ 数据路径
 │    笔记本 → 家庭路由器(NAT) → ISP → 互联网 → 公司防火墙
 │      → 公司 VPN 网关(解封 ESP) → 公司核心交换机
 │      → 内网 Git 服务器 10.10.0.5
 │
 │ ④ 响应返回 (反向路径)
 │    Git 服务器 → VPN 网关(ESP 封装) → 互联网 → 笔记本(解封)
 ▼
```

**双层封装示意**：
```
                     ┌─── 公网可见 ──────────────────────────────┐
                     │                                           │
[外层 IP: 公网路由]   │  [ESP Header] [内层 IP: 10.20→10.10]     │
Src: 223.x.x.x      │  (加密)       [TCP | SSH | Git Data]      │
Dst: 120.x.x.x      │               (原始内网包，被加密保护)      │
                     │                                           │
                     └───────────────────────────────────────────┘

 中间路由器只看到外层 IP，不知道内部通信的是 10.20.0.100 ↔ 10.10.0.5
 VPN 网关解封后，取出内层包转发到内网
```

**关键协议协作**：
| 阶段 | 协议 | 说明 |
|------|------|------|
| 密钥协商 | IKE (UDP:500/4500) | Diffie-Hellman 密钥交换 |
| 数据加密 | ESP (IP Protocol 50) | 封装安全载荷，加密+认证 |
| NAT 穿透 | UDP Encapsulation (4500) | ESP 包在 UDP 中穿越 NAT |
| 内网路由 | OSPF / 静态路由 | VPN 网关到 Git 服务器的内网路由 |

---

### 案例 3：公网用户 → 内网 Web → 数据库 的典型三层架构

**场景**：公网用户下单，请求经 CDN → WAF → Nginx → 应用服务器 → 数据库，横跨公网、DMZ、内网三个网络区域。

```
[公网用户手机] (4G/5G, 运营商分配公网 IP)
 │
 │ ① HTTPS 请求 api.shop.com/api/order
 │    DNS 解析 → CNAME → CDN 边缘节点 IP
 │
 │ ② CDN 边缘节点 (就近接入)
 │    TCP + TLS → CDN 节点
 │    CDN 检查缓存 → 未命中 → 回源
 │
 │ ③ CDN 回源 → WAF (Web 应用防火墙)
 │    CDN → [互联网骨干] → 公司公网入口
 │    WAF 检查: SQL 注入、XSS、频率限制
 │    WAF 透传 → DMZ 区 Nginx
 │
 │ ④ Nginx 反向代理 (DMZ 区)
 │    Nginx (172.16.1.10) → upstream 应用服务器集群
 │    选择 app-server-03 (10.0.2.30)
 │    ※ DMZ 与内网之间有防火墙策略，仅允许特定端口
 │
 │ ⑤ 应用服务器处理 (内网区)
 │    app-server-03 执行下单逻辑
 │    → 连接 MySQL 主库 10.0.3.10:3306
 │      [TCP 三次握手] → [MySQL 协议]
 │      INSERT INTO orders ...
 │    → 连接 Redis 10.0.3.20:6379 更新库存缓存
 │      [TCP] → [RESP 协议]
 │      DECR stock:item_001
 │
 │ ⑥ 响应原路返回
 │    MySQL ACK → app-server → Nginx → WAF → CDN → 用户
 ▼
```

**网络区域与安全边界**：
```
 ┌─────────┐    ┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
 │  公网    │    │  CDN    │    │   DMZ    │    │   内网    │    │ 数据库区  │
 │         │    │  边缘   │    │          │    │  应用层   │    │          │
 │ 用户    │───→│  节点   │───→│ WAF      │───→│ App      │───→│ MySQL    │
 │ 手机    │    │         │    │ Nginx    │    │ Server   │    │ Redis    │
 │         │    │         │    │          │    │          │    │          │
 │公网 IP  │    │CDN IP   │    │172.16.x  │    │10.0.2.x  │    │10.0.3.x  │
 └─────────┘    └─────────┘    └──────────┘    └───────────┘    └──────────┘
      │              │              │                │               │
   运营商网络    CDN 骨干网     防火墙①         防火墙②         数据库防火墙
                              (公网→DMZ)     (DMZ→内网)      (内网→DB区)
```

**每一跳的协议栈变化**：
| 跳转 | 源 IP | 目的 IP | 传输层 | 应用层 | 特殊处理 |
|------|-------|---------|--------|--------|----------|
| 用户→CDN | 用户公网IP | CDN边缘IP | TCP+TLS | HTTPS | CDN Anycast |
| CDN→WAF | CDN回源IP | 公司公网IP | TCP+TLS | HTTPS | X-Forwarded-For |
| WAF→Nginx | WAF内网IP | 172.16.1.10 | TCP | HTTP | 反向代理 |
| Nginx→App | 172.16.1.10 | 10.0.2.30 | TCP | HTTP | upstream 选择 |
| App→MySQL | 10.0.2.30 | 10.0.3.10 | TCP | MySQL协议 | 连接池 |
| App→Redis | 10.0.2.30 | 10.0.3.20 | TCP | RESP协议 | 单线程模型 |

---

### 案例 4：内网 DHCP 获取网络配置 + ARP 冲突检测

**场景**：一台新电脑插入公司有线网络，从零开始获得网络连通性。

```
[新电脑开机，网卡启动]
 │
 │ ① 物理层协商
 │    自动协商速率: 1000BASE-T 全双工 (osi/L1)
 │    ※ 双绞线 Cat5e 以上才支持千兆
 │
 │ ② DHCP 获取 IP (四步 DORA)
 │    此时电脑没有任何 IP 地址！
 │
 │    Discover: src=0.0.0.0:68 → dst=255.255.255.255:67 (广播)
 │      "我是 MAC=AA:BB:CC:DD:EE:FF，谁能给我分配 IP？"
 │      ※ 交换机泛洪广播帧到所有端口 (osi/L2)
 │
 │    Offer: DHCP 服务器 10.0.0.1 响应
 │      "给你 10.0.2.88/24，网关 10.0.0.1，DNS 10.0.0.2，租约 8h"
 │
 │    Request: 广播接受 (通知其他 DHCP 服务器)
 │
 │    ACK: 确认分配，配置生效
 │      ※ 同时获得: IP, 掩码, 网关, DNS, NTP, 域名
 │
 │ ③ 免费 ARP 冲突检测
 │    电脑发送 ARP Request: "Who has 10.0.2.88? Tell 10.0.2.88"
 │      源 IP = 目标 IP = 自己的 IP
 │      ※ 如果收到 Reply，说明 IP 已被占用，需要重新 DHCP
 │      ※ 同时通知网络中所有设备更新 ARP 缓存
 │
 │ ④ 首次访问外网
 │    ping 8.8.8.8
 │    → ARP: 网关 10.0.0.1 的 MAC 地址？
 │      ARP Request (广播) → 网关 Reply: 00:1A:2B:3C:4D:5E
 │    → 构造 ICMP Echo Request
 │      [ETH: dst=网关MAC] [IP: dst=8.8.8.8, TTL=64] [ICMP: Echo]
 │    → 帧发往交换机 → 交换机根据 MAC 表转发到网关端口
 │    → 路由器查路由表 → 下一跳 → ARP → 继续转发...
 ▼
```

**从"零"到"通"的完整协议链**：
```
物理层自动协商 (L1)
  → DHCP 获取 IP 配置 (L7, 基于 UDP 广播)
    → 免费 ARP 检测冲突 (L2/L3 边界)
      → ARP 解析网关 MAC (L2/L3 边界)
        → ICMP Echo 测试连通 (直接封装在 IP 中)
          → IP 路由到外网 (L3, 路由器逐跳转发)
            → BGP/OSPF 选路 (路由器间的路由协议)
```

---

### 案例 5：同一数据中心内的东西向流量（微服务调用）

**场景**：K8s 集群中，前端 Pod (10.244.1.5) 调用后端微服务 Pod (10.244.2.8)，流量经过 Overlay 网络。

```
[前端 Pod 10.244.1.5] (Node A: 192.168.1.10)
 │
 │ ① HTTP 请求
 │    GET http://user-service:8080/api/users/123
 │    DNS 解析 (CoreDNS) → 10.96.0.50 (ClusterIP)
 │    kube-proxy (iptables) DNAT → 10.244.2.8:8080 (后端 Pod)
 │
 │ ② Overlay 封装 (如 VXLAN / Calico)
 │    原始包:
 │      [TCP | HTTP Data]  Src: 10.244.1.5 → Dst: 10.244.2.8
 │
 │    VXLAN 封装后:
 │      [外层 ETH] [外层 IP: 192.168.1.10 → 192.168.1.20]
 │      [VXLAN Header] [内层 ETH] [内层 IP: 10.244.1.5 → 10.244.2.8]
 │      [TCP | HTTP Data]
 │      ※ Pod 网络是虚拟的，实际在宿主机物理网络上传输
 │
 │ ③ 物理网络传输
 │    Node A (192.168.1.10) → ToR 交换机 → Node B (192.168.1.20)
 │    ※ 交换机只看到外层 IP 和 MAC，不知道 Pod 网络
 │
 │ ④ Node B 解封
 │    去除 VXLAN 封装 → 取出原始包 → 转发到目标 Pod
 ▼
[后端 Pod 10.244.2.8]
```

**Overlay 双层封装**：
```
┌─── 物理网络 (Underlay) ──────────────────────────────────────┐
│                                                               │
│  外层 ETH  │  外层 IP        │ VXLAN │ 内层 ETH │ 内层 IP    │
│  Node MAC  │ 192.168.1.10   │Header │ Pod MAC  │ 10.244.1.5 │
│  → ToR MAC │ → 192.168.1.20 │       │          │ →10.244.2.8│
│            │                 │       │          │            │
│            │  交换机据此转发  │       │  Pod 网络据此路由      │
│            │  (Underlay)    │       │  (Overlay)           │
│                                                               │
│  [TCP Header | HTTP Request Data]                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

### 案例路径总结

| 案例 | 核心看点 | 涉及的关键协议 |
|------|---------|---------------|
| ① 异地负载均衡 | DNS 智能调度、NAT、多层代理、TLS 终止 | DNS, TCP, TLS, HTTP, BGP, NAT |
| ② VPN 远程接入 | 隧道封装/解封、双层 IP、加密传输 | IPSec/ESP, IKE, UDP, IP |
| ③ 三层架构穿透 | 安全区域隔离、防火墙策略、多协议栈切换 | HTTPS, MySQL, RESP, TCP |
| ④ 从零获取连通性 | DHCP 四步流程、ARP 冲突检测、首次路由 | DHCP, ARP, ICMP, IP, Ethernet |
| ⑤ 数据中心东西向 | Overlay 网络、虚拟 IP、双层封装 | VXLAN, IP, TCP, HTTP, ARP |
