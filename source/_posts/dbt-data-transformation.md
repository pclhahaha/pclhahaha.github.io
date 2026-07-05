---
title: dbt 数据转换实践
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - dbt
categories:
  - 数据工程
---

dbt（data build tool）是数据工程领域最流行的转换工具。它的核心理念是"SQL + Jinja模板 + 版本控制"——用软件工程的方式管理数据转换代码。

## 一、为什么需要 dbt

传统 ETL/ELT 在 Airflow 中用 Python 写转换逻辑，问题：
- Python + SQL 混在一起，代码冗长
- 变更管理靠注释和文档（不可靠）
- 依赖关系隐蔽在 Job 调度中

dbt 的答案：只用 SQL 写转换逻辑，Jinja 模板处理参数化，自动管理表之间的依赖关系，内置数据质量测试和数据文档生成。

## 二、核心概念

```
dbt 项目结构:
  models/
    staging/       ← 原始数据标准化
    intermediate/  ← 中间转换
    marts/         ← 最终业务表
  tests/           ← 数据质量测试
  macros/          ← 可复用的 Jinja 宏
```

### Model（模型）

```sql
-- models/marts/orders_daily.sql
{{ config(materialized='table') }}

WITH raw_orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),
raw_users AS (
    SELECT * FROM {{ ref('stg_users') }}
)
SELECT
    DATE(o.created_at) AS dt,
    u.country,
    COUNT(*) AS order_cnt,
    SUM(o.amount) AS total_amount
FROM raw_orders o
JOIN raw_users u ON o.user_id = u.user_id
GROUP BY 1, 2
```

`{{ ref('stg_orders') }}` 自动管理表间依赖——dbt 知道 `orders_daily` 依赖 `stg_orders` 和 `stg_users`，生成正确的执行顺序。

### Materialization

| 类型 | 说明 |
|------|------|
| `table` | 每次 DROP + CREATE（刷新整表） |
| `incremental` | 只处理增量数据（`WHERE dt > max_date`），节省计算 |
| `view` | 数据库视图（不存储数据） |
| `ephemeral` | CTE 内联（不创建独立表） |

### Test

```yaml
# models/schema.yml
models:
  - name: orders_daily
    columns:
      - name: dt
        tests:
          - not_null
      - name: order_cnt
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
      - name: country
        tests:
          - relationships:
              to: ref('dim_countries')
              field: country_code
```

## 三、Airflow + dbt 集成

```python
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator

dbt_run = DbtCloudRunJobOperator(
    task_id="dbt_daily_run",
    job_id=12345,
    check_interval=60,
    timeout=3600
)
```

或直接在 Airflow 中调用 dbt CLI：

```python
dbt_task = BashOperator(
    task_id="dbt_run",
    bash_command="dbt run --select marts.+ --profiles-dir /opt/dbt"
)
```

## 四、dbt vs 纯 Python ETL

| 维度 | dbt | Python ETL (Spark/Pandas) |
|------|-----|--------------------------|
| 上手门槛 | 低（会 SQL 就行） | 中（需要编程能力） |
| 测试 | 内置 | 需自己写 |
| 文档 | 自动生成 | 需手动维护 |
| 执行引擎 | 依赖数仓（Redshift/BQ/Snowflake） | 自带计算引擎 |
| 适用 | ELT（T 在数仓中做） | ETL（T 在外部做） |

## 五、小结

dbt 的价值在于"让 SQL 转换变得可工程化"——自动依赖管理、内置测试、文档生成、版本控制。如果你已经在用 Redshift/Snowflake/BigQuery 做数仓，dbt 是最自然的 T 层工具。
