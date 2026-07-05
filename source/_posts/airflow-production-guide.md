---
title: Airflow 生产级实践
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Airflow
  - DAG
  - 调度
  - ETL
categories:
  - 数据工程
---

Airflow 是数据工程领域最主流的调度引擎。它用 Python 代码定义 DAG（有向无环图），管理任务间的依赖关系和执行顺序。

## 一、核心概念

```
┌──────────────── DAG: daily_etl ────────────────┐
│                                                  │
│  [extract_data] ──→ [transform] ──→ [load_dw]   │
│       │                                     │    │
│       └──→ [backup_source]                  │    │
│                                                  │
│  schedule: @daily                                 │
│  start_date: 2026-01-01                          │
└──────────────────────────────────────────────────┘
```

| 概念 | 说明 |
|------|------|
| **DAG** | 工作流定义，Python 文件 |
| **Task** | DAG 中的一个执行步骤 |
| **Operator** | 任务类型（Python/Bash/Spark/Email） |
| **Sensor** | 等待外部条件满足的 Task（文件到达、分区就绪） |
| **Executor** | 任务执行引擎（Local/Celery/K8s） |
| **XCom** | Task 间传递小量数据的机制 |

## 二、DAG 设计原则

### 2.1 幂等性——最重要的设计原则

同一个 DAG Run 被重跑多次，结果必须一致。手段就是**分区覆盖**——每次运行只覆盖目标分区的数据：

```python
@task
def transform_and_write(execution_date):
    df = spark.read.parquet(f"s3://bucket/raw/dt={execution_date}/")
    result = df.transform(...)
    result.write.mode("overwrite") \
          .parquet(f"s3://bucket/gold/dt={execution_date}/")
```

### 2.2 回填（Backfill）

Airflow 的核心能力之一是回溯执行历史区间。回填的注意事项：

- 每次 run 的执行窗口要明确（`execution_date` → `next_execution_date`）
- 全量回填前先用一两天数据验证逻辑正确
- 注意 API 速率限制——上游数据源对历史数据的拉取可能有限流

### 2.3 DAG 超时与 SLA

```python
dag = DAG(
    "daily_etl",
    dagrun_timeout=timedelta(hours=2),   # 整体超时 2h
    sla_miss_callback=alert_if_missed,   # SLA 未达触发告警
)

task = PythonOperator(
    task_id="process_data",
    execution_timeout=timedelta(hours=1),  # 单个 Task 超时 1h
    sla=timedelta(hours=3)                 # 预期 3h 内完成
)
```

## 三、Spark on EMR Operator

```python
from airflow.providers.amazon.aws.operators.emr import EmrAddStepsOperator

spark_step = EmrAddStepsOperator(
    task_id="spark_transform",
    job_flow_id="{{ var.value.emr_cluster_id }}",
    steps=[{
        "Name": "Spark ETL",
        "ActionOnFailure": "CANCEL_AND_WAIT",
        "HadoopJarStep": {
            "Jar": "command-runner.jar",
            "Args": [
                "spark-submit",
                "--class", "com.example.ETLJob",
                "--conf", "spark.dynamicAllocation.enabled=true",
                "s3://bucket/jars/app.jar",
                "--date", "{{ ds }}"  # Airflow 内置变量，执行日期
            ]
        }
    }]
)
```

## 四、Sensors——等待外部依赖

Sensors 解决"上游数据还没就绪，下游不能跑"的问题：

```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_for_data = S3KeySensor(
    task_id="wait_for_raw_data",
    bucket_key="raw/events/dt={{ ds }}/*.parquet",
    wildcard_match=True,          # 支持通配符
    timeout=60 * 60,             # 最多等 1 小时
    poke_interval=300,           # 每 5 分钟检查一次
    mode="reschedule"            # 检查间隙释放 worker slot
)
```

**注意**：用 `mode="reschedule"` 而非默认 `mode="poke"`。——poke 模式在等待期间占用一个 worker slot，大量 sensor 会耗尽 worker 池导致其他 task 无法执行。reschedule 模式在两次检查之间释放 slot，避免了这个问题。

## 五、生产级配置要点

| 配置 | 建议值 | 说明 |
|------|--------|------|
| `parallelism` | worker × 32 | 全局最大并发 task 数 |
| `dag_concurrency` | 16 | 单个 DAG 的最大并发 task 数 |
| `max_active_runs_per_dag` | 1 | 防止同一 DAG 多 Run 并发写入冲突 |
| `catchup` | False（多数场景） | 避免部署后回填整个历史区间 |
| `schedule_interval` | `@daily` 或 `0 2 * * *` | 凌晨低峰期执行，错峰降低集群资源竞争 |

## 六、监控与告警

```python
def on_failure_callback(context):
    """Task 失败时发送通知"""
    task = context["task_instance"]
    send_slack_alert(f"Task {task.task_id} in {task.dag_id} failed")

task = PythonOperator(
    task_id="transform",
    on_failure_callback=on_failure_callback,
    retries=3,                    # 最多重试 3 次
    retry_delay=timedelta(minutes=5),
    retry_exponential_backoff=True  # 指数退避: 5min→10min→20min
)
```

## 七、小结

Airflow 的核心价值在于"用代码定义工作流 + 自动回溯执行"。生产级使用的关键：分区覆盖保证幂等性、Sensor 用 reschedule 模式、Spark 任务通过 EMR Operator 调起而非 Airflow 本机执行、DAG 超时和 SLA 双重保障。
