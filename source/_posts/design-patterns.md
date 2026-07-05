---
title: 设计模式 -- 后端实践
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 设计模式
  - Java
  - 架构
categories:
  - CS基础
---
## 一、设计模式概述

### 1.1 什么是设计模式

设计模式是前人总结的、针对特定问题的可复用解决方案——一套命名良好的编程惯例，目的是降低耦合、提高可维护性。

但**设计模式不是银弹**。模式的价值在于提供通用词汇表，让工程师用"单例""策略""责任链"快速对齐思路，而不是让每行代码都套用一种模式。

### 1.2 六大设计原则

在谈具体模式之前，必须先理解背后的原则。模式是手段，原则才是目的。

**SOLID 五原则**

| 原则 | 英文 | 核心含义 | 一句话记忆 |
|------|------|----------|-----------|
| 单一职责 | SRP | 一个类只做一件事 | 不要让别人修改你的类时顺带改你的功能 |
| 开闭原则 | OCP | 对扩展开放，对修改关闭 | 加新功能靠新增代码，不靠改旧代码 |
| 里氏替换 | LSP | 子类可以替换父类且行为不变 | 方形的矩形可以当矩形用吗？不行 |
| 接口隔离 | ISP | 接口应该小而专 | 别让实现类被迫实现它不需要的方法 |
| 依赖倒置 | DIP | 依赖抽象而非具体实现 | 高层模块不应依赖低层模块，两者都依赖接口 |

**迪米特法则（Law of Demeter / 最少知识原则）**

也叫"不要和陌生人说话"。一个对象应该对其他对象有尽可能少的了解——只与直接的朋友通信。

```java
// 违反迪米特法则：链式穿透多个对象
String zip = order.getCustomer().getAddress().getZip();

// 遵循迪米特法则：通过中间对象暴露所需数据
String zip = order.getCustomerZip();
```

迪米特法则的核心价值在于降低耦合——调用者不需要知道被调用者的内部结构。但也别走向极端，为每个字段都包一层 getter 会导致类膨胀。

六大原则不是孤立的教条，而是相互补充的实践指南。实际编码中，优先遵守 SRP 和 DIP，其他原则往往会在过程中自然满足。

### 1.3 设计模式三大分类

GoF 将 23 种设计模式划分为三类：

| 分类 | 关注点 | 核心思路 | 典型代表 |
|------|--------|---------|----------|
| 创建型 | 对象怎么创建 | 将对象的创建与使用分离 | 单例、工厂、建造者、原型 |
| 结构型 | 类/对象怎么组合 | 通过继承/组合构建更大结构 | 代理、适配器、装饰器、门面 |
| 行为型 | 类/对象怎么协作 | 关注对象之间的责任分配和通信 | 观察者、策略、模板方法、责任链、状态 |

以下各章节**只聚焦后端开发中最常用的模式**，略过那些"考试常见但生产鲜见"的模式。

---

## 二、创建型模式

创建型模式的核心问题：**如何在不指定具体类的情况下创建对象**。Spring 的 Bean 容器本质上就是一个巨大的工厂 + 单例注册表。


---

### 2.2 工厂方法 & 抽象工厂

**意图**：定义一个创建对象的接口，让子类决定实例化哪个类。工厂方法使类的实例化延迟到子类。

直接 `new` 的问题：调用方需要知道具体类名，而具体类可能会变。工厂将这些变化封装起来。

**工厂方法 vs 抽象工厂**

| 维度 | 工厂方法 | 抽象工厂 |
|------|---------|---------|
| 产品数量 | 单一产品 | 产品族（一组相关产品） |
| 实现方式 | 继承，子类覆盖工厂方法 | 组合，注入不同的工厂实例 |
| 扩展点 | 新增具体产品 = 加一个工厂子类 | 新增产品族 = 加一个工厂实现 |
| 典型场景 | 一个方法返回一个对象 | 一组相关对象需要配套使用 |

**Spring BeanFactory —— 工厂方法的集大成者**

```java
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
}
```

`getBean()` 是工厂方法，具体的创建逻辑分散在各实现类中（`DefaultListableBeanFactory` 负责注册和获取，`AbstractAutowireCapableBeanFactory` 负责实例化和依赖注入）。`ApplicationContext` 继承 `BeanFactory`，形成经典的"工厂方法层次结构"。

**MyBatis SqlSessionFactory**

```java
SqlSessionFactory factory = new SqlSessionFactoryBuilder()
    .build(Resources.getResourceAsStream("mybatis-config.xml"));
// SqlSessionFactory 用工厂方法创建 SqlSession
// SqlSessionFactoryBuilder 用建造者模式从 XML 构建出 SqlSessionFactory
```

