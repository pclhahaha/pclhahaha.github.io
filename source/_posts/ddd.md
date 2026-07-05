---
title: 领域驱动设计 (DDD)
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - DDD
  - 领域驱动设计
  - 架构
  - 微服务
categories:
  - 分布式
---
## 一、DDD 概述

### 1.1 为什么需要 DDD

大多数 Java 项目的代码结构是 `controller → service → dao → model`，这就是经典的**贫血模型**——对象只有数据没有行为，业务逻辑全部塞在 Service 层。项目初期开发快，但随着业务复杂化：

- Service 类膨胀到 3000+ 行，方法之间相互调用，代码读起来像解谜。
- 业务规则散落各处：判断订单能否取消的逻辑在 OrderService、RefundService、LogisticsService 各有一份。
- 新人打开 `OrderDO`，只有几十个 getter/setter，真正的业务含义藏在 Service 里的某一行中。

**充血模型**把数据和行为放在一起：`Order` 对象自己知道怎么取消、怎么添加订单项、怎么计算总价。DDD 的核心思想是：**将业务逻辑建模为领域模型，让代码结构与业务结构保持一致**。

### 1.2 DDD 不是银弹

| 场景 | 为什么不适用 |
|---|---|
| 简单的 CRUD 系统（后台管理、数据面板） | 业务逻辑简单，贫血模型 + MyBatis Generator 足矣 |
| 纯数据管道（ETL、日志处理） | 没有复杂的业务规则需要建模 |
| 团队 2-3 人且业务不会扩张 | DDD 的学习成本和沟通成本高于收益 |
| 技术驱动的项目（中间件、基础设施） | 领域逻辑不是核心 |

**DDD 适用场景**：电商订单履约、保险理赔、银行贷款审批、供应链调度——业务流程长、状态多、规则复杂、需要多角色协作。

> 判断标准：如果你的代码里频繁出现 `if status == xxx then do yyy`，且 status 有十几种，那就是 DDD 的信号。

### 1.3 DDD 与微服务的关系

DDD 和微服务是两个独立概念，但在实践中高度互补：

- **DDD 解决"怎么拆"的问题**：限界上下文天然就是微服务拆分的边界。
- **微服务解决"怎么部署"的问题**：每个限界上下文独立部署、独立扩展。

没有 DDD 的拆分容易变成"拆表"——按数据库表拆服务，一个业务流程要调 7-8 个服务。通过限界上下文划分，服务边界就是业务能力边界：

```
DDD 提供划分依据           微服务提供技术落地
┌──────────────┐         ┌──────────────┐
│ 限界上下文    │   ──→   │  微服务 A     │
│ 聚合         │   ──→   │  数据所有权    │
│ 领域事件     │   ──→   │  消息队列     │
└──────────────┘         └──────────────┘
```

## 二、战略设计

战略设计关注宏观建模——系统怎么划分、上下文之间怎么协作，回答"我们在做什么"和"边界在哪里"。

### 2.1 统一语言 (Ubiquitous Language)

最常见的沟通场景：

```
产品经理："用户下单后如果 30 分钟没付钱，订单就自动关掉。"
开发："所以是 update order set status = 'CANCELLED' where create_time < ?"
产品经理："不是 cancelled，是关闭，我们叫'超时未支付自动关闭'。"
```

问题出在：**代码里的词汇和业务语言不在一个语境里**。统一语言要求：

1. 领域专家和开发团队一起定义**词汇表**，形成项目 Wiki 的术语表。
2. 代码中类名、方法名直接使用业务术语，不是翻译。
3. 术语在限界上下文中才有意义——同一个词在不同上下文可以有不同含义。

实例——电商订单的统一语言：

| 业务术语 | 代码映射 | 说明 |
|---|---|---|
| 订单 | `Order` (聚合根) | 用户提交的购买请求 |
| 订单项 | `OrderItem` (实体) | 订单中的商品条目 |
| 已提交 | `OrderStatus.SUBMITTED` | 订单已创建等待支付 |
| 已支付 | `OrderStatus.PAID` | 买家已完成付款 |
| 已发货 | `OrderStatus.SHIPPED` | 仓库已出库 |
| 履约 | `FulfillmentService` | 从支付到签收的全过程 |

