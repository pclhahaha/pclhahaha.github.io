---
title: MyBatis Mapper 动态代理
date: 2026-07-05
updated: 2026-07-05
tags:
  - MyBatis
  - 动态代理
  - MapperProxy
categories:
  - Java
  - MyBatis
---

```java
public class MapperRegistry {
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

    public <T> void addMapper(Class<T> type) {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        new MapperAnnotationBuilder(config, type).parse();  // 解析 @Select 等注解
    }

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return knownMappers.get(type).newInstance(sqlSession);
    }
}

public class MapperProxyFactory<T> {
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

    public T newInstance(SqlSession sqlSession) {
        return (T) Proxy.newProxyInstance(
            mapperInterface.getClassLoader(),
            new Class<?>[]{mapperInterface},
            new MapperProxy<>(sqlSession, mapperInterface, methodCache));
    }
}
```
