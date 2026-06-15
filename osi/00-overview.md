# OSI 七层模型总览

## 概述
OSI（Open Systems Interconnection）模型由 ISO 于 1984 年发布，是网络通信的参考框架。它将通信过程分为七层，每层职责明确、相互独立。

## 七层一览

| 层号 | 名称 | 英文 | 核心职责 | 典型协议 | 数据单元 |
|------|------|------|----------|----------|----------|
| 7 | 应用层 | Application | 为用户应用提供网络服务接口 | HTTP, FTP, SMTP, DNS | 数据 (Data) |
| 6 | 表示层 | Presentation | 数据格式转换、加密/解密、压缩 | SSL/TLS, JPEG, ASCII | 数据 (Data) |
| 5 | 会话层 | Session | 建立/管理/终止会话 | NetBIOS, RPC, PPTP | 数据 (Data) |
| 4 | 传输层 | Transport | 端到端的可靠/不可靠数据传输 | TCP, UDP | 段 (Segment) |
| 3 | 网络层 | Network | 逻辑寻址与路由选择 | IP, ICMP, OSPF | 包 (Packet) |
| 2 | 数据链路层 | Data Link | 物理寻址、帧传输、差错检测 | Ethernet, PPP, Wi-Fi | 帧 (Frame) |
| 1 | 物理层 | Physical | 比特流的物理传输 | 电缆, 光纤, 无线电波 | 比特 (Bit) |

## 数据封装过程

```
应用层    [Data]
  ↓ 加头部
表示层    [Data]
  ↓
会话层    [Data]
  ↓
传输层    [TCP/UDP Header | Data]         → Segment
  ↓
网络层    [IP Header | TCP Header | Data] → Packet
  ↓
数据链路层[ETH Header | IP Header | TCP Header | Data | ETH Trailer] → Frame
  ↓
物理层    010110100101...                  → Bits
```

## 核心设计原则
1. **分层解耦**：每层只需知道相邻层的接口，不需了解其他层实现
2. **标准化接口**：层与层之间通过服务访问点（SAP）交互
3. **对等通信**：发送方第 N 层与接收方第 N 层逻辑上直接通信（通过协议）
4. **逐层封装/解封装**：发送方逐层加头，接收方逐层去头

## 与 TCP/IP 模型的对应关系

| OSI 层 | TCP/IP 层 |
|--------|-----------|
| 应用层 + 表示层 + 会话层 | 应用层 |
| 传输层 | 传输层 |
| 网络层 | 网际层 |
| 数据链路层 + 物理层 | 网络接口层 |

> OSI 是理论参考模型；TCP/IP 是实际工程实现，互联网基于 TCP/IP 构建。

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| TCP/IP 模型总览 | [tcpip/00-overview.md](../tcpip/00-overview.md) | OSI 是理论框架，TCP/IP 是工程实现 |
| 各层详解 | [L7](L7-application.md) [L6](L6-presentation.md) [L5](L5-session.md) [L4](L4-transport.md) [L3](L3-network.md) [L2](L2-datalink.md) [L1](L1-physical.md) | 点击跳转到对应层详解 |
| 全链路封装 | [README 第二节](../README.md#二数据从应用到线路全链路封装) | 追踪数据从应用到物理介质的完整旅程 |
| 实战案例 | [cases/](../cases/) | 真实场景下的数据路径剖析 |
