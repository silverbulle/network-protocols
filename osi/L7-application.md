# L7 - 应用层 (Application Layer)

## 职责
为用户应用程序提供网络服务接口，是用户与网络交互的最顶层。

## 核心功能
- **网络服务接口**：为应用提供访问网络资源的标准接口
- **资源访问**：文件传输、电子邮件、远程登录等
- **目录服务**：分布式数据库查询（如 DNS）

## 主要协议

### HTTP / HTTPS
- **端口**：80 / 443
- **用途**：Web 内容传输
- **特点**：请求-响应模型，无状态（HTTP），HTTPS 基于 TLS 加密
- **详见**：`protocols/http.md`

### FTP / SFTP
- **端口**：21（控制）/ 20（数据）/ 22（SFTP）
- **用途**：文件传输
- **特点**：双连接（控制连接 + 数据连接），支持主动/被动模式

### SMTP / POP3 / IMAP
- **端口**：25 / 110 / 143
- **用途**：电子邮件发送与接收
- **特点**：SMTP 负责发送，POP3/IMAP 负责接收；IMAP 支持服务端管理

### DNS
- **端口**：53
- **用途**：域名到 IP 地址的解析
- **特点**：UDP 为主，大包用 TCP；递归查询 + 迭代查询
- **详见**：`protocols/dns.md`

### SSH / Telnet
- **端口**：22 / 23
- **用途**：远程终端访问
- **特点**：SSH 加密，Telnet 明文（已不推荐使用）

### SNMP
- **端口**：161（UDP）
- **用途**：网络设备管理
- **特点**：基于 MIB 的管理信息结构

## 报文示例（HTTP 请求）
```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html
Connection: keep-alive
```

---

## 应用层协议依赖关系详解

应用层协议不是孤立存在的，每个协议都依赖下层提供的服务，同时彼此之间也形成复杂的依赖网络。理解这些依赖是排障、抓包、设计系统的关键。

### 1. 协议栈全景图

```
┌────────────────────────────────────────────────────────────┐
│ 应用层 (L7)                                                 │
│                                                             │
│   HTTP/1.1 HTTP/2   FTP    SMTP POP3 IMAP   SSH    SNMP     │
│   DNS      DHCP   NTP    LDAP RADIUS       Telnet          │
│                                                             │
├────────────────────────────────────────────────────────────┤
│ 安全层 (L6 概念)                                            │
│   TLS 1.2/1.3    SSH Transport    IPsec ESP    MACsec       │
│   (HTTPS/SMTPS/IMAPS/FTPS/LDAPS/DoT)                       │
│                                                             │
├────────────────────────────────────────────────────────────┤
│ QUIC (UDP 上自带加密, 替代 TCP+TLS)                         │
│   HTTP/3 · WebTransport · DNS over QUIC                     │
│                                                             │
├────────────────────────────────────────────────────────────┤
│ 传输层 (L4)                                                 │
│   TCP (可靠)              UDP (轻量)                        │
│   HTTP/FTP/SMTP/SSH/...   DNS/DHCP/NTP/SNMP/QUIC           │
│                                                             │
├────────────────────────────────────────────────────────────┤
│ 网络层 (L3):  IP (v4/v6) · ICMP · ARP                       │
├────────────────────────────────────────────────────────────┤
│ 链路层 (L2):  Ethernet · Wi-Fi · PPP · VLAN                 │
└────────────────────────────────────────────────────────────┘
```

### 2. 各协议依赖矩阵

