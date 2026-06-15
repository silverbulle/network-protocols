# ARP 协议详解

## 概述
ARP (Address Resolution Protocol, RFC 826) 用于将 IP 地址解析为 MAC 地址。工作在数据链路层与网络层之间。

### 协议栈位置与关联

```
网络层   ┌──────────────────┐
         │    IP (v4)       │
         └────────┬─────────┘
                  │ "下一跳 IP 的 MAC 是？"
                  ▼
L2/L3    ┌──────────────────┐
边界     │     ARP ← 本文件  │  广播查询，单播应答
         └────────┬─────────┘
                  │
链路层   ┌────────┴─────────┐
         │   Ethernet 帧     │
         └──────────────────┘
```

| 关系 | 文档 | 说明 |
|------|------|------|
| 为 IP 服务 | [IP](ip.md) | IP 发包前需 ARP 解析下一跳 MAC |
| 工作在链路层 | [tcpip/link.md](../tcpip/link.md) | ARP 报文封装在以太网帧中（EtherType=0x0806） |
| IPv6 替代 | [IP - NDP](ip.md#ndp-neighbor-discovery-protocol-消息) | IPv6 用 ICMPv6 NDP 替代了 ARP |
| 安全威胁 | [tcpip/link.md](../tcpip/link.md#二层交换) | ARP 欺骗与 DHCP Snooping 防御 |
| OSI 层 | [L2](../osi/L2-datalink.md) · [L3](../osi/L3-network.md) | 横跨两层的桥梁协议 |

## 工作原理

### 地址解析流程
```
已知: 目的 IP = 192.168.1.100
需要: 目的 MAC = ?

1. 查 ARP 缓存表 → 未找到
2. 构造 ARP Request (广播)
3. 发送到 FF:FF:FF:FF:FF:FF
4. 192.168.1.100 收到后单播 ARP Reply
5. 更新 ARP 缓存
```

## ARP 报文格式

### 以太网中的 ARP 帧
```
┌────────────┬────────────┬──────────┬──────────────────────────────┐
│ Dest MAC   │ Src MAC    │ EtherType│        ARP Payload           │
│ FF:FF:FF   │ AA:BB:CC   │ 0x0806   │                              │
│ FF:FF:FF   │ DD:EE:FF   │          │                              │
│ (6 bytes)  │ (6 bytes)  │ (2B)     │                              │
└────────────┴────────────┴──────────┴──────────────────────────────┘
```

### ARP 报文结构
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Hardware Type        |         Protocol Type         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| HW Addr Len  | Proto Addr Len |          Operation            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Sender Hardware Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Sender Protocol Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Target Hardware Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Target Protocol Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 字段说明

| 字段 | 长度 | 说明 |
|------|------|------|
| Hardware Type | 16 bit | 硬件类型（1 = 以太网） |
| Protocol Type | 16 bit | 协议类型（0x0800 = IPv4） |
| HW Addr Len | 8 bit | 硬件地址长度（6 = MAC） |
| Proto Addr Len | 8 bit | 协议地址长度（4 = IPv4） |
| Operation | 16 bit | 1=Request, 2=Reply |
| Sender HW Addr | 48 bit | 发送方 MAC |
| Sender Proto Addr | 32 bit | 发送方 IP |
| Target HW Addr | 48 bit | 目标 MAC（Request 时为 0） |
| Target Proto Addr | 32 bit | 目标 IP |

## ARP 缓存

### 缓存管理
```bash
# 查看 ARP 缓存
$ arp -a
Interface: 192.168.1.10
  Internet Address      Physical Address      Type
  192.168.1.1           00-1a-2b-3c-4d-5e     dynamic
  192.168.1.100         aa-bb-cc-dd-ee-ff     dynamic
  192.168.1.255         ff-ff-ff-ff-ff-ff     static

# 添加静态条目
$ arp -s 192.168.1.1 00-1a-2b-3c-4d-5e

# 删除条目
$ arp -d 192.168.1.100
```

### 缓存超时
- 动态条目：通常 2-20 分钟无活动后删除
- 静态条目：永久保留（重启后丢失）

## 代理 ARP (Proxy ARP)
```
Host A (192.168.1.10) 想访问 192.168.2.20
                          │
                     [路由器 R]
                     接口1: 192.168.1.1
                     接口2: 192.168.2.1
                          │
                   192.168.2.20

A 广播 ARP Request: "Who has 192.168.2.20?"
R 代替回答: "192.168.2.20 is at R's MAC address"
A 将包发给 R，由 R 转发
```

## 免费 ARP (Gratuitous ARP)
```
主机发送 ARP Request 查询自己的 IP:
  Sender IP = 192.168.1.10, Target IP = 192.168.1.10

用途:
1. IP 冲突检测（若收到 Reply 则 IP 已被占用）
2. 通知网络更新 ARP 缓存（IP 或 MAC 变更后）
3. VRRP/HSRP 主备切换时更新 MAC
```

## RARP (Reverse ARP)
- 功能：已知 MAC 地址，查询 IP 地址（ARP 的反向）
- 已弃用，被 BOOTP/DHCP 替代

## InARP (Inverse ARP)
- 用于帧中继/ATM 网络
- 已知虚电路标识，查询对端 IP

## ARP 安全威胁

### ARP 欺骗 (ARP Spoofing/Poisoning)
```
攻击者 M:
  向 A 发送: "192.168.1.1 is at M's MAC" (伪造)
  向 B 发送: "192.168.1.2 is at M's MAC" (伪造)

结果: A 和 B 的流量都经过 M
M 可以: 窃听、篡改、中间人攻击
```

### 防御措施
| 方法 | 说明 |
|------|------|
| 静态 ARP 绑定 | 手动配置 IP-MAC 映射 |
| DAI (Dynamic ARP Inspection) | 交换机检测 ARP 合法性 |
| ARP 防火墙 | 软件层面检测异常 ARP |
| 802.1X | 端口级认证 |
