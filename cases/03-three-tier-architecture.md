# 案例 3：公网用户 → 内网 Web → 数据库（典型三层架构）

## 场景
公网用户下单，请求经 CDN → WAF → Nginx → 应用服务器 → 数据库，横跨公网、DMZ、内网三个网络区域。

## 数据路径

```
[公网用户手机] (4G/5G, 运营商分配公网 IP)
 │
 │ ① HTTPS 请求 api.shop.com/api/order
 │    DNS 解析 → CNAME → CDN 边缘节点 IP
 │
 │ ② CDN 边缘节点 (就近接入)
 │    TCP + TLS → CDN 节点
 │    CDN 检查缓存 → 未命中 → 回源
 │
 │ ③ CDN 回源 → WAF (Web 应用防火墙)
 │    CDN → [互联网骨干] → 公司公网入口
 │    WAF 检查: SQL 注入、XSS、频率限制
 │    WAF 透传 → DMZ 区 Nginx
 │
 │ ④ Nginx 反向代理 (DMZ 区)
 │    Nginx (172.16.1.10) → upstream 应用服务器集群
 │    选择 app-server-03 (10.0.2.30)
 │    ※ DMZ 与内网之间有防火墙策略，仅允许特定端口
 │
 │ ⑤ 应用服务器处理 (内网区)
 │    app-server-03 执行下单逻辑
 │    → 连接 MySQL 主库 10.0.3.10:3306
 │      [TCP 三次握手] → [MySQL 协议]
 │      INSERT INTO orders ...
 │    → 连接 Redis 10.0.3.20:6379 更新库存缓存
 │      [TCP] → [RESP 协议]
 │      DECR stock:item_001
 │
 │ ⑥ 响应原路返回
 │    MySQL ACK → app-server → Nginx → WAF → CDN → 用户
 ▼
```

## 网络区域与安全边界

```
 ┌─────────┐    ┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
 │  公网    │    │  CDN    │    │   DMZ    │    │   内网    │    │ 数据库区  │
 │         │    │  边缘   │    │          │    │  应用层   │    │          │
 │ 用户    │───→│  节点   │───→│ WAF      │───→│ App      │───→│ MySQL    │
 │ 手机    │    │         │    │ Nginx    │    │ Server   │    │ Redis    │
 │         │    │         │    │          │    │          │    │          │
 │公网 IP  │    │CDN IP   │    │172.16.x  │    │10.0.2.x  │    │10.0.3.x  │
 └─────────┘    └─────────┘    └──────────┘    └───────────┘    └──────────┘
      │              │              │                │               │
   运营商网络    CDN 骨干网     防火墙①         防火墙②         数据库防火墙
                              (公网→DMZ)     (DMZ→内网)      (内网→DB区)
```

## 每一跳的协议栈变化

| 跳转 | 源 IP | 目的 IP | 传输层 | 应用层 | 特殊处理 |
|------|-------|---------|--------|--------|----------|
| 用户→CDN | 用户公网IP | CDN边缘IP | TCP+TLS | HTTPS | CDN Anycast |
| CDN→WAF | CDN回源IP | 公司公网IP | TCP+TLS | HTTPS | X-Forwarded-For |
| WAF→Nginx | WAF内网IP | 172.16.1.10 | TCP | HTTP | 反向代理 |
| Nginx→App | 172.16.1.10 | 10.0.2.30 | TCP | HTTP | upstream 选择 |
| App→MySQL | 10.0.2.30 | 10.0.3.10 | TCP | MySQL协议 | 连接池 |
| App→Redis | 10.0.2.30 | 10.0.3.20 | TCP | RESP协议 | 单线程模型 |

## 涉及的协议文档

| 协议 | 在本知识库中的位置 | 在此案例中的角色 |
|------|-------------------|-----------------|
| DNS | [protocols/dns.md](../protocols/dns.md) | CNAME 解析到 CDN 节点 |
| TCP | [protocols/tcp.md](../protocols/tcp.md) | 全链路可靠传输 |
| TLS | [protocols/http.md](../protocols/http.md) | 公网段 HTTPS 加密 |
| HTTP | [protocols/http.md](../protocols/http.md) | Nginx 反向代理、upstream 转发 |
| IP | [protocols/ip.md](../protocols/ip.md) | 多网段路由，NAT 转换 |
