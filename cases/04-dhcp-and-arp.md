# 案例 4：内网 DHCP 获取网络配置 + ARP 冲突检测

## 场景
一台新电脑插入公司有线网络，从零开始获得网络连通性。

## 数据路径

```
[新电脑开机，网卡启动]
 │
 │ ① 物理层协商
 │    自动协商速率: 1000BASE-T 全双工
 │    ※ 双绞线 Cat5e 以上才支持千兆
 │
 │ ② DHCP 获取 IP (四步 DORA)
 │    此时电脑没有任何 IP 地址！
 │
 │    Discover: src=0.0.0.0:68 → dst=255.255.255.255:67 (广播)
 │      "我是 MAC=AA:BB:CC:DD:EE:FF，谁能给我分配 IP？"
 │      ※ 交换机泛洪广播帧到所有端口
 │
 │    Offer: DHCP 服务器 10.0.0.1 响应
 │      "给你 10.0.2.88/24，网关 10.0.0.1，DNS 10.0.0.2，租约 8h"
 │
 │    Request: 广播接受 (通知其他 DHCP 服务器)
 │
 │    ACK: 确认分配，配置生效
 │      ※ 同时获得: IP, 掩码, 网关, DNS, NTP, 域名
 │
 │ ③ 免费 ARP 冲突检测
 │    电脑发送 ARP Request: "Who has 10.0.2.88? Tell 10.0.2.88"
 │      源 IP = 目标 IP = 自己的 IP
 │      ※ 如果收到 Reply，说明 IP 已被占用，需要重新 DHCP
 │      ※ 同时通知网络中所有设备更新 ARP 缓存
 │
 │ ④ 首次访问外网
 │    ping 8.8.8.8
 │    → ARP: 网关 10.0.0.1 的 MAC 地址？
 │      ARP Request (广播) → 网关 Reply: 00:1A:2B:3C:4D:5E
 │    → 构造 ICMP Echo Request
 │      [ETH: dst=网关MAC] [IP: dst=8.8.8.8, TTL=64] [ICMP: Echo]
 │    → 帧发往交换机 → 交换机根据 MAC 表转发到网关端口
 │    → 路由器查路由表 → 下一跳 → ARP → 继续转发...
 ▼
```

## 从"零"到"通"的完整协议链

```
物理层自动协商 (L1)
  → DHCP 获取 IP 配置 (L7, 基于 UDP 广播)
    → 免费 ARP 检测冲突 (L2/L3 边界)
      → ARP 解析网关 MAC (L2/L3 边界)
        → ICMP Echo 测试连通 (直接封装在 IP 中)
          → IP 路由到外网 (L3, 路由器逐跳转发)
            → BGP/OSPF 选路 (路由器间的路由协议)
```

## 涉及的协议文档

| 协议 | 在本知识库中的位置 | 在此案例中的角色 |
|------|-------------------|-----------------|
| DHCP | [protocols/dhcp.md](../protocols/dhcp.md) | 四步 DORA 获取 IP 及全部网络配置 |
| ARP | [protocols/arp.md](../protocols/arp.md) | 免费 ARP 冲突检测 + 网关 MAC 解析 |
| ICMP | [protocols/icmp.md](../protocols/icmp.md) | ping 测试外网连通性 |
| IP | [protocols/ip.md](../protocols/ip.md) | 路由寻址、TTL 转发 |
| Ethernet | [tcpip/link.md](../tcpip/link.md) | 帧封装、交换机广播泛洪、MAC 表转发 |
| 物理层 | [osi/L1-physical.md](../osi/L1-physical.md) | 自动协商速率和双工模式 |
