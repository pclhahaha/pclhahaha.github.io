---
title: Spring AOP — JDK动态代理 vs CGLIB
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - Spring
  - AOP
  - JDK动态代理
  - CGLIB
categories:
  - Java
  - Spring
---

Spring AOP 的底层实现基于动态代理。Spring 会根据目标对象是否实现了接口，自动选择 JDK 动态代理或 CGLIB 代理。理解两者的区别和 Spring 的选择逻辑，是深入 Spring AOP 的基础。

## 一、JDK 动态代理

JDK 动态代理是 Java 原生的代理机制，基于接口实现。

### 1.1 工作原理

```
客户端 → [Proxy (implements Interface)] → [InvocationHandler] → [真实对象]
```

核心 API：

```java
// 创建代理对象
MyService proxy = (MyService) Proxy.newProxyInstance(
    target.getClass().getClassLoader(),   // 类加载器
    target.getClass().getInterfaces(),    // 目标接口列表
    new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) 
            throws Throwable {
            System.out.println("前置增强");
            Object result = method.invoke(target, args);  // 调用真实方法
            System.out.println("后置增强");
            return result;
        }
    }
);
```

### 1.2 特点

| 特点 | 说明 |
|------|------|
| 必须实现接口 | 只能代理实现了至少一个接口的类 |
| 性能 | 反射调用，略慢于 CGLIB |
| JDK 内置 | 无需第三方依赖 |
| 代理对象类型 | `$Proxy0 extends Proxy implements MyService` |

### 1.3 底层原理

```java
// ProxyGenerator 生成的字节码（简化版）
public final class $Proxy0 extends Proxy implements MyService {
    private static Method m3; // MyService.doSomething()

    public void doSomething() {
        h.invoke(this, m3, null);  // h = InvocationHandler
    }
}
```

`Proxy.newProxyInstance()` 在运行时动态生成一个实现了目标接口的代理类，该类持有 `InvocationHandler` 引用，所有方法调用都转发到 `InvocationHandler.invoke()`。

## 二、CGLIB 代理

CGLIB（Code Generation Library）是基于字节码生成的代理方式，可以代理没有实现接口的类。

### 2.1 工作原理

```
客户端 → [Proxy (extends Target)] → [MethodInterceptor] → [真实对象]
```

核心 API：

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(TargetClass.class);           // 设置父类
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, 
                           MethodProxy proxy) throws Throwable {
        System.out.println("前置增强");
        Object result = proxy.invokeSuper(obj, args);  // 调用父类方法
        System.out.println("后置增强");
        return result;
    }
});
TargetClass proxy = (TargetClass) enhancer.create();
```

### 2.2 特点

| 特点 | 说明 |
|------|------|
| 不需要接口 | 可以代理任何非 final 类 |
| final 方法限制 | 不能代理 final 方法（JVM 不允许重写） |
| 性能 | 使用 FastClass 机制，比 JDK 反射快 |
| 额外依赖 | 需要 cglib 和 asm 库 |
| 代理对象类型 | `TargetClass$$EnhancerByCGLIB$$xxxx` |

### 2.3 底层原理

CGLIB 使用 ASM 字节码框架，在运行时生成目标类的子类：

```java
// CGLIB 生成的字节码（简化版）
public class TargetClass$$EnhancerByCGLIB$$xxxx extends TargetClass {
    private MethodInterceptor interceptor;  // 回调

    @Override
    public void doSomething() {
        // 调用 interceptor.intercept()
        // interceptor 再决定是否调用 super.doSomething()
    }
}
```

CGLIB 的 FastClass 机制避免了反射：通过为每个方法分配一个 index，调用时通过 switch-case 分发，比反射快 3-5 倍。

## 三、Spring 代理选择策略

### 3.1 Spring Boot 2.x 之后的默认行为

| 场景 | Spring 使用的代理 |
|------|------------------|
| 目标类实现了接口 | **JDK 动态代理** |
| 目标类未实现接口 | **CGLIB** |
| `proxyTargetClass = true` | 强制使用 **CGLIB** |
| Spring Boot 2.0+ 默认 | **CGLIB**（通过 `spring.aop.proxy-target-class=true`） |

在 Spring Boot 2.0 之前，默认使用 JDK 动态代理；2.0 之后改为 CGLIB。

### 3.2 切换代理方式

```yaml
# application.yml
spring:
  aop:
    proxy-target-class: true   # true = CGLIB, false = JDK Proxy（但 AOP 需要接口）
```

### 3.3 AOP 失效的常见场景

**场景 1：内部方法调用（self-invocation）**

```java
@Service
public class UserService {
    @Transactional
    public void methodA() {
        this.methodB();  // 直接调用，不走代理！
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // 事务不起作用！
    }
}
```

原因：`this.methodB()` 是直接调用，绕过了代理对象。解决方法：注入自身、使用 `AopContext.currentProxy()`、或将 methodB 移到另一个 Service。

**场景 2：非 public 方法**

```java
@Service
public class UserService {
    @Transactional
    private void privateMethod() { }  // Spring AOP 不支持 private 方法
}
```

JDK 动态代理要求方法在接口中（天然 public），CGLIB 也无法代理 private 方法（字节码层面的限制）。

**场景 3：final 类或 final 方法**

```java
@Service
public final class UserService {  // CGLIB 无法继承 final 类
    @Transactional
    public void method() { }
}
```

CGLIB 通过生成子类实现代理，final 类无法被继承。

## 四、性能对比

一般场景下两者差异很小，以下是约百万次调用的对比：

| 指标 | JDK 动态代理 | CGLIB |
|------|------------|-------|
| 创建代理（首次） | 快 | 慢（生成字节码） |
| 方法调用（后续） | 略慢（反射） | 快（FastClass 索引） |
| 内存占用 | 小 | 较大（代理类对象） |

在 Spring 的实际使用中，这个性能差异微乎其微——AOP 主要用于事务管理、日志记录等横切关注点，这些操作本身的耗时远大于代理调用的开销。

## 五、小结

Spring AOP 的两种代理模式各有优势：JDK 动态代理基于接口，无需第三方依赖，实现更纯净；CGLIB 基于继承，更灵活但有一些限制（final 类/方法）。Spring Boot 2.0 后默认使用 CGLIB，因为实际开发中大部分 Service 类并不专门定义接口。理解 AOP 的代理机制不仅能帮你写出正确的代码，也能在遇到"事务失效"、"切面不生效"等问题时快速定位根因。
