---
title: 数据工程成本优化
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - 成本优化
  - AWS
categories:
  - 数据工程
---

数据工程的月度账单很容易失控——一个忘记关的 EMR 集群周末跑了两天烧掉 $5000，一个没有生命周期策略的 S3 bucket 存了 5 年的冷数据但没人访问。控制成本的起点是**可视化每一项支出的来源**。

## 一、成本构成

| 项目 | 占比（典型） | 优化空间 |
|------|-------------|----------|
| EMR EC2 实例 | 50-60% | 最大 |
| S3 存储 | 15-20% | 中等 |
| Redshift 集群 | 10-15% | 中等 |
| Glue/Athena 查询 | 5-10% | 中等 |
| 数据传输（跨AZ/跨区域） | 5-10% | 大 |

## 二、EMR 优化

### Spot + On-Demand 混合

```json
{
  "InstanceFleets": [
    {
      "InstanceFleetType": "CORE",
      "TargetOnDemandCapacity": 2,
      "TargetSpotCapacity": 6,
      "InstanceTypeConfigs": [
        {"InstanceType": "r6g.2xlarge"},
        {"InstanceType": "r5.2xlarge"},
        {"InstanceType": "r7g.2xlarge"}
      ]
    }
  ]
}
```

- 使用 Instance Fleets 混合多种实例类型提高 Spot 获取率
- Core 节点（运行 HDFS/Namenode）用 On-Demand，Task 节点用 Spot
- 单可用区部署，避免跨 AZ 数据传输费

### Auto Scaling 策略

```yaml
# 低峰期（凌晨）自动缩容到最小
ScaleDownRule:
  Trigger: YARNMemoryAvailablePercentage > 80 for 300s
  Action: 减少 2 个 Task 节点
  Cooldown: 300s

# 高峰期缩容条件更严格
ScaleUpRule:
  Trigger: YARNMemoryAvailablePercentage < 20 for 120s
  Action: 增加 2 个 Task 节点
```

### 作业调度优化

```python
# Airflow DAG - 错峰执行
dag = DAG(
    "daily_etl",
    schedule_interval="0 3 * * *",  # 凌晨 3 点，集群低峰
    ...
)
```

## 三、S3 优化

| 优化 | 节省 | 实现 |
|------|------|------|
| S3 Intelligent-Tiering | 自动分层，省 20-40% | 开启即可，自动监控访问模式 |
| 生命周期策略 | 归档冷数据，省 50-80% | 30天后→IA，90天后→Glacier |
| 删除未完成的分段上传 | 清理碎文件 | 生命周期规则中设置 AbortIncompleteMultipartUpload |
| 合并小文件 | 减少 LIST/PUT 请求费 | Spark coalesce + 定时合并任务 |
| 压缩格式 | 省存储 40-60% | Parquet + ZSTD |

## 四、Redshift 优化

- **暂停集群**：开发/测试集群在非工作时间暂停
- **RA3 实例**：计算和存储分离，存储按 S3 计费（更便宜）
- **Concurrency Scaling**：只在高峰期启用，按使用量付费
- **Spectrum**：冷数据放 S3 直接用 Spectrum 查，不入 Redshift 本地

## 五、监控与告警

```python
# AWS Budget Alert
budget = Budget(
    amount=5000,  # 月度预算 $5000
    threshold=0.8,  # 80% 时告警
    notification=["slack", "email"]
)
```

关键监控指标：
- 每日按服务拆分的费用（AWS Cost Explorer）
- Spot 中断率（过高说明价格设置太低）
- EMR 集群空闲时间（长时间空闲自动终止）

## 六、成本优化清单

| 优化 | 预估节省 | 实施难度 |
|------|----------|----------|
| Task 节点全用 Spot | 60-70% | 低 |
| S3 生命周期策略 | 30-50% | 低 |
| 合并小文件 | 10-20% | 中 |
| 错峰调度 | 10-20% | 低 |
| 压缩格式 (ZSTD) | 20-40% | 低 |
| Redshift 暂停非生产集群 | 30-50% | 低 |
| 自动终止空闲 EMR | 5-10% | 中 |

## 七、小结

数据工程成本优化的核心是"用 Spot 跑计算、用生命周期管存储、用监控抓异常"。前三项做到，一般能省 40-60% 的月账单。
