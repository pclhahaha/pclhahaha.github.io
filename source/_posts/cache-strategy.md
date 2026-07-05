---
title: 缓存策略总论
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 缓存
  - Redis
  - CDN
  - 系统设计
categories:
  - 系统设计
---

缓存是后端性能优化的第一手段。从 CPU 的 L1 Cache 到浏览器的 Service Worker，缓存无处不在。本文串起各级缓存的原理和策略。

## 一、缓存的层级

```
请求延时         缓存层级
  ↓
 1ms     ←  浏览器缓存（Service Worker, HTTP Cache）
 2ms     ←  CDN 边缘缓存
 5ms     ←  反向代理缓存（Nginx）
 10ms    ←  本地缓存（Caffeine/Guava Cache）
 20ms    ←  分布式缓存（Redis/Memcached）
 50ms    ←  数据库查询缓存（MySQL Query Cache/Buffer Pool）
100ms    ←  磁盘
```

**核心原则**：离用户越近，延迟越低，容量越小。CDN 边缘节点覆盖全球但不存全量数据，本地 JVM 缓存延迟微秒级但容量只有几十 MB。

## 二、缓存模式

### 2.1 Cache-Aside（旁路缓存）

```
读：
  App → Cache → Miss → DB → 写入 Cache → 返回

写：
  App → DB（更新） → Cache（删除）
```

最常见的模式。应用自己管理缓存的加载和失效。

**为什么写时删除而非更新？** 删除后下次读自然重建，避免并发更新导致的不一致。

```java
public User getUser(Long id) {
    String key = "user:" + id;
    User user = (User) redis.get(key);
    if (user != null) return user;
    
    user = db.findById(id);
    if (user != null) {
        redis.setex(key, 3600, user);
    }
    return user;
}
```

### 2.2 Read-Through / Write-Through

```
读：App → Cache（自动查 DB）→ 返回
写：App → Cache（同时写 DB）→ 返回
```

缓存层封装了数据加载逻辑，应用不需要关心缓存和数据库的协作。常见于 Redis + Lua 脚本或专门的缓存框架。

### 2.3 Write-Behind（异步写回）

```
写：App → Cache（写入成功）→ 异步批量 → DB
```

写入只写到缓存就返回，异步批量刷到 DB。性能最高，但宕机可能丢数据。适合对一致性要求不高的场景，如用户行为日志、页面访问计数。

## 三、缓存三大问题

| 问题 | 现象 | 解法 |
|------|------|------|
| **穿透** | 查询不存在的数据，缓存和 DB 都没有 | 布隆过滤器、空值缓存 |
| **击穿** | 热点 key 过期瞬间，大量请求直达 DB | 互斥锁、永不过期+异步刷新 |
| **雪崩** | 大量 key 同时过期，DB 压力骤增 | TTL 加随机值、多级缓存、限流 |

### 3.1 穿透——布隆过滤器

```
请求 → BloomFilter.contains(key)
        ├─ false → 一定不存在 → 直接拒绝（不查 DB）
        └─ true  → 可能存在 → 查缓存 → 查 DB
```

布隆过滤器是概率数据结构——可能误判"存在"，但不会漏判"不存在"。用于快速过滤掉大多数穿透请求。

### 3.2 击穿——互斥锁

```java
public String getWithLock(String key) {
    String val = redis.get(key);
    if (val != null) return val;

    String lockKey = "lock:" + key;
    if (redis.setnx(lockKey, "1")) {
        redis.expire(lockKey, 10);
        val = db.query(key);
        redis.set(key, val);
        redis.del(lockKey);
        return val;
    }
    Thread.sleep(100);
    return getWithLock(key);  // 重试
}
```

### 3.3 雪崩——随机 TTL

```java
int baseTTL = 3600;
int randomTTL = ThreadLocalRandom.current().nextInt(600); // 0-600s
redis.setex(key, baseTTL + randomTTL, value);
```

## 四、缓存一致性

缓存和数据库双写时的数据一致性是分布式系统的经典难题。

| 策略 | 做法 | 不一致窗口 | 适用场景 |
|------|------|-----------|----------|
| 先删缓存再更新 DB | 删除 → 更新 | 大 | 不推荐 |
| 先更新 DB 再删缓存 | 更新 → 删除 | 很小 | 一般场景 |
| 延迟双删 | 删 → 更新 → 等 N 秒 → 再删 | 可控 | 并发高的读写场景 |
| Canal + MQ | 监听 binlog → 异步更新缓存 | 极小 | 强一致性要求 |

**推荐**：绝大多数场景用"先更新 DB 再删缓存"，配合重试机制。对一致性要求极高的场景用 Canal/MQ 或直接让它走主库查询。

## 五、淘汰策略

Redis 提供了多种内存淘汰策略：

| 策略 | 行为 |
|------|------|
| `noeviction` | 内存满时拒绝写入 |
| `allkeys-lru` | 从所有 key 中淘汰最久未用的 |
| `volatile-lru` | 从设了 TTL 的 key 中淘汰最久未用的 |
| `allkeys-lfu` | 从所有 key 中淘汰最少使用的 |
| `volatile-ttl` | 淘汰 TTL 最短的 |

**推荐**：缓存场景用 `allkeys-lru`；既有缓存又有持久化数据用 `volatile-lru`。

## 六、CDN 缓存

CDN 缓存是离用户最近的一级缓存。关键配置：

```
Cache-Control: public, max-age=86400          # 浏览器+CDN 都缓存
Cache-Control: public, s-maxage=3600          # 仅 CDN 缓存 1h，浏览器不缓存
Cache-Control: no-cache                       # 每次都验证（但可缓存）
Cache-Control: no-store                       # 完全不缓存
```

CDN 缓存清除：
- **Purge**：手动或 API 清除指定 URL 的缓存
- **版本化 URL**：`/app.v2.js` → `/app.v3.js`，天然绕过缓存
- **Cache Key 差异化**：基于 Header/Cookie 内容生成不同缓存版本

## 七、多级缓存

实际生产环境中通常不止一级缓存：

```
Nginx 本地缓存（Top 100，5s TTL）
  ↓ Miss
Redis Cluster（全量热点，1-30min TTL）
  ↓ Miss
Caffeine 本地缓存（热点中的热点，1min TTL）
  ↓ Miss
MySQL（持久层）
```

每一级都拦截一部分流量，逐级减压。

## 八、小结

缓存的本质是"空间换时间"和"近处换远处"。Cache-Aside 是最通用的模式，三大问题有固定的解法，CDN 是最容易被忽视但效果最明显的一级缓存。做好缓存能让系统的读吞吐量提升 10-100 倍——但记住，缓存是数据一致性的负债，设计时需要明确回答"读到旧数据能否接受"。
