---
title: 限流算法对比
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - 限流
  - Sentinel
  - Guava RateLimiter
categories:
  - 分布式
---

在分布式系统中，限流是保护服务不被突发流量击垮的关键手段。核心目标是：在保证系统承载能力的前提下，尽可能多地处理请求。

### 10.1 计数器（固定窗口）

最简单的方式：在固定的时间窗口内计数，超过阈值则拒绝。

```
窗口: [12:00:00 - 12:00:01)
计数器: 0 → 1 → 2 → ... → 100 → 拒绝
12:00:01: 计数器重置为 0
```

**临界问题**：在窗口边界处（如 12:00:00.900 - 12:00:01.100），两个窗口各允许 100 次，实际通过了 200 次——是期望值的两倍。

### 10.2 滑动窗口

固定窗口的改进版，将大窗口切分为多个小格子：

```
将 1 秒窗口切分为 10 个格子（每个 100ms）
当前时间: 12:00:01.350
计算范围: [12:00:00.350, 12:00:01.350] 内命中 10 个格子中的计数
```

滑动窗口更平滑，基本消除了固定窗口的临界突变问题，是生产环境的常用方案。

### 10.3 漏桶（Leaky Bucket）

漏桶算法的核心：

- 请求以任意速率到达，先放入**固定容量的桶**（队列）
- 桶以**恒定速率**漏出（处理）请求
- 桶满后，新请求被丢弃

**特点**：强制输出速率恒定，能有效应对突发流量（突发请求由桶容量缓存），但峰值处理能力被严格限制。适合需要平滑输出的场景（如消息队列生产速率控制）。

### 10.4 令牌桶（Token Bucket）

令牌桶是漏桶的"逆操作"，也是 Guava RateLimiter 和 Sentinel 限流的基础：

- 系统以**固定速率**向桶中放入令牌
- 桶有最大容量（`maxTokens`），满后丢弃多余令牌
- 请求到达时从桶中取令牌，取到则通过，否则拒绝或等待

```java
// Guava RateLimiter 令牌桶实现
RateLimiter limiter = RateLimiter.create(10.0);  // 每秒 10 个许可

public void handleRequest() {
    if (limiter.tryAcquire(100, TimeUnit.MILLISECONDS)) {
        // 处理请求
    } else {
        // 限流
        throw new RateLimitException("too many requests");
    }
}

// 预热模式：启动时低速率，逐步增加到稳定速率
RateLimiter warmupLimiter = RateLimiter.create(10.0, 5, TimeUnit.SECONDS);
```

#### SmoothBursty vs SmoothWarmingUp

Guava RateLimiter 提供两种令牌获取模式：

- **SmoothBursty**：默认模式，允许突发流量——桶中的令牌可以一次性取完，然后进入等待
- **SmoothWarmingUp**：预热模式——系统刚启动时令牌生成速率较低，逐步增加到稳定值，适合需要预热的系统（如缓存冷启动）

### 10.5 Sentinel

Sentinel 是阿里巴巴开源的流量治理组件，以"流量为切入点"提供：

- **限流**：支持 QPS/并发线程数/关联资源限流，基于滑动窗口统计
- **熔断**：基于响应时间/异常比例/异常数熔断
- **热点参数限流**：对"热点"参数值（如爆款商品 ID）进行更细粒度的限流
- **系统自适应限流**：根据系统 Load、CPU 使用率、平均 RT 等自动调整阈值

Sentinel 的限流算法基于改进的滑动窗口（LeapArray），使用环形数组存储各时间窗口的统计数据，滑动时只更新头尾两个窗口，时间复杂度 O(1)。

```java
// Sentinel 限流规则配置
FlowRule rule = new FlowRule();
rule.setResource("orderService");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);  // QPS 上限 100
FlowRuleManager.loadRules(Collections.singletonList(rule));

// 使用
Entry entry = null;
try {
    entry = SphU.entry("orderService");
    // 业务逻辑
} catch (BlockException e) {
    // 被限流或熔断
} finally {
    if (entry != null) entry.exit();
}
```

### 10.6 限流算法对比

| 算法 | 平滑度 | 突发容忍 | 实现复杂度 | 适用场景 |
|------|--------|----------|------------|----------|
| 固定窗口 | 低 | 否 | 极低 | 简单场景 |
| 滑动窗口 | 中 | 否 | 低 | 通用限流 |
| 漏桶 | 高 | 是（由桶容量） | 低 | 流量整型、消息队列 |
| 令牌桶 | 中高 | 是 | 中 | 突发流量场景、网关 |
| Sentinel | 高 | 可配置 | 中 | 生产级流量治理 |


- **数据完整性检测**：通过 checksum 检测数据损坏并自动从正常副本恢复；通过 chunk version number 检测并回收过期的陈旧副本
