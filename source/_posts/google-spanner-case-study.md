---
title: Google Spanner 架构——万亿行数据五个9的高可用
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 系统设计
  - Spanner
  - Google
  - NewSQL
  - 分布式数据库
  - TrueTime
categories:
  - 系统设计
---

Google Spanner 是全球最知名的 NewSQL 数据库，管理着万亿行数据并提供 99.999%（五个9）的可用性。它是第一个在全球范围内提供外部一致性的分布式数据库——这意味着不论数据在世界的哪个角落被修改，所有用户看到的顺序都是一致的。

## 一、解决的问题

传统数据库在可扩展性和一致性之间需要做取舍：

| 方案 | 一致性 | 可扩展性 | 代表 |
|------|--------|----------|------|
| 单机 SQL | 强一致 | 差 | PostgreSQL |
| 分库分表 | 最终一致 | 好 | MySQL 分片 |
| NoSQL | 最终一致 | 极好 | Cassandra |
| **NewSQL** | **强一致** | **极好** | **Spanner** |

Spanner 的目标：在一个全球分布的环境下，提供关系型数据库的 ACID 事务保证，同时具备 NoSQL 的水平扩展能力。

## 二、核心架构

### 2.1 部署拓扑

```
                    [全球 Paxos 集群]
                    ┌─────────────────┐
                    │   Spanner 集群   │
                    │  (Zone A)        │
                    │  ┌───────────┐   │
                    │  │ Paxos 副本 │   │
                    │  │ 强一致读/写 │   │
                    │  └───────────┘   │
                    └─────────────────┘
                             │
                    ┌─────────────────┐
                    │   Spanner 集群   │
                    │  (Zone B)        │
                    │  ┌───────────┐   │
                    │  │ Paxos 副本 │   │
                    │  └───────────┘   │
                    └─────────────────┘
                             │
                    ┌─────────────────┐
                    │   Spanner 集群   │
                    │  (Zone C)        │
                    │  ┌───────────┐   │
                    │  │ Paxos 副本 │   │
                    │  └───────────┘   │
                    └─────────────────┘
```

每个 Spanner 部署被称为一个 **Universe**，每个 Universe 包含多个 Zone。每个 Zone 内有一组 Paxos 副本提供读写服务。

### 2.2 数据分片

数据被切分成 **Split**，每个 Split 包含一系列连续的主键范围：

```
Split A:  [user:000000, user:500000)
Split B:  [user:500000, user:999999)
Split C:  [user:1000000, user:1499999)
```

每个 Split 由一个 Paxos 组管理，Paxos 组的多个副本分布在不同 Zone，保证容灾能力。

### 2.3 读写路径

**写操作**：

```
1. 客户端 → 找到 Split 的 Paxos Leader
2. Leader → 通过 Paxos 协议写入日志
3. 多数派确认 → 写入成功 → 返回客户端
4. Leader 异步应用到存储引擎
```

**读操作**：

```
1. 客户端 → 找到 Split 的 Paxos Leader 或 Follower
2. 如果读 Follower：检查本地时间戳是否 ≥ 读请求的时间戳
   ├─ 是 → 直接返回（外部一致）
   └─ 否 → 等待到足够的时间戳后返回
```

## 三、TrueTime——外部一致性的基石

Spanner 最独特的设计是 **TrueTime**：一个全球精密的时钟同步系统。传统分布式系统不可能实现完全一致的时间同步，Spanner 使用 GPS 和原子钟为每个数据中心提供精确的时间参考。

### 3.1 时间不确性

即使有 GPS 和原子钟，时钟同步也存在不确定性。TrueTime 不报告一个准确的时间点，而是报告一个时间区间：

```
TT.now() → [earliest, latest]

例如: [2026-07-05 10:00:00.000, 2026-07-05 10:00:00.007]
```

这意味着真实时间有 99.9999% 的置信度落在 [earliest, latest] 区间内，区间宽度通常 < 7 毫秒。

### 3.2 如何用不确定性保证确定性

提交等待（Commit Wait）协议是 Spanner 保证外部一致性的关键：

```
事务 T 的提交时间戳: s
提交等待: 在提交后等待，直到 TT.now().earliest > s

也就是说，等待一段时间确保"任何比 s 大的时间戳都已经在真实世界中过去了"。
```

这样保证了：如果事务 A 的提交时间戳 < 事务 B 的提交时间戳，那么 A 在所有节点上都发生在 B 之前。这提供了一种近似完美的外部一致性，概率满足。

## 四、Paxos 副本与数据复制

Spanner 使用 Paxos 协议管理每个 Split 的数据复制。Paxos 组的成员分布在多个数据中心：

```
Split A 的 Paxos 组 (3 副本):
  Zone A: Leader (处理所有写 + 强一致读)
  Zone B: Follower (只读，复制延迟 ~10ms)
  Zone C: Follower (只读，复制延迟 ~50ms)
```

当 Leader 所在的 Zone 故障时，Paxos 协议保证剩余的副本能在秒级选出新 Leader，满足 5 个 9 的可用性要求。

## 五、Schema 设计

Spanner 支持标准 SQL 并且扩展了分布式特有的功能：

```sql
CREATE TABLE Users (
  user_id   INT64 NOT NULL,
  name      STRING(100),
  email     STRING(100),
  created_at TIMESTAMP,
) PRIMARY KEY (user_id);

-- 交错表：子表与父表共享主键前缀，数据物理上相邻
CREATE TABLE UserOrders (
  user_id   INT64 NOT NULL,
  order_id  INT64 NOT NULL,
  amount    FLOAT64,
) PRIMARY KEY (user_id, order_id),
  INTERLEAVE IN PARENT Users;
```

**交错表（Interleaving）**是 Spanner 特有的设计：将父子表的数据物理上存储在一起，使得"根据用户查其所有订单"只需要一次磁盘扫描，而不需要分布式 JOIN。

## 六、性能优化

| 优化策略 | 说明 |
|----------|------|
| **交错表** | 消除分布式 JOIN |
| **只读副本** | Follower 可以处理只读事务，分散读压力 |
| **批量写入** | 聚合多个小写入为一次 Paxos 提交 |
| **客户端路由** | 客户端智能路由到最近的 Follower 读取 |
| **热点检测** | 自动检测并分裂热点 Split |

## 七、小结

Google Spanner 解决了分布式数据库的"圣杯问题"：在保持 SQL 语义和 ACID 保证的前提下实现全球级扩展性。这得益于两项创新：TrueTime 使用物理时间（GPS/原子钟）取代逻辑时钟，为分布式事务提供了确定性的时间参考；Paxos 副本 + 交错表则在高可用和高性能之间找到平衡。

Spanner 的开源等价物包括 CockroachDB 和 TiDB——它们都受到 Spanner 论文的启发，但使用混合逻辑时钟（HLC）代替 GPS 时钟，在易部署性和一致性之间做出不同的取舍。
