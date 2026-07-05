---
title: Hive 与 Presto/Trino 实战
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Hive
  - Presto
  - Trino
  - SQL
categories:
  - 数据工程
---

Hive 是大数据 SQL 查询引擎的鼻祖，但今天它的角色已经从"查询引擎"转变为"元数据管理 + ETL 管道"。Presto/Trino 则凭借内存计算能力接替了交互式查询的角色。

## 一、Hive 的核心概念

```
用户 SQL → Hive Metastore (MySQL) → 获取表 Schema/分区信息
                                       ↓
                              Hive 执行引擎
                              (MapReduce/Tez/Spark)
                                       ↓
                              HDFS / S3 上的数据文件
```

### 表类型

| 类型 | 数据位置 | 删除表 | 适用场景 |
|------|----------|--------|----------|
| **MANAGED TABLE** | Hive 仓库目录 | 数据也被删除 | 临时表 |
| **EXTERNAL TABLE** | 外部路径（如 S3） | 只删元数据，数据保留 | 数据湖标准做法 |

```sql
-- External table（数据湖推荐）
CREATE EXTERNAL TABLE events (
    event_id    BIGINT,
    user_id     BIGINT,
    event_type  STRING
)
PARTITIONED BY (dt STRING)
STORED AS PARQUET
LOCATION 's3://bucket/events/';
```

### 分区管理

```sql
-- 手动添加分区
ALTER TABLE events ADD PARTITION (dt='2026-07-01') 
LOCATION 's3://bucket/events/dt=2026-07-01/';

-- 自动恢复所有分区（元数据同步）
MSCK REPAIR TABLE events;
```

注意：`MSCK REPAIR` 在大量分区时会非常慢——它需要列出 S3 上的所有前缀。如果分区数超过几千个，应考虑直接使用 Iceberg 等表格式自管理分区的元数据，而非依赖 Hive Metastore 的分区机制。

## 二、Hive on S3 的坑

### 小文件导致慢查询

Hive 的 MapReduce 执行引擎默认为每个文件启动一个 Mapper。如果 S3 上有 10000 个小文件，就会启动 10000 个 Mapper——极度浪费且慢。

**解决方案**：
- ETL 输出时 `coalesce` 合并文件
- 定期运行小文件合并任务
- 使用 `hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat`（自动在 Map 端合并小文件）

### Hive Metastore 膨胀

每个分区都在 MySQL Metastore 中占用一条记录。数百万分区 → Metastore 查询变慢。

**解决方案**：用 Iceberg/Hudi 等表格式替代 Hive 原生分区管理。

## 三、Presto/Trino——交互式查询引擎

Trino（前身 PrestoSQL）是一个分布式 SQL 查询引擎，与 Hive 的关键区别：

| 维度 | Hive | Trino |
|------|------|-------|
| 执行模型 | MapReduce/Tez（磁盘中间结果） | 纯内存管道（Pipeline） |
| 延迟 | 分钟级 | 秒级（交互式查询） |
| 容错 | 支持重跑失败 Task | 查询失败需要重新提交 |
| 适合 | 批处理 ETL | BI 报表、Ad-hoc 查询 |

Trino 的核心理念是 **"纯内存、无磁盘"的流水线执行**——数据在内存中从一个算子流向另一个算子，中间结果不落盘。这带来了极快的响应速度，但代价是无法处理超大内存的数据集和单节点故障导致全查询重跑。

```sql
-- Trino 查询（和 Hive SQL 几乎一样）
SELECT user_id, COUNT(*) as cnt
FROM hive.events
WHERE dt >= '2026-07-01'
GROUP BY user_id
ORDER BY cnt DESC
LIMIT 100;
```

Trino 可以连接多个数据源做联邦查询：

```sql
-- 关联 Hive 表（S3）和 PostgreSQL 表
SELECT h.user_id, p.user_name, COUNT(*) as cnt
FROM hive.events h
JOIN postgresql.public.users p ON h.user_id = p.id
WHERE h.dt = '2026-07-05'
GROUP BY h.user_id, p.user_name;
```

## 四、执行引擎演进

```
Hive on MapReduce（2012）→ Hive on Tez（2015）→ Hive on Spark（2016）→ 
Presto/Trino 交互式查询（2018+）→ Spark SQL 批处理（当前主力）
```

当前最佳实践：
- **ETL 管道**：Spark on EMR（性能 + 生态）
- **交互式查询**：Trino（亚秒-秒级）
- **元数据管理**：Hive Metastore / AWS Glue Catalog
- **表格式**：Iceberg（取代传统 Hive 分区）

## 五、小结

Hive 的定位已经从全能选手转变为元数据入口，真正的计算能力由 Spark 和 Trino 承担。实践中常见的是"Hive Metastore 管理表定义 + Spark 做 ETL + Trino 做 BI 查询"的组合。