MyBatis 展示了**工厂方法 + 建造者的组合使用**。

**JDK 中的工厂方法**

```java
Calendar cal = Calendar.getInstance();          // 工厂方法
ExecutorService pool = Executors.newFixedThreadPool(10);  // 静态工厂方法
```

---

### 2.3 建造者模式

**意图**：将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

**何时使用**：
- 构造函数参数超过 4 个
- 参数有可选值，且多种组合
- 需要构建不可变对象

**Lombok @Builder**

```java
@Builder
public class OrderRequest {
    private String orderId;
    private Long userId;
    private BigDecimal amount;
    private String payChannel;    // 可选
    private String couponCode;     // 可选
    private String remark;         // 可选
}

// 使用
OrderRequest request = OrderRequest.builder()
    .orderId("ORD-001")
    .userId(1001L)
    .amount(new BigDecimal("99.00"))
    .payChannel("WECHAT")
    .remark("加急配送")
    .build();
```

Lombok 编译期生成静态内部类 `OrderRequestBuilder`，每个字段对应一个 setter 方法返回 `this` 实现链式调用，`build()` 方法调用全参构造器创建目标对象。

**OkHttp Request.Builder**

```java
Request request = new Request.Builder()
    .url("https://api.example.com/data")
    .header("Authorization", "Bearer token")
    .post(RequestBody.create(json, MediaType.parse("application/json")))
    .build();
```

OkHttp 的 `Request` 是不可变对象，大量可选配置（头、超时、缓存策略）通过 Builder 构建，避免参数爆炸。

**建造者模式 vs 工厂模式**

| 对比维度 | 建造者 | 工厂 |
|---------|--------|------|
| 关注点 | 如何一步步组装复杂对象 | 直接创建对象，不关心步骤 |
| 参数数量 | 很多，且可选 | 少，通常是必须的 |
| 构建过程 | 分步骤，可以灵活组合 | 一次性完成 |
| 返回时机 | 调用 build() 时才创建 | 调用工厂方法即返回 |

---

### 2.4 原型模式

**意图**：通过复制已有对象来创建新对象，而不是通过 new。

**Java 中的核心接口**：`Cloneable` + `Object.clone()`

```java
public class ReportTemplate implements Cloneable {
    private String header;
    private String footer;
    private List<String> columns;

    @Override
    public ReportTemplate clone() {
        try {
            ReportTemplate cloned = (ReportTemplate) super.clone();
            // 深拷贝：集合字段需要单独处理
            cloned.columns = new ArrayList<>(this.columns);
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

注意 `super.clone()` 是**浅拷贝**——引用类型字段只复制引用。如果对象包含集合或其他可变对象，需要手动实现深拷贝。

**Spring @Scope("prototype")**

```java
@Component
@Scope("prototype")
public class ShoppingCart { /* ... */ }

// 每次 getBean 都返回一个新实例
ShoppingCart cart1 = context.getBean(ShoppingCart.class);
ShoppingCart cart2 = context.getBean(ShoppingCart.class);
// cart1 != cart2
```

Spring 的 `prototype` scope 并非严格意义上的 clone，而是每次重新创建。但从效果上看，它实现了"通过已有定义来获取新对象"的语义。

> `Cloneable` 被普遍认为是失败的设计（clone 方法靠约定而非接口定义）。实际项目推荐 JSON 序列化/反序列化或 MapStruct 做深拷贝。

---

## 三、结构型模式

结构型模式关注如何将类或对象组合成更大的结构。它们的核心价值在于**解耦接口与实现**，让系统更容易扩展。


---

### 3.2 适配器模式

**意图**：将一个接口转换成客户期望的另一个接口，使原本不兼容的接口能一起工作。

**UML 简述**：适配器持有被适配对象，实现目标接口，将客户调用转换为对 Adaptee 的调用。

**SpringMVC HandlerAdapter**

这是适配器模式在后端框架中最经典的应用。SpringMVC 中有多种 Handler 类型（`@Controller`、`HttpRequestHandler`、`Servlet`），每种都有自己的执行方式：

```java
public interface HandlerAdapter {
    boolean supports(Object handler);               // 是否支持该 handler
    ModelAndView handle(HttpServletRequest req,     // 执行 handler
                        HttpServletResponse resp,
                        Object handler) throws Exception;
}

// 实现类示例
public class RequestMappingHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }
    // ... 处理 @RequestMapping 注解的方法
}