> 核心原则：**如果领域专家和你说的不是同一种语言，那你建的模就是错的**。

### 2.2 限界上下文 (Bounded Context)

限界上下文是 DDD 中最核心的概念。一个限界上下文就是一个**语义明确的业务能力边界**——边界内部的模型是内聚的、一致的和自洽的。

**如何划分——四个维度**：

**1. 领域专家访谈。** 和采购聊商品入库、和仓管聊出库、和运营聊订单——对话内容的边界往往就是上下文的边界。

**2. 业务边界。** 当一个概念在不同场景下语义变化时，就是上下文边界。比如"商品"：

| 上下文 | "商品"含义 | 关心的属性 |
|---|---|---|
| 商品上下文 | 商品定义 | SKU、名称、规格、图片、品牌 |
| 订单上下文 | 下单快照 | 下单时价格、数量、商品名 |
| 库存上下文 | 物理库存 | SKU、库存量、储位、批次 |
| 物流上下文 | 待运输包裹 | 重量、体积、发货地址 |

如果全部塞进 `t_product(200 columns)`，就是标准的大泥球。

**3. 组织结构（康威定律）。** 商品团队和订单团队是两个独立小组，它们的系统大概率也需要分开。

**4. 数据边界。** 如果两张表总是一起被修改（强一致性事务），它们大概率在同一个上下文中。

**电商系统上下文划分**：

```
┌──────────┬──────────┬───────────┬───────────┬──────────┬──────────┐
│ 商品上下文│ 订单上下文│ 支付上下文 │ 库存上下文 │ 物流上下文│ 用户上下文│
│ Product  │ Order    │ Payment   │ Inventory │ Logistics│ User     │
├──────────┼──────────┼───────────┼───────────┼──────────┼──────────┤
│ 商品信息  │ 订单管理  │ 支付      │ 库存管理   │ 发货      │ 用户     │
│ 类目管理  │ 购物车    │ 退款      │ 入库/出库  │ 签收      │ 地址     │
│ 品牌管理  │ 订单状态  │ 对账      │ 库存预留   │ 轨迹追踪  │ 会员     │
└──────────┴──────────┴───────────┴───────────┴──────────┴──────────┘
```

### 2.3 上下文映射 (Context Map)

划分上下文只是第一步，上下文之间如何协作才是实战难点：

**合作关系 (Partnership)**：同一团队维护，目标一致，紧密协作。如支付和退款在同一个小团队。

**共享内核 (Shared Kernel)**：共享一部分模型和数据，各自有扩展。如用户登录态在各子系统中共享。共享内核应尽量小，只共享核心概念（UserId, Address）。

**客户-供应商 (Customer-Supplier)**：上游供能力，下游用能力。下游需求驱动上游接口。如订单（客户）依赖商品（供应商）查询商品信息。关键原则：**下游（客户）定义接口契约，上游（供应商）实现**。

**防腐层 (Anti-Corruption Layer)**：当下游依赖外部/遗留系统而模型不一致时，引入翻译层：

```java
// 防腐层接口——只暴露订单上下文需要的语义
public interface RiskAssessmentService {
    RiskAssessmentResult assess(Order order);
}

// 防腐层实现——将 Order 模型翻译为外部系统的格式
@Service
public class RiskAssessmentAdapter implements RiskAssessmentService {
    public RiskAssessmentResult assess(Order order) {
        RiskRequest request = RiskRequest.builder()
            .userId(order.getUserId().toString())
            .orderAmount(order.getTotalAmount().toPlainString())
            .build();
        RiskResponse response = riskSystemClient.evaluate(request);
        return new RiskAssessmentResult(
            response.getRiskLevel() <= 2,
            response.getReason()
        );
    }
}
```

**开放主机服务 / 发布语言 (OHS/PL)**：一个上下文被多个下游消费时，提供标准化 API + 标准数据格式。如商品上下文提供 RESTful API + 商品 JSON Schema。

**各行其道 (Separate Ways)**：两个上下文不需要集成就不集成。

| 关系类型 | 技术实现 |
|---|---|
| 合作关系 / 共享内核 | 进程内调用（同服务） |
| 客户-供应商 | RPC / HTTP，下游定义接口契约 |
| 防腐层 | 独立模块，翻译外部模型 |
| 开放主机 | 标准化 REST API / gRPC + 协议定义 |

