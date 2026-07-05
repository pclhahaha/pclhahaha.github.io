---
title: Prometheus + Grafana 监控
date: 2026-07-05
updated: 2026-07-05
tags:
  - 监控
  - Prometheus
  - Grafana
  - PromQL
categories:
  - 可观测性
---

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   Node   │  │   JMX    │  │ 自定义   │     ┌──────────┐
│ Exporter │  │ Exporter │  │ Exporter │     │Pushgateway│
└────┬─────┘  └────┬─────┘  └────┬─────┘     └────┬─────┘
     │             │             │                │ (短任务)
     └─────────────┼─────────────┼────────────────┘
                   │  /metrics   │
                   ▼             ▼
          ┌────────────────────────────┐
          │    Prometheus Server        │
          │ ┌──────────────────────┐    │    ┌───────────┐
          │ │ Service Discovery    │    │───▶│Alertmanager│
          │ │ (K8s/Consul/DNS)    │    │    └───────────┘
          │ └──────────────────────┘    │    ┌───────────┐
          │ ┌──────────────────────┐    │───▶│  Grafana   │
          │ │    TSDB (本地+Remote)│    │    └───────────┘
          │ └──────────────────────┘    │
          └────────────────────────────┘
```

**Pull 模型优势**：服务端控制采集频率，拉取失败本身即"目标不可达"的健康信号。K8S 下通过 Pod Annotation 动态发现：
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

#### 常用 Exporter

| Exporter | 用途 |
|----------|------|
| Node Exporter | CPU/内存/磁盘/网络 |
| JMX Exporter | JVM 指标（Agent 或独立进程） |
| Blackbox Exporter | HTTP/TCP/ICMP 探活 |
| PostgreSQL/Redis/Kafka Exporter | 中间件指标 |
