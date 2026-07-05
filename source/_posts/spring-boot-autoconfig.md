---
title: Spring Boot 自动配置原理
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - Spring Boot
  - 自动配置
categories:
  - Java
  - Spring
---

Spring Boot 最核心的特性是"自动配置"——你引入 `spring-boot-starter-web`，Web 环境就自动配置好了；引入 `spring-boot-starter-data-redis`，Redis 连接自动就绪。这一切的背后的机制并不复杂：注解驱动 + SPI 配置。

## 一、@SpringBootApplication 入口

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@SpringBootApplication` 是一个组合注解，包含三个核心注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@SpringBootConfiguration    // 等同于 @Configuration
@EnableAutoConfiguration    // 自动配置的核心
@ComponentScan(excludeFilters = { ... })  // 组件扫描
public @interface SpringBootApplication { }
```

其中 `@EnableAutoConfiguration` 是自动配置的入口。

## 二、@EnableAutoConfiguration 的工作原理

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration { }
```

关键在 `@Import(AutoConfigurationImportSelector.class)`。`AutoConfigurationImportSelector` 实现了 `DeferredImportSelector` 接口，在 Spring 容器 refresh 过程中被调用。

其核心逻辑：

```java
// AutoConfigurationImportSelector.getCandidateConfigurations()
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, 
                                                   AnnotationAttributes attributes) {
    // 1. 读取 META-INF/spring/xxx.imports 文件（Spring Boot 3.x）
    //    或 META-INF/spring.factories 文件（Spring Boot 2.x）
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    
    // 2. 去重、排序（@AutoConfigureOrder, @AutoConfigureBefore/After）
    // 3. 按 @Conditional 过滤
    return configurations;
}
```

## 三、spring.factories 与 xxx.imports

### Spring Boot 2.x：`META-INF/spring.factories`

```properties
# spring-boot-autoconfigure.jar!/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
...
```

每个 starter 在自己的 `spring.factories` 文件中声明它提供的自动配置类。Spring Boot 启动时加载所有 jar 中的这个文件，汇总所有自动配置类名。

### Spring Boot 3.x：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

Spring Boot 3.x 改用更简洁的格式：

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

每行一个自动配置类全限定名，不再需要 Key-Value 格式。

## 四、条件注解——精准控制配置生效时机

自动配置类不一定会生效。每个自动配置类上都标注了 `@ConditionalOnXxx` 注解，只有当条件满足时才会被加载：

| 注解 | 行为 | 典型场景 |
|------|------|----------|
| `@ConditionalOnClass` | 类路径中有特定类 | `RedisAutoConfiguration` 要求 `RedisConnection` 类存在 |
| `@ConditionalOnMissingBean` | 容器中没有该类型的 Bean | 用户可以自己定义 Bean 覆盖默认配置 |
| `@ConditionalOnProperty` | 配置文件中存在特定属性 | `spring.redis.enabled=true` |
| `@ConditionalOnBean` | 容器中存在该类型的 Bean | 只在某个 Bean 已注册时才生效 |
| `@ConditionalOnJava` | Java 版本匹配 | 特定版本优化的配置 |

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)                 // classpath 有 Redis
@EnableConfigurationProperties(RedisProperties.class)      // 绑定配置属性
@ConditionalOnMissingBean(name = "redisTemplate")          // 用户未自定义
public class RedisAutoConfiguration {

    @Bean
    public RedisTemplate<Object, Object> redisTemplate(...) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

这个自动配置类的生效条件是：classpath 中存在 `RedisOperations` 类（即用户引入了 redis starter）、用户没有自定义 `redisTemplate` Bean。

## 五、自定义 Starter

创建一个自定义 Starter 只需要三步：

```
my-spring-boot-starter/
├── pom.xml
├── src/main/java/
│   └── com/example/
│       ├── MyAutoConfiguration.java       # 自动配置类
│       └── MyProperties.java              # 配置属性
└── src/main/resources/
    └── META-INF/spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

**Step 1：定义配置属性**

```java
@ConfigurationProperties(prefix = "myapp")
public class MyProperties {
    private String name = "default";
    private int timeout = 5000;
    // getters and setters
}
```

**Step 2：编写自动配置类**

```java
@AutoConfiguration
@EnableConfigurationProperties(MyProperties.class)
@ConditionalOnProperty(prefix = "myapp", name = "enabled", matchIfMissing = true)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getName(), properties.getTimeout());
    }
}
```

**Step 3：注册到自动配置**

在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中添加一行：

```
com.example.MyAutoConfiguration
```

完成。用户在 `application.yml` 中就可以配置了：

```yaml
myapp:
  name: production
  timeout: 3000
```

## 六、自动配置执行顺序

Spring Boot 在 `AbstractApplicationContext.refresh()` 过程中处理自动配置：

```
1. invokeBeanFactoryPostProcessors(beanFactory)
   └─ ConfigurationClassPostProcessor 处理 @Configuration 类
       └─ AutoConfigurationImportSelector 加载所有自动配置类
           │
2. 过滤：按 @Conditional 条件筛选，只保留满足条件的配置类
           │
3. 排序：按 @AutoConfigureOrder 和 @AutoConfigureBefore/After 排序
           │
4. 加载：将选中的 @Configuration 类中的 @Bean 方法注册到容器
           │
5. 用户自定义配置覆盖自动配置（条件注解 + Order 保证）
```

## 七、小结

Spring Boot 自动配置的原理可以归纳为："约定大于配置 + SPI 机制 + 条件注解"：

- **约定大于配置**：引入 starter → 自动获得最佳实践配置
- **SPI 机制**：`spring.factories` / `.imports` 文件让所有 jar 声明自己能提供的配置
- **条件注解**：`@ConditionalOnXxx` 保证只在合适的场景下启用配置
- **可覆盖**：用户自定义的 Bean 优先于自动配置的 Bean

理解了这三个机制，你就理解了为什么 Spring Boot 能"开箱即用"——它并没有魔法，只是精心设计了配置的发现、筛选和加载流程。
