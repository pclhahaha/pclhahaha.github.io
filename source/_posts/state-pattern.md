---
title: 状态模式 vs 策略模式
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 状态模式
  - 状态机
  - Spring State Machine
categories:
  - 设计模式
---

**意图**：允许对象在内部状态改变时改变它的行为，对象看起来好像修改了它的类。

**为什么需要状态模式？** 订单、工单、审批流等业务场景中，对象在不同状态下有不同的行为规则。最简单的实现方式是 `if-else` 判断状态，但当状态超过 5 个时，代码会变成不可维护的意大利面条。

**UML 简述**：Context 持有当前 State，State 对象决定何时流转到下一个状态。

**订单状态机实现**

```java
// 状态接口
public interface OrderState {
    void pay(OrderContext ctx);       // 支付
    void ship(OrderContext ctx);      // 发货
    void confirm(OrderContext ctx);   // 确认收货
    void cancel(OrderContext ctx);    // 取消
}

// 待支付状态
public class PendingState implements OrderState {
    @Override
    public void pay(OrderContext ctx) {
        System.out.println("支付成功");
        ctx.setState(new ShippedState());  // 状态流转
    }

    @Override
    public void ship(OrderContext ctx) {
        throw new IllegalStateException("待支付状态不能发货");
    }

    @Override
    public void cancel(OrderContext ctx) {
        System.out.println("订单已取消");
        ctx.setState(new CancelledState());
    }
    // confirm() 同理抛异常...
}

// 上下文——持有当前状态
public class OrderContext {
    private OrderState state = new PendingState();  // 初始状态

    public void setState(OrderState state) {
        this.state = state;
    }

    public void pay()    { state.pay(this); }
    public void ship()   { state.ship(this); }
    public void confirm(){ state.confirm(this); }
    public void cancel() { state.cancel(this); }
}
```

**状态模式 vs 策略模式的本质区别**

这是另一个常见混淆点。两者的类结构几乎相同，但意图完全不同：

| 维度 | 状态模式 | 策略模式 |
|------|---------|---------|
| 关注点 | 对象**内部状态**变化导致行为变化 | 对象**外部算法**的替换 |
| 谁决定切换 | 状态对象自己决定何时流转 | 由调用方/上下文决定使用哪个策略 |
| 切换频率 | 频繁，且有严格的转换规则 | 通常在初始化时选定，运行时很少变 |
| 相互感知 | 状态对象持有对上下文的引用 | 策略对象通常不知道上下文的存在 |

**Spring State Machine**：对于复杂流程，Spring 提供了专业的状态机框架：

```java
@Configuration
@EnableStateMachine
public class OrderStateMachineConfig
        extends StateMachineConfigurerAdapter<String, String> {
    @Override
    public void configure(StateMachineStateConfigurer<String, String> states) {
        states.withStates()
            .initial("PENDING")
            .states(new HashSet<>(Arrays.asList(
                "PENDING", "PAID", "SHIPPED", "COMPLETED", "CANCELLED")));
    }
    @Override
    public void configure(StateMachineTransitionConfigurer<String, String> transitions) {
        transitions
            .withExternal().source("PENDING").target("PAID").event("PAY")
            .and()
            .withExternal().source("PAID").target("SHIPPED").event("SHIP");
    }
}
```

---
