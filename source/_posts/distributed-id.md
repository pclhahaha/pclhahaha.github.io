---
title: 分布式 ID 生成方案
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - 分布式ID
  - Snowflake
  - Leaf
  - UUID
categories:
  - 分布式
---

### 1.1 为什么需要分布式 ID

在单库系统中，数据库自增主键可以满足 ID 生成的唯一性需求。但分库分表后，多个数据库独立增长会导致 ID 冲突；而业务 ID 通常需要全局唯一、趋势递增（利于索引），因此需要专门的分布式 ID 生成方案。

### 1.2 UUID

```
550e8400-e29b-41d4-a716-446655440000
```

| 优点 | 缺点 |
|------|------|
| 本地生成，零网络开销 | 非自增、字符串占空间大（36字符） |
| 全局唯一，无单点 | 作为 InnoDB 聚簇索引时页分裂严重 |
| 实现简单 | 不包含时间信息，不可排序 |

适用场景：非主键的唯一标识（如日志 traceId、临时 token）。

### 1.3 数据库自增ID

通过独立的 ID 生成表（或号段表）获取全局自增 ID：

```sql
CREATE TABLE id_generator (
    biz_tag VARCHAR(32) PRIMARY KEY,
    max_id  BIGINT NOT NULL,
    step    INT DEFAULT 1000
);

-- 批量获取号段（原子操作）
UPDATE id_generator SET max_id = max_id + step WHERE biz_tag = 'order';
SELECT max_id FROM id_generator WHERE biz_tag = 'order';
```

**问题**：DB 成为单点瓶颈。一次 DB 交互只能获取 `step` 个 ID，用完后再请求。

### 1.4 号段模式（Leaf-segment）

美团 Leaf 的号段模式是对数据库方案的升级：

- 客户端先批量获取一个号段（如 [0, 1000]），缓存在本地
- 本地内存中基于号段分配 ID，无需每次访问 DB
- **双 buffer 优化**：当前号段消费到 10% 时，异步预加载下一个号段到备用 buffer，号段切换时无缝衔接

```java
// 双 buffer 号段模式核心
public class SegmentBuffer {
    private Segment[] segments = new Segment[2];  // 双 buffer
    private volatile int currentPos;              // 当前使用的 segment 索引
    
    public long getId() {
        long id = segments[currentPos].nextId();
        if (segments[currentPos].usageRate() > 0.9) {
            // 异步预加载备用 buffer
            asyncLoadSegment(1 - currentPos);
        }
        if (segments[currentPos].isExhausted()) {
            currentPos = 1 - currentPos;  // 切换 buffer
        }
        return id;
    }
}
```

优点：解决了 DB 的性能瓶颈，可实现 10w+ QPS；缺点：ID 不是严格全局自增（多实例各自消耗号段，时间线可能交错）。

### 1.5 Snowflake（雪花算法）

Twitter 开源的经典方案，ID 为 64 位长整型：

```
┌─┬───────────────────────────────────────┬──────────┬────────────────┐
│0│           41-bit timestamp            │10-bit    │   12-bit       │
│ │           (毫秒，约69年)              │ workerId │  sequence      │
└─┴───────────────────────────────────────┴──────────┴────────────────┘
```

每毫秒单机可生成 4096 个 ID，整体趋势递增。

#### 时钟回拨问题及解决方案

Snowflake 严重依赖机器时钟。时钟回拨会导致 ID 重复：

**方案一：抛异常**（Leaf-snowflake 默认）：
- 检测到回拨后拒绝服务，等待时钟追上

**方案二：使用历史时间戳**：
- 回拨发生时，沿用之前的时间戳生成 sequence，直到时钟追上

**方案三：扩展位 + 防重**：
- 在 ID 中增加 1-bit clock-back 标识，记录回拨事件

**方案四：借用未来时间**（百度 UidGenerator）：
- 基于 RingBuffer 预生成 ID，缓存最近一段时间的时间戳，回拨时直接使用缓存

```java
// 时钟回拨检测核心逻辑
public synchronized long nextId() {
    long currentTime = System.currentTimeMillis();
    if (currentTime < lastTimestamp) {
        long offset = lastTimestamp - currentTime;
        if (offset <= MAX_BACKWARD_MS) {  // 容忍小范围回拨（< 5ms）
            // 等待时钟追上
            Thread.sleep(offset);
            currentTime = System.currentTimeMillis();
        } else {
            throw new ClockBackwardsException("Clock moved backwards: " + offset);
        }
    }
    if (currentTime == lastTimestamp) {
        sequence = (sequence + 1) & SEQUENCE_MASK;
        if (sequence == 0) {
            currentTime = waitNextMillis(lastTimestamp);
        }
    } else {
        sequence = 0;
    }
    lastTimestamp = currentTime;
    return (currentTime - EPOCH) << TIMESTAMP_SHIFT
         | workerId << WORKER_ID_SHIFT
         | sequence;
}
```

### 1.6 Redis 自增

利用 Redis 单线程特性与 `INCR` / `INCRBY` 命令：

```
INCR order_id      → 返回 1001
INCRBY order_id 10 → 返回 1011（预取号段）
```

| 优点 | 缺点 |
|------|------|
| 高性能、天然递增 | 依赖 Redis 持久化（RDB/AOF），宕机可能丢失 |
| 实现极简 | 非严格趋势递增（主从切换后可能回退） |
| 可横向扩展（分段 key） | 需额外维护 Redis 集群 |

### 1.7 各方案对比

| 方案 | 趋势递增 | 高可用 | 性能 | 依赖 | 适用场景 |
|------|----------|--------|------|------|----------|
| UUID | 否 | 极高 | 极高 | 无 | 日志 traceId |
| DB 自增 | 是 | 低 | 低 | DB | 小规模 |
| Leaf-segment | 近似 | 高 | 极高 | DB + 自有服务 | 大规模分库分表 |
| Snowflake | 近似（时间序） | 高 | 极高 | 无（本地生成） | 通用高并发 |
| Redis | 是 | 中 | 高 | Redis | 已有 Redis 的中小规模 |

### 1.8 美团 Leaf 与百度 UidGenerator

**美团 Leaf**：双模式架构——号段模式 + Snowflake 模式，通过 ZooKeeper 注册 workerId 并做心跳续约。

**百度 UidGenerator**：
- **DefaultUidGenerator**：标准 Snowflake 实现，基于数据库表分配 workerId
- **CachedUidGenerator**：基于 RingBuffer（Disruptor 思想），提前批量生成 ID 并缓存在环形队列中，消除 Snowflake 的 synchronized 竞争；支持容忍时钟回拨
