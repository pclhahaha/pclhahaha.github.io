---
title: 数据质量与血缘
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - 数据质量
  - 数据血缘
categories:
  - 数据工程
---

数据工程有一条铁律：**Garbage In, Garbage Out**。数据管道再快、数仓再大，如果数据本身是错的，产出的报表和决策都是错的。

## 一、数据质量六维度

| 维度 | 检查内容 | 示例 |
|------|----------|------|
| **完整性** | 必填字段是否为空 | `user_id IS NULL` 的行数 |
| **准确性** | 数据是否符合业务规则 | 订单金额 > 0 |
| **一致性** | 跨表/跨系统的数据是否一致 | ODS 和 DW 同一订单的金额一致 |
| **时效性** | 数据是否按时到达 | 每小时分区是否在整点后 15 分钟内就绪 |
| **唯一性** | 主键是否重复 | `COUNT(*) != COUNT(DISTINCT pk)` |
| **有效性** | 数据是否符合格式 | email 是否包含 `@` |

## 二、实现方案

### Great Expectations（开源首选）

```python
import great_expectations as gx

# 定义期望
validator.expect_column_values_to_not_be_null("user_id")
validator.expect_column_values_to_be_between("amount", min_value=0, max_value=100000)
validator.expect_column_values_to_match_regex("email", r"^.+@.+\..+$")
validator.expect_column_unique_value_count_to_be_between("id", min_value=10000)

# 生成报告
validator.save_expectation_suite()
checkpoint = gx.checkpoint.SimpleCheckpoint("daily_check")
checkpoint.run()
```

### dbt tests

```yaml
# models/schema.yml
models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: amount
        tests:
          - accepted_values:
              values: ['negative', 'positive']
```

### 自定义 SQL 检查

```sql
-- 检查：今天的订单数不应该比昨天少 50% 以上
WITH daily AS (
  SELECT dt, COUNT(*) as cnt FROM orders WHERE dt IN (CURRENT_DATE, CURRENT_DATE-1) GROUP BY dt
)
SELECT * FROM daily WHERE cnt < 0.5 * LAG(cnt) OVER (ORDER BY dt)
```

## 三、数据血缘

数据血缘回答的核心问题：**这个报表的数字从哪来的？如果源表变了会影响谁？**

### 字段级血缘

```
源表 A.user_id → ETL Job 1 → ODS 表 B.user_id → ETL Job 2 → DW orders.user_id → BI Report
```

### 实现方式

| 工具 | 特点 |
|------|------|
| **dbt** | SQL 解析自动生成血缘（内置 `dbt docs`） |
| **OpenLineage** | 开源标准协议，Spark/Airflow/dbt 通用 |
| **Atlas** | 企业级数据治理平台 |
| **自建** | 从 Spark 执行计划 + Airflow DAG 提取 |

### dbt 血缘示例

```bash
dbt docs generate
dbt docs serve  # 可视化 DAG，点击任意节点看上下游依赖
```

## 四、告警策略

| 级别 | 触发条件 | 动作 |
|------|----------|------|
| P0 | 核心表数据丢失、延迟 > 2h | PagerDuty 电话 |
| P1 | 数据量异常（波动 > 30%） | Slack 告警 |
| P2 | 个别字段空值率上升 | 自动记录、周报汇总 |

## 五、小结

数据质量的底线是**不产生错误决策**。核心实践：关键表必做 not_null + unique 检查、量级异常的自动告警、血缘追踪到字段级。不要试图一开始就做到 100% 覆盖——从核心表开始，逐步扩展。