public class HttpRequestHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HttpRequestHandler;
    }
    // ... 处理 HttpRequestHandler 类型
}
```

`DispatcherServlet.doDispatch()` 中的核心逻辑：

```java
// 简化后的源码逻辑
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
// 遍历所有 HandlerAdapter，调用 supports() 找到匹配的适配器
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

**SLF4J —— 日志门面的适配器层**

SLF4J 的绑定层（如 `slf4j-log4j12`）是纯适配器：应用 -> SLF4J API(门面) -> slf4j-logback(适配器) / slf4j-log4j12(适配器) -> Logback / Log4j 1.x。每个绑定包内部将 SLF4J 的 `Logger.info()` 转换为具体日志框架的 `logger.log()`。

---

### 3.3 装饰器模式

**意图**：动态地给对象添加额外的职责，提供比继承更灵活的扩展方式。

**与代理模式的区别**：这是另一个面试高频题。装饰器和代理的类结构几乎一样，但**意图完全不同**：
- 代理：控制访问（鉴权、延迟加载、远程调用）——代理决定"是否调用"
- 装饰器：增强功能（加日志、加密、压缩）——装饰器必然调用目标，并在前后加功能

**Java IO —— 装饰器模式的集大成者**

Java IO 流的类层次是装饰器模式的经典范例：

```java
// 核心组件
InputStream fileInput = new FileInputStream("data.txt");

// 一层层装饰
InputStream buffered = new BufferedInputStream(fileInput);    // 增加缓冲
InputStream dataInput = new DataInputStream(buffered);        // 增加基本类型读取

// 一行等价写法
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("data.txt")
    )
);
```

每个 `FilterInputStream` 的子类都是一个装饰器，持有另一个 `InputStream` 的引用，在调用被装饰流的方法前后添加行为：

```java
public class BufferedInputStream extends FilterInputStream {
    // 持有 InputStream in (继承自 FilterInputStream)

    @Override
    public int read() throws IOException {
        if (缓冲区为空) {
            fill();  // 批量预读 → 减少系统调用
        }
        return 缓冲区中读取一个字节;
    }
}
```

**Java Collections 中的装饰器**

```java
// 将线程不安全的集合装饰成线程安全的
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// 将可变集合装饰成不可变的
List<String> unmodifiable = Collections.unmodifiableList(list);

// 实现原理——内部是一个包装类
static class SynchronizedList<E> implements List<E> {
    final List<E> list;  // 被装饰的对象
    final Object mutex;

    public boolean add(E e) {
        synchronized (mutex) {
            return list.add(e);  // 在原有方法外包裹同步
        }
    }
}
```

**装饰器 vs 代理 vs 适配器**

| 模式 | 增强行为？ | 改变接口？ | 核心目的 |
|------|----------|----------|---------|
| 装饰器 | 是 | 否（接口相同） | 动态添加功能 |
| 代理 | 可有可无 | 否（接口相同） | 控制访问 |
| 适配器 | 否 | 是（接口不同） | 接口转换 |

---

### 3.4 门面模式

**意图**：为子系统中的一组接口提供一个统一的高层接口，使子系统更容易使用。

门面模式可能是最被低估的设计模式之一。它不涉及复杂的类结构，但价值巨大——**减少调用方需要知道的类的数量**，降低使用门槛。

**SLF4J 日志门面**

SLF4J 的门面层：

```java
// 应用程序只需要面对 SLF4J 这一个"门面"
Logger logger = LoggerFactory.getLogger(MyClass.class);
logger.info("用户登录成功, userId={}", userId);

// 底层的 Logback/Log4j2 对调用方完全透明
// 切换日志框架只需要更换依赖，代码零改动
```

这就是门面模式的价值——应用代码只依赖 SLF4J API，不直接依赖任何具体日志实现。

**门面在现代架构中的演变：网关**

微服务架构中，API 网关将多个服务的接口聚合成统一入口，对客户端屏蔽后端复杂性——与门面"为子系统提供统一接口"的意图完全一致。

**门面 vs 适配器 vs 代理 速记**

```
门面（Facade）  → 简化接口（把复杂的变简单）
适配器（Adapter） → 转换接口（把不兼容的变兼容）
代理（Proxy）    → 控制接口（在访问前后加上控制逻辑）
```

---

## 四、行为型模式

行为型模式关注算法和对象间的职责分配。它们是日常业务代码中最常被使用的模式类别。


---


---


---


---


---

## 五、设计模式在框架中的应用速查表

以下表格覆盖了本文讨论的所有模式在后端主流框架/库中的具体落点：

