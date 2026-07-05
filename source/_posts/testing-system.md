---
title: 后端测试体系
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 测试
  - 单元测试
  - 集成测试
  - 性能测试
categories:
  - 工程实践
---

测试不是"有时间就写"的可选任务，而是保障系统质量的工程纪律。一个成熟的后端测试体系应该覆盖从代码到生产的每一层。

## 一、测试金字塔

```
        ┌─────────┐
        │  E2E    │  少量（关键业务流程）
        │ 5%      │
       ┌┴─────────┴┐
       │ 集成测试   │  中等（跨服务/跨模块）
       │  15%      │
      ┌┴───────────┴┐
      │  单元测试    │  最多（单函数/单类）
      │    80%      │
     └─────────────┘
```

金字塔的核心思想：**越底层的测试越多、越快、越稳定**。E2E 测试虽然价值高，但慢且脆弱，只覆盖核心流程。

## 二、单元测试

### 2.1 什么值得测

| 应该测 | 不用测 |
|--------|--------|
| 业务逻辑（计算、校验、状态转换） | getter/setter |
| 工具类方法（字符串处理、格式转换） | 框架代码（Controller 路由映射） |
| 复杂条件分支 | 简单的一行委托调用 |

```java
// ✅ 值得测试
public Money calculateDiscount(Order order) {
    if (order.getTotal().greaterThan(new Money("1000"))) {
        return order.getTotal().multiply(0.1);
    }
    return Money.ZERO;
}

// ❌ 不值得专门测试
public void setOrder(Order order) { this.order = order; }
```

### 2.2 Mock vs Stub

| 技术 | 用途 | 示例 |
|------|------|------|
| **Stub** | 返回预设值，不验证调用 | `when(repo.findById(1L)).thenReturn(user)` |
| **Mock** | 验证是否调用、调用几次、参数正确 | `verify(repo, times(1)).save(any())` |

过度使用 Mock 会让测试变成"Mock 的实现测试"而非"业务逻辑测试"。优先用真实对象测试纯粹的业务逻辑，只在需要隔离的外部依赖（数据库、RPC、消息队列）时用 Mock。

### 2.3 JUnit 5 + Mockito 示例

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepo;
    @Mock PaymentGateway paymentGateway;
    @InjectMocks OrderService orderService;

    @Test
    void shouldCreateOrderWhenPaymentSucceeds() {
        // Given
        var request = new CreateOrderRequest(1L, "item-123", new Money("100"));
        var payment = Payment.success();
        when(paymentGateway.charge(any())).thenReturn(payment);

        // When
        var order = orderService.createOrder(request);

        // Then
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(orderRepo).save(any(Order.class));
    }

    @Test
    void shouldMarkFailedWhenPaymentDeclined() {
        when(paymentGateway.charge(any())).thenReturn(Payment.failed());

        assertThrows(PaymentFailedException.class,
            () -> orderService.createOrder(request));
    }
}
```

## 三、集成测试

集成测试验证多个组件协作是否正确——数据库连接、消息队列、外部 API。

### 3.1 数据库测试

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired UserRepository repo;

    @Test
    void shouldPersistAndRetrieve() {
        var user = new User("test@example.com", "Test");
        repo.save(user);

        var found = repo.findByEmail("test@example.com");
        assertThat(found).isPresent();
    }
}
```

用 **Testcontainers** 启动真实的数据库容器，比 H2 内存数据库更能暴露兼容性问题（如 MySQL/PostgreSQL 语法差异）。测试完成后容器自动销毁，不污染环境。

### 3.2 API 测试

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class UserControllerTest {

    @Autowired TestRestTemplate rest;

    @Test
    void shouldReturn200ForValidUser() {
        var response = rest.getForEntity("/users/1", UserDTO.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getName()).isEqualTo("pcl");
    }
}
```

### 3.3 契约测试

当服务 A 调用服务 B 时，A 需要知道 B 的 API 格式。Pact 等契约测试工具可以验证 A 的期望与 B 的实际输出一致——在微服务环境下比端到端的 E2E 测试更轻量、更可靠。

## 四、性能测试

### 4.1 压力测试

```
wrk -t8 -c200 -d30s https://api.example.com/users/1
```

关注指标：
- QPS 峰值
- P50/P99 延迟
- 错误率（< 0.1%）
- CPU/内存/网络使用率

### 4.2 基准测试

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
public class SerializationBenchmark {
    @Benchmark
    public byte[] protobuf() { return protoSerializer.serialize(obj); }

    @Benchmark
    public byte[] json() { return jsonSerializer.write(obj); }
}
```

用 JMH 对关键路径做微基准测试，量化优化效果。

## 五、测试覆盖率

| 指标 | 建议 |
|------|------|
| 行覆盖率 | > 80% |
| 分支覆盖率 | > 70% |
| 核心业务路径 | 100% |

覆盖率不是目标，暴露未测试的风险代码才是。高覆盖率 + 差断言 = 假安全感。

## 六、CI 中的测试策略

```
push → [单元测试(2min)] → [集成测试(5min)] → [代码扫描(3min)] → deploy to staging → [E2E测试(15min)]
      失败阻断             失败阻断           警告             自动部署            失败阻断
```

- 单元测试和集成测试在 PR 阶段运行，失败阻断合并
- E2E 测试通常在 staging 环境运行，避免阻塞开发者
- 每层测试耗时尽量控制在 15 分钟内

## 七、小结

后端测试体系的核心原则：**底层多写、上层少写、核心路径全覆盖**。80% 的单元测试保证快反馈，15% 的集成验证组件协作，5% 的 E2E 保障关键链路。测试不是为了数字，是为了在半夜被 PagerDuty 叫醒时，你的第一反应是"哪个依赖又挂了"而非"是不是我上周的提交搞坏了"。
