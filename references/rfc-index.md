# RFC 文档索引

## 核心 RFC

### TCP/IP 协议族

| RFC | 标题 | 协议 | 状态 |
|-----|------|------|------|
| RFC 768 | User Datagram Protocol | UDP | 标准 |
| RFC 791 | Internet Protocol | IPv4 | 标准 |
| RFC 792 | Internet Control Message Protocol | ICMP | 标准 |
| RFC 793 | Transmission Control Protocol | TCP | 标准 |
| RFC 826 | Ethernet Address Resolution Protocol | ARP | 标准 |
| RFC 950 | Internet Standard Subnetting Procedure | 子网划分 | 标准 |

### IPv6

| RFC | 标题 | 说明 |
|-----|------|------|
| RFC 8200 | IPv6 Specification | IPv6 核心规范 (取代 RFC 2460) |
| RFC 4291 | IPv6 Addressing Architecture | IPv6 地址架构 |
| RFC 4861 | Neighbor Discovery for IPv6 | NDP (取代 ARP) |
| RFC 4862 | IPv6 Stateless Address Autoconfiguration | SLAAC |
| RFC 5952 | IPv6 Address Text Representation | 地址表示法 |

### 路由协议

| RFC | 标题 | 协议 |
|-----|------|------|
| RFC 2328 | OSPF Version 2 | OSPF |
| RFC 2453 | RIP Version 2 | RIPv2 |
| RFC 4271 | A Border Gateway Protocol 4 | BGP-4 |
| RFC 5340 | OSPF for IPv6 | OSPFv3 |

### 应用层协议

| RFC | 标题 | 协议 |
|-----|------|------|
| RFC 1034 | Domain Names - Concepts and Facilities | DNS |
| RFC 1035 | Domain Names - Implementation and Specification | DNS |
| RFC 2131 | Dynamic Host Configuration Protocol | DHCP |
| RFC 2616 | Hypertext Transfer Protocol -- HTTP/1.1 | HTTP/1.1 (已过时) |
| RFC 7230-7235 | HTTP/1.1 (更新) | HTTP/1.1 |
| RFC 7540 | Hypertext Transfer Protocol Version 2 | HTTP/2 |
| RFC 9114 | HTTP/3 | HTTP/3 |
| RFC 5321 | Simple Mail Transfer Protocol | SMTP |
| RFC 3501 | Internet Message Access Protocol - Version 4rev1 | IMAP |

### 安全与加密

| RFC | 标题 | 说明 |
|-----|------|------|
| RFC 5246 | The Transport Layer Security (TLS) Protocol 1.2 | TLS 1.2 |
| RFC 8446 | The Transport Layer Security (TLS) Protocol 1.3 | TLS 1.3 |
| RFC 4301 | Security Architecture for the Internet Protocol | IPsec |
| RFC 4302 | IP Authentication Header | AH |
| RFC 4303 | IP Encapsulating Security Payload | ESP |

### 网络管理

| RFC | 标题 | 说明 |
|-----|------|------|
| RFC 1157 | Simple Network Management Protocol | SNMPv1 |
| RFC 3410-3418 | SNMPv3 | SNMPv3 |
| RFC 1213 | MIB-II | 管理信息库 |

## TCP 扩展与优化

| RFC | 标题 | 说明 |
|-----|------|------|
| RFC 1323 | TCP Extensions for High Performance | 时间戳、窗口缩放 |
| RFC 2018 | TCP Selective Acknowledgments | SACK |
| RFC 2581 | TCP Congestion Control | 拥塞控制 |
| RFC 5681 | TCP Congestion Control (更新) | 拥塞控制更新 |
| RFC 7413 | TCP Fast Open | TFO |
| RFC 6824 | TCP Extensions for Multipath Operation | MPTCP |

## 重要 BCP / FYI

| RFC | 类型 | 说明 |
|-----|------|------|
| RFC 1918 | BCP | 私有地址分配 |
| RFC 2119 | BCP | 关键词定义 (MUST, SHOULD, MAY) |
| RFC 3339 | FYI | 日期时间格式 |
| RFC 4632 | BCP | CIDR 地址架构 |
| RFC 7942 | BCP | 实施状态节 |

## 如何查找 RFC
- 在线: https://www.rfc-editor.org/
- 格式: https://www.rfc-editor.org/rfc/rfc{number}.txt
- 搜索: https://datatracker.ietf.org/

> RFC 状态说明：
> - **Standard**: 正式标准
> - **Proposed Standard**: 提议标准（最常见的"标准"）
> - **Experimental**: 实验性
> - **Informational**: 信息性
> - **Historic**: 历史（已过时）
> - **BCP**: 最佳当前实践
