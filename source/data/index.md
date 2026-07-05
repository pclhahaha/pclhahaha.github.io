---
title: 数据工程知识图谱
date: 2026-07-05 12:00:00
type: page
comments: false
---

<style>
.knowledge-graph { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; max-width: 960px; margin: 0 auto; }
.kg-header { text-align: center; margin-bottom: 32px; }
.kg-header h1 { font-size: 28px; margin-bottom: 8px; }
.kg-header p { color: #666; font-size: 14px; }
.kg-stats { display: flex; justify-content: center; gap: 32px; margin: 20px 0; }
.kg-stat { text-align: center; }
.kg-stat-num { font-size: 32px; font-weight: 700; color: #333; }
.kg-stat-label { font-size: 12px; color: #999; margin-top: 2px; }
.kg-section { margin-bottom: 36px; }
.kg-section-title { font-size: 20px; font-weight: 700; padding-bottom: 8px; border-bottom: 2px solid #eee; margin-bottom: 16px; }
.kg-category { margin-bottom: 20px; }
.kg-cat-title { font-size: 15px; font-weight: 600; color: #555; margin-bottom: 8px; }
.kg-links { display: flex; flex-wrap: wrap; gap: 6px 12px; }
.kg-link { font-size: 13px; color: #333; text-decoration: none; padding: 3px 10px; background: #f7f7f7; border-radius: 4px; transition: all 0.15s; white-space: nowrap; }
.kg-link:hover { background: #e3e3e3; color: #000; }
@media (max-width: 600px) { .kg-stats { gap: 16px; } .kg-stat-num { font-size: 24px; } }
</style>

<div class="knowledge-graph">

<div class="kg-header">
  <h1>数据工程知识图谱</h1>
  <p>数据湖 · 数仓 · ETL · 调度 · 流处理</p>
  <div class="kg-stats">
    <div class="kg-stat"><div class="kg-stat-num">5</div><div class="kg-stat-label">篇文章</div></div>
    <div class="kg-stat"><div class="kg-stat-num">4</div><div class="kg-stat-label">分类</div></div>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">⚙️ 计算引擎</div>
  <div class="kg-category">
    <div class="kg-cat-title">批处理</div>
    <div class="kg-links">
      <a class="kg-link" href="/spark-emr-guide/">Spark on EMR 深度解析</a>
      <a class="kg-link" href="#" style="color:#999">Spark SQL 调优（待补充）</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">交互式查询</div>
    <div class="kg-links">
      <a class="kg-link" href="/hive-trino-guide/">Hive 与 Trino 实战</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">流处理</div>
    <div class="kg-links">
      <a class="kg-link" href="#" style="color:#999">Flink 实时计算（待补充）</a>
      <a class="kg-link" href="#" style="color:#999">Kafka Streams（待补充）</a>
    </div>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">🗄️ 存储层</div>
  <div class="kg-category">
    <div class="kg-cat-title">数据湖</div>
    <div class="kg-links">
      <a class="kg-link" href="/s3-data-lake-guide/">S3 数据湖最佳实践</a>
      <a class="kg-link" href="#" style="color:#999">Iceberg / Delta Lake / Hudi（待补充）</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">数据仓库</div>
    <div class="kg-links">
      <a class="kg-link" href="/redshift-warehouse-guide/">Redshift 数仓实战</a>
      <a class="kg-link" href="#" style="color:#999">Snowflake / BigQuery（待补充）</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">数据建模</div>
    <div class="kg-links">
      <a class="kg-link" href="#" style="color:#999">星型/雪花/Vault（待补充）</a>
    </div>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">⏱️ 调度与编排</div>
  <div class="kg-links">
    <a class="kg-link" href="/airflow-production-guide/">Airflow 生产级实践</a>
    <a class="kg-link" href="#" style="color:#999">DAG 设计模式（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">dbt 数据转换（待补充）</a>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">📊 数据治理</div>
  <div class="kg-links">
    <a class="kg-link" href="#" style="color:#999">数据质量（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">数据血缘（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">成本优化（待补充）</a>
  </div>
</div>

<div style="text-align:center; margin-top:48px;">
  <a href="/" style="color:#555; text-decoration:none; font-size:14px; border:1px solid #ddd; padding:8px 24px; border-radius:4px;">← 返回后端知识图谱</a>
</div>

</div>
