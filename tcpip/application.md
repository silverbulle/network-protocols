# TCP/IP - 应用层 (Application Layer)

## 概述
TCP/IP 应用层整合了 OSI 的应用层、表示层和会话层功能，直接为用户应用提供网络服务。

## 核心协议

### Web 协议族
| 协议 | 端口 | 说明 |
|------|------|------|
| HTTP/1.1 | 80 | 文本协议，持久连接 |
| HTTP/2 | 443 | 二进制帧，多路复用，服务器推送 |
| HTTP/3 | 443 | 基于 QUIC (UDP)，无队头阻塞 |
| HTTPS | 443 | HTTP + TLS 加密 |
| WebSocket | 80/443 | 全双工长连接 |

### 邮件协议族
| 协议 | 端口 | 方向 | 说明 |
|------|------|------|------|
| SMTP | 25/587 | 发送 | 简单邮件传输协议 |
| POP3 | 110/995 | 接收 | 下载到本地后删除服务端副本 |
| IMAP | 143/993 | 接收 | 服务端管理，多端同步 |

### 文件传输
| 协议 | 端口 | 说明 |
|------|------|------|
| FTP | 20/21 | 传统文件传输（明文） |
| SFTP | 22 | 基于 SSH 的安全文件传输 |
| SCP | 22 | 安全复制 |
| TFTP | 69 (UDP) | 简单文件传输（无认证，用于 PXE 启动） |

### 域名与目录
| 协议 | 端口 | 说明 |
|------|------|------|
| DNS | 53 | 域名解析 |
| LDAP | 389/636 | 目录访问协议 |
| NTP | 123 (UDP) | 网络时间同步 |

### 远程管理
| 协议 | 端口 | 说明 |
|------|------|------|
| SSH | 22 | 安全远程终端 |
| Telnet | 23 | 远程终端（明文，已弃用） |
| SNMP | 161 (UDP) | 网络设备管理 |
| RDP | 3389 | 远程桌面 |

### 实时通信
| 协议 | 端口 | 说明 |
|------|------|------|
| SIP | 5060 | VoIP 会话初始化 |
| RTP | 动态 | 实时传输协议 |
| XMPP | 5222 | 即时通讯 |

## 应用层协议设计模式

### 请求-响应模式（HTTP）
```
Client → Server: Request
Server → Client: Response
```
- 简单、无状态
- 适合短交互

### 推送模式（WebSocket）
```
Client ↔ Server: 双向持久数据流
```
- 全双工、低延迟
- 适合实时应用

### 发布-订阅模式（MQTT）
```
Publisher → Broker ← Subscriber
```
- 解耦生产者和消费者
- 适合 IoT 场景

## HTTP 详解

### 请求方法
| 方法 | 幂等 | 安全 | 语义 |
|------|------|------|------|
| GET | ✓ | ✓ | 获取资源 |
| POST | ✗ | ✗ | 提交数据 |
| PUT | ✓ | ✗ | 全量更新 |
| PATCH | ✗ | ✗ | 部分更新 |
| DELETE | ✓ | ✗ | 删除资源 |
| HEAD | ✓ | ✓ | 获取头部（不含body） |
| OPTIONS | ✓ | ✓ | 查询支持的方法 |

### 状态码分类
| 范围 | 类别 | 常见状态码 |
|------|------|-----------|
| 1xx | 信息 | 100 Continue, 101 Switching Protocols |
| 2xx | 成功 | 200 OK, 201 Created, 204 No Content |
| 3xx | 重定向 | 301 Moved, 302 Found, 304 Not Modified |
| 4xx | 客户端错误 | 400 Bad Request, 403 Forbidden, 404 Not Found |
| 5xx | 服务器错误 | 500 Internal Error, 502 Bad Gateway, 503 Unavailable |

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| OSI 对应 | [L7](../osi/L7-application.md) · [L6](../osi/L6-presentation.md) · [L5](../osi/L5-session.md) | TCP/IP 应用层 = OSI L5+L6+L7 |
| 下层 | [transport.md](transport.md) | 应用层协议依赖传输层的 TCP 或 UDP |
| 协议详解 | [HTTP](../protocols/http.md) · [DNS](../protocols/dns.md) · [DHCP](../protocols/dhcp.md) | 典型应用层协议深入解析 |
| 加密层 | [HTTP - HTTPS](../protocols/http.md#https-http-over-tls) | TLS 在应用层与传输层之间提供加密 |
| 案例 | [案例3: 三层架构](../cases/03-three-tier-architecture.md) | 公网用户 → CDN → WAF → App → DB 全链路 |
