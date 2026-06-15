# HTTP 协议详解

## 概述
HTTP (HyperText Transfer Protocol) 是 Web 的基础应用层协议，基于请求-响应模型。

### 协议栈位置与关联

```
应用层   ┌──────────────────────────────────┐
         │   HTTP/1.1  HTTP/2  HTTP/3       │  ← 本文件
         └────────┬─────────────────────────┘
                  │
表示层            ▼
         ┌──────────────┐
         │  TLS 1.2/1.3 │  HTTPS = HTTP + TLS
         └──────┬───────┘
                │
传输层          ▼
         ┌─────┐         ┌─────┐
         │ TCP │         │ UDP │  HTTP/3 基于 QUIC → UDP
         └──┬──┘         └──┬──┘
            │               │
网络层      ▼               ▼
         ┌──────────────────────┐
         │      IP (v4/v6)      │
         └──────────────────────┘
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 依赖 TCP | [TCP](tcp.md) | HTTP/1.1 和 HTTP/2 基于 TCP 可靠传输 |
| 依赖 UDP | [UDP](udp.md) | HTTP/3 基于 QUIC，QUIC 运行在 UDP 之上 |
| TLS 加密 | [L6 表示层](../osi/L6-presentation.md) | TLS 握手在概念上属于 OSI 表示层 |
| DNS 先行 | [DNS](dns.md) | HTTP 请求前先 DNS 解析目标域名 |
| OSI 层 | [L7 应用层](../osi/L7-application.md) | |
| TCP/IP 层 | [tcpip/application.md](../tcpip/application.md#web-协议族) | |
| 案例 | [案例1](../cases/01-remote-load-balancing.md) | 异地负载均衡的 HTTP 完整路径 |

## 版本演进

| 版本 | 年份 | 关键特性 |
|------|------|----------|
| HTTP/0.9 | 1991 | 仅 GET，纯文本 |
| HTTP/1.0 | 1996 | 头部、状态码、多种方法 |
| HTTP/1.1 | 1997 | 持久连接、分块传输、缓存控制 |
| HTTP/2 | 2015 | 二进制帧、多路复用、服务器推送 |
| HTTP/3 | 2022 | 基于 QUIC (UDP)、无队头阻塞 |

## HTTP/1.1 详解

### 请求格式
```
<Method> <URI> HTTP/<Version>
<Header>: <Value>
<Header>: <Value>

<Body>
```

### 响应格式
```
HTTP/<Version> <Status-Code> <Reason-Phrase>
<Header>: <Value>
<Header>: <Value>

<Body>
```

### 请求方法

| 方法 | 幂等 | 安全 | 请求体 | 说明 |
|------|------|------|--------|------|
| GET | ✓ | ✓ | 无 | 获取资源 |
| HEAD | ✓ | ✓ | 无 | 仅获取头部 |
| POST | ✗ | ✗ | 有 | 提交数据/创建资源 |
| PUT | ✓ | ✗ | 有 | 全量替换资源 |
| PATCH | ✗ | ✗ | 有 | 部分更新资源 |
| DELETE | ✓ | ✗ | 无 | 删除资源 |
| OPTIONS | ✓ | ✓ | 无 | 查询支持的方法 |
| CONNECT | ✗ | ✗ | 无 | 建立隧道 |
| TRACE | ✓ | ✓ | 无 | 回显请求（调试） |

### 状态码

| 分类 | 范围 | 常见 |
|------|------|------|
| 1xx 信息 | 100-199 | 100 Continue, 101 Switching Protocols |
| 2xx 成功 | 200-299 | 200 OK, 201 Created, 204 No Content |
| 3xx 重定向 | 300-399 | 301 永久移动, 302 临时移动, 304 未修改 |
| 4xx 客户端错误 | 400-499 | 400 错误请求, 401 未认证, 403 禁止, 404 未找到 |
| 5xx 服务器错误 | 500-599 | 500 内部错误, 502 网关错误, 503 服务不可用 |

### 常见头部

#### 请求头
| 头部 | 说明 | 示例 |
|------|------|------|
| Host | 目标主机 | Host: www.example.com |
| User-Agent | 客户端标识 | User-Agent: Mozilla/5.0 |
| Accept | 可接受的媒体类型 | Accept: text/html, application/json |
| Content-Type | 请求体类型 | Content-Type: application/json |
| Authorization | 认证凭据 | Authorization: Bearer <token> |
| Cookie | Cookie | Cookie: session=abc123 |
| Cache-Control | 缓存指令 | Cache-Control: no-cache |
| If-None-Match | 条件请求 | If-None-Match: "abc" |

#### 响应头
| 头部 | 说明 | 示例 |
|------|------|------|
| Content-Type | 响应体类型 | Content-Type: text/html; charset=utf-8 |
| Content-Length | 响应体长度 | Content-Length: 1234 |
| Set-Cookie | 设置 Cookie | Set-Cookie: session=abc; Path=/ |
| Location | 重定向地址 | Location: /new-page |
| Cache-Control | 缓存策略 | Cache-Control: max-age=3600 |
| ETag | 资源指纹 | ETag: "abc123" |
| Strict-Transport-Security | HSTS | max-age=31536000; includeSubDomains |

### 持久连接 (Keep-Alive)
```
HTTP/1.0: 默认每个请求新建 TCP 连接
HTTP/1.1: 默认持久连接，Connection: keep-alive

