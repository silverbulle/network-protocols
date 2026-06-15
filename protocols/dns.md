# DNS 协议详解

## 概述
DNS (Domain Name System, RFC 1034/1035) 是互联网的分布式目录服务，将域名映射为 IP 地址及其他资源记录。

### 协议栈位置与关联

```
应用层   ┌──────────────────────────────────┐
         │   DNS ← 本文件                    │
         │   HTTP  DHCP  SMTP  (都依赖DNS)   │
         └────────┬─────────────────────────┘
                  │
传输层            ▼
         ┌─────┐  ┌─────┐
         │ UDP │  │ TCP │  查询用 UDP，大包/区域传输用 TCP
         └──┬──┘  └──┬──┘
            │        │
网络层      ▼        ▼
         ┌──────────────┐
         │  IP (v4/v6)  │
         └──────────────┘
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 依赖传输层 | [UDP](udp.md) · [TCP](tcp.md) | 查询用 UDP:53，响应>512B 或区域传输用 TCP |
| 被依赖 | [HTTP](http.md) · [DHCP](dhcp.md) | 几乎所有应用层协议都先经过 DNS 解析 |
| DHCP 提供 DNS | [DHCP - Option 6](dhcp.md#常用选项) | DHCP 分配 IP 时同时告知 DNS 服务器地址 |
| 安全扩展 | [HTTP - HSTS](http.md#cookie-与-session) | DNSSEC 验证 DNS 响应真实性 |
| OSI 层 | [L7 应用层](../osi/L7-application.md) | |
| TCP/IP 层 | [tcpip/application.md](../tcpip/application.md#域名与目录) | |

## DNS 架构

### 层次结构
```
                    . (根)
                   /|\
              com  org  net  ...
              /|\
         google  apple  amazon
          /|\
     www  mail  api
```

### 查询流程
```
1. 浏览器查询 www.google.com
2. 检查浏览器 DNS 缓存
3. 检查操作系统 DNS 缓存
4. 检查 hosts 文件
5. 向本地 DNS 服务器（递归解析器）发起查询
6. 递归解析器:
   a. 查询根服务器 → 得到 .com TLD 地址
   b. 查询 .com TLD → 得到 google.com 权威服务器
   c. 查询 google.com 权威服务器 → 得到 www.google.com 的 IP
7. 返回结果并缓存
```

## DNS 报文格式

### 请求/响应报文
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          ID                   |QR|Opcode|AA|TC|RD|RA| Z| RCODE|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         QDCOUNT               |         ANCOUNT               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         NSCOUNT               |         ARCOUNT               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
/                          Questions                            /
/                                                               /
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
/                          Answers                              /
/                                                               /
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
/                     Authority Records                         /
/                                                               /
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
/                   Additional Records                          /
/                                                               /
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 头部标志位
| 标志 | 位 | 说明 |
|------|-----|------|
| QR | 1 | 0=Query, 1=Response |
| Opcode | 4 | 0=标准查询, 1=反向查询, 2=状态查询 |
| AA | 1 | Authoritative Answer |
| TC | 1 | Truncated (应答被截断) |
| RD | 1 | Recursion Desired |
| RA | 1 | Recursion Available |
| RCODE | 4 | 响应码 (0=成功, 3=NXDOMAIN等) |

### 域名编码
```
www.google.com 编码为:
[3] w w w [6] g o o g l e [3] c o m [0]

每个标签前加长度字节，以 0 结尾
支持指针压缩: 0xC00C 指向偏移 12 处的域名
```

## 资源记录类型

| 类型 | 值 | 说明 | 示例 |
|------|-----|------|------|
| A | 1 | IPv4 地址 | www → 93.184.216.34 |
| AAAA | 28 | IPv6 地址 | www → 2606:2800:220:1:: |
| CNAME | 5 | 别名 | www → lb.example.com |
| MX | 15 | 邮件交换 | @ → mail.example.com (优先级10) |
| NS | 2 | 名称服务器 | example.com → ns1.example.com |
| SOA | 6 | 授权起始 | 包含序列号、刷新间隔等 |
| TXT | 16 | 文本记录 | SPF、DKIM、域名验证 |
| PTR | 12 | 反向解析 | 34.216.184.93 → www.example.com |
| SRV | 33 | 服务定位 | _sip._tcp → sip.example.com:5060 |
| CAA | 257 | CA 授权 | 允许哪些 CA 签发证书 |

### SOA 记录详解
```
example.com.  IN  SOA  ns1.example.com. admin.example.com. (
    2024010101  ; Serial (序列号)
    3600        ; Refresh (辅服务器刷新间隔: 1h)
    900         ; Retry (重试间隔: 15min)
    604800      ; Expire (过期时间: 1周)
    86400       ; Minimum TTL (默认TTL: 1天)
)
```

### MX 记录
```
example.com.  IN  MX  10  mail1.example.com.
example.com.  IN  MX  20  mail2.example.com.

优先级数字越小越优先
```

## DNS 传输协议

| 场景 | 协议 | 端口 |
|------|------|------|
| 标准查询 | UDP | 53 |
| 响应 > 512 字节 | TCP | 53 |
| 区域传输 | TCP | 53 |
| DNS over TLS (DoT) | TCP | 853 |
| DNS over HTTPS (DoH) | HTTPS | 443 |
| DNS over QUIC (DoQ) | QUIC | 853 |

## DNSSEC (DNS 安全扩展)
```
目标: 验证 DNS 响应的真实性，防止 DNS 欺骗

机制:
1. 每个区域有密钥对 (KSK + ZSK)
2. 记录签名 → RRSIG 记录
3. 公钥发布 → DNSKEY 记录
4. 信任链: 根 → TLD → 域名 (DS 记录)

新增记录类型:
- DNSKEY: 公钥
- RRSIG: 记录签名
- DS: 委托签名
- NSEC/NSEC3: 证明记录不存在
```

## 反向 DNS 解析
```
IP: 93.184.216.34
查询: 34.216.184.93.in-addr.arpa 的 PTR 记录
结果: www.example.com

IPv6:
IP: 2001:db8::1
查询: 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa
```

## DNS 缓存与 TTL
- **TTL (Time To Live)**：记录缓存时间（秒）
- **递归缓存**：DNS 服务器缓存查询结果
- **浏览器缓存**：Chrome `chrome://net-internals/#dns`
- **系统缓存**：`ipconfig /flushdns` (Windows)

## 常见 DNS 攻击

| 攻击 | 说明 | 防御 |
|------|------|------|
| DNS 欺骗 | 伪造响应，重定向流量 | DNSSEC |
| DNS 放大攻击 | 利用开放解析器发起 DDoS | 限制递归、响应速率限制 |
| DNS 隧道 | 通过 DNS 协议传输数据 | 监控异常查询模式 |
| 域名劫持 | 修改域名注册信息 | 注册商锁定、MFA |
| 缓存投毒 | 向缓存注入虚假记录 | 随机化端口和 ID |