| 协议 | 传输层 | 加密/安全 | 依赖 DNS | 依赖其他 L7 | 端口 |
|------|--------|-----------|----------|-------------|------|
| **HTTP/1.1, HTTP/2** | TCP | 可选 TLS (HTTPS) | ✅ 必需 | — | 80 / 443 |
| **HTTP/3** | UDP (经 QUIC) | QUIC 内置加密 | ✅ | — | 443 |
| **WebSocket** | TCP | 可选 TLS (wss://) | ✅ | 建立在 HTTP 升级之上 | 80 / 443 |
| **gRPC** | TCP | 可选 TLS | ✅ | 建立在 HTTP/2 之上 | 任意 |
| **FTP (Control)** | TCP | 可选 TLS (FTPS) | ✅ | — | 21 |
| **FTP (Data)** | TCP | 同 Control | — | 由 FTP Control 协商 | 20 (Active) / 随机 (Passive) |
| **SFTP** | TCP | SSH 加密 | ✅ | 运行在 SSH 通道上 | 22 |
| **SCP** | TCP | SSH 加密 | ✅ | SSH | 22 |
| **SMTP** | TCP | 可选 STARTTLS | ✅ (MX) | — | 25 / 587 / 465 |
| **POP3** | TCP | 可选 TLS | ✅ | — | 110 / 995 |
| **IMAP** | TCP | 可选 TLS | ✅ | — | 143 / 993 |
| **DNS** | **UDP 优先，TCP 备用** | 可选 DoT/DoH | — | — | 53 / 853 (DoT) |
| **DoH (DNS over HTTPS)** | TCP | TLS | ✅ | **依赖 HTTP/2** | 443 |
| **DoT (DNS over TLS)** | TCP | TLS | — | — | 853 |
| **DoQ (DNS over QUIC)** | UDP | QUIC | — | — | 853 |
| **DHCP** | UDP | ❌ | — | — | 67/68 (v4) · 546/547 (v6) |
| **DHCPv6** | UDP | ❌ | — | 可依赖 NDP | 546/547 |
| **NTP** | UDP | 可选 NTS (Network Time Security) | ✅ | — | 123 |
| **SNMP** | UDP | v3 自带加密 (USM) | 可选 | — | 161/162 |
| **SSH** | TCP | **自含 SSH Transport 加密** | ✅ | — | 22 |
| **Telnet** | TCP | ❌ 明文 | ✅ | — | 23 |
| **LDAP** | TCP | 可选 TLS (LDAPS) / StartTLS | ✅ | — | 389 / 636 |
| **RADIUS** | UDP | 弱 (共享密钥 + 属性隐藏) | ✅ | — | 1812/1813 |
| **TACACS+** | TCP | 部分加密 | ✅ | — | 49 |
| **Kerberos** | UDP/TCP | 对称加密 (票据) | ✅ | — | 88 |
| **SIP** | UDP/TCP | 可选 TLS (SIPS) + SRTP 媒体加密 | ✅ | — | 5060 / 5061 |
| **RTP / RTCP** | UDP | 可选 SRTP | — | SIP/H.323 协商 | 动态偶数端口 |
| **MQTT** | TCP | 可选 TLS | ✅ | — | 1883 / 8883 |
| **AMQP** | TCP | 可选 TLS | ✅ | — | 5672 / 5671 |
| **QUIC** | UDP | **强制加密** | — | — | 任意 |

### 3. 典型依赖链剖析

#### 链 A：HTTPS 请求（经典 Web）
```
浏览器访问 https://example.com
  │
  ├─ [1] DNS (UDP 53)              → 解析 example.com 的 IP
  │    └─ 依赖: UDP + IP
  │
  ├─ [2] TCP 三次握手 (TCP 443)    → 建立可靠连接
  │    └─ 依赖: IP + 路由 + ARP
  │
  ├─ [3] TLS 1.3 握手              → 协商密钥 + 证书验证
  │    └─ 依赖: TCP + 证书链（X.509 + OCSP/CRL 可能再触发 HTTP）
  │
  ├─ [4] HTTP/2 请求/响应          → 加密通道内的应用数据
  │    └─ 依赖: TLS + TCP
  │
  └─ [5] (可能触发) WebSocket/HTTP/2 Push/资源加载（再次 DNS+TCP+TLS）
```

#### 链 B：HTTP/3（现代 Web）
```
浏览器访问 https://example.com (HTTP/3)
  │
  ├─ [1] DNS (UDP 53)              → 解析 IP + 检测 Alt-Svc
  │
  ├─ [2] QUIC 握手 (UDP 443)       → 一个 RTT 内完成连接+加密
  │    └─ 依赖: UDP + IP
  │    └─ QUIC 自带: 可靠传输 + 加密 + 多路复用（无需 TCP+TLS）
  │
  └─ [3] HTTP/3 请求/响应          → 跑在 QUIC Stream 上
```

> HTTP/3 合并了传统链的 [2]+[3]+[4] 三步，这是 0-RTT 提速的根本

#### 链 C：邮件收发
```
发件人 → MUA ──SMTP──► MSA ──SMTP──► MX ──SMTP──► 收件 MTA
                          │                          │
                          └─ STARTTLS 加密            └─ 收件人 MUA 拉取
                                                         │
                                                ┌────────┴────────┐
                                                │ IMAP (143/993)  │
                                                │ 或 POP3 (110/995)│
                                                └─────────────────┘

  SMTP 发送依赖:
    - DNS MX 记录查询（找到目标 MTA）
    - DNS A/AAAA（解析 MX 主机名）
    - DNS SPF TXT 记录（反垃圾）
    - DNS DKIM TXT 记录（签名）
    - DNS DMARC TXT 记录（策略）
    - OCSP（验证证书）
```

#### 链 D：SSH 远程登录
```
ssh user@host.example.com
  │
  ├─ [1] DNS: 解析主机名
  │
  ├─ [2] TCP: 三次握手 (22)
  │
  ├─ [3] SSH Transport Layer:
  │     - 密钥交换 (ECDH/Curve25519)
  │     - 服务器主机密钥认证 (ed25519/rsa)
  │     - 协商加密算法 (chacha20-poly1305 / aes-gcm)
  │
  ├─ [4] SSH Authentication Layer:
  │     - 公钥认证 / 密码 / GSSAPI (Kerberos)
  │
  └─ [5] SSH Connection Layer:
        - 多个 Channel (session, port-forward, SFTP subsystem)
        - SFTP 实际是 SSH Channel 里的子系统
        - SCP 也是 SSH Channel
```

#### 链 E：Wi-Fi + 802.1X + DHCP + HTTP
```
笔记本连企业 Wi-Fi
  │
  ├─ [L2] Wi-Fi 关联 (Open / WPA3-SAE)
  │
  ├─ [L2] 802.1X/EAP 认证 (端口未认证前只允许 EAPoL)
  │     └─ Supplicant ↔ Authenticator (AP) ↔ RADIUS Server
  │     └─ RADIUS 查询 LDAP/AD 用户库
  │     └─ 认证成功 → 动态下发 VLAN + 密钥
  │
  ├─ [L3] DHCP 获取 IP (UDP 67/68)
  │     └─ 依赖 L2 已通过认证
  │
  ├─ [L3] DNS 解析 (UDP 53)
  │
  └─ [L7] 访问 HTTPS (TLS + HTTP/2)
```

#### 链 F：VoIP 通话 (SIP + RTP)
```
A 呼叫 B
  │
  ├─ [1] SIP INVITE (UDP/TCP 5060 或 TLS 5061)
  │     ├─ DNS 查 SIP 服务器 (SRV 记录 _sip._udp.example.com)
  │     └─ SDP 携带媒体能力（编解码、端口、RTP 参数）
  │
  ├─ [2] SIP 服务器转发 INVITE → B
  │     └─ 可能走 SIP Trunk (跨运营商)
  │
  ├─ [3] B 回 180 Ringing / 200 OK
  │
  ├─ [4] A 发 ACK，开始媒体传输
  │     └─ RTP/UDP 偶数端口 → 语音流（实时，不容忍重传）
  │     └─ RTCP/UDP 相邻端口 → 控制/统计
  │     └─ 可选 SRTP 加密
  │
  └─ [5] BYE 结束通话

  RTP 为什么用 UDP 不用 TCP？
    - 语音/视频延迟敏感，重传会破坏实时性
    - 丢几包比卡顿 200ms 更可接受
```

### 4. 协议家族与分层

#### 基于 TCP 的"文本协议"家族
```
┌────────────┐  ┌─────────┐  ┌──────────┐
│ HTTP/1.1   │  │ SMTP    │  │ FTP Ctrl │   文本命令 + 响应码
└─────┬──────┘  └────┬────┘  └────┬─────┘
      │              │            │
      └──────────────┼────────────┘
                     ▼
                    TCP
                     ▼
                    IP
```

#### 基于 HTTP 的"二次协议"家族（现代 Web 架构）
```
┌─────────┐  ┌──────────┐  ┌───────────┐  ┌─────────┐  ┌───────┐
│ gRPC    │  │ WebSocket│  │ DoH       │  │ SSE     │  │ GraphQL│
│ (HTTP/2)│  │ (HTTP升) │  │ (HTTP/2)  │  │(HTTP流式)│  │(HTTP) │
└────┬────┘  └────┬─────┘  └────┬──────┘  └────┬────┘  └───┬───┘
     │            │             │              │            │
     └────────────┴─────┬───────┴──────────────┴────────────┘
                        ▼
                    HTTP / HTTP/2 / HTTP/3
                        ▼
              ┌──────────┴───────────┐
              ▼                      ▼
           TCP+TLS                QUIC (UDP)
```

#### 基于 SSH 的子系统家族
```
┌──────┐  ┌──────┐  ┌──────┐  ┌────────┐  ┌────────┐
│ SFTP │  │ SCP  │  │ SSH  │  │ SSH    │  │ SSH    │
│      │  │      │  │ Shell│  │ Tunnel │  │ Agent  │
└──┬───┘  └──┬───┘  └──┬───┘  └───┬────┘  └───┬────┘
   │         │         │          │            │
   └─────────┴────┬────┴──────────┴────────────┘
                  ▼
         SSH Connection Layer (Channel)
                  ▼
         SSH Authentication Layer
                  ▼
         SSH Transport Layer (加密 + MAC)
                  ▼
                TCP
```

#### AAA 协议家族（认证/授权/计费）
```
┌──────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐
│ RADIUS   │  │ TACACS+   │  │ Diameter  │  │ LDAP/AD  │
│ (UDP)    │  │ (TCP)     │  │ (TCP/SCTP)│  │ (TCP)    │
└─────┬────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘
      └──────────────┼──────────────┼─────────────┘
                     ▼
              被以下协议调用:
         802.1X · PPP CHAP/EAP · SSH · VPN
```

### 5. DNS —— 几乎所有 L7 协议的隐式依赖

DNS 是最被低估的 L7 基础设施。**除了 DHCP、纯 IP 直连场景，几乎所有 L7 协议都依赖 DNS**：

| DNS 记录类型 | 被哪些 L7 协议使用 |
|-------------|-------------------|
| **A / AAAA** | HTTP、SMTP、SSH、FTP、LDAP、MQTT、NTP... 几乎所有 |
| **MX** | SMTP（找邮件服务器） |
| **SRV** | SIP (_sip._tcp)、LDAP (_ldap._tcp)、Minecraft、Kerberos |
| **TXT** | SPF / DKIM / DMARC（邮件反垃圾）、ACME（Let's Encrypt 证书） |
| **PTR** | SMTP 反查（反垃圾）、日志反解析 |
| **NAPTR** | SIP、ENUM（电话号码→SIP URI） |
| **CNAME** | 别名（CDN、负载均衡） |
| **SOA / NS** | 区域管理 |

> **DNS 故障 = 所有依赖它的 L7 协议一起瘫痪**。这就是为什么企业网会部署本地 DNS 缓存（如 CoreDNS、Unbound）

### 6. 安全依赖链

| 加密/安全协议 | 依赖 | 被依赖 |
|--------------|------|--------|
| **TLS 1.2/1.3** | TCP + 证书 (X.509) + 随机数源 | HTTPS, SMTPS, IMAPS, POP3S, FTPS, LDAPS, DoT, gRPC |
| **SSH Transport** | TCP + 主机密钥 | SSH, SFTP, SCP, SSH Tunnel |
| **QUIC** | UDP + 证书 | HTTP/3, DoQ, WebTransport |
| **IPsec ESP** | IP + IKEv2 (UDP 500/4500) | 站点 VPN、远程 VPN |
| **MACsec** | L2 + MKA | 数据中心交换机互联 |
| **STARTTLS** | 明文协议 + TLS 升级 | SMTP/IMAP/LDAP/FTP（先明文后加密） |
| **SRTP** | UDP + 密钥（由 SIP/SDP 协商） | VoIP 媒体流 |

### 7. 隐式依赖与陷阱

| 现象 | 隐式依赖 | 排障思路 |
|------|---------|----------|
| HTTP 请求超时 | 可能是 DNS 超时，不是 TCP 问题 | 先 `dig` 看 DNS |
| HTTPS 证书错误 | 依赖系统时间 + 根证书库 | 检查 NTP 同步、证书链 |
| SMTP 邮件被拒 | 依赖 PTR + SPF + DKIM + DMARC | 查所有 4 种 DNS 记录 |
| FTP 被动模式连不上数据端口 | 依赖 NAT/防火墙放行随机端口 | 检查 FTP ALG、被动端口范围 |
| SIP 呼叫单通 | RTP 端口被防火墙挡 | SIP SDP 协商的 IP/端口要可达 |
| Kerberos 失败 | **时钟偏差 > 5 分钟就失败** | 必须配 NTP |
| DoH 首次连接 | 依赖 DNS 解析 DoH 服务器自己（鸡生蛋） | 常用 bootstrap IP 或 HTTPS DNS 记录 |
| QUIC 被阻断 | UDP 443 被运营商防火墙丢 | 自动回退 HTTP/2 over TCP |

### 8. 协议演进：从明文到加密

| 明文版本 | 加密版本 | 加密机制 | 端口变化 |
|----------|---------|---------|---------|
| HTTP | HTTPS | TLS | 80 → 443 |
| FTP | FTPS | TLS (Implicit/Explicit) | 21 → 990 (Imp) 或 21 + STARTTLS |
| FTP | SFTP | SSH | 21 → 22 (完全不同的协议) |
| SMTP | SMTPS / STARTTLS | TLS | 25 → 465 / 587 |
| POP3 | POP3S | TLS | 110 → 995 |
| IMAP | IMAPS | TLS | 143 → 993 |
| LDAP | LDAPS | TLS | 389 → 636 |
| Telnet | SSH | SSH | 23 → 22 |
| DNS | DoT / DoH / DoQ | TLS/HTTPS/QUIC | 53 → 853/443/853 |
| SNMP v1/v2c | SNMPv3 | USM (HMAC+AES) | 161 (同端口) |
| Syslog | Syslog over TLS | TLS | 514 → 6514 |
| MQTT | MQTTS | TLS | 1883 → 8883 |

> 现代最佳实践：**默认强制加密**。TLS 1.3 已经接近明文连接的性能（1-RTT / 0-RTT）

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| TCP/IP 对应 | [tcpip/application.md](../tcpip/application.md) | L5+L6+L7 合并为 TCP/IP 应用层 |
| 下层 | [L6 表示层](L6-presentation.md) | 数据格式转换、加密（TLS 概念上属于此层） |
| 相关协议 | [HTTP](../protocols/http.md) · [DNS](../protocols/dns.md) · [DHCP](../protocols/dhcp.md) | 典型应用层协议 |
| 依赖传输层 | [L4 传输层](L4-transport.md) | 应用层协议需要传输层提供端到端服务 |
| L7 协议依赖详解 | [协议依赖关系详解](#应用层协议依赖关系详解) | 依赖矩阵、典型依赖链、协议家族、安全依赖、隐式陷阱 |
| 加密层级 | [protocols/encryption-layers.md L7](../protocols/encryption-layers.md#l7-应用层加密) | PGP/Signal/ECH 等端到端加密 |
