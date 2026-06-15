# TCP/IP 四层模型总览

## 概述
TCP/IP 模型是互联网的实际协议栈，由 DARPA 于 1970s 开发。相比 OSI 的理论七层，TCP/IP 采用更实用的四层结构。

## 四层结构

| TCP/IP 层 | 对应 OSI 层 | 核心协议 | 功能 |
|-----------|-------------|----------|------|
| 应用层 | L7 + L6 + L5 | HTTP, DNS, FTP, SSH | 应用协议与数据交换 |
| 传输层 | L4 | TCP, UDP | 端到端数据传输 |
| 网际层 | L3 | IP, ICMP, ARP | 寻址与路由 |
| 网络接口层 | L2 + L1 | Ethernet, Wi-Fi, PPP | 物理传输 |

## 与 OSI 的关键区别
1. **实用优先**：TCP/IP 先有协议后有模型，OSI 先有模型后有协议
2. **层次简化**：合并了会话层和表示层到应用层
3. **无严格分层要求**：实际实现中层次边界更灵活
4. **端到端原则**：智能在端点，网络保持简单

## 协议栈封装
```
应用层     [Application Data]
              ↓
传输层     [TCP/UDP Header | Application Data]
              ↓
网际层     [IP Header | TCP/UDP Header | Application Data]
              ↓
网络接口层 [ETH Header | IP Header | TCP/UDP Header | Data | FCS]
              ↓
物理介质   bits on the wire
```

## 互联网架构的五大设计原则
1. **端到端原则**：差错控制和智能功能放在端系统
2. **尽力交付**：网络不保证可靠性，由端系统保证
3. **无状态核心**：路由器不维护连接状态
4. **分层解耦**：每层独立演进
5. **可伸缩性**：支持从几台到数十亿台设备的扩展

---

## 关联导航

| 关系 | 文档 | 说明 |
|------|------|------|
| OSI 模型总览 | [osi/00-overview.md](../osi/00-overview.md) | OSI 七层是理论参考，TCP/IP 四层是工程实现 |
| 应用层 | [application.md](application.md) | HTTP、DNS、DHCP 等应用层协议族 |
| 传输层 | [transport.md](transport.md) | TCP/UDP 对比、拥塞控制、端口体系 |
| 网际层 | [internet.md](internet.md) | IP 寻址、路由协议、ICMP、ARP |
| 网络接口层 | [link.md](link.md) | 以太网、Wi-Fi、PPP、二层交换 |
| 实战案例 | [cases/](../cases/) | 真实场景下的数据路径剖析 |