## 三、战术设计

战略设计解决"是什么"和"边界在哪"，战术设计解决"怎么实现"。

### 3.1 实体 (Entity)

实体是**具有唯一标识**的对象，属性可变但标识不变。同一个人 5 岁和 30 岁身高体重完全不同，但身份证号没变，所以是同一实体。

```java
public class Order {
    private OrderId id;             // 唯一标识，不变
    private OrderStatus status;     // 可变
    private List<OrderItem> items;  // 可变
    private Money totalAmount;      // 可变

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        return id.equals(((Order) o).id);
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

关键原则：实体通过唯一标识判断相等性，`equals`/`hashCode` 仅基于标识实现；实体可变且应该有行为方法，而非 `setXxx()`。

### 3.2 值对象 (Value Object)

值对象是**没有唯一标识**的不可变对象，用属性值定义相等性。典型：金额、地址、电话号码。

```java
public class Money {
    private final BigDecimal amount;

    public Money(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount must not be negative");
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
    }

    public Money add(Money other) {
        return new Money(this.amount.add(other.amount));
    }

    public Money multiply(int factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)));
    }

    @Override
    public boolean equals(Object o) { /* 比较 amount */ }
    @Override
    public int hashCode() { return Objects.hash(amount); }
}
```

特征：不可变（无 setter，修改返回新对象）；无标识（两张一样的 100 元无所谓哪一张）；自验证（构造时校验）。常见候选：金额、坐标范围、电话号码、邮箱、百分比、时间段。

> 口诀：**如果你关心它"是什么"而不是"是谁"，它就是值对象**。

### 3.3 聚合 (Aggregate) 与聚合根 (Aggregate Root)

聚合是**一组必须保持一致性的对象，对外只暴露一个入口——聚合根**。

```
聚合 "Order"
┌──────────────────────────────────────────┐
│  Order (聚合根)                           │
│  ├── orderId                             │
│  ├── status                              │
│  ├── totalAmount                          │
│  ├── items: List<OrderItem> (实体)        │
│  │    └── 只能通过 Order 访问/修改         │
│  └── shippingAddress: Address (值对象)    │
└──────────────────────────────────────────┘
```

```java
public class Order {

    private OrderId id;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();
    private Money totalAmount = Money.ZERO;

    // 聚合根控制所有的修改入口
    public void addItem(ProductId productId, String name, int quantity, Money price) {
        if (status != OrderStatus.DRAFT) {
            throw new OrderException("Only draft order can add items");
        }
        items.add(new OrderItem(productId, name, quantity, price));
        recalculateTotal();
    }

    public void submit() {
        if (items.isEmpty()) throw new OrderException("Cannot submit empty order");
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(this.id));
    }

    public void pay() {
        if (status != OrderStatus.SUBMITTED) throw new OrderException("Only submitted order can be paid");
        this.status = OrderStatus.PAID;
        registerEvent(new OrderPaidEvent(this.id, this.totalAmount));
    }

    public void ship(TrackingNumber trackingNumber) {
        if (status != OrderStatus.PAID) throw new OrderException("Must be paid before shipping");
        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderShippedEvent(this.id, trackingNumber));
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubTotal)
            .reduce(Money.ZERO, Money::add);
    }

    // 不暴露 setItems / setStatus
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
}
```

**聚合设计的四条黄金法则**：

1. **聚合尽量小**——只包含必须强一致的对象。大聚合等于大写锁。
2. **通过 ID 引用其他聚合，而非对象引用**——`Order` 持有 `UserId` 而非 `User` 对象。
3. **聚合内部强一致性，聚合间最终一致性**——事务只在单个聚合内生效。
4. **聚合根控制所有访问**——外部不能绕过 `Order` 直接修改 `OrderItem`。

```
错误（聚合过大）：                        正确（小聚合 + ID 引用）：
┌──────────────────────────────┐        ┌─────────┐     ┌─────────┐
│ Order 聚合 (含 User/Product等) │        │ Order   │     │ Product │
│ → 并发冲突、性能差             │        │ orderId │     │productId│
└──────────────────────────────┘        │ userId  │──→  │ name    │
                                        │productId│←    │ price   │
                                        └─────────┘     └─────────┘
