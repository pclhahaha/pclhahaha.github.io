---
title: Kafka Streams 流处理
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 数据工程
  - Kafka
  - 流处理
categories:
  - 数据工程
---

Kafka Streams 是 Kafka 生态内置的流处理库。与 Flink 的外部集群模式不同，Kafka Streams 只是一个 Java 库——你把它嵌入到任何 Java 应用中，无需独立的集群。

## 一、Kafka Streams vs Flink

| 维度 | Kafka Streams | Flink |
|------|--------------|-------|
| 部署 | Java 库，嵌入应用 | 独立集群 |
| 状态 | 本地 RocksDB | 本地/远程 RocksDB |
| 扩展 | 应用实例数 = Kafka 分区数 | 独立扩缩 |
| SQL | KSQL (Kafka SQL) | Flink SQL (完整 ANSI) |
| 精确一次 | 支持 | 支持 |
| 延迟 | 毫秒级 | 毫秒级 |
| 适合 | 简单流处理、Kafka 重度用户 | 复杂流处理、多数据源 |

## 二、核心概念

### KStream vs KTable

```java
// KStream: 事件流，每个事件独立
KStream<String, Order> orders = builder.stream("orders");

// KTable: 变更日志，每个 key 只保留最新值
KTable<String, User> users = builder.table("users");

// 流表 JOIN：事件到达时关联当前最新的用户信息
orders.join(users, (order, user) -> enrich(order, user))
      .to("enriched-orders");
```

### 核心操作

```java
KStream<String, PageView> views = builder.stream("pageviews");

// 窗口聚合：每 5 分钟统计 PV
views.groupByKey()
     .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
     .count()
     .toStream()
     .to("pviews-per-5min");

// 过滤 + 转换 + 分流
views.filter((key, view) -> view.getDuration() > 10000)
     .mapValues(v -> new EngagedView(v))
     .to("engaged-views");
```

## 三、状态管理

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Event> events = builder.stream("events");

// 有状态处理：使用 KeyValueStore
events.groupByKey()
      .aggregate(
          () -> new HashSet<>(),                               // 初始化
          (key, event, seen) -> { seen.add(event.getId()); return seen; },  // 更新
          Materialized.as("seen-events-store")                 // 状态存储名
      );
```

默认使用本地 RocksDB 存储状态。如果应用重启，状态从 Kafka 内部的 changelog topic 恢复——Kafka Streams 自动把每次状态变更写入一个专用的内部 topic。

## 四、KSQL——流式 SQL

```sql
-- 创建流
CREATE STREAM orders (
    order_id BIGINT,
    user_id BIGINT,
    amount DECIMAL(10,2)
) WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='JSON');

-- 实时聚合
CREATE TABLE hourly_stats AS
SELECT user_id, COUNT(*) AS order_count, SUM(amount) AS total
FROM orders
WINDOW TUMBLING (SIZE 1 HOUR)
GROUP BY user_id;

-- 流表 JOIN
SELECT o.order_id, u.name, o.amount
FROM orders o JOIN users u ON o.user_id = u.user_id;
```

## 五、精确一次

Kafka Streams 的精确一次依赖于 Kafka 的事务机制：

```
1. 消费消息 → 处理 → 写入输出 topic + 提交 offset
   这三步在同一个 Kafka 事务中完成
2. 失败 → 事务回滚，offset 不提交 → 重试
3. 成功 → 事务提交
```

## 六、适用场景

| 场景 | 选择 |
|------|------|
| 数据源只有 Kafka | **Kafka Streams** |
| 多数据源（Kafka + DB + File） | Flink |
| 团队不熟悉 Java | Flink SQL / KSQL |
| 复杂事件处理（CEP） | Flink |
| 已有的 Spring Boot 服务加流处理 | **Kafka Streams** |

## 七、小结

Kafka Streams 的核心价值是"零运维的流处理"——没有独立集群、没有额外的部署步骤。适合 Kafka 重度用户、简单到中等复杂度的流处理场景。一旦需要多数据源 JOIN、CEP 或复杂窗口，Flink 就是更好的选择。
