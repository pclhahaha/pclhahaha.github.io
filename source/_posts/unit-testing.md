---
title: 单元测试金字塔
date: 2026-07-05
updated: 2026-07-05
tags:
  - 测试
  - JUnit
  - Mockito
  - TDD
categories:
  - 工程实践
---

### 4.1 JUnit 5 + Mockito 基础

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepository;
    @Mock private PaymentGateway paymentGateway;
    @InjectMocks private OrderService orderService;

    @Test
    @DisplayName("支付成功应更新订单状态")
    void shouldUpdateOrderStatusWhenPaymentSucceeds() {
        Order order = mock(Order.class);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentGateway.pay(1L)).thenReturn(PaymentResult.success());

        orderService.pay(1L);
        verify(order).markAsPaid();
    }

    @Test
    @DisplayName("订单不存在时应抛出异常")
    void shouldThrowWhenOrderNotFound() {
        when(orderRepository.findById(1L)).thenReturn(Optional.empty());
        assertThrows(OrderNotFoundException.class, () -> orderService.pay(1L));
    }
}
```

### 4.2 测试金字塔

```
         ┌──────┐
         │ E2E  │ ← 极少
         │ 集成  │ ← 较少
         │ 单元  │ ← 大量
         └──────┘
```

| 层级 | 速度 | 稳定性 | 占比 | 工具 |
|------|:---:|:---:|:---:|------|
| 单元测试 | 毫秒 | 极高 | 70%+ | JUnit 5 + Mockito |
| 集成测试 | 秒 | 中 | 20% | Spring Boot Test + Testcontainers |
| E2E 测试 | 分钟 | 低 | 10% | REST Assured / Selenium |

不要追求精确比例，**追求反馈速度**：单元测试 200ms 内完成，发现错误立即反馈。

### 4.3 什么值得测

```
不值得测 →                        ← 最值得测 →
简单 getter/setter   核心算法/金额计算   复杂状态机/边界条件/异常路径
```

核心规则：涉及钱必测；边界（null/空/零/负/超大值）必测；异常路径必测。不测 getter/setter、框架代码、一眼可确认的简单逻辑。

### 4.4 TDD 争议与实践

TDD 的理想流程：写失败的测试 → 写最小代码通过 → 重构。现实态度：**在复杂逻辑和核心算法上 TDD，在简单 CRUD 上后补测试。** TDD 真正的价值不在"先写测试"，而在于**驱动你思考接口设计**——你必须先回答"输入是什么？输出是什么？边界是什么？"——这本身就是一次 mini 设计评审。

### 4.5 测试覆盖率

```java
// 行覆盖 ≠ 分支覆盖
if (score >= 90) return "A";
if (score >= 80) return "B";   // 只测 95,75 → 行 100%，分支 67%
return "C";
```

核心业务 90%+/85%+（行/分支），公共库 90%+，CRUD 70%+，Controller 40%+。**覆盖率是手段不是目的**——用它发现未测试代码。CI 中设新代码阈值 ≥80%。

---


```yaml
`