```

### 3.4 领域服务 (Domain Service)

当某个业务操作**不属于任何一个实体或值对象**时，交给领域服务。判断标准：这个操作涉及多个聚合，而且没有一个自然的所有者。

```java
// 转账不属于 Account 的职责（Account 不知道自己转给谁）
// 也不属于 Money（Money 不知道账户），所以是领域服务
@Service
public class TransferService {
    private final AccountRepository accountRepository;

    public void transfer(AccountId fromId, AccountId toId, Money amount) {
        Account from = accountRepository.findById(fromId).orElseThrow(...);
        Account to = accountRepository.findById(toId).orElseThrow(...);
        from.debit(amount);
        to.credit(amount);
        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

### 3.5 领域事件 (Domain Event)

领域事件是聚合内已发生的**有业务意义**的状态变化——不是技术事件（"数据库写入成功"），而是业务事件（"订单已支付"）。

```java
public class OrderPaidEvent extends DomainEvent {
    private final OrderId orderId;
    private final Money amount;
    // constructor / getter
}

// 聚合内部产生事件
public void pay() {
    this.status = OrderStatus.PAID;
    registerEvent(new OrderPaidEvent(this.id, this.totalAmount));
}

// 应用层在持久化后发布
@Transactional
public void payOrder(OrderId orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow(...);
    order.pay();
    orderRepository.save(order);
    order.getEvents().forEach(eventPublisher::publish);
}
```

核心价值：解耦聚合（发券/积分/短信各自监听而非主动调用）；事件溯源（存储事件序列而非快照）；审计追踪。

### 3.6 资源库 (Repository)

Repository 是聚合的持久化接口，隐藏存储细节，让领域层**像操作内存集合一样操作持久化数据**。

**Repository vs DAO**：

| | DAO | Repository |
|---|---|---|
| 粒度 | 按数据库表 | 按聚合 |
| 返回 | DO / 数据库实体 | 领域对象 (聚合) |
| 职责 | 数据库 CRUD | 聚合的持久化与重建 |
| 所属层 | 基础设施层 | 领域层接口（实现在基础设施层） |

```java
// 领域层接口
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

// 基础设施层实现
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderMapper orderMapper;
    private final OrderItemMapper itemMapper;

    @Override
    public Optional<Order> findById(OrderId id) {
        OrderDO orderDO = orderMapper.selectById(id.getValue());
        if (orderDO == null) return Optional.empty();
        List<OrderItemDO> itemDOs = itemMapper.selectByOrderId(id.getValue());
        return Optional.of(Order.reconstitute(orderDO, itemDOs));
    }

    @Override
    public void save(Order order) {
        OrderDO orderDO = OrderDO.fromDomain(order);
        orderMapper.upsert(orderDO);
        itemMapper.deleteByOrderId(order.getId().getValue());
        order.getItems().forEach(item -> itemMapper.insert(OrderItemDO.from(item)));
    }
}
```

关键点：Repository 接口在领域层，实现在基础设施层；`save` 保存整个聚合而非单表；重建聚合需工厂方法绕过业务校验。

### 3.7 工厂 (Factory)

当聚合创建逻辑复杂时用工厂封装。简单 `new Order(id)` 不需要工厂——**只有构造有明显复杂度时（跨聚合依赖、多步构建）才需要**。

## 四、架构分层

### 4.1 四层架构与依赖倒置

```
┌──────────────────────────────────────────────────┐
│              接口层 (Interface)                   │  Controller, DTO, 参数校验
├──────────────────────────────────────────────────┤
│              应用层 (Application)                 │  用例编排、事务管理、权限校验
├──────────────────────────────────────────────────┤
│               领域层 (Domain)                    │  Entity, ValueObject, Aggregate,
│                                                  │  DomainService, Repository 接口
├──────────────────────────────────────────────────┤
│             基础设施层 (Infrastructure)           │  RepositoryImpl, MQ, 外部 API
└──────────────────────────────────────────────────┘
```

**依赖方向：接口层 → 应用层 → 领域层 ← 基础设施层**（下层的依赖指向下层，但基础设施层通过接口实现指向领域层）。

```
传统三层（依赖正置）                  DDD 四层（依赖倒置）
┌─────────┐                      ┌─────────────┐
│ Service  │──依赖──→ DAO         │  领域层      │
└─────────┘                      │ (定义接口)    │
   领域层知道基础设施                 └──────┬──────┘
                                          │  实现
                                   ┌──────┴──────┐
                                   │  基础设施层   │
                                   └─────────────┘
```

**各层职责划分**：

```java
// 接口层——只做请求处理、协议转换
@RestController
public class OrderController {
    @PostMapping("/orders")
    public Result<OrderVO> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        return Result.success(applicationService.createOrder(toCommand(request)));
    }
}

// 应用层——用例编排、事务管理
@Service
public class OrderApplicationService {
    @Transactional  // 事务在应用层
    public OrderId createOrder(CreateOrderCommand cmd) {
        Order order = new Order(OrderId.generate(), cmd.getBuyerId());
        cmd.getItems().forEach(i -> order.addItem(i.getProductId(), i.getQuantity(), i.getPrice()));
        order.submit();
        orderRepository.save(order);
        order.getEvents().forEach(eventPublisher::publish);
        return order.getId();
    }
}

// 领域层——纯业务逻辑，零框架依赖
public class Order {
    public void submit() {
        if (items.isEmpty()) throw new OrderException("Cannot submit empty order");
        this.status = OrderStatus.SUBMITTED;
    }
}
```

**代码归属判断标准**：

| 代码内容 | 应属于 |
|---|---|
| `if (amount < 0) throw ...` | 领域层 |
| 编排多个聚合 + `repository.save` + 发事件 | 应用层 |
| `request.getHeader("Authorization")` | 接口层 |
| `jdbcTemplate.update(...)` | 基础设施层 |

### 4.2 @Transactional 应该在哪一层？

**答案：应用层。** 事务是技术关注点，不是业务关注点。`Order.pay()` 只是在说"我现在已支付"，它不知道也不应该知道自己在数据库事务中。领域层应纯 Java 模块，不依赖 Spring 或任何框架——copy 到新项目不加依赖就能编译通过。

### 4.3 六边形架构（端口适配器）

六边形架构的概念：领域逻辑是核心，所有外部依赖（DB、MQ、HTTP）通过端口（接口）和适配器（实现）连接到核心。

```
        ┌──────────────────────────────┐
        │       适配器 (Adapter)        │
        │   HTTP / MQ / CLI / Test    │
        ├──────────────────────────────┤
        │       端口 (Port)            │
   ┌────┴────┐                  ┌──────┴───┐
   │ 输入端口  │                  │ 输出端口   │
   └──────────┘                  └──────┬───┘
        │                               │
   ┌────┴───────────────────────────────┴───┐
   │           领域层 (Domain)              │
   │      业务规则不依赖任何外部框架          │
   └───────────────────────────────────────┘
```

落地建议：不需教条画六边形，关键是做到 (1) 领域层零外部依赖 (2) 所有外部依赖通过接口倒置。做到这两点，换数据库/换 MQ/换 RPC 框架时领域代码一行不用改。


**事件溯源 (Event Sourcing)** 是 CQRS 进阶版：存储所有事件序列而非当前状态。适用于审计要求极高的金融场景，但复杂度高，不要轻易使用。

### 4.5 Spring Boot 项目包结构

```
com.example.order/
├── interfaces/                         # 接口层
│   ├── rest/OrderController.java       # REST 接口
│   ├── dto/                            # 入参 DTO / 出参 VO
│   └── mq/OrderEventListener.java      # 消息监听
├── application/                        # 应用层
│   ├── service/OrderApplicationService.java
│   ├── command/                        # 应用层命令对象
│   └── event/EventPublisher.java
├── domain/                             # 领域层
│   ├── model/order/
│   │   ├── Order.java                  # 聚合根
│   │   ├── OrderId.java                # 值对象
│   │   ├── OrderItem.java              # 实体
│   │   ├── OrderStatus.java            # 枚举
│   │   └── Money.java                  # 值对象
│   ├── service/PricingService.java     # 领域服务接口
│   ├── event/                          # 领域事件
│   └── repository/OrderRepository.java # 仓储接口
└── infrastructure/                     # 基础设施层
    ├── persistence/
    │   ├── OrderRepositoryImpl.java
    │   ├── mapper/OrderMapper.java
    │   └── converter/OrderConverter.java
    ├── messaging/RabbitMQEventPublisher.java
    └── external/RiskAssessmentAdapter.java
```

## 五、DDD 与微服务拆分

### 5.1 限界上下文 → 微服务的映射

```
限界上下文 ────────────────→ 微服务
┌──────────────┐           ┌──────────────┐
│   订单上下文  │    ─→     │ order-service │
│   Order聚合  │           │  数据库:       │
└──────────────┘           │  order_db     │
                           └──────────────┘
┌──────────────┐           ┌──────────────┐
│   商品上下文  │    ─→     │product-service│
│   Product聚合│           └──────────────┘
└──────────────┘
```

理论上一上下文对应一微服务，实践中需权衡：极简上下文可合并到相邻服务；庞大上下文可进一步拆分。最终标准：**一个微服务 = 一个独立可部署单元，由一个自治团队负责**。

### 5.2 聚合内强一致，聚合间最终一致

- **单聚合内修改** → 本地数据库事务。
- **跨聚合修改** → 领域事件 + 最终一致性。

```
订单支付流程：
Order.pay()  →  OrderPaidEvent  →  InventoryService.deductStock()
(本地事务)       (MQ 消息)            (库存服务自己的本地事务)

               →  OrderPaidEvent  →  CouponService.markAsUsed()
               →  OrderPaidEvent  →  NotificationService.sendSMS()
```

### 5.3 分布式事务 → Saga 模式

跨服务业务流程不能再用 `@Transactional`。在 DDD 中采用** Saga 模式**（补偿事务）：

```
正向：订单创建 → 库存预留 → 支付扣款 → 积分发放
某个步骤失败后的补偿：
支付扣款失败 → 补偿：释放库存预留 → 补偿：取消订单
```

```java
@Service
public class OrderSagaOrchestrator {
    public void process(CreateOrderCommand cmd) {
        Order order = orderService.createDraft(cmd);
        try {
            inventoryService.reserve(order.getItems());
        } catch (Exception e) {
            orderService.cancel(order.getId());
            throw e;
        }
        try {
            paymentService.charge(order);
        } catch (Exception e) {
            inventoryService.release(order.getItems());
            orderService.cancel(order.getId());
            throw e;
        }
    }
}
```

**心态转变**：从"一个分布式事务搞定一切"到"每步本地事务 + 失败补偿"——金融系统中的冲正就是这个模式。

## 六、DDD 落地实践

### 6.1 Application Service vs Domain Service

| | Application Service | Domain Service |
|---|---|---|
| 职责 | 用例编排、事务管理 | 跨实体/聚合的纯业务逻辑 |
| 调用 Repository | 可以 | 理论上可，不推荐 |
| 调用外部服务 | 可以 | 不行（通过接口倒置） |
| 管理事务 | 必须 | 绝不 |
| 典型例子 | `createOrder()` / `payOrder()` | `PricingService` / `TransferService` |

```java
// 应用层——编排
@Transactional
public void payOrder(OrderId orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow(...);
    order.pay();
    orderRepository.save(order);
    order.getEvents().forEach(eventPublisher::publish);
}

// 领域服务——纯业务逻辑
public class OrderPricingService {
    public Money calculateTotal(List<ItemRequest> items, List<Coupon> coupons) {
        Money subtotal = items.stream()
            .map(i -> i.getPrice().multiply(i.getQuantity()))
            .reduce(Money.ZERO, Money::add);
        // 优惠券处理...
        return subtotal;
    }
}
```

### 6.2 Repository 实现（MyBatis 示例）

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderMapper orderMapper;
    private final OrderItemMapper itemMapper;

    @Override
    public Optional<Order> findById(OrderId id) {
        OrderDO orderDO = orderMapper.selectById(id.getValue());
        if (orderDO == null) return Optional.empty();
        List<OrderItemDO> itemDOs = itemMapper.selectByOrderId(id.getValue());
        return Optional.of(toDomain(orderDO, itemDOs));
    }

    @Override
    public void save(Order order) {
        OrderDO orderDO = OrderDO.fromDomain(order);
        if (orderMapper.countById(order.getId().getValue()) > 0) {
            orderMapper.update(orderDO);
        } else {
            orderMapper.insert(orderDO);
        }
        itemMapper.deleteByOrderId(order.getId().getValue());
        order.getItems().forEach(item ->
            itemMapper.insert(OrderItemDO.from(item, order.getId())));
    }

    private Order toDomain(OrderDO orderDO, List<OrderItemDO> itemDOs) {
        List<OrderItem> items = itemDOs.stream()
            .map(this::toItemDomain)
            .collect(Collectors.toList());
        return Order.reconstitute(
            new OrderId(orderDO.getId()),
            OrderStatus.valueOf(orderDO.getStatus()),
            items,
            new Money(new BigDecimal(orderDO.getTotalAmount()))
        );
    }
}
```

### 6.3 JPA 充血模型的注意事项

```java
@Entity
@Table(name = "orders")
public class Order {
    @EmbeddedId
    private OrderId id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    protected Order() {}   // JPA only, protected

