---
title: Redshift 数仓实战
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Redshift
  - 数据仓库
  - AWS
categories:
  - 数据工程
---

Redshift 是 AWS 的云原生数据仓库，基于 PostgreSQL 8.0 分支开发，核心设计是列式存储 + 大规模并行处理（MPP）。它的目标场景不是 OLTP，而是对 PB 级数据做复杂的分析查询。

## 一、架构

```
Leader Node（查询解析 + 执行计划 + 结果聚合）
    │
    ├── Compute Node 1 ── Slice 0, Slice 1
    ├── Compute Node 2 ── Slice 2, Slice 3
    └── Compute Node N ── Slice 2N-2, Slice 2N-1
```

- **Leader Node**：接收查询，生成执行计划，聚合结果
- **Compute Node**：存储数据并执行查询。每个节点分为多个 Slice（分区），每个 Slice 获得表的一部分数据
- **RA3 架构**：计算和存储分离——数据存储在 S3 上（Redshift Managed Storage），计算节点只缓存热数据

## 二、数据分布（Distribution Key）

分布键决定了数据如何在计算节点间分布——这是 Redshift 性能最关键的设计决策。

```sql
CREATE TABLE orders (
    order_id    BIGINT,
    user_id     BIGINT DISTKEY,     -- 按 user_id 分布
    amount      DECIMAL(18,2),
    created_at  TIMESTAMP
) DISTSTYLE KEY DISTKEY(user_id);
```

| 分布方式 | 说明 | 适用场景 |
|----------|------|----------|
| **KEY** | 按指定列的哈希分布 | JOIN 键相同的行落在同一节点，避免数据跨节点重分布 |
| **ALL** | 全量复制到每个节点 | 小维度表（< 10万行） |
| **EVEN** | 轮询分布 | 没有明显 JOIN 键的大表 |
| **AUTO** | Redshift 自动选择 | 不确定时使用 |

注意：分布键选错会导致大量数据在节点间重分布（Redistribute），这是 Redshift 查询慢的第一大原因。

## 三、排序键（Sort Key）

排序键决定了数据在每个 Slice 内的物理存储顺序——直接影响查询**需要扫描多少数据**。

```sql
CREATE TABLE events (
    event_id    BIGINT,
    user_id     BIGINT,
    event_type  VARCHAR(50),
    dt          DATE
) DISTKEY(user_id) COMPOUND SORTKEY(dt, event_type);
```

| 排序方式 | 说明 |
|----------|------|
| **Compound** | 分层排序——先按 dt 排序，相同 dt 内按 event_type 排序。适合 `WHERE dt = '2026-07-01' AND event_type = 'click'` 这种前缀过滤 |
| **Interleaved** | 交叉排序——给每列同等权重。适合每列都单独过滤但很少组合过滤的场景 |

选择标准：**把 WHERE 条件中最常出现的列放在最前面**。如果查询总是带 `WHERE dt = ?`，dt 就是最好的排序列开头。

## 四、Vacuum 与 Analyze

Redshift 需要定期维护才能保持性能：

```sql
-- VACUUM：回收被删除/更新行占用的空间，重新排序
VACUUM FULL events;         -- 完全重排（耗时）
VACUUM DELETE ONLY events;  -- 仅回收空间（更快）

-- ANALYZE：更新统计信息，帮助查询优化器做出更好的计划
ANALYZE events;
ANALYZE events PREDICATE COLUMNS;  -- 只为过滤列更新统计
```

自动化建议：维护窗口内对前一天的表执行 VACUUM + ANALYZE。

## 五、查询优化

### 避免跨节点数据移动

```sql
-- ❌ EXPLAIN 会显示 DS_DIST_NONE（重分布）
SELECT * FROM orders o JOIN users u ON o.city = u.city;
-- 分布键 order_id 和 user_id 不匹配，需要重分布

-- ✅ 用 DISTKEY 做 JOIN，DS_DIST_NONE（无数据移动）
SELECT * FROM orders o JOIN users u ON o.user_id = u.user_id;
```

### 避免 Leader Node 压力

```sql
-- ❌ Leader 节点需要聚合所有节点结果后排序
SELECT * FROM events ORDER BY event_time DESC LIMIT 100;

-- ✅ 加上 dt 分区过滤，数据只在少量 Slice 内排序然后合并
SELECT * FROM events WHERE dt >= '2026-07-01' ORDER BY event_time DESC LIMIT 100;
```

## 六、Spectrum——查询 S3 数据湖

Redshift Spectrum 可以直接查询 S3 上的数据而无需加载到 Redshift 本地磁盘：

```sql
CREATE EXTERNAL TABLE spectrum.events (
    event_id    BIGINT,
    user_id     BIGINT,
    event_type  VARCHAR(50),
    dt          DATE
)
PARTITIONED BY (dt DATE)
STORED AS PARQUET
LOCATION 's3://bucket/events/';
```

适用场景：**冷数据放 S3 用 Spectrum 查，热数据在 Redshift 本地查**——降低存储成本且无需维护 VACUUM。

## 七、小结

Redshift 性能优化三原则：选对 DISTKEY 避免数据移动、选对 SORTKEY 减少扫描量、定期 VACUUM+ANALYZE 保持统计信息准确。在这三项都做好后，再考虑物化视图和 concurrency scaling 等进阶优化。
