---
title: MyBatis 插件机制 (PageHelper 原理)
date: 2026-07-05
updated: 2026-07-05
tags:
  - MyBatis
  - 插件
  - PageHelper
  - Interceptor
categories:
  - Java
  - MyBatis
---

### 5.1 责任链 + 动态代理

MyBatis 插件通过 JDK 动态代理实现。多个插件依次嵌套形成责任链：

```
外部调用 → PluginB.invoke()
              → InterceptorB 前置 → PluginA.invoke()
                  → InterceptorA 前置 → 原始方法 → InterceptorA 后置
              → InterceptorB 后置
```

可拦截四个接口：`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`。

### 5.2 Plugin.wrap() 源码

```java
public class Plugin implements InvocationHandler {
    private final Object target;
    private final Interceptor interceptor;
    private final Map<Class<?>, Set<Method>> signatureMap;  // 从 @Signature 解析

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Set<Method> methods = signatureMap.get(method.getDeclaringClass());
        if (methods != null && methods.contains(method)) {
            return interceptor.intercept(new Invocation(target, method, args));  // ★ 拦截
        }
        return method.invoke(target, args);  // 透明透传
    }

    public static Object wrap(Object target, Interceptor interceptor) {
        Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
        Class<?>[] interfaces = getAllInterfaces(target.getClass(), signatureMap);
        if (interfaces.length > 0) {
            return Proxy.newProxyInstance(
                target.getClass().getClassLoader(), interfaces,
                new Plugin(target, interceptor, signatureMap));
        }
        return target;
    }
}
```

Executor 在 `Configuration.newExecutor()` 中包装，StatementHandler 在 `Configuration.newStatementHandler()` 中包装。

### 5.3 PageHelper 分页插件原理

拦截 `Executor.query()` 实现物理分页：

1. `PageHelper.startPage(pageNum, pageSize)` → ThreadLocal 存储分页参数
2. 插件拦截器检测到 ThreadLocal 有 Page → **不直接放行**
3. 先执行 COUNT 查询获取总记录数
4. 解析原始 SQL，按方言生成分页 SQL（MySQL: `LIMIT offset,size`，Oracle: `ROWNUM`）
5. 反射替换 `BoundSql` 中的 SQL 文本 → 调 `invocation.proceed()` 执行
6. 封装 `PageInfo` 返回

> **ThreadLocal 陷阱**：`startPage()` 后必须紧跟 Mapper 方法，否则参数可能被同线程后续查询误消费。

### 5.4 自定义插件：慢 SQL 告警

```java
@Intercepts(@Signature(type = Executor.class, method = "query",
    args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}))
public class SlowSqlPlugin implements Interceptor {
    private long threshold;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long start = System.currentTimeMillis();
        try { return invocation.proceed(); }
        finally {
            long elapsed = System.currentTimeMillis() - start;
            if (elapsed > threshold) {
                MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
                log.warn("慢SQL: [{}] 耗时{}ms", ms.getId(), elapsed);
            }
        }
    }
    public Object plugin(Object target) { return Plugin.wrap(target, this); }
    public void setProperties(Properties props) {
        this.threshold = Long.parseLong(props.getProperty("threshold", "1000"));
    }
}
```

---