    // 业务方法，不用 setter
    public void pay() {
        this.status = OrderStatus.PAID;  // field access
    }
}
```

两个选择：(1) JPA 直接映射——领域对象即 `@Entity`，接受无参构造约束但不暴露 public setter；(2) 领域对象与数据对象分离——领域层纯 POJO，基础设施层独立 DO + Converter。推荐后者，但前者对于小项目足够。

## 七、常见误区与建议

### 7.1 不要过度设计

如果业务规则能在 3 行 if-else 内描述清楚，不需要 DDD。后台管理系统的用户 CRUD 上全套 Entity/ValueObject/Aggregate/DomainService/DomainEvent，维护成本远高于简单的 Controller → Service → Mapper。

### 7.2 不是所有对象都要 DDD 化

```java
// 没必要
public class PageRequest extends ValueObject {  // ← 过度设计
    private final int page;
    private final int size;
}
```

`PageRequest` 就是数据载体。战术模式只用在**真正需要建模的业务概念**上。

### 7.3 聚合不能太大

大聚合是落地中最常见的反模式——订单聚合里放用户、商品详情、优惠券、收货地址……15 个实体 12 张表。后果：每次保存性能差、不同订单因关联同一商品而行锁冲突、加载整个聚合内存压力大。

**聚合只包含必须强一致的关联**。商品改名不影响历史订单快照——订单保存的是下单时的快照值。

### 7.4 不要为了 DDD 而 DDD

见过太多这样的代码：

```java
public class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand> { ... }
public class OrderCreatedEventHandler implements DomainEventHandler<OrderCreatedEvent> { ... }
// 文件名比业务代码长，做的事就是 orderService.create()
```

DDD 的核心是**业务建模**，不是技术模式。从核心开始：统一语言 → 限界上下文 → 在核心上下文用聚合封装业务规则 → 渐进式引入 CQRS / 领域事件。

> **删掉所有 DDD 名词后，代码是否仍清晰表达业务逻辑？如果是，就做对了。**

### 7.5 团队层面建议

- DDD 需要领域专家深度参与。产品经理只会扔 PRD，DDD 推不动。
- **别在全公司铺开 DDD**。在最重要的 1-2 个核心域深度应用，其他子域继续 CRUD。
- 代码审查聚焦"业务语义是否准确"，而非"是否用了正确的 DDD 模式"——后者是本末倒置。
- 领域层代码应可脱离框架运行——不依赖 Spring、MyBatis、任何基础设施。

```
┌───────────────────────────────────────────────────┐
│                DDD 落地核心口诀                    │
├───────────────────────────────────────────────────┤
│ 战略先行：统一语言 + 限界上下文 → 划定边界再动手    │
│ 战术服务业务：只对复杂规则使用聚合/值对象          │
│ 依赖倒置：领域层零框架依赖                         │
│ 聚合在事务内，聚合间最终一致                       │
│ 从核心域开始，渐进式推广                           │
│ 不要求完美，先让代码说人话                         │
└───────────────────────────────────────────────────┘
```

## 参考资料

- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003)
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013)
- Martin Fowler, [BoundedContext](https://martinfowler.com/bliki/BoundedContext.html)
- Microsoft, [.NET Microservices Architecture](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)
- Chris Richardson, [Microservices Patterns](https://microservices.io/patterns/)
