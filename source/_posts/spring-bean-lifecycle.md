---
title: Spring Bean 完整生命周期
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - Spring
  - Bean生命周期
  - BeanPostProcessor
categories:
  - Java
  - Spring
---

Spring Bean 的生命周期是 Spring 框架最核心的概念之一。从 Bean 的定义加载，到实例化、属性填充、初始化，再到最终销毁，Spring 在每个阶段都提供了扩展点。理解这些扩展点，是在 Spring 框架中做深度定制的关键。

## 一、生命周期全景

```
1. BeanDefinition 加载（XML/@Component/@Bean）
2. BeanFactoryPostProcessor 处理（修改 BeanDefinition）
3. 实例化（反射调用构造器）
4. 属性填充（@Autowired 注入）
5. Aware 回调（BeanNameAware → BeanFactoryAware → ApplicationContextAware）
6. BeanPostProcessor.postProcessBeforeInitialization()
7. @PostConstruct / InitializingBean.afterPropertiesSet()
8. init-method（XML 中指定或 @Bean 的 initMethod 属性）
9. BeanPostProcessor.postProcessAfterInitialization() ← AOP 在此处生成代理
10. Bean 就绪，可被使用
11. @PreDestroy / DisposableBean.destroy()
12. destroy-method（XML 中指定或 @Bean 的 destroyMethod 属性）
```

## 二、关键扩展点详解

### 2.1 BeanFactoryPostProcessor

在 Bean 实例化之前修改 BeanDefinition：

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 可以修改任意 Bean 的定义元数据
        BeanDefinition bd = beanFactory.getBeanDefinition("userService");
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
    }
}
```

Spring 内部的关键实现是 `ConfigurationClassPostProcessor`——它扫描 `@Configuration` 类，解析 `@Bean` 方法，并注册相应的 BeanDefinition。这是 Spring 能识别 `@Configuration` 注解的基础。

### 2.2 BeanPostProcessor

在 Bean 实例化后、初始化前后进行拦截，是 Spring 最强大的扩展机制。

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof MyService) {
            System.out.println(beanName + " 初始化前");
        }
        return bean;  // 可以返回原始 bean 或包装后的 bean
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(MyAnnotation.class)) {
            // 为特定 Bean 创建代理
            return Proxy.newProxyInstance(...);
        }
        return bean;
    }
}
```

Spring AOP 的实现正是基于 `BeanPostProcessor`——`AnnotationAwareAspectJAutoProxyCreator` 在 `postProcessAfterInitialization` 中为目标 Bean 生成代理对象。

### 2.3 Aware 接口

Aware 接口让 Bean 可以注入 Spring 容器内部对象：

```java
@Component
public class MyService implements ApplicationContextAware, BeanNameAware {

    private ApplicationContext ctx;
    private String beanName;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
        // 现在可以通过 ctx 获取其他 Bean
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        // beanName = "myService"
    }
}
```

常见的 Aware 接口：

| Aware 接口 | 注入对象 | 用途 |
|-----------|----------|------|
| BeanNameAware | Bean 的名称 | 记录日志、标识 |
| BeanFactoryAware | BeanFactory | 手动获取 Bean |
| ApplicationContextAware | ApplicationContext | 获取上下文、发布事件 |
| EnvironmentAware | Environment | 读取环境配置 |
| ResourceLoaderAware | ResourceLoader | 加载资源文件 |

## 三、初始化与销毁

### 3.1 三种初始化方式

| 方式 | 示例 | 优先级 | 推荐度 |
|------|------|--------|--------|
| `@PostConstruct` | 方法上标注 | 最先执行 | 推荐 |
| `InitializingBean.afterPropertiesSet()` | 实现接口 | 其次 | 不推荐（耦合 Spring） |
| `init-method` | XML/@Bean 属性 | 最后 | 推荐（解耦） |

```java
@Component
public class MyService {
    
    @PostConstruct
    public void init() {
        System.out.println("1. @PostConstruct 执行");
    }
}

// 或者
@Bean(initMethod = "init")
public MyService myService() {
    return new MyService();
}
```

三种方式的执行顺序：`@PostConstruct` → `afterPropertiesSet()` → `init-method`。

### 3.2 三种销毁方式

| 方式 | 优先级 |
|------|--------|
| `@PreDestroy` | 最先 |
| `DisposableBean.destroy()` | 其次 |
| `destroy-method` | 最后 |

Spring 容器优雅关闭时会调用这些销毁回调。

## 四、循环依赖与三级缓存

### 4.1 什么是循环依赖

```java
@Component
public class A {
    @Autowired private B b;
}

@Component
public class B {
    @Autowired private A a;
}
```

Spring 通过**三级缓存**解决单例 Bean 的循环依赖：

```
singletonObjects               ← 一级缓存：完全创建好的 Bean
earlySingletonObjects           ← 二级缓存：提前暴露的 Bean 引用
singletonFactories              ← 三级缓存：生成提前引用的工厂（ObjectFactory）
```

### 4.2 解决流程

```
1. 创建 A → 放入三级缓存（singletonFactories）
2. 填充 A 的属性 → 发现需要 B → 去创建 B
3. 创建 B → 放入三级缓存
4. 填充 B 的属性 → 发现需要 A → 从三级缓存中获取 A 的提前引用（ObjectFactory.getObject()）
   → 将 A 移入二级缓存（earlySingletonObjects）
5. B 创建完成 → 放入一级缓存
6. 继续填充 A 的属性（现在 B 可用了）
7. A 创建完成 → 放入一级缓存
```

### 4.3 为什么需要三级缓存

一级缓存（singletonObjects）存储完全初始化完成的 Bean；二级缓存（earlySingletonObjects）存储提前暴露但未完成的 Bean引用；三级缓存（singletonFactories）存储能生成提前Bean引用的工厂。三级缓存中存的是 `ObjectFactory` 而非直接的 Bean 引用，是为了支持 AOP——在获取提前引用时，`ObjectFactory.getObject()` 可以先检查是否需要为这个 Bean 生成代理，如果需要，返回代理对象而非原始对象。这正是 AOP 和循环依赖可以共存的保障。

### 4.4 限制条件

- 只有单例 Bean 支持循环依赖（prototype 不行）
- 构造器注入无法解决循环依赖（因为构造器执行时 Bean 还未暴露）
- 建议尽量避免循环依赖——它通常暗示设计问题

## 五、小结

Spring Bean 的生命周期是一个精心设计的扩展架构——从 BeanDefinition 加载到最终的销毁，每个阶段都有对应的扩展点。BeanFactoryPostProcessor 修改元数据，BeanPostProcessor 拦截 Bean 实例，Aware 注入容器资源，三级缓存解决循环依赖——这些机制共同构成了 Spring 高度可扩展的 IOC 容器骨架。理解这张"扩展地图"，是在 Spring 上做框架级定制的第一课。