| 设计模式 | JDK | Spring | 其他框架 |
|----------|-----|--------|---------|
| 单例 | `Runtime`, `Desktop` | Bean 默认 Scope | Logback `LoggerFactory` |
| 工厂方法 | `Calendar.getInstance()` | `BeanFactory.getBean()` | MyBatis `SqlSessionFactory` |
| 抽象工厂 | `DocumentBuilderFactory` | `AopProxyFactory` | - |
| 建造者 | `StringBuilder`, `Stream.Builder` | `RestTemplateBuilder` | Lombok `@Builder`, OkHttp `Request.Builder` |
| 原型 | `Cloneable` | `@Scope("prototype")` | - |
| 代理 | `Proxy.newProxyInstance()` | Spring AOP (JDK/CGLIB) | Feign 声明式调用 |
| 适配器 | `InputStreamReader` | `HandlerAdapter` | SLF4J 绑定层 |
| 装饰器 | `BufferedInputStream`, `CheckedInputStream` | `TransactionAwareCacheDecorator` | - |
| 门面 | - | `JdbcTemplate`(部分) | SLF4J API |
| 观察者 | `Observer/Observable`(已弃用) | `@EventListener` / `ApplicationEvent` | MQ 发布订阅 |
| 策略 | `Comparator` | `ResourceLoader` | - |
| 模板方法 | `InputStream.read(byte[])` | `AbstractApplicationContext.refresh()` | `JdbcTemplate` |
| 责任链 | - | `HandlerInterceptor` | Servlet `Filter`, Netty `ChannelPipeline` |
| 状态 | - | Spring State Machine | - |

---

## 六、实际项目中的取舍

### 6.1 什么时候不该用设计模式

学会了模式之后最容易犯的错误就是**到处套模式**。以下场景慎重：

**1) 过度设计**

一个只有 3 个 if 分支的业务逻辑，不需要拆成 5 个类+3 个接口。随着系统演进加分支再来重构。

```java
// 过度设计——为了一个简单的转换，写了 4 个类
// String → Integer 只需要 Integer.parseInt() 就能搞定
```

**2) 性能敏感路径**

模式带来的间接调用、对象创建和抽象层次，在热点路径上是实打实的开销：
- 代理模式的反射调用有性能损耗
- 装饰器模式的层层包装增加对象创建开销
- 观察者模式的同步通知可能阻塞关键路径

**3) 团队水平参差**

如果一个模式的抽象让团队一半以上的人看不懂，那就别用。代码首先是给人读的，其次才是给机器执行的。

### 6.2 KISS vs DRY 的平衡

| 原则 | 含义 | 极端问题 |
|------|------|---------|
| KISS | Keep It Simple, Stupid | 代码重复、不够抽象 |
| DRY | Don't Repeat Yourself | 过度抽象、耦合爆炸 |

两者的平衡点在于 **Rule of Three**：

> 第一次：直接写代码  
> 第二次：容忍重复，但开始留意规律  
> 第三次：抽象重构，引入合适的模式

不等第三次就抽象的代价往往是"为了消除 5 行重复代码，引入了 100 行框架代码"。

### 6.3 模式选择的决策树

遇到设计问题时，按以下顺序思考：

1. **先问是否真的需要模式**——能不能用更简单的方式解决？
2. **确定问题类型**——是对象创建、结构组装，还是行为协作？
3. **寻找框架已经提供的机制**——Spring 可能已经用某种模式解决了你的问题
4. **参考速查表中的实践**——看看类似场景下优秀的框架是怎么做的
5. **保持简单**——能用组合就不用继承，能用一个类就不用三个

### 6.4 学习建议

对于后端工程师而言，学习设计模式的最佳路径是：

1. **不要抱着 GoF 书死磕**——先看框架源码，再回头对照模式定义
2. **从 Spring 源码入手**——阅读 `AbstractApplicationContext.refresh()` 理解模板方法，阅读 `DispatcherServlet.doDispatch()` 理解适配器，阅读 AOP 代理创建理解代理模式
3. **在 Code Review 中实践**——看到 5 层 if-else 时思考是否该用策略模式，看到重复的 try-catch 时思考是否该用模板方法
4. **记住模式是词汇表，不是教条**——代码中"用了什么模式"不重要，代码是否易于理解和维护才重要

最后，用一句话总结设计模式的本质：

> **设计模式不是为了让你多写代码，而是为了让你在正确的时机写出正确的抽象。** 如果在某个场景下一个简单的 if-else 就能清晰表达业务逻辑，那它就是一种好的设计——模式只是手段，清晰的代码才是目的。

---

*本文覆盖了后端开发中最高频的设计模式，每个模式都附带了框架源码级的实践案例。建议收藏作为速查参考，在 Code Review 或设计评审时对照翻阅。*
