---
title: "可观测性: 监控/日志/链路追踪"
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 可观测性
  - 监控
  - 日志
  - 链路追踪
  - Prometheus
categories:
  - 基础设施
---
## 一、可观测性概述

### 1.1 概念与定位

可观测性（Observability）源自控制论：**系统内部状态能否由外部输出唯一确定**。在分布式系统中，它回答的是"为什么出问题"，而传统监控只回答"出问题了吗"。

| 维度 | 监控 (Monitoring) | 可观测性 (Observability) |
|------|-------------------|--------------------------|
| 核心问题 | 系统出问题了吗？ | 为什么会出问题？ |
| 数据特征 | 预定义指标，低基数 | 高基数、高维度遥测数据 |
| 探索能力 | 已知未知 (known unknowns) | 未知未知 (unknown unknowns) |
| 触发方式 | 被动接收已知故障信号 | 主动探索任意维度系统行为 |

### 1.2 三大支柱

```
                   可观测性
          ┌──────────┼──────────┐
          ▼          ▼          ▼
     监控(Metrics) 日志(Logging) 追踪(Tracing)
     ┌──────┐    ┌──────┐    ┌──────┐
     │指标聚合│    │事件记录│    │请求链路│
     │趋势预测│    │上下文  │    │延迟分析│
     │告警规则│    │自由查询│    │依赖拓扑│
     └──────┘    └──────┘    └──────┘
```

### 1.3 三支柱对比

| 维度 | Metrics | Logging | Tracing |
|------|---------|---------|---------|
| 数据体积 | 最小（聚合后） | 最大 | 中等 |
| 存储成本 | 低（TSDB 压缩） | 高（全文索引） | 中 |
| 信息密度 | 低 | 极高 | 极高（调用关系） |
| 典型保留期 | 1-3 年 | 7-30 天（热） | 7-15 天 |
| 典型工具 | Prometheus + Grafana | ELK / Loki | Jaeger / SkyWalking |
| 典型查询 | "P99 延迟是否超阈值？" | "为什么这个请求返回 500？" | "慢在哪一个 Span？" |

三者互补——成熟的平台通过 **TraceId 和 Labels** 打通：Metrics 发现异常 → Tracing 找到异常 Trace → 日志定位根因。

### 1.4 为什么需要可观测性

- **请求链路复杂**：一次 API 调用穿越 API Gateway → BFF → 订单 → 库存 → 支付 → MQ → 通知，任意环节故障客户端都只看到"慢了/失败了"
- **故障模式多样**：网络分区、GC 停顿、线程池耗尽、连接池泄漏、慢 SQL、缓存雪崩
- **部署频率加速**：CI/CD 从月级提升到日级/时级，每次变更都可能引入新故障
- **跨团队协作**：单次请求横跨多个团队，统一的可观测平台是协作基础

## 二、监控 (Metrics)

### 2.1 四种指标类型

#### Counter（计数器）
只增不减的累积值。用 `rate()` 求变化率：
```
http_requests_total{method="GET", status="200"} 3382150
```

#### Gauge（仪表盘）
可增可减的瞬时快照：
```
jvm_memory_used_bytes{area="heap"} 2.56e+08
db_connection_pool_active 12
```

#### Histogram（直方图）
将观测值分配到预定义桶，记录总数与总和：
```
http_request_duration_seconds_bucket{le="0.005"}  240
http_request_duration_seconds_bucket{le="0.01"}   334
http_request_duration_seconds_bucket{le="+Inf"}  10000
http_request_duration_seconds_sum  234.56
http_request_duration_seconds_count 10000
```
通过 `histogram_quantile()` 计算任意分位值。桶设置关键——太少精度差，太多成本高，建议对齐 SLO 阈值。

#### Summary（摘要）
**客户端**直接计算 φ-分位值，不可跨服务聚合：

| 特性 | Histogram | Summary |
|------|-----------|---------|
| 分位数计算 | 服务端 | 客户端 |
| 跨服务聚合 | 支持 | 不支持 |
| 任意分位数 | 支持 | 仅预定义 |
| 推荐场景 | 大部分场景 | 需精确分位的单点监控 |

**优先使用 Histogram**。

### 2.2 Prometheus 架构

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

### 2.3 PromQL 核心查询

```promql
# QPS（5 分钟窗口）
rate(http_requests_total[5m])

# P99 延迟
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# 按服务拆分的 P95
histogram_quantile(0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)

# 错误率
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# 预测磁盘满时间（4 小时后）
predict_linear(node_filesystem_free_bytes[1h], 4 * 3600) < 0

# Top 5 内存占用
topk(5, container_memory_working_set_bytes)
```

