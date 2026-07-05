---
title: Flink 实时计算
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Flink
  - 流处理
categories:
  - 数据工程
---

Flink 是当前最主流的流处理引擎。相比 Spark Streaming 的微批处理（micro-batch），Flink 是真正的逐事件处理，延迟可以从秒级降到毫秒级。

## 一、Flink vs Spark Streaming

| 维度 | Spark Streaming | Flink |
|------|----------------|-------|
| 处理模型 | 微批（micro-batch，通常1-10秒） | 逐事件（event-by-event） |
| 延迟 | 秒级 | 毫秒级 |
| 状态管理 | 有限（依赖外部存储） | 内置 RocksDB 状态后端 |
| 精确一次 | 支持（但有限制） | 原生支持 |
| 窗口 | 基于批次时间 | 事件时间 + Watermark |
| SQL 支持 | Spark SQL | Flink SQL（ANSI SQL 兼容） |

## 二、核心概念

### Watermark——处理乱序数据

现实世界中事件到达顺序可能是乱的。Watermark 是 Flink 判断事件时间窗口"已完成"的机制：

```
事件时间: 10:00  10:01  10:03  10:02(延迟到达)  10:04  10:05
                                              ↑
                              Watermark = 10:04 → 10:00-10:03 的窗口可以关闭了
```

```sql
CREATE TABLE events (
    event_time TIMESTAMP(3),
    user_id STRING,
    action STRING,
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
);
```

### 精确一次（Exactly-Once）

Flink 通过 Checkpoint 机制实现精确一次语义：

```
1. Flink 定期触发 Checkpoint（比如每 60 秒）
2. 暂停数据处理 → 保存所有算子的当前状态（RocksDB Snapshot）→ 保存 Kafka offset
3. 状态保存到持久化存储（S3/HDFS）
4. 故障恢复时：从最近的 Checkpoint 恢复状态 + 从保存的 offset 重放 Kafka
```

## 三、Flink SQL——流批一体

```sql
-- 实时统计每 5 分钟的 PV
SELECT
    TUMBLE_START(event_time, INTERVAL '5' MINUTE) AS window_start,
    COUNT(*) AS pv
FROM events
GROUP BY TUMBLE(event_time, INTERVAL '5' MINUTE);

-- Top-N 热门商品（实时更新）
SELECT *
FROM (
    SELECT item_id, COUNT(*) as cnt,
           ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) as rn
    FROM orders
    GROUP BY item_id
) WHERE rn <= 100;
```

## 四、状态管理与 RocksDB

Flink 的算子状态默认存储在内存中。大规模状态（如用户会话、去重集合）使用 RocksDB 作为状态后端：

```yaml
state.backend: rocksdb
state.backend.rocksdb.localdir: /tmp/flink-state
state.checkpoints.dir: s3://bucket/flink/checkpoints/
```

- RocksDB 将状态持久化到本地 SSD，内存只做缓存
- Checkpoint 时 RocksDB 做增量快照到 S3——只上传变化的部分，节省网络和存储
- 故障恢复时从 S3 拉取最近 Checkpoint 的 RocksDB 文件

## 五、部署模式

| 模式 | 适用 | 说明 |
|------|------|------|
| **Session** | 开发测试 | 共享集群资源，多个 Job 共享 JM/TM |
| **Application** | 生产推荐 | 一个 Job 一个集群，资源隔离 |

```bash
# 提交 Flink Job
flink run-application -t yarn-application \
  -Djobmanager.memory.process.size=4096m \
  -Dtaskmanager.memory.process.size=8192m \
  -Dtaskmanager.numberOfTaskSlots=4 \
  -c com.example.StreamJob \
  s3://bucket/jars/stream-job.jar
```

## 六、小结

Flink 的核心价值是"低延迟 + 精确一次 + 事件时间语义"。实时数仓、实时风控、实时推荐这些对延迟敏感的场景，Flink 是首选方案。如果你的延迟要求是"秒级"，Spark Streaming 可能够用；如果是"毫秒级"，必须 Flink。
