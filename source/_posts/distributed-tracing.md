---
title: 分布式链路追踪
date: 2026-07-05
updated: 2026-07-05
tags:
  - 链路追踪
  - Jaeger
  - SkyWalking
  - OpenTelemetry
categories:
  - 可观测性
---

### 1.1 核心概念

```
Trace Id: a1b2c3d4e5f67890
├─ Span A: API Gateway           duration: 350ms
│  ├─ Span B: OrderService.createOrder   duration: 200ms
│  │  ├─ Span D: MySQL SELECT    duration: 80ms
│  │  └─ Span E: Redis GET       duration: 10ms
│  └─ Span C: InventoryService.checkStock  duration: 120ms
```

- **Trace**：一次完整请求链，由一组 Span 组成
- **Span**：单个逻辑操作，含起止时间、Tag、事件
- **SpanContext**：跨进程传递上下文（TraceId + SpanId + Baggage）
- **上下文传播**：通过 HTTP Header `traceparent` 或 gRPC Metadata 传递

### 1.2 OpenTelemetry 标准

OTel 统一了 OpenTracing + OpenCensus，成为可观测性**采集标准**：

```
Application Code
    │
    ▼
┌─────────────────┐
│  OTel API       │  手动埋点
│  OTel SDK       │  采样/导出/处理
└────────┬────────┘
         │
    ┌────┴────┬──────────┐
    ▼         ▼          ▼
  OTLP    Jaeger    Zipkin   ← 可导出到任何后端
    │       Thrift     HTTP
    ▼         ▼          ▼
┌─────────────────────────────┐
│    Collector (可选)          │ 接收/处理/导出流水线
└─────────────────────────────┘
         │
    ┌────┴──────────┐
    ▼               ▼
 Jaeger(Traces)  Prometheus(Metrics)
```

**核心价值**：厂商无关；统一 Traces + Metrics + Logs API；Java Agent 自动埋点 Spring/gRPC/Kafka/Redis。

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.traces.exporter=otlp \
     -jar order-service.jar
```

### 1.3 Jaeger 架构

```
Service A/B/C + Jaeger Client
        │ Thrift/UDP
        ▼
┌─────────────────┐
│  Jaeger Agent    │  DaemonSet，每 Node 一个，批处理转发
└────────┬────────┘
         │ gRPC
         ▼
┌─────────────────┐
│ Jaeger Collector │  验证/索引 → 写 Kafka → 写 ES/Cassandra
└─────────────────┘
         │
    ┌────┴──────┐
    ▼           ▼
 Ingester    Query Service + UI
(Kafka→DB)
```

| 存储 | 适用场景 | 说明 |
|------|---------|------|
| **Elasticsearch** | 生产首选 | 全文检索，与日志共用 ES 集群 |
| Cassandra | 超大流量 | 线性扩展，运维复杂 |
| ClickHouse | 新选择 | 列存高压缩，OLAP 分析快 |

### 1.4 SkyWalking

国人开源 APM，Java Agent 自动埋点极强，几乎零代码侵入。

```
Service + SW Agent ──gRPC──▶ OAP(Receiver→Analyzer→Aggregator) ──▶ ES/MySQL ──▶ UI(RocketBot)
```

**核心特性**：服务拓扑图（实时调用量/延迟/成功率）、JVM 级指标、慢端点检测、内置告警。

```bash
java -javaagent:skywalking-agent.jar \
     -DSW_AGENT_NAME=order-service \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800 \
     -jar order-service.jar
```

自定义 Span：
```java
@Trace(operationName = "customBusinessLogic")
public void processOrder(Order order) {
    ActiveSpan.tag("orderId", order.getId());
}
```

### 1.5 Spring Cloud Sleuth + Zipkin

```yaml
spring:
  application:
    name: order-service
  sleuth:
    sampler:
      probability: 0.1    # 10% 采样（生产）
    baggage:
      remote-fields:       # 跨服务传播自定义字段
        - userId
  zipkin:
    base-url: http://zipkin:9411
    sender:
      type: kafka          # 通过 Kafka 解耦
```

Sleuth 自动将 `traceId` 和 `spanId` 注入 MDC，日志输出：
```
[order-service,a1b2c3d4e5f67890,a1b2c3d4e5f67890] INFO - order created
[inventory-service,a1b2c3d4e5f67890,1234567890abcdef] INFO - stock deducted
```

### 1.6 采样策略

全量追踪成本极高。生产推荐混合策略：
```
入口层：10-30% 随机采样
    ↓
规则层：错误 Trace → 100% 保留；延迟 > 1s → 100% 保留
    ↓
收集层：异常服务提高采样率，正常流量降低
```

| 策略 | 说明 | 适用 |
|------|------|------|
| 固定采样 | 固定比例（10%） | 成本可预测 |
| 概率采样 | 根据延迟/错误动态调整 | 异常优先保留 |
| 头部采样 | Trace 入口决定 | 实现简单 |
| 尾部采样 | Trace 完成后根据结果决定 | 一定保留异常 |

Jaeger 配置：`sampler.type: probabilistic; sampler.param: 0.1`，关键操作 `perOperationStrategies` 设 100%。
