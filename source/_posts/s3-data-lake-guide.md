---
title: S3 数据湖最佳实践
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - S3
  - 数据湖
  - AWS
categories:
  - 数据工程
---

S3 是现代数据湖的默认存储底座。但它不是文件系统——把 HDFS 的使用习惯搬到 S3 上会踩坑。这篇文章聚焦数据工程师日常面对的 S3 问题。

## 一、S3 不是文件系统

| 维度 | HDFS | S3 |
|------|------|-----|
| 一致性 | 强一致（写入立即可见） | 最终一致（写入后有短暂延迟，已大幅改善） |
| Rename | 原子操作，O(1) | Copy + Delete，O(N)，非原子 |
| List | 快（本地目录结构） | 慢（HTTP LIST 请求，按请求数计费） |
| 追加写 | 支持 | 不支持（对象不可变） |
| 适合模式 | 流式写、追加 | 一次写多次读、不可变 |

## 二、分区设计

好的分区是 S3 性能的基石：

```
s3://bucket/events/
  dt=2026-07-01/
    hour=00/
    hour=01/
  dt=2026-07-02/
    ...
```

**分区粒度**：
- 太粗（按年）→ 单分区数据量过大，查询要扫全量
- 太细（按分钟）→ 数百万分区，LIST 操作成为瓶颈
- 推荐：按天分区，高流量加按小时子分区

**分区下推**（Partition Pruning）：

```sql
-- ✅ 好：Spark 只会 LIST dt=2026-07-05/ 下的文件
SELECT * FROM events WHERE dt = '2026-07-05'

-- ❌ 坏：无法下推分区，需要扫全部 dt 分区
SELECT * FROM events WHERE date_format(event_time, 'yyyy-MM-dd') = '2026-07-05'
```

## 三、文件大小与格式

**小文件是第一大性能杀手**。S3 的 LIST 操作收费且慢，每个文件都需要一次 HEAD 请求获取元数据。

```
推荐：256MB-1GB/文件
最小：> 50MB/文件（低于此阈值考虑合并）
```

**压缩格式选择**：

| 格式 | 可分割 | 压缩率 | 适合 |
|------|--------|--------|------|
| Parquet + Snappy | ✅ | 中等 | Spark 查询首选 |
| Parquet + ZSTD | ✅ | 高 | 存储优化 |
| gzip | ❌ | 高 | 小文件归档 |
| ORC + ZLIB | ✅ | 高 | Hive 查询 |

**可分割（Splittable）很重要**：gzip 压缩的文件不能分割——一个 10GB 的 gzip 文件只能由一个 Task 处理，无法并行读取，导致单 Task 执行过慢甚至 OOM。Parquet/ORC 天然支持按 Row Group/Stripe 并行读取。

## 四、Iceberg——表格式管理

原始 parquet 文件缺少表语义——没有 schema 演化、没有 ACID、不知道哪些文件属于哪个分区。

Iceberg 在 Parquet 之上添加了表元数据层：

```
s3://bucket/warehouse/events/
  metadata/
    v1.json       ← 表结构 + 快照
    v2.json       ← 新增分区信息
    snap-*.avro   ← 文件清单
  data/
    dt=2026-07-01/
      file-1.parquet
      file-2.parquet
```

核心能力：
- **时间旅行**：`SELECT * FROM events FOR SYSTEM_VERSION AS OF '2026-07-01'`
- **Schema 演化**：加列、改列类型（部分兼容），不需要重建表
- **分区演化**：修改分区策略，元数据更新即可，无需重写数据文件
- **ACID**：通过乐观并发控制支持多写入者、多读取者

## 五、存储分层

```
S3 Standard → S3 Infrequent Access → S3 Glacier
   热数据         温数据（30天+）       冷归档（90天+）
   ~$0.023/GB      ~$0.012/GB          ~$0.004/GB
```

用 S3 生命周期策略自动迁移：

```json
{
  "Rules": [{
    "Id": "archive-old-data",
    "Prefix": "dt=2025/",
    "Transitions": [
      {"Days": 30,  "StorageClass": "STANDARD_IA"},
      {"Days": 90,  "StorageClass": "GLACIER"}
    ]
  }]
}
```

注意：查询 Glacier 数据前需要先恢复（Restore），耗时数分钟到数小时。

## 六、成本优化

| 优化 | 节省 |
|------|------|
| 合并小文件 | 减少 LIST/PUT 请求（$0.005/1000次） |
| 用 S3 Select 而非全量下载 | 只传输需要的数据 |
| S3 Transfer Acceleration | 远距离上传提速（按加速流量收费） |
| VPC Endpoint | 免去 NAT 网关的数据传输费 |

## 七、小结

S3 数据湖的核心实践：按天分区 + 256MB 以上 Parquet 文件 + Iceberg 表管理 + 生命周期自动分层。做好这四点，性能和成本都能控制住。