关闭连接: Connection: close
```

### 分块传输 (Chunked Transfer)
```
Transfer-Encoding: chunked

23\r\n
This is the first chunk\r\n
1A\r\n
and this is the second\r\n
0\r\n
\r\n
```

## HTTP/2

### 核心特性
1. **二进制分帧**：消息分割为二进制帧
2. **多路复用**：单连接并行多个请求/响应
3. **头部压缩**：HPACK 算法
4. **服务器推送**：主动推送资源
5. **流优先级**：控制资源加载顺序

### 帧结构
```
+-----------------------------------------------+
|                 Length (24)                    |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+
|R|                 Stream ID (31)               |
+-+---------------------------------------------+
|                 Frame Payload                 |
+-----------------------------------------------+
```

### 帧类型
| 类型 | 说明 |
|------|------|
| DATA | 请求/响应体 |
| HEADERS | 请求/响应头 |
| PRIORITY | 流优先级 |
| RST_STREAM | 终止流 |
| SETTINGS | 连接设置 |
| PUSH_PROMISE | 服务器推送承诺 |
| PING | 心跳 |
| GOAWAY | 优雅关闭 |
| WINDOW_UPDATE | 流量控制 |

### HTTP/1.1 vs HTTP/2
```
HTTP/1.1:
  连接1: GET /a → Response | GET /b → Response | GET /c → Response (串行)
  连接2: GET /d → Response (需要多连接)

HTTP/2:
  连接1: Stream 1: GET /a ─┐
        Stream 2: GET /b ──┤ 并行
        Stream 3: GET /c ──┤
        Stream 4: GET /d ──┘ (单连接多路复用)
```

## HTTP/3 (基于 QUIC)

### 核心改进
1. **基于 UDP**：避免 TCP 队头阻塞
2. **0-RTT / 1-RTT 连接**：更快的握手
3. **内置加密**：TLS 1.3 集成在 QUIC 中
4. **连接迁移**：IP/端口变化不中断连接

### QUIC vs TCP+TLS
```
TCP+TLS:
  TCP 握手 (1 RTT) + TLS 握手 (1-2 RTT) + HTTP 请求 = 2-3 RTT

QUIC:
  QUIC 握手 + TLS (1 RTT) + HTTP 请求 = 1 RTT
  后续连接: 0 RTT
```

## HTTPS (HTTP over TLS)

### TLS 握手 (1.3)
```
Client                                    Server
  │                                         │
  │─── ClientHello ────────────────────────→│
  │    (supported_versions, key_share)      │
  │                                         │
  │←── ServerHello ────────────────────────│
  │    (selected_version, key_share)        │
  │←── EncryptedExtensions ────────────────│
  │←── Certificate ────────────────────────│
  │←── CertificateVerify ──────────────────│
  │←── Finished ───────────────────────────│
  │                                         │
  │─── Finished ───────────────────────────→│
  │                                         │
  │════ 加密通信 ══════════════════════════│
```

## 缓存机制

### 强缓存
```
Cache-Control: max-age=3600     → 1小时内直接使用缓存
Cache-Control: no-cache         → 每次需验证
Cache-Control: no-store         → 不缓存
Expires: Wed, 01 Jan 2025 00:00:00 GMT  → 过期时间
```

### 协商缓存
```
请求: If-None-Match: "abc123"    → ETag 验证
请求: If-Modified-Since: Wed...  → 时间验证
响应: 304 Not Modified           → 缓存有效
```

## Cookie 与 Session
```
服务器设置: Set-Cookie: session=abc; Path=/; HttpOnly; Secure
客户端发送: Cookie: session=abc

Cookie 属性:
- Domain/Path: 作用域
- Expires/Max-Age: 过期时间
- Secure: 仅 HTTPS 传输
- HttpOnly: JS 不可访问
- SameSite: 跨站请求限制
```