**rate vs irate**：`rate` 取窗口内所有点做线性回归，平滑；`irate` 只取最后两点，灵敏。告警用 irate，看板用 rate。

### 2.4 监控方法论

#### 四大黄金信号（Google SRE）

| 信号 | 指标 | 告警示例 |
|------|------|---------|
| **延迟 (Latency)** | P99/P999 延迟 | > 500ms 持续 5min，看分位数不要看均值 |
| **流量 (Traffic)** | QPS | 环比变化 > 50% |
| **错误 (Errors)** | 5xx 比例 | > 1% 持续 3min，区分 4xx（客户端）和 5xx（服务端） |
| **饱和度 (Saturation)** | CPU/内存/连接池 | CPU > 80%，关注排队指标而非仅利用率 |

#### USE 方法（资源视角）
对 CPU/内存/磁盘/网络，逐个检查：**Utilization（利用率）→ Saturation（饱和度）→ Errors（错误数）**。要求 100% 覆盖所有物理资源。

#### RED 方法（服务视角）
每个微服务看：**Rate（QPS）→ Errors（错误率）→ Duration（P99 延迟）**。三个面板排列在同一行，扫一眼就能定位异常服务。

**两者结合**：RED 告诉哪个服务有问题，USE 告诉机器是否资源不足。

### 2.5 Grafana 看板设计

#### 三层分层
```
第一层：全局视图 — 全局 QPS/错误率/P99、各服务健康状态红/黄/绿
第二层：服务视图 — RED 面板 + JVM(Heap/GC/Thread) + DB(连接池/慢查询)
第三层：实例视图 — CPU/内存/网络/磁盘 + GC 日志 + 线程堆栈
```

#### 设计原则
- **从左到右，从粗到细**：左 = 概览，右 = 细分
- **颜色语义一致**：绿/黄/红，能力可建设在变色时就开始关注
- **同页相关性**：CPU、内存、GC、QPS、延迟放同一行，方便关联
- **画 SLO 线**：在 Y 轴上画 SLO 线（如延迟 200ms），一眼看达标情况

### 2.6 JVM 监控

```bash
# JMX Exporter 作为 Java Agent 运行
java -javaagent:jmx_prometheus_javaagent-0.20.0.jar=9404:config.yml -jar app.jar
```

关键指标与 PromQL：

| 指标 | PromQL | 告警阈值 |
|------|--------|---------|
| Heap 使用率 | `jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}` | > 85% |
| GC 频率 | `rate(jvm_gc_CollectionCount{gc="G1 Young Generation"}[5m])` | > 10 次/s |
| Full GC 耗时 | `rate(jvm_gc_CollectionTime{gc=~".*Old.*"}[10m])` | > 5 s/min |
| 线程数 | `jvm_threads_current` | 监控趋势防泄漏 |

**典型异常模式**：
1. 锯齿 GC + Heap 缓慢上升 → 内存泄漏，应 dump 分析
2. Full GC 飙升 + Old Gen 不回收 → 即将 OOM，立即重启 + dump
3. 线程数持续上涨 → 线程泄漏，看线程名定位
4. Metaspace 增长 → 动态类加载过多（CGLib 代理等），需限制

### 2.7 Alertmanager 告警

告警规则示例：
```yaml
groups:
  - name: application
    rules:
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} P99延迟超过1s"
          runbook_url: "https://wiki.internal/runbooks/HighLatency"

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
          sum(rate(http_requests_total[5m])) > 0.01
        for: 3m
        labels:
          severity: critical

      - alert: FrequentFullGC
        expr: rate(jvm_gc_CollectionTime{gc=~".*Old.*"}[10m]) > 5
        for: 10m
        labels:
          severity: warning
```

Alertmanager 路由配置核心参数：
- `group_wait: 30s`：聚合同组告警后再发
- `group_interval: 5m`：同组新告警每 5 分钟通知一次
- `repeat_interval: 4h`：相同告警 4 小时重复一次，防告警疲劳

**告警分级**：Critical(P0，PagerDuty/电话) | Warning(P1，Slack/飞书) | Info(P2，邮件/Ticket)

**告警疲劳铁律**：每条告警有 `runbook_url`；必须有 `for` 持续时间；每月做告警审查；尽量用环比变化率替代固定阈值。

## 三、日志 (Logging)

### 3.1 日志规范

#### 日志级别

