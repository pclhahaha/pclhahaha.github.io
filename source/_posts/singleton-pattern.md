---
title: 单例模式 — DCL/枚举/Holder
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 单例
  - DCL
  - volatile
categories:
  - 设计模式
---

**意图**：保证一个类只有一个实例，并提供一个全局访问点。核心思路：构造器私有 + 静态方法返回唯一实例。

**实现方式对比**

| 方式 | 线程安全 | 懒加载 | 反序列化安全 | 反射安全 | 推荐度 |
|------|---------|--------|------------|---------|-------|
| 饿汉式 | ✓ | ✗ | ✗ | ✗ | ★★ |
| 懒汉式(synchronized) | ✓ | ✓ | ✗ | ✗ | ★★ |
| DCL + volatile | ✓ | ✓ | ✗ | ✗ | ★★★★ |
| 静态内部类 | ✓ | ✓ | ✗ | ✗ | ★★★★ |
| 枚举 | ✓ | ✗(类加载时) | ✓ | ✓ | ★★★★★ |

**1) DCL（双重检查锁定）+ volatile**

```java
public class DclSingleton {
    private static volatile DclSingleton instance; // volatile 防止指令重排序
    private DclSingleton() {}

    public static DclSingleton getInstance() {
        if (instance == null) {
            synchronized (DclSingleton.class) {
                if (instance == null) {
                    instance = new DclSingleton();
                }
            }
        }
        return instance;
    }
}
```

为什么需要 `volatile`？`new DclSingleton()` 不是原子操作，JVM 会将其分解为三步：
1. 分配内存空间
2. 初始化对象
3. 将引用指向内存地址

JIT 可能将步骤 2 和 3 重排序。如果线程 A 执行到步骤 3 但未执行步骤 2，线程 B 在第一次检查时发现 `instance != null`，直接返回一个未初始化完毕的对象——这就是著名的**指令重排序问题**。`volatile` 通过内存屏障禁止这种重排序。

**2) 静态内部类（Holder 模式）**

```java
public class HolderSingleton {
    private HolderSingleton() {}

    private static class Holder {
        private static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;  // 首次访问 Holder 类时才触发加载
    }
}
```

利用了 JVM 类加载机制的延迟初始化——`Holder` 类只有在 `getInstance()` 首次被调用时才会被加载和初始化，天然线程安全。

**3) 枚举（推荐方式）**

```java
public enum EnumSingleton {
    INSTANCE;

    public void doSomething() {
        // 业务逻辑
    }
}
```

枚举是 Java 中实现单例的最佳方式：
- 序列化安全：JVM 保证反序列化时不会创建新实例
- 反射安全：`Constructor.newInstance()` 源码中对枚举类型直接抛出 `IllegalArgumentException`
- 但注意：枚举实例在类加载时创建，不支持懒加载

**破坏与防御**

1. **反射破坏**：`setAccessible(true)` 调用私有构造器。防御：构造器中判断 instance 已存在则抛异常。
2. **序列化破坏**：反序列化时通过 `readResolve()` 返回已有实例。枚举天然免疫上述两种破坏。

**框架中的位置**

- Spring Bean 默认 Scope 是 `singleton`，IoC 容器中每个 BeanDefinition 对应唯一实例
- JDK `java.lang.Runtime` 是饿汉式单例
- Logback/Log4j2 的 LoggerFactory 内部使用单例

---
