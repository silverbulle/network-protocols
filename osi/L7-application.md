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

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| TCP/IP 对应 | [tcpip/application.md](../tcpip/application.md) | L5+L6+L7 合并为 TCP/IP 应用层 |
| 下层 | [L6 表示层](L6-presentation.md) | 数据格式转换、加密 |
| 相关协议 | [HTTP](../protocols/http.md) · [DNS](../protocols/dns.md) · [DHCP](../protocols/dhcp.md) | 典型应用层协议 |
| 依赖传输层 | [L4 传输层](L4-transport.md) | 应用层协议需要传输层提供端到端服务 |
