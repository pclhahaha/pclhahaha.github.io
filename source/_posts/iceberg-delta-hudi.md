---
title: Iceberg / Delta Lake / Hudi 表格式对比
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Iceberg
  - Delta Lake
  - Hudi
  - 数据湖
categories:
  - 数据工程
---

数据湖的原始形态是直接在 S3/HDFS 上写 Parquet 文件——没有表语义、没有 ACID、没有时间旅行。Iceberg、Delta Lake、Hudi 这三大表格式在原始 Parquet 之上添加了一层元数据管理，让数据湖有了数仓的可靠性。

## 一、为什么需要表格式

```
无表格式的 S3 Parquet:
  s3://bucket/events/dt=2026-07-01/
    part-0001.parquet
    part-0002.parquet
  s3://bucket/events/dt=2026-07-02/
    part-0003.parquet

问题：
  - 没有 schema 管理：不知道表有哪些列、类型是什么
  - 没有事务：写入过程中其他人读到不完整数据
  - 没有时间旅行：无法回退
  - 分区变更需重写数据
```

```
有表格式（如 Iceberg）:
  s3://bucket/events/
    metadata/
      v1.json → 当前版本的表结构+文件清单
      snap-*.avro → 每次写入的快照
    data/
      dt=2026-07-01/file-1.parquet
```

## 二、三方案对比

| 特性 | Iceberg | Delta Lake | Hudi |
|------|---------|------------|------|
| **创始** | Netflix | Databricks | Uber |
| **ACID** | ✅ 乐观并发 | ✅ 乐观并发 | ✅ OCC/MVCC |
| **时间旅行** | ✅ Snapshot | ✅ Version | ✅ Commit Timeline |
| **Schema 演化** | ✅ 完整 | ✅ 完整 | ✅ 完整 |
| **分区演化** | ✅ 元数据变更即可 | ❌ 需重写数据 | ❌ 需重写数据 |
| **生态** | 中立，Spark/Flink/Trino/Presto | Spark 为主，Trino 有限 | Spark/Flink/Hive |
| **流式写入** | 有限 | 有限 | **最佳**（Upsert/Delete原生支持） |
| **文件格式** | Parquet/ORC/Avro | Parquet | Parquet/ORC |

## 三、Iceberg——最中立的方案

Iceberg 由 Netflix 捐赠给 Apache，是三者中最具生态中立性的方案。它的关键优势：

```sql
-- 时间旅行
SELECT * FROM events FOR SYSTEM_VERSION AS OF '2026-07-01 00:00:00';

-- Schema 演化（安全添加列）
ALTER TABLE events ADD COLUMN new_field STRING;

-- 分区演化（不重写数据改分区策略）
ALTER TABLE events REPLACE PARTITION FIELD (days(dt));
```

**分区演化**是 Iceberg 的独特能力——可以改变分区策略而无需重写历史数据，新策略只对未来的写入生效，查询时 Iceberg 透明处理新旧分区。

## 四、Delta Lake——Spark 首选

Delta Lake 由 Databricks 主导，与 Spark 深度集成：

```python
# Delta Lake 写入
df.write.format("delta").mode("overwrite").save("s3://bucket/table/")

# 时间旅行
spark.read.format("delta").option("versionAsOf", 5).load("s3://bucket/table/")

# MERGE (Upsert)
delta_table.alias("t").merge(
    updates.alias("u"), "t.id = u.id"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()

# Vacuum（清理旧版本文件）
delta_table.vacuum(retentionHours=168)
```

Delta Lake 与 Databricks 的 Photon 引擎深度绑定，在 Databricks 环境下性能无可匹敌，但在开源 Spark 上的性能与 Iceberg 相当。VACUUM 操作需定期执行，否则版本文件会无限增长。

## 五、Hudi——流式场景首选

Hudi 由 Uber 开发，设计初衷是解决流式数据的 Upsert 问题：

```python
# Hudi 流式写入
df.write.format("hudi") \
    .option("hoodie.table.name", "events") \
    .option("hoodie.datasource.write.operation", "upsert") \
    .option("hoodie.datasource.write.recordkey.field", "id") \
    .option("hoodie.datasource.write.precombine.field", "ts") \
    .mode("append") \
    .save("s3://bucket/events/")
```

Hudi 的两种表类型：

| 类型 | 读优化 | 写优化 | 适用 |
|------|--------|--------|------|
| **COW** (Copy on Write) | 列存 Parquet | 全量重写 | 读多写少 |
| **MOR** (Merge on Read) | Base Parquet + Delta Avro | 增量追加 | 写多读少、实时更新 |

MOR 表在写入时只追加增量日志（Avro 格式），查询时合并 Base 和 Delta 的数据——牺牲少量读性能换取近实时的写入延迟。

## 六、选型建议

| 场景 | 推荐 |
|------|------|
| 通用数据湖、多引擎环境 | **Iceberg**（最中立） |
| Databricks/Spark 为主 | **Delta Lake**（原生集成） |
| 流式 Upsert、CDC | **Hudi**（流式写入最强） |
| AWS EMR + S3 | **Iceberg** 或 **Hudi** |

## 七、小结

三种表格式的本质都是在 Parquet 之上加"表管理层"。Iceberg 赢在中立性和分区演化，Delta Lake 赢在 Spark 生态和 Databricks 支持，Hudi 赢在流式 Upsert。选型时最重要的不是功能对比表，而是你的计算引擎生态——选和已有基础设施集成最好的。