| 级别 | 触发场景 | 生产行为 |
|------|---------|---------|
| **ERROR** | DB连接失败、业务异常 | 立即告警 |
| **WARN** | 重试成功、连接池接近耗尽 | 定期 Review |
| **INFO** | 关键业务流程节点 | 默认开启 |
| **DEBUG** | 调试信息 | 默认关闭 |
| **TRACE** | 每步执行细节 | 永不开启于生产 |

生产环境：Root Logger = WARN，业务包 = INFO，按需开 DEBUG。

#### 结构化日志

传统文本日志人读，结构化日志**机器可解析**。推荐 JSON 格式输出到 stdout：

```json
{
  "timestamp": "2026-07-05T14:30:22.125Z",
  "level": "INFO",
  "logger": "com.myc.OrderService",
  "thread": "http-nio-8080-exec-3",
  "traceId": "a1b2c3d4e5f67890",
  "message": "Order created",
  "mdc": { "userId": "12345", "orderId": "ORD-001", "amount": 99.90, "duration_ms": 120 }
}
```

Logback 配置使用 Logstash Encoder：
```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>userId</includeMdcKeyName>
    </encoder>
</appender>
```

这让你在 Kibana 中按 `mdc.userId:12345` 搜索该用户所有操作，按 `mdc.duration_ms:>1000` 找慢操作。

