---
title: MyBatis 一/二级缓存
date: 2026-07-05
updated: 2026-07-05
tags:
  - MyBatis
  - 缓存
  - PerpetualCache
  - CacheKey
categories:
  - Java
  - MyBatis
---

### 6.1 一级缓存（SqlSession 级别）

一级缓存默认开启且无法关闭，存储结构 `PerpetualCache`（本质 `HashMap<CacheKey, Object>`）。

```java
// BaseExecutor.query() — 一级缓存核心
List<E> list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
if (list != null) return list;
list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
```

**CacheKey 组成**：MappedStatement.id + RowBounds(offset,limit) + SQL 字符串 + 参数值。

**失效场景**：不同 SqlSession（不共享）、执行 INSERT/UPDATE/DELETE（`update()` 调 `clearLocalCache()`）、`flushCache="true"`、事务 `commit()`。

> **跨 SqlSession 脏读**：线程 A 的 SqlSession 缓存了 `user(id=1)`，线程 B 修改提交后，线程 A 再次查询从一级缓存取到旧值。这是与 Spring 集成时最常见的问题。

### 6.2 二级缓存：不推荐的原因

二级缓存是 namespace 级别，多个 SqlSession 共享。开启需两处配置：

```xml
<setting name="cacheEnabled" value="true"/>           <!-- mybatis-config.xml -->
<cache eviction="LRU" flushInterval="60000" size="512" readOnly="false"/> <!-- Mapper.xml -->
```

写入时机：`CachingExecutor` 暂存到 `TransactionalCache`，事务提交后才刷入正式 Cache，回滚则丢弃。

**不推荐生产的四个原因**：

1. `flushCache` 只能全量清空整个 namespace，做不到"更新 user(id=1) 只失效 user(id=1) 的缓存"
2. 多表关联场景：User 更新了但 JOIN Order 的缓存结果不会自动失效
3. `readOnly="false"` 通过序列化返回副本，大对象开销不可忽视
4. 多实例部署时缓存不同步，集成 Redis 又会引入分布式一致性复杂度

> **生产建议**：优先用独立缓存层（Redis/Caffeine），粒度可控、TTL 灵活。MyBatis 二级缓存仅适合极少变更的字典表或配置表。

### 6.3 自定义缓存

实现 `Cache` 接口即可集成外部缓存：

```java
public class RedisCache implements Cache {
    public void putObject(Object key, Object value) {
        jedis.setex(serializeKey(key), 300, serialize(value));
    }
    public Object getObject(Object key) {
        return deserialize(jedis.get(serializeKey(key)));
    }
    public Object removeObject(Object key) { return jedis.del(serializeKey(key)); }
}
```

```xml
<cache type="com.example.cache.RedisCache"/>
```

---
