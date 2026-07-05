---
title: Apache Spark 深度解析
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Spark
  - EMR
  - 大数据
categories:
  - 数据工程
---

## 一、Spark 架构概览

```
Driver Program
    │
SparkContext
    │
Cluster Manager (YARN / K8s / Standalone)
    │
┌───┴───┬───────┬───────┐
Worker  Worker  Worker  Worker
│       │       │       │
Executor Executor Executor Executor
(Task)  (Task)  (Task)  (Task)
```

- **Driver**：运行 main 方法，解析用户代码，生成 DAG，调度 Task
- **Executor**：运行在 Worker 节点上，执行 Task，缓存 RDD/DataFrame
- **Cluster Manager**：资源调度器。在 EMR 上通常是 YARN

## 二、RDD vs DataFrame vs Dataset

| | RDD | DataFrame | Dataset |
|------|-----|-----------|---------|
| 类型安全 | 是（编译时） | 否（运行时） | 是（编译时） |
| 优化 | 无 | Catalyst 优化器 | Catalyst 优化器 |
| 序列化 | Java/Kryo | Tungsten 二进制 | Encoder |
| API | 函数式 | 声明式 SQL | 两者兼具 |
| 推荐度 | 仅底层操作 | **首选** | 需要类型安全时 |

**现在写 Spark 首选 DataFrame API**——它在 Spark 2.0 后是主力 API，Catalyst 优化器能自动做谓词下推、列裁剪、常量折叠等优化。

## 三、Spark 作业执行流程

```
用户代码 → [逻辑计划] → [Catalyst 优化] → [物理计划] → [DAG] → [Stage] → [Task]
```

### DAG 与 Shuffle

Spark 将作业划分为 Stage，Stage 的边界是 **宽依赖（Shuffle）** 操作：

```python
# 读取 → filter (窄依赖) → groupBy (宽依赖，触发 Shuffle) → count
df = spark.read.parquet("s3://bucket/events/")
df.filter(col("status") == "active") \
  .groupBy("user_id") \
  .count() \
  .show()
```

宽依赖操作的代价：数据在网络上全量重新分区（Shuffle Write → Shuffle Read），是 Spark 作业中最慢的环节。

## 四、Spark on EMR

在 AWS EMR 上跑 Spark 的核心配置：

```json
{
  "Classification": "spark-defaults",
  "Properties": {
    "spark.executor.memory": "8g",
    "spark.executor.cores": "4",
    "spark.executor.instances": "20",
    "spark.dynamicAllocation.enabled": "true",
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true"
  }
}
```

### 关键调优参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `spark.sql.adaptive.enabled` | true | AQE 自适应优化，Spark 3.0 后必开 |
| `spark.sql.adaptive.coalescePartitions.enabled` | true | 自动合并小分区 |
| `spark.sql.shuffle.partitions` | executor数×cores×2~3 | Shuffle 分区数，过大会产生大量小文件 |
| `spark.executor.memoryOverhead` | max(384MB, 0.1×executor.memory) | off-heap 内存，避免 YARN 杀容器 |

### Shuffle 倾斜处理

数据倾斜是 Spark 第一大性能杀手——某个 partition 的数据量远大于其他 partition，导致一个 Task 拖慢整个 Stage。

```python
# 方案1：加盐打散
df.withColumn("salted_key", concat(col("key"), lit("_"), (rand() * 10).cast("int"))) \
  .groupBy("salted_key").agg(...) \
  .groupBy(substring("salted_key", 1, -2)).agg(...)  # 去盐二次聚合

# 方案2：AQE 自动处理（Spark 3.2+ 默认开启）
# spark.sql.adaptive.skewJoin.enabled = true
```

## 五、S3 最佳实践

### 小文件问题

Spark 写 S3 最大的坑是小文件——每个 Task 输出一个文件，1000 个 Task = 1000 个文件。S3 的 LIST 操作按请求数计费，小文件会严重影响后续读取性能。

```python
# 写入前先 coalesce 或 repartition
df.coalesce(10).write.parquet("s3://bucket/output/")  # 只输出 10 个文件

# 或使用分区写入
df.write.partitionBy("dt").parquet("s3://bucket/output/")
```

**S3 写入机制**：Spark 写 S3 先写临时文件，rename 操作在 S3 上不是原子的（实际上是 copy + delete），大文件 rename 很慢。使用 **EMRFS** 或 **S3A Committer** 可以优化：

```
--conf spark.hadoop.fs.s3a.committer.name=magic
--conf spark.sql.sources.commitProtocolClass=org.apache.spark.internal.io.cloud.PathOutputCommitProtocol
```

### 文件格式选择

| 格式 | 特点 | 场景 |
|------|------|------|
| **Parquet** | 列存、压缩率高、谓词下推 | Spark/Hive 查询首选 |
| **ORC** | 列存、Hive 原生支持好 | Hive 为主的场景 |
| **Avro** | 行存、Schema 演化 | CDC 日志、Kafka 消息 |
| **Delta Lake / Iceberg / Hudi** | 表格式、ACID、时间旅行 | 数据湖上的表管理 |

## 六、EMR 作业提交

```bash
# 通过 Airflow 或 Step Functions 提交
aws emr add-steps \
  --cluster-id j-XXXX \
  --steps Type=Spark,Name="Daily ETL",\
    ActionOnFailure=CONTINUE,\
    Args=["--class","com.example.ETLJob",\
          "s3://bucket/jars/app.jar",\
          "--date","2026-07-05"]
```

**成本优化**：
- 非高峰期用 Spot 实例（省 60-90%）
- 核心节点（HDFS/元数据）用 On-Demand，Task 节点用 Spot
- 使用 EMR Auto Scaling 根据 YARN 队列自动扩缩
- 配合 Instance Fleets 混合多种实例类型提高 Spot 可用性

## 七、小结

Spark on EMR 的核心挑战：Shuffle 调优（分区数、倾斜处理）、S3 文件管理（小文件、提交协议）、以及成本控制（Spot + Auto Scaling）。DataFrame API + Catalyst 优化器在多数场景下已足够高效，除非你需要更底层的控制才退回 RDD。
