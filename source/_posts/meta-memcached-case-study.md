---
title: Meta Memcached 架构——十亿请求/秒的缓存系统
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 系统设计
  - 缓存
  - Memcached
  - Meta
categories:
  - 系统设计
---

Meta（原 Facebook）的社交图谱是世界上最大的图结构之一。为了在数十亿用户和数万亿条关系上提供亚毫秒级的读取速度，Meta 运行着全球最大的 Memcached 集群——部署在上万台服务器上，处理每秒数十亿次请求。

## 一、为什么是 Memcached

Meta 早在 2008 年就选择了 Memcached 而非 Redis。原因：

| 考量 | Memcached | Redis |
|------|-----------|-------|
| **内存效率** | 极简 slab 分配器，额外开销极小 | 丰富的对象模型，有额外内存开销 |
| **多线程** | 原生多线程，多核利用率高 | 早期单线程（6.0 后才引入多线程 IO） |
| **简单性** | 纯 K-V，不支持持久化 | 支持持久化、复制、集群 |
| **运维** | 无状态，重启即清空 | 有数据持久化，重启需恢复 |

这些设计决定了 Memcached 是**纯缓存**的理想选择——数据可以丢失，速度和内存效率更重要。

## 二、整体架构

```
                        [Web 服务器集群]
                         (PHP/Facebook Hack)
                              │
                    ┌─────────┼─────────┐
                    │         │         │
              [Memcached 集群]  │   [MySQL 集群]
              (分布式缓存层)   │   (持久化存储)
                    │         │         │
                    └─────────┼─────────┘
                              │
                    [TAO 图数据库]
              (关联 Memcached + MySQL)
```

## 三、读路径

```
1. Web 服务器收到用户请求 "获取用户 A 的好友列表"
2. 计算 Cache Key: "friendlist:{userA}"
3. 查询 Memcached → 
   ├─ 命中 → 直接返回（T < 1ms）
   └─ 未命中 → 
       4. 查询 TAO/MySQL 获取好友列表
       5. 将结果写入 Memcached (TTL = 1h)
       6. 返回结果
```

## 四、写路径

Meta 对缓存一致性的处理是**旁路缓存**模式：

```
1. 更新 MySQL: UPDATE friends SET status = 'active' WHERE user_id = ?
2. 删除 Memcached 中的相关 Key: DELETE friendlist:{userid}
   （注意：是删除，不是更新。下次读时自动重建缓存）
```

为什么是删除而非更新？

- 在并发环境下，先更新后删除避免了数据不一致窗口
- 如果写多读少，更新缓存是浪费（写了之后可能没人读就过期了）
- 删除比更新简单（不需要知道新值）

如果删除失败怎么办？Meta 通过 **Lease 机制**防止缓存不一致问题。

## 五、Lease 机制防缓存不一致

核心问题：两个并发请求导致的缓存与数据库不一致（"thundering herd"的变种）：

```
时序:
T1: 用户 A 的缓存过期 → 请求1 查询 MySQL（旧版本）
T2: 请求2 更新 MySQL（新版本）→ 删除缓存
T3: 请求1 将 MySQL 返回的旧版本写入缓存
结果: 缓存中是旧版本，数据库是新版本——永久不一致直到缓存下次过期
```

**Lease 解决方案**：

```
1. 请求1 发现缓存未命中 → Memcached 颁发一个 Lease Token（有效期 5s）
2. 请求1 查询 MySQL，拿到旧版本数据
3. 在 T2 时刻，请求2 更新 MySQL + 删除缓存 + 使 Lease Token 失效
4. 请求1 携带 Lease Token 写入缓存
5. Memcached 发现 Lease Token 已失效 → 拒绝写入
6. 下次请求重新填充缓存 → 从 MySQL 读最新值
```

## 六、Memcached 集群管理

Meta 的 Memcached 集群分布在上万台服务器上。客户端通过一致性哈希确定 key 所在节点：

```
集群分片策略:
  ┌─ [分片0] → memcache-node-1, memcache-node-2 (主备)
  ├─ [分片1] → memcache-node-3, memcache-node-4
  ├─ ...
  └─ [分片N] → memcache-node-(2N-1), memcache-node-2N

Key routing:
  node = hash(key) % N  (一致性哈希环)
```

## 七、Gutter Pool——雪崩保护

当某个 Memcached 节点宕机时，按一致性哈希分配的流量会全部落到下一个节点上，可能导致级联故障。Meta 引入了**Gutter Pool**概念：

```
正常情况：
  key → 主 Memcached 节点 → 命中/未命中 → 正常流程

节点宕机：
  key → 主 Memcached 节点(宕机) → 
        ├─ Gutter Pool（备用的小型缓存集群，临时承接流量）
        └─ MySQL 查询
```

Gutter Pool 是每个 Web 服务器本地的几个备用 Memcached 实例，只有在主集群不可用时才启用。宕机节点的缓存数据会丢失，但 Gutter Pool 防止了 MySQL 被流量瞬间打垮。

## 八、单机优化

Meta 对 Memcached 做了深度定制：

| 优化 | 说明 |
|------|------|
| **UDP 协议** | 读取使用 UDP（减少连接建立开销），写入使用 TCP |
| **Slab 自动调整** | Facebook 修改的 slab 自动平衡算法 |
| **连接池复用** | 每个 Web 服务器维持与 Memcached 的长连接池 |
| **批量请求** | 使用 `get_multi` 批量拉取多个 key |

## 九、关键数据

| 指标 | 数值 |
|------|------|
| 峰值 QPS | 数十亿/秒 |
| 缓存命中率 | > 98%（大量热点数据） |
| 缓存延迟 | < 1ms (单机) |
| 集群规模 | 上万个 Memcached 实例 |
| 总缓存容量 | 多个 PB |

## 十、小结

Meta 的 Memcached 架构证明了：简单的系统可以处理极其复杂的负载——只要设计得当。核心经验包括：Lease 机制解决缓存不一致问题，Gutter Pool 防止雪崩故障，UDP 读取降低协议开销——这些都是在超大规模场景下才能验证的设计智慧，也是系统设计的经典范式。
