---
title: Snowflake vs BigQuery 对比
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Snowflake
  - BigQuery
categories:
  - 数据工程
---

Snowflake 和 BigQuery 是云原生数仓的两极——一个跨云中立，一个深度绑定 GCP。Redshift 用户常会想"要不要换"。

## 一、核心差异

| 维度 | Snowflake | BigQuery | Redshift |
|------|-----------|----------|----------|
| 架构 | 存算分离（共享存储） | 存算分离（完全无服务器） | 存算耦合（RA3 部分分离） |
| 计算 | Virtual Warehouse（需选规格） | Slots（自动弹性，按需付费） | 固定集群（需手动扩缩） |
| 扩展 | 手动扩缩 Warehouse | 自动弹性（无感） | 手动添加/移除节点 |
| 计费 | 计算 + 存储 分开 | 按扫描数据量 或 按 Slot | 按 EC2 实例 |
| 跨云 | AWS/Azure/GCP | 仅 GCP | AWS |
| SQL | 标准 SQL + 扩展 | 标准 SQL (ANSI兼容) | PostgreSQL 变体 |

## 二、Snowflake 架构

```
┌─────────────────────────────────────┐
│         Cloud Services Layer         │
│   (认证、查询优化、元数据管理)        │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│     Virtual Warehouse (计算层)       │
│  XS/S/M/L/XL (1-128 节点)           │
│  可独立扩缩、暂停/恢复               │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│      Storage Layer (存储层)          │
│  列式压缩、自动分区（微分区）         │
│  S3/GCS/Azure Blob 持久化            │
└─────────────────────────────────────┘
```

关键特性：
- **Zero-Copy Cloning**：瞬间克隆 TB 级数据库（只复制元数据，共享底层文件）
- **Time Travel**：最多回溯 90 天（企业版）
- **Data Sharing**：跨账户共享数据（无需复制数据）
- **微分区**：自动将数据切分为 50-500MB 的微分区，无需手动设计分区键

## 三、BigQuery 架构

```
┌─────────────────────────────────────┐
│         BigQuery Engine              │
│   (Dremel 查询引擎 + Colossus 存储)   │
│   完全 Serverless，无需管理资源        │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│         Colossus (Google 分布式 FS)   │
│  列式存储 + Capacitor 文件格式         │
│  自动分区 + 自动集群                  │
└─────────────────────────────────────┘
```

关键特性：
- **完全 Serverless**：无 Warehouse 概念，无需选择规格
- **Slot 自动弹性**：按需分配 Slot 计算资源（最大可到数千 Slot）
- **BI Engine**：内存加速层，亚秒级查询
- **BigLake**：直接在 GCS/S3/Azure 上查 Iceberg 表
- **ML 集成**：BigQuery ML（直接 SQL 建模型），Vertex AI 连接器

## 四、成本对比

### Snowflake
- 计算：$2-4/Snowflake Credit/小时（按 Warehouse 规格 × 运行时间）
- 存储：~$23/TB/月（压缩后）
- 典型月费（10TB）：计算 $3000-5000 + 存储 $230

### BigQuery
- 按扫描：$5/TB 扫描（适合临时查询）
- 按 Slot：$2000/100 Slots/月（适合固定工作负载）
- 存储：~$20/TB/月（活跃），$10/TB/月（长期）
- 典型月费（10TB）：固定分析 $2000-4000 + 存储 $200

### Redshift
- EC2 实例费：~$1000-5000/月（RA3 集群）
- 存储：Redshift Managed Storage 按 S3 计费 + 本地缓存
- 典型月费（10TB）：~$3000-8000

## 五、选型建议

| 场景 | 推荐 |
|------|------|
| 已在 AWS，需要跨云中立 | Snowflake |
| 已在 GCP，追求 Serverless | BigQuery |
| 已在 AWS，成本敏感 | Redshift (RA3 + Spectrum) |
| 高峰低谷明显的分析 | BigQuery (自动弹性) |
| 需要数据共享给合作伙伴 | Snowflake (Data Sharing) |
| 需要内置 ML 能力 | BigQuery (BigQuery ML) |

## 六、小结

三者都是优秀的云数仓。Snowflake 赢在跨云和生态成熟度，BigQuery 赢在完全 Serverless 和 GCP 深度集成，Redshift 赢在 AWS 生态内成本优化。换数仓的最大成本不是软件费，而是迁移——在现有基础设施上优化通常比迁移更划算。
