---
title: 流量治理 (限流/熔断/降级)
date: 2026-07-05
updated: 2026-07-05
tags:
  - 微服务
  - 限流
  - 熔断
  - 降级
  - Sentinel
categories:
  - 微服务
---

### 1.1 限流

> 限流的令牌桶/漏桶/滑动窗口等算法详见 [分布式系统理论基础](distributed-theory.md) 的"流量控制"章节。

在微服务架构中，限流的落点通常在三层：
1. **网关层**：全局限流，防止流量打死任何下游
2. **RPC 框架层**：Dubbo/Sentinel 拦截器，服务级限流
3. **业务层**：对特定接口/资源（如秒杀 SKU）的精准限流

```java
// Sentinel 限流注解
@SentinelResource(value = "createOrder", blockHandler = "createOrderFallback")
public Order createOrder(CreateOrderRequest request) {
    // 业务逻辑
}

public Order createOrderFallback(CreateOrderRequest request, BlockException e) {
    // 被限流后的降级处理
    return Order.fail("系统繁忙，请稍后再试");
}
```

### 1.2 熔断（Circuit Breaker）

熔断器的灵感来自电路保险丝：当电流过大时自动断开，保护电路。在微服务中同理——当某个下游服务故障率达到阈值，熔断器断开，后续请求直接失败（快速失败），不继续压垮它。

**Hystrix 断路器状态机**：

```
                请求失败数超过阈值
    ┌─────────┐ ──────────────────────► ┌─────────┐
    │  CLOSED │                          │   OPEN  │
    │ (闭合)   │ ◄─────────────────────── │ (断开)   │
    │         │    超时后恢复检测         │         │
    └─────────┘                          └─────────┘
         ▲                                    │
         │                                    │ 探测请求成功
         │         ┌─────────────┐            │
         └─────────│ HALF-OPEN   │◄───────────┘
                   │ (半开)       │
                   └─────────────┘
                   探测请求失败 → 重新 OPEN
```

**核心参数**：
```java
HystrixCommand.Setter
    .withCircuitBreakerEnabled(true)
    .withCircuitBreakerRequestVolumeThreshold(20)       // 请求数阈值（最低 20 次）
    .withCircuitBreakerErrorThresholdPercentage(50)     // 错误率 50% 触发熔断
    .withCircuitBreakerSleepWindowInMilliseconds(5000); // 熔断后 5s 进入半开
```

**注意事项**：
- 熔断器不是越多越好——每个熔断器维护独立的状态和线程池，开销不可忽视
- 不要对**所有异常**都触发熔断计数——业务异常（如"商品已下架"）不是下游故障，不应参与熔断统计
- 熔断颗粒度：一般以服务为单位，细粒度的可以到方法级别

### 1.3 降级（Fallback）

降级是熔断/限流被触发后的兜底逻辑。当系统判定当前不可靠时，宁可返回一个降级结果，也不让上游超时等待。

**分类**：

| 类型       | 说明                                   | 示例                         |
| ---------- | -------------------------------------- | ---------------------------- |
| 容错降级   | 被熔断/限流时触发                      | 返回缓存数据 / 提示"请重试"  |
| 优雅降级   | 主动关闭非核心功能，保证核心可用       | 双11时关闭评论、推荐         |
| 异步降级   | 同步调用失败转为异步补偿               | 打款失败 → 记录 → T+1 重试   |

```java
// Hystrix 降级（已不推荐，改用 Sentinel）
@HystrixCommand(fallbackMethod = "getUserFallback")
public User getUser(Long id) {
    return userService.getUser(id); // 可能失败
}

public User getUserFallback(Long id) {
    // 降级：返回缓存 / 默认用户
    return cacheService.getUserFromCache(id);
}
```

### 1.4 Sentinel

Sentinel（阿里开源）是取代 Hystrix 的更好选择，Hystrix 已进入维护模式。Sentinel 的核心能力：

| 能力       | 说明                                           |
| ---------- | ---------------------------------------------- |
| 流控       | QPS/并发线程数/匀速排队（漏桶效果）            |
| 熔断降级   | 慢调用比例/异常比例/异常数                     |
| 热点参数限流 | 对特定参数值（如 SKU ID）单独限流             |
| 系统保护   | 系统 Load/CPU/RT 超过阈值时自动限流            |
| 规则持久化 | 推送规则到 Nacos/Apollo，动态生效              |
| 控制台     | 可视化 Dashboard，实时监控秒级数据             |

**Sentinel vs Hystrix**：

| 维度           | Hystrix                    | Sentinel                |
| -------------- | -------------------------- | ----------------------- |
| 隔离策略       | 线程池/信号量              | 信号量（轻量）          |
| 熔断粒度       | 服务级                     | 接口/方法/参数级        |
| 规则配置       | 代码硬编码                 | 控制台动态下发          |
| 实时监控       | Hystrix Dashboard          | Sentinel Dashboard      |
| 自适应         | 无                         | 系统自适应保护          |
| 维护状态       | 停更                       | 活跃                    |

```java
// Sentinel 流控规则
FlowRule rule = new FlowRule();
rule.setResource("createOrder");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS 限流
rule.setCount(100);                           // 每秒 100 次
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP); // 预热
rule.setWarmUpPeriodSec(10);
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

**最佳实践**：规则统一在 Sentinel Dashboard 管理，通过 Nacos 做规则持久化，确保 Dashboard 重启后规则不丢失。

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080
      datasource:
        ds1:
          nacos:
            server-addr: 127.0.0.1:8848
            data-id: ${spring.application.name}-sentinel-rules
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```

### 1.5 隔离

隔离是防止"服务内一个接口的异常拖整个服务"的关键。

**线程池隔离**：
- 每个依赖服务分配独立的线程池
- 优点：完全隔离，下游慢不会耗尽上游的线程
- 缺点：线程上下文切换开销大，OOM 风险（线程栈内存）

**信号量隔离**：
- 限制并发数，使用调用线程执行
- 优点：轻量，无线程切换开销
- 缺点：不隔离超时——下游慢会阻塞调用线程

```
线程池隔离（Hystrix）:                    信号量隔离（Sentinel）:

┌──────────────┐                        ┌──────────────┐
│  调用线程      │                        │  调用线程      │
│      ▼         │                        │      ▼         │
│  HystrixCmd    │                         │   Sentinel     │
│      ▼         │                        │   检查信号量    │
│ ┌───────────┐  │                        │      ▼         │
│ │ 线程池 (10) │ │                        │  直接执行调用   │
│ │    ▼       │  │                        │  (若超过阈值则  │
│ │  下游服务   │  │                        │   直接Block)   │
│ └───────────┘  │                        └──────────────┘
└──────────────┘
```

**生产建议**：
- 大多数场景用信号量隔离（Sentinel 默认），性能更好
- 当依赖的服务经常有慢查询且不可控时，用线程池隔离——宁可开销大一点，不能让整个服务阻塞

---
