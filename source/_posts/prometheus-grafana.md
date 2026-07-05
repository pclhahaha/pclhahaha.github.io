---
title: Prometheus + Grafana 监控
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 监控
  - Prometheus
  - Grafana
  - PromQL
categories:
  - 可观测性
---

Prometheus 是云原生生态中最主流的监控系统，Grafana 则是最流行的可视化面板。两者结合构成了现代后端服务的标准监控方案。

## 一、Prometheus 架构

Prometheus 采用 Pull 模型——由服务端主动拉取目标的 metrics 端点，而非等客户端推送。

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   Node   │  │   JMX    │  │ 自定义   │     ┌──────────┐
│ Exporter │  │ Exporter │  │ Exporter │     │Pushgateway│
└────┬─────┘  └────┬─────┘  └────┬─────┘     └────┬─────┘
     │             │             │                │ (短任务推送)
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
          │ │    TSDB (时序数据库)  │    │    └───────────┘
          │ └──────────────────────┘    │
          └────────────────────────────┘
```

**Pull 模型的优势**：服务端控制采集频率，拉取失败本身就是"目标不可达"的健康信号。在 Kubernetes 下，通过 Pod Annotation 动态发现：

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

## 二、Metrics 类型

Prometheus 支持四种核心 Metrics 类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| **Counter** | 只增不减的计数器 | HTTP 请求总数、错误数 |
| **Gauge** | 可增可减的瞬时值 | CPU 使用率、内存使用量、连接数 |
| **Histogram** | 数据分布（可计算分位数） | 请求延迟分布（P50/P99） |
| **Summary** | 客户端计算的分位数 | 请求延迟（P50/P90/P99） |

**Histogram vs Summary**：Histogram 在服务端聚合时更灵活（可以跨实例计算总体的百分位），Summary 在客户端计算分位数不可聚合。通常推荐 Histogram。

```go
// Go 示例：注册 Metrics
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "http_requests_total"},
        []string{"method", "path", "status"},
    )
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )
)
```

## 三、PromQL 查询语言

PromQL 是 Prometheus 的函数式查询语言。

```promql
# 当前 QPS（每秒请求增长率）
rate(http_requests_total[5m])

# CPU 使用率百分比
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 过去 5 分钟的 P99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 错误率
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# JVM 堆内存使用率
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100
```

## 四、常用 Exporter

| Exporter | 采集对象 |
|----------|----------|
| Node Exporter | Linux 主机 CPU/内存/磁盘/网络 |
| JMX Exporter | JVM 指标（GC、线程、堆内存） |
| Blackbox Exporter | HTTP/TCP/ICMP 探活探测 |
| PostgreSQL Exporter | 连接数、查询延迟、缓存命中率 |
| Redis Exporter | 内存使用、命中率、键数量 |
| Kafka Exporter | Consumer Lag、Partition 偏移量 |

## 五、Alertmanager 告警

Prometheus 根据 PromQL 规则触发告警，推送到 Alertmanager。Alertmanager 负责告警的分组、抑制、静默和路由：

```yaml
# prometheus 告警规则
groups:
  - name: instance_down
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "实例 {{ $labels.instance }} 宕机"
          description: "已持续 {{ $value }} 秒"

# Alertmanager 路由
route:
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match:
        severity: warning
      receiver: 'slack'
```

## 六、Grafana 可视化

Grafana 从 Prometheus 数据源拉取数据，用仪表盘呈现：

- 折线图：QPS、延迟趋势
- 热力图：延迟分布直方图
- 统计面板：当前在线用户数
- 状态面板：服务健康状态

Grafana 支持变量（Variables），允许用户动态切换数据源、筛选标签——一个 DashBoard 可以适配多个环境和集群。

## 七、小结

Prometheus + Grafana 的组合已经成为云原生监控的事实标准。Pull 模型简化了服务端架构，PromQL 提供了强大的时序查询能力，四种 Metrics 类型覆盖了绝大多数的监控场景。对于追求更高可靠性的系统，可以考虑引入 Thanos 或 Cortex 实现 Prometheus 的跨集群聚合和长期存储。
