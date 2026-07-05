---
title: Airflow DAG 设计模式
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Airflow
  - DAG
categories:
  - 数据工程
---

Airflow 的 DAG 不是"写出来能跑就行"——好的 DAG 设计让管道可维护、可回溯、可观测。这篇文章聚焦 DAG 的设计模式和常见反模式。

## 一、设计模式

### 1.1 按数据域拆分 DAG

```python
# ❌ 一个巨大的 DAG 做所有事
with DAG("monolithic_etl"):
    extract_users() >> extract_orders() >> transform_all() >> load_all()

# ✅ 按数据域拆分
dag_users = DAG("etl_users", schedule="@daily")
dag_orders = DAG("etl_orders", schedule="@daily")
dag_aggregates = DAG("etl_aggregates", schedule="@daily")
```

拆分的好处：一个 DAG 卡住不影响其他数据域的产出；独立回填某个域；不同域可设置不同 schedule。

### 1.2 Task Groups——逻辑分组

```python
from airflow.utils.task_group import TaskGroup

with TaskGroup("extract", dag=dag) as extract_group:
    extract_users = PythonOperator(task_id="extract_users", ...)
    extract_orders = PythonOperator(task_id="extract_orders", ...)

with TaskGroup("transform", dag=dag) as transform_group:
    transform_users = PythonOperator(task_id="transform_users", ...)
    transform_orders = PythonOperator(task_id="transform_orders", ...)

extract_group >> transform_group
```

### 1.3 动态 DAG 生成

```python
# 为每个数据源自动生成结构相同的 DAG
SOURCES = ["users", "orders", "products", "payments"]

for source in SOURCES:
    dag_id = f"ingest_{source}"
    globals()[dag_id] = create_ingest_dag(dag_id, source, schedule="@hourly")
```

适用于：多个数据源有完全相同的 ETL 模式。

### 1.4 跨 DAG 依赖——Dataset/Sensor

```python
# DAG A：产出数据集
with DAG("produce_data", schedule=None) as dag_a:
    @task(outlets=[Dataset("s3://bucket/output/dt={{ ds }}/")])
    def produce():
        ...

# DAG B：等待 DAG A 产出
with DAG("consume_data", schedule=[Dataset("s3://bucket/output/dt={{ ds }}/")]) as dag_b:
    ...
```

## 二、反模式

### 2.1 Top-Level Code

```python
# ❌ DAG 文件顶层有重操作
conn = create_db_connection()          # 每次解析 DAG 都执行！
users = load_config_from_api()         # API 调用也每次执行！

with DAG("example") as dag:
    ...

# ✅ 延迟到 Task 执行
with DAG("example") as dag:
    @task
    def load_config():
        users = load_config_from_api()  # 只在 Task 执行时调用
        return users
```

Airflow 每 30 秒解析一次所有 DAG 文件。顶层代码每次解析都执行——意味着 API 调用、数据库连接、文件读取都会被高频触发。

### 2.2 Star Schema DAG

```
                    [big_task]
                   /    |    \
           [t1] [t2] ...[t20] [t21] ... [t50]
```

50 个子任务依赖一个父任务。问题：`big_task` 的任何小改动需要验证 50 个下游的稳定性；难以重跑部分子任务。

解法：按功能分组，用 TriggerDagRunOperator 拆成子 DAG。

### 2.3 Hard-Coded Paths

```python
# ❌
output_path = "/data/2026/07/05/output/"

# ✅
output_path = f"/data/dt={{ ds }}/output/"
```

## 三、小结

好的 DAG 设计 = 独立数据域 + Task Group 逻辑分组 + 动态生成减少重复 + 跨 DAG 用 Dataset 触发。Top-Level Code、Star Schema 和硬编码路径是最常见的三个坑。
