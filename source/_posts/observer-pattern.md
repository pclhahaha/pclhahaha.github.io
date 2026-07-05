---
title: 观察者模式 — Spring Event / MQ
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 观察者
  - Spring Event
  - MQ
categories:
  - 设计模式
---

**意图**：定义对象间的一对多依赖，当一个对象状态变化时，所有依赖者自动得到通知。

**UML 简述**：Subject 持有多个 Observer，状态变化时通知所有已注册的 Observer 调用其 update 方法。

**JDK 内置实现**：`Observable` + `Observer`，Java 9+ 已弃用——类而非接口、线程不安全、违背组合优于继承。现代 Java 中观察者模式通过事件机制落地。

**Spring Event 机制**

Spring 的事件机制是观察者模式在 IoC 容器中的标准实现：

```java
// 1. 事件
public class OrderPaidEvent extends ApplicationEvent {
    private final Long orderId;
    public OrderPaidEvent(Object source, Long orderId) {
        super(source);
        this.orderId = orderId;
    }
}

// 2. 监听器（观察者）
@Component
public class SmsListener {
    @EventListener
    public void handle(OrderPaidEvent event) {
        System.out.println("发送短信, 订单: " + event.getOrderId());
    }
}

@Component
public class CouponListener {
    @EventListener @Async
    public void handle(OrderPaidEvent event) {
        System.out.println("发放优惠券, 订单: " + event.getOrderId());
    }
}

// 3. 发布事件
@Service
public class OrderService {
    @Autowired private ApplicationEventPublisher publisher;
    public void pay(Long orderId) {
        publisher.publishEvent(new OrderPaidEvent(this, orderId));
    }
}
```

Spring Event 核心流程：`publisher.publishEvent()` → `ApplicationEventMulticaster.multicastEvent()` → 遍历匹配的 Listener → 同步/异步调用。加 `@Async` + `@EnableAsync` 即可异步执行。

**MQ 发布订阅 —— 跨进程的观察者**：MQ 将观察者模式扩展到分布式环境——Publisher → MQ Broker → Subscriber 1 / Subscriber 2，解决了单进程内 Subject-Observer 的耦合问题。

**三种观察者实现对比**

| 实现方式 | 耦合度 | 可靠性 | 适用场景 |
|---------|--------|--------|---------|
| JDK Observable | 紧耦合（同线程） | 低 | 几乎不用 |
| Spring Event | 松耦合（同容器） | 中 | 单应用内业务解耦 |
| MQ 发布订阅 | 无耦合（跨进程） | 高（持久化+重试） | 跨服务异步解耦 |

---