### 3.2 日志采集经典架构

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ App Pod  │  │ App Pod  │  │ App Pod  │   stdout/stderr
└────┬─────┘  └────┬─────┘  └────┬─────┘
     └──────────────┼──────────────┘
                    ▼
     ┌─────────────────────────┐
     │ Fluent Bit DaemonSet    │  每 Node 一个，tail /var/log/containers/*.log
     └────────────┬────────────┘
                  ▼
     ┌─────────────────────────┐
     │         Kafka           │  削峰填谷、持久化缓冲、多消费者
     └────────────┬────────────┘
                  ▼
     ┌─────────────────────────┐
     │  Logstash / Fluentd     │  清洗、脱敏、格式转换
     └────────────┬────────────┘
                  ▼
     ┌────────────────────────────┐
     │  Elasticsearch (Hot-Warm)  │  ILM：7天热→30天温→90天冷→删除
     └────────────┬───────────────┘
                  ▼
     ┌────────────────────────────┐
     │          Kibana            │  搜索 & 可视化
     └────────────────────────────┘
```

**为什么需要 Kafka**：削峰填谷防 ES 被压垮、解耦采集与消费、支持多消费者（ES + 安全审计 + 实时计算）。

**Logstash Pipeline 示例**：
```ruby
input {
  kafka { bootstrap_servers => "kafka:9092" topics => ["app-logs"] codec => json }
}
filter {
  mutate {
    gsub => [ "message", "(password=)[^&]+", "\1***" ]  # 脱敏
  }
  if [level] == "DEBUG" { drop {} }  # 丢弃 DEBUG 节省存储
}
output {
  elasticsearch { hosts => ["es:9200"] index => "logs-app-%{+YYYY.MM.dd}" }
}
```

### 3.3 EFK vs ELK

| 组件 | Logstash | Fluentd | Fluent Bit |
|------|----------|---------|------------|
| 语言 | JRuby (JVM) | CRuby | C |
| 内存 | ~500MB | ~100MB | ~10MB |
| 适用场景 | 复杂日志处理 | K8S 日志聚合 | 边缘/容器采集 |

推荐：Fluent Bit DaemonSet 采集 → Fluentd 聚合路由 → ES/Kafka/S3。

### 3.4 日志与链路关联 (TraceId 注入)

这是打通日志和追踪的关键。在请求入口生成/透传 TraceId 写入 MDC：

```java
@Component
public class TraceFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) {
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString().replace("-", "").substring(0, 16);
        }
        response.setHeader("X-Trace-Id", traceId);
        MDC.put("traceId", traceId);
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();  // 线程池复用必须清理！
        }
    }
}
```

对应的 Logback pattern：`[%X{traceId:-}]`，在 Kibana 中搜索 `traceId:"a1b2c3d4e5f67890"` 即可看到完整日志链。

### 3.5 日志最佳实践

1. **禁止 `System.out.println()`** — 不受框架管理，无 TraceId
2. **禁止循环打日志** — for 里打 DEBUG 瞬间打满磁盘
3. **异常必须记录上下文** — `log.error("创建订单失败", e)` 而非 `e.printStackTrace()`
4. **敏感信息脱敏** — 密码、Token、身份证、手机号、API Key，用 `@JsonIgnore`、`@ToString.Exclude` 或 Logback `%replace` 做多层防护
5. **关键路径记录耗时** — `log.info("操作耗时: {}μs", duration)`
6. **ES 索引管理** — 使用 ILM：Hot(0-7天,SSD) → Warm(8-30天,HDD) → Cold(冻结) → Delete

## 四、链路追踪 (Tracing)

### 4.1 核心概念

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

### 4.2 OpenTelemetry 标准

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

### 4.3 Jaeger 架构

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

### 4.4 SkyWalking

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

### 4.5 Spring Cloud Sleuth + Zipkin

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

### 4.6 采样策略

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

## 五、可观测性平台设计

### 5.1 整体架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                          应用层                                       │
│  Spring Boot(+Sleuth/Micrometer/Logback) / Go(+OTel/Prom/Zap)        │
└──────────────┬───────────────────┬────────────────────┬──────────────┘
               │ Logs              │ Metrics            │ Traces
               ▼                   ▼                    ▼
┌────────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Fluent Bit DaemonSet│ │ Prometheus Server │ │ OTel Collector   │
│ (采集 stdout Log)   │ │ (Pull /metrics)   │ │ (接收 OTLP/gRPC) │
└─────────┬──────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                    │
          ▼                     ▼                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Kafka Cluster (缓冲层)                       │
│              topic: logs | metrics-remote-write | traces             │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
│ Elasticsearch    │ │Prometheus TSDB│ │ ClickHouse   │
│ (日志全文检索)    │ │ (指标时序存储)│ │ (Trace 采样) │
└────────┬─────────┘ └───────┬──────┘ └──────┬───────┘
         │                  │               │
         ▼                  ▼               ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
│     Kibana       │ │   Grafana    │ │  Jaeger UI   │
│   (日志搜索)      │ │  (指标看板)  │ │ (追踪展示)    │
└──────────────────┘ └──────────────┘ └──────────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │  Alertmanager    │ → PagerDuty / 飞书 / Slack
                  └──────────────────┘
```

### 5.2 开源方案选型

| 方案 | 日志 | 指标 | 追踪 | 告警 | 适用团队 | 运维成本 |
|------|------|------|------|------|---------|---------|
| ELK + Prometheus + Jaeger | ES | Prometheus | Jaeger | Alertmanager | 中大型 | 高 |
| Grafana LGTM | Loki | Mimir | Tempo | Grafana Alerting | 中型 | 中 |
| SkyWalking 全家桶 | - | SkyWalking | SkyWalking | SkyWalking | 小型 Java | 低 |
| Elastic Stack 8+ | ES | ES | Elastic APM | Kibana Alerting | 中大型 | 中 |

**推荐**：Java 团队 → SkyWalking + Prometheus + Grafana + ELK；多语言 → OTel + Grafana LGTM。

### 5.3 核心告警规则

#### 基础设施
```yaml
- alert: HighCPUUsage
  expr: avg by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m])) * 100 > 80
  for: 10m

- alert: DiskRunningFull
  expr: predict_linear(node_filesystem_free_bytes{mountpoint="/"}[6h], 4*3600) < 0
  for: 5m

- alert: NodeDown
  expr: up == 0
  for: 2m
  labels: { severity: critical }
```

#### 应用层
```yaml
- alert: HighP99Latency
  expr: |
    histogram_quantile(0.99,
      sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
    ) > 2
  for: 5m

- alert: High5xxRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
    /
    sum(rate(http_requests_total[5m])) by (service) > 0.01
  for: 3m
  labels: { severity: critical }

- alert: QPSDrop
  expr: |
    rate(http_requests_total[5m])
    /
    rate(http_requests_total[5m] offset 1h) < 0.5
  for: 10m
```

#### JVM
```yaml
- alert: FrequentFullGC
  expr: rate(jvm_gc_CollectionTime{gc=~".*Old.*"}[10m]) > 5
  for: 10m

- alert: HighHeapUsage
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
  for: 5m

- alert: ThreadLeak
  expr: deriv(jvm_threads_current[30m]) > 10
  for: 30m
```

#### 中间件
```yaml
- alert: KafkaConsumerLag
  expr: kafka_consumer_group_lag_sum > 100000
  for: 15m

- alert: DBConnectionPoolExhausted
  expr: db_connection_pool_active / db_connection_pool_max > 0.9
  for: 5m
  labels: { severity: critical }
```

## 六、生产实践

### 6.1 故障排查流程

```
收到告警
  │
  ▼
Step 1 — 确认影响范围 (30s)
  Grafana 全局看板：多少服务变红？单服务异常还是全局？
  确定时间窗口和异常起点
  │
  ▼
Step 2 — 看指标缩小范围 (2min)
  打开 RED Dashboard：QPS/Error/Duration 哪个异常？
  打开 USE Dashboard：CPU/内存/磁盘/网络 哪个高？
  看依赖中间件：DB 连接池？Redis 命中率？
  │
  ▼
Step 3 — 用追踪定位延迟 (2min)
  Jaeger/SkyWalking UI → 搜索慢 Trace（按 Duration 排序）
  展开 Span 树：哪个 Span 耗时最长？哪个有重试？
  记录异常 Span 的 TraceId
  │
  ▼
Step 4 — 用日志定位根因 (2min)
  Kibana Discover → 搜索 TraceId
  查看 ERROR/WARN 日志、异常入参、下游完整错误信息、异常堆栈
  日志不够 → 开 DEBUG 观察后续请求
  │
  ▼
Step 5 — 决策 (1min)
  已知问题（慢SQL）→ 走优化流程
  新引入问题 → 立即回滚
  依赖服务故障 → 通知对应团队 + 临时降级/熔断
  资源不足 → 扩容
```

**核心**：TraceId 将追踪与日志串起来——没有 TraceId 只能在 Kibana 中大海捞针。

### 6.2 常见故障模式

#### 模式一：内存泄漏 → OOM
```
信号：Heap 不停上升无回落（锯齿变阶梯）→ Full GC 频率暴涨 → P99 延迟升高 → Pod OOMKilled
排查：确认哪个服务 → jmap -dump 导 heap dump → MAT 分析
预防：-XX:+HeapDumpOnOutOfMemoryError，Heap 使用率 > 85% 预警
```

#### 模式二：数据库连接池耗尽
```
信号：DB Connection Pool Active = Max(20/20) → Pending > 0 → 获取连接超时 → "Cannot get connection"
排查：数据库端连接分布 → 慢查询持有连接？→ 代码未在 finally 释放？
配置：hikari.leak-detection-threshold=10000（10s泄露告警）
```

#### 模式三：GC 停顿导致毛刺
```
信号：P99 每隔几分钟出现尖刺，P50 正常，时间与 Full GC 完全吻合
排查：jstat -gc <pid> 1000 → 是 YGC 还是 FGC？
  YGC → Eden 区太小，增大 -XX:NewRatio
  FGC → 大对象直接进 Old Gen，调 G1 参数
常用：-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=45
```

#### 模式四：Kafka 消费积压
```
信号：consumer_group_lag 持续上涨，消费者 CPU 正常
排查：消费者状态 → 消费逻辑慢在哪？
  DB 写入慢 → 批量写入
  下游 API 慢 → 异步化 + 增加并发
  某条消息异常卡住 → skip + 死信队列
应急：临时增加消费者实例/分区数
```

### 6.3 SLO 与错误预算

可观测性的最终目的是保障可靠性。SLO 是连接可观测平台与业务的桥梁：

```
服务: 订单服务
─────────────────────────
SLI                  SLO
─────────────────────────
可用性               99.95%（月）
P99 延迟              < 500ms
P50 延迟              < 100ms
5xx 错误率            < 0.1%
─────────────────────────
Error Budget = 1 - 0.9995 = 0.05% × 月分钟数 ≈ 22 分钟/月
```

**错误预算的核心用法**：预算充足时可以大胆发布，预算耗尽时冻结发布。告警按预算消耗速率触发，而非固定阈值。

```promql
# 预算消耗速率告警（当前短窗口消耗速率 > 10x 预算允许速率）
(
  1 - sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
) > ((1 - 0.9995) * 10)
```

### 6.4 可观测性文化建设

1. **先有可观测性，后上线服务**：Dashboard + 告警 + 日志采集 = 服务上线的硬性前置条件
2. **On-Call 授权**：值班工程师有权主动回滚/降级/限流，不需要审批。快速恢复 > 根因分析
3. **Postmortem 文化**：P0/P1 故障后记录时间线、根因、Action Items。追因不追责
4. **告警代码化管理**：告警规则和 runbook 纳入 Git，随服务代码一起 Review
5. **Dashboard 即文档**：新成员了解服务 → README + Dashboard + Runbook 三件套
6. **混沌工程定期演练**：非高峰期主动注入故障（Kill Pod、注入延迟、断网），验证告警和 runbook 是否有效

---

> 可观测性的终极目标不是"发现问题"，而是**让系统在问题发生时，自己能"说"清楚哪里出了问题**。一个好的平台，应在用户投诉前就发现异常，并在第一张 Grafana 截图内提示根因方向。
