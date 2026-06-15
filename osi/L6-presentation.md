# L6 - 表示层 (Presentation Layer)

## 职责
处理数据的语法和语义，确保不同系统间的数据能正确理解和交换。

## 核心功能

### 1. 数据格式转换
不同系统可能使用不同的内部数据表示，表示层负责转换：
- **字符编码**：ASCII ↔ UTF-8 ↔ EBCDIC
- **字节序**：大端序 (Big-Endian) ↔ 小端序 (Little-Endian)
- **数据结构**：XML、JSON、ASN.1 等序列化/反序列化

### 2. 数据加密与解密
- **SSL/TLS**：虽然 TLS 在协议栈中位于传输层之上，但其加密功能在概念上属于表示层
- **对称加密**：AES, DES, 3DES
- **非对称加密**：RSA, ECC, DH
- **哈希**：SHA-256, MD5

### 3. 数据压缩与解压缩
- **文本压缩**：gzip, deflate, brotli
- **图像压缩**：JPEG, PNG, GIF
- **视频压缩**：MPEG, H.264, H.265
- **音频压缩**：MP3, AAC, Opus

## 常见数据格式标准

| 格式 | 用途 | 说明 |
|------|------|------|
| ASCII | 英文字符编码 | 7-bit，128 个字符 |
| UTF-8 | 通用字符编码 | 变长编码，兼容 ASCII |
| EBCDIC | IBM 大型机编码 | 8-bit，与 ASCII 不兼容 |
| JPEG | 有损图像压缩 | ISO/IEC 10918 |
| MPEG | 音视频压缩 | ISO/IEC 系列标准 |
| ASN.1 | 抽象语法表示 | 用于 SNMP、X.509 等 |
| XDR | 外部数据表示 | 用于 NFS、RPC |

## TLS 握手过程（概念上属于表示层）
```
Client                          Server
  │                                │
  │──── ClientHello ──────────────→│  支持的密码套件、随机数
  │                                │
  │←──── ServerHello ─────────────│  选定密码套件、随机数
  │                                │
  │←──── Certificate ─────────────│  服务器证书
  │                                │
  │←──── ServerKeyExchange ───────│  (可选)
  │                                │
  │──── ClientKeyExchange ────────→│  预主密钥
  │                                │
  │──── ChangeCipherSpec ─────────→│
  │──── Finished ─────────────────→│
  │                                │
  │←──── ChangeCipherSpec ────────│
  │←──── Finished ────────────────│
  │                                │
  │════ 加密通信开始 ══════════════│
```

> 实际中，表示层的功能通常由应用程序或中间件实现，而非独立的网络层。

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| TCP/IP 对应 | [tcpip/application.md](../tcpip/application.md) | 表示层功能在 TCP/IP 中由应用层或中间件实现 |
| 上层 | [L7 应用层](L7-application.md) | 为应用层提供数据格式服务 |
| 下层 | [L5 会话层](L5-session.md) | 会话管理 |
| 典型实现 | [HTTP (TLS)](../protocols/http.md#https-http-over-tls) | TLS 加密在概念上属于表示层 |
| 加密跨层总览 | [protocols/encryption-layers.md](../protocols/encryption-layers.md) | L1-L7 各层加密协议对比，TLS 位于 L4/L7 之间 |
