---
title: DNS 域名系统
date: 2026-07-05
updated: 2026-07-05
tags:
  - DNS
  - 域名解析
  - CDN
  - 网络协议
categories:
  - CS基础
  - 网络
---

### 1.1 迭代查询 vs 递归查询

- **递归查询**：客户端→本地 DNS→根 DNS→.com NS→最终结果一层层回传。最简单但对递归 DNS 负担大
- **迭代查询**：本地 DNS 逐个查询根→`.com` NS→权威 NS，组装结果后返回。运营商 DNS→权威 DNS 通常用迭代

### 1.2 DNS 缓存层次

| 缓存位置 | TTL 行为 | 说明 |
|----------|----------|------|
| 浏览器缓存 | 独立，~1min | Chrome: `chrome://net-internals/#dns` |
| OS 缓存 | 遵从记录 TTL | `ipconfig /displaydns` |
| 递归 DNS | 可能延长 TTL | 8.8.8.8, 114.114.114.114 |
| 权威 DNS | 无缓存 | 直接返回查询结果 |

注意：部分递归 DNS 会忽略过短的 TTL(TTL 延展)，也存在负缓存(NXDOMAIN 结果)。

### 1.3 CDN 原理

CDN 通过 DNS 将用户导向最近的边缘节点：

```
用户 → 递归 DNS → CDN 权威 DNS(智能 DNS)
    → 根据来源 IP(地理位置/运营商/负载) 返回最近边缘节点 IP
    → 用户访问边缘节点
    → 未命中 → 回源 → 缓存
```

**后端关注**：
- 缓存穿透：大量 CDN miss 直接打回源站，需做好限流降级
- 缓存一致性：内容更新后需 Purge 或等 TTL 过期
- 日志：源站通过 `X-Forwarded-For` / `X-Real-IP` 获取真实 IP

### 1.4 DNS 优化

1. **减少查询次数**：域名收敛、HTTP/2 多路复用
2. **DNS Prefetch**：`<link rel="dns-prefetch" href="//api.example.com">`
3. **HTTPDNS**：绕过运营商 DNS，直接向 HTTPDNS 服务商请求，避免劫持并获得准确调度
4. **Anycast DNS**：多节点共享 IP，BGP 自动路由最近节点
5. **长连接+连接池**：减少建连频率

### 1.5 DNS 负载均衡

DNS Round-Robin 返回多个 A 记录轮询排序，实现粗粒度负载均衡：

```
example.com. 300 IN A 192.168.1.1
example.com. 300 IN A 192.168.1.2
example.com. 300 IN A 192.168.1.3
```

局限：缓存导致不均匀、无故障剔除、无法感知实时负载。GSLB 结合健康检查+地理位置更智能，但后端内部仍依赖 L4/L7 负载均衡器。

---
