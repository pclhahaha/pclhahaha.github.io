---
title: 策略模式 — 消除 if-else
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 策略
  - Comparator
  - Spring
categories:
  - 设计模式
---

**意图**：定义一系列算法，把它们一个个封装起来，并且使它们可以相互替换。

策略模式是消除 `if-else` 的利器，也是后端业务代码中最实用的模式。

**UML 简述**：定义策略接口和多个实现，Context 持有策略引用，通过替换不同策略来改变行为。

**经典示例：支付策略**

```java
public interface PaymentStrategy { boolean pay(BigDecimal amount); }

@Component("ALIPAY")
public class AlipayPayment implements PaymentStrategy { /* 调用支付宝 */ }

@Component("WECHAT")  
public class WechatPayment implements PaymentStrategy { /* 调用微信支付 */ }

@Service
public class PaymentService {
    @Autowired
    private Map<String, PaymentStrategy> strategyMap; // Spring 自动注入所有实现

    public boolean pay(String channel, BigDecimal amount) {
        PaymentStrategy strategy = strategyMap.get(channel);
        if (strategy == null) throw new IllegalArgumentException("不支持的支付渠道: " + channel);
        return strategy.pay(amount);
    }
}
```

利用 Spring 的依赖注入，所有 `PaymentStrategy` 的实现类会自动注入到 `Map` 中（key 是 Bean 名称）。**新增支付渠道只需要加一个类，零修改已有代码** —— 这就是 OCP 的体现。

**JDK Comparator —— 策略模式的经典**

```java
// 策略：不同的比较规则
List<User> users = new ArrayList<>();

// 策略 A：按年龄排序
users.sort(Comparator.comparingInt(User::getAge));

// 策略 B：按姓名排序
users.sort(Comparator.comparing(User::getName));

// 策略 C：按创建时间倒序
users.sort(Comparator.comparing(User::getCreateTime).reversed());
```

`Comparator` 接口就是策略接口，每次 `sort()` 调用传入不同的 Lambda 就是不同的策略。

**Spring 中的策略模式——Resource 加载**

```java
// ResourceLoader 使用策略模式选择不同的 Resource 实现
ResourceLoader loader = new DefaultResourceLoader();

// classpath: → ClassPathResource
Resource res1 = loader.getResource("classpath:application.yml");

// file: → FileSystemResource
Resource res2 = loader.getResource("file:/opt/config/app.yml");

// http: → UrlResource
Resource res3 = loader.getResource("https://example.com/config.yml");
```

**策略模式 vs 工厂模式**

| 模式 | 关注点 | 返回 |
|------|--------|------|
| 策略 | 封装可互换的**行为/算法** | 通常无返回值，执行操作 |
| 工厂 | 封装可互换的**创建逻辑** | 返回创建好的对象 |

两者经常组合使用：工厂负责选择策略，策略负责执行逻辑。支付示例中的 `strategyMap.get(channel)` 就是最简单的工厂。

---
`
