---
title: 代理模式 — JDK动态代理 vs CGLIB
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 代理
  - JDK动态代理
  - CGLIB
  - AOP
categories:
  - 设计模式
---

**意图**：为另一个对象提供一个替身或占位符，以控制对原对象的访问。

**为什么重要**：代理是 Spring AOP 的底层基石，是理解 Spring 框架最关键的几个模式之一。

**UML 简述**：代理持有真实对象的引用，实现相同的接口，在调用前后添加额外逻辑。

**JDK 动态代理 vs CGLIB**

这是面试中的高频问题。两者的本质区别在于实现机制：

| 维度 | JDK 动态代理 | CGLIB 代理 |
|------|------------|-----------|
| 机制 | 基于接口，运行时生成 `$Proxy` 类实现目标接口 | 基于继承，运行时生成目标类的子类 |
| 要求 | 目标类必须实现接口 | 目标类不能是 final，方法不能是 final |
| 创建方式 | `Proxy.newProxyInstance()` | `Enhancer.create()` |
| invoke 方式 | `InvocationHandler.invoke()` | `MethodInterceptor.intercept()` |
| 性能 | JDK 6 后优化明显，性能接近 CGLIB | 早期比 JDK 代理快，现在差距很小 |
| Spring 默认 | 目标有接口时默认使用 | 目标无接口时使用 |

**JDK 动态代理示例**：

```java
public interface UserService {
    void save(User user);
}

public class UserServiceImpl implements UserService {
    @Override
    public void save(User user) {
        System.out.println("保存用户: " + user.getName());
    }
}

// 代理处理器
public class LogHandler implements InvocationHandler {
    private final Object target;

    public LogHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(">>> 开始调用: " + method.getName());
        long start = System.currentTimeMillis();
        Object result = method.invoke(target, args);
        long cost = System.currentTimeMillis() - start;
        System.out.println(">>> 调用完成: " + method.getName() + ", 耗时: " + cost + "ms");
        return result;
    }
}

// 创建代理
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{UserService.class},
    new LogHandler(new UserServiceImpl())
);
proxy.save(new User());  // 所有调用都会被 LogHandler 拦截
```

**Spring AOP 代理选择策略**

Spring 的代理选择是一个典型的"策略 + 工厂"组合：

```
如果目标类实现了接口 → 默认使用 JDK 动态代理
否则 → 使用 CGLIB 代理
Spring Boot 2.x 起默认使用 CGLIB（因为更少限制）
```

可以强制指定：`@EnableAspectJAutoProxy(proxyTargetClass = true)` —— 始终使用 CGLIB。

**AOP 核心链路**（以 `@Transactional` 为例）：

```
1. Bean 初始化后的 postProcessAfterInitialization 
   → 创建代理对象（JDK/CGLIB）
2. 调用代理对象的方法
   → TransactionInterceptor.invoke()
3. 开启事务 → invoke 目标方法 → 提交/回滚事务
4. 返回结果给调用者
```

理解这个过程是排查 AOP 失效问题的前提。最常见的问题场景：
- **同类方法调用**：`this.methodB()` 不经过代理，AOP 不生效
- **私有方法**：代理无法拦截 private 方法
- **static/final 方法**：CGLIB 无法代理

**代理模式的变体**

| 变体 | 用途 | 示例 |
|------|------|------|
| 远程代理 | 为远程对象提供本地代表 | RMI Stub, Feign 接口 |
| 虚拟代理 | 延迟加载开销大的对象 | Hibernate Lazy Loading |
| 保护代理 | 控制访问权限 | Spring Security 方法级权限 |
| 缓存代理 | 缓存计算结果 | `@Cacheable` |

---
