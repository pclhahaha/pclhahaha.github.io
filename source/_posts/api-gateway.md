---
title: API 网关
date: 2026-07-05
updated: 2026-07-05
tags:
  - 微服务
  - API网关
  - Spring Cloud Gateway
categories:
  - 微服务
---

### 1.1 网关的作用

网关是所有外部请求进入微服务体系的第一道关口，做的是"横切关注点"的抽象：

```
外部请求
  │
  ▼
┌──────────────────────────────────────────┐
│                  网关                     │
│                                          │
│  1. 路由转发  →  /api/order/** → order-svc
│  2. 鉴权认证  →  JWT 校验 / OAuth2       │
│  3. 限流      →  令牌桶 / 漏桶           │
│  4. 日志      →  请求/响应日志统一采集    │
│  5. 协议转换  →  HTTP → gRPC / WebSocket │
│  6. 负载均衡  →  Round Robin 分发        │
│  7. 灰度路由  →  Header 染色 → 金丝雀实例 │
│                                          │
└──────────────────────────────────────────┘
  │
  ▼
内部微服务
```

网关的本质是把每个微服务都需要处理的基础设施逻辑上提到网关层，让业务开发关注"做什么"而不是"怎么做身份校验/限流/日志"。但注意：**网关不宜承载重业务逻辑**——它是流量入口，逻辑越重延迟越大，影响全站。

### 1.2 Spring Cloud Gateway

Spring Cloud Gateway 基于 **WebFlux（Reactor + Netty）**，非阻塞 I/O，性能和资源利用率远超传统 Servlet 容器。

**核心三要素**：

| 概念      | 说明                                | 示例                            |
| --------- | ----------------------------------- | ------------------------------- |
| Route     | 路由规则，ID + 目标 URI + 断言 + 过滤器 | 把 `/api/order/**` 转到 order 服务 |
| Predicate | 断言，匹配路由条件                    | `Path=/api/order/**`            |
| Filter    | 过滤器，对请求/响应做加工             | `AddRequestHeader=X-Color, blue` |

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service            # lb:// 启用负载均衡
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=1                  # 去掉 /api
            - AddRequestHeader=X-Source, gateway
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: RequestRateLimiter       # 限流
              args:
                redis-rate-limiter:
                  replenishRate: 10          # 每秒填充令牌数
                  burstCapacity: 20          # 令牌桶容量
        - id: canary-route
          uri: lb://order-service-canary
          predicates:
            - Path=/api/order/**
            - Header=X-Canary, true          # Header 中含有 X-Canary=true
```

**自定义 GatewayFilter**：

```java
@Component
public class AuthFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (StringUtils.isEmpty(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // JWT 校验逻辑...
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100; // 优先级最高
    }
}
```

### 1.3 Zuul 1.x vs Zuul 2.x vs Gateway

| 维度       | Zuul 1.x              | Zuul 2.x                 | Spring Cloud Gateway |
| ---------- | --------------------- | ------------------------ | -------------------- |
| 底层实现   | Servlet 2.5（阻塞）   | Netty（非阻塞）          | WebFlux + Netty      |
| 线程模型   | 每连接一个线程         | 事件驱动                  | 事件驱动             |
| 性能       | 差（线程切换开销大）   | 好                        | 好                   |
| Spring 支持| Spring Cloud Netflix  | 非 Spring 官方            | Spring 官方          |
| 维护状态   | 仅修 Bug              | 慢                        | 活跃                 |

**结论**：新项目直接用 **Spring Cloud Gateway**。它不仅是 Spring 官方力推，底层 WebFlux 也代表着 Java 未来的方向（响应式编程）。

---
