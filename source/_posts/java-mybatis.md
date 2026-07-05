---
title: MyBatis 源码解析
date: 2025-01-01 00:00:00
updated: 2025-01-01 00:00:00
tags:
  - MyBatis
  - ORM
  - 源码分析
  - Java
categories:
  - Java
---

[TOC]

## 一、ORM 概述

### 1.1 JDBC 痛点

传统 JDBC 编程存在四个核心痛点：代码重复（每次操作都要获取连接、创建 Statement、处理 ResultSet、关闭资源）、硬编码 SQL（SQL 散落在 Java 代码中，改一行 SQL 需重新编译部署）、手动结果映射（逐字段调用 get/set）、资源管理繁琐（Connection/Statement/ResultSet 关闭必须写在 finally 中）。

### 1.2 MyBatis vs Hibernate vs JPA

| 维度 | MyBatis | Hibernate / JPA |
|------|---------|-----------------|
| 定位 | SQL Mapping 框架 | 全自动 ORM |
| SQL 控制 | 开发者完全掌控，支持动态 SQL | HQL/JPQL 自动生成 |
| 学习曲线 | 低，会写 SQL 即可 | 高，需理解 Session/缓存/懒加载 |
| 性能调优 | 直接优化 SQL，DBA 友好 | SQL 不可见，易产生 N+1 |
| 动态 SQL | 强大的 XML 标签（if/choose/foreach） | Criteria API，表达能力偏弱 |
| 适用场景 | 复杂查询、报表、遗留数据库 | 标准 CRUD、领域模型驱动 |

MyBatis 在国内互联网公司占据主流，核心原因：互联网业务**查询多变、报表复杂、性能敏感**。当 DBA 拿着慢查询 SQL 沟通时，MyBatis 开发者可立刻定位到具体 Mapper.xml。

> **选型建议**：SQL 复杂度高、性能要求苛刻 → MyBatis。标准 CRUD + DDD → JPA/Hibernate 或 MyBatis-Plus。

---

## 二、MyBatis 核心架构

### 2.1 功能架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                        接口层 (API)                               │
│   SqlSession (增删改查/事务)    Mapper 接口 (动态代理)              │
└────────────┬──────────────────────────────────┬──────────────────┘
             │                                  │
┌────────────▼──────────────────────────────────▼──────────────────┐
│                      核心处理层                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ Config Parser│  │ SQL 解析引擎 │  │ SQL 执行引擎            │  │
│  │ XMLConfigBld│  │ SqlNode Tree │  │ Executor               │  │
│  │ XMLMapperBld│  │ DynamicSqlSrc│  │ StatementHandler       │  │
│  └──────┬───────┘  └──────┬───────┘  │ ParameterHandler       │  │
│         │                 │          │ ResultSetHandler       │  │
│         └────────┬────────┘          └───────────┬────────────┘  │
│                  ▼                               ▼               │
│           MappedStatement (SqlCommand + SqlSource + ResultMap)   │
└────────────┬──────────────────────────────────┬──────────────────┘
             │                                  │
┌────────────▼──────────────────────────────────▼──────────────────┐
│                     基础支撑层                                     │
│   连接池 (Pooled/Unpooled)    事务管理 (Jdbc/Managed)             │
│   缓存 (一级/二级)            反射工具箱 (Reflector/MetaObject)     │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 调用链与启动入口

```
SqlSessionFactoryBuilder
  └─► SqlSessionFactory (DefaultSqlSessionFactory)
        └─► SqlSession → Executor → StatementHandler
              ├── ParameterHandler (TypeHandler)
              └── ResultSetHandler
```

启动入口为 `SqlSessionFactoryBuilder.build(InputStream)`：

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());  // parser.parse() 返回 Configuration
}
```

`XMLConfigBuilder` 继承 `BaseBuilder`，使用 XPath 逐节点解析，解析顺序：properties → settings → typeAliases → typeHandlers → objectFactory → plugins → environments → databaseIdProvider → mappers。

```java
private void parseConfiguration(XNode root) {
    propertiesElement(root.evalNode("properties"));           // 1. 属性
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomLogImpl(settings);                              // 2. 日志
    typeAliasesElement(root.evalNode("typeAliases"));         // 3. 别名
    pluginElement(root.evalNode("plugins"));                  // 4. 插件
    settingsElement(settings);                                // 5. 全局设置
    environmentsElement(root.evalNode("environments"));       // 6. 数据源+事务
    typeHandlerElement(root.evalNode("typeHandlers"));        // 7. 类型处理器
    mapperElement(root.evalNode("mappers"));                  // 8. Mapper 注册 ★
}
```

---

## 三、初始化流程

### 3.1 Mapper.xml 解析

`mapperElement()` 支持四种注册方式：resource（XML 路径）、url（远程 XML）、class（接口类名）、package（包扫描）。最终委托给 `XMLMapperBuilder.parse()`：

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
        configurationElement(parser.evalNode("/mapper"));
        configuration.addLoadedResource(resource);
        bindMapperForNamespace();  // namespace ↔ Mapper 接口 绑定
    }
    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
}

private void configurationElement(XNode context) {
    String namespace = context.getStringAttribute("namespace");
    builderAssistant.setCurrentNamespace(namespace);
    cacheRefElement(context.evalNode("cache-ref"));       // 缓存引用
    cacheElement(context.evalNode("cache"));              // 二级缓存配置
    resultMapElements(context.evalNodes("/mapper/resultMap"));  // ★ 结果映射定义
    sqlElement(context.evalNodes("/mapper/sql"));         // ★ SQL 片段 (<include>)
    buildStatementFromContext(                             // ★ CRUD 语句
        context.evalNodes("select|insert|update|delete"));
}
```

### 3.2 MappedStatement 构建

每个 `<select>`/`<insert>`/`<update>`/`<delete>` 由 `XMLStatementBuilder` 解析为一个 `MappedStatement`，注册到 `Configuration.mappedStatements`（`Map<String, MappedStatement>`），key 为 `namespace.id`。

SQL 经 `XMLLanguageDriver` 处理生成 `SqlSource`：静态 SQL → `StaticSqlSource`，纯 `#{}` SQL → `RawSqlSource`，含动态标签 → `DynamicSqlSource`。

```java
public final class MappedStatement {
    private String id;                      // namespace.statementId
    private SqlSource sqlSource;            // ★ SQL 来源
    private SqlCommandType sqlCommandType;  // SELECT/INSERT/UPDATE/DELETE
    private StatementType statementType;    // STATEMENT/PREPARED/CALLABLE
    private List<ResultMap> resultMaps;     // ★ 结果映射
    private Cache cache;                    // 二级缓存
    private boolean flushCacheRequired;
    private boolean useCache;
    private KeyGenerator keyGenerator;      // 主键生成策略
}
```

---

## 四、执行流程：一条 SQL 的完整旅程

### 4.1 完整时序图

```
Mapper 接口调用
  │
  ▼
MapperProxy.invoke()                         ← JDK 动态代理拦截
  │
  ├── Object 方法 → 直接执行
  └── Mapper 方法 → MapperMethod.execute()
       │
       ├── INSERT/UPDATE/DELETE → sqlSession.insert/update/delete()
       └── SELECT → sqlSession.selectOne()/selectList()
            │
            ▼
DefaultSqlSession → 获取 MappedStatement → 委托 Executor
            │
            ▼
CachingExecutor (装饰器)
  ├── 生成 CacheKey → 查二级缓存
  │     ├── 命中 → 返回
  │     └── 未命中 → delegate.query()
  │
  ▼
BaseExecutor (Simple/Reuse/Batch)
  ├── 查一级缓存 (localCache / PerpetualCache)
  │     ├── 命中 → 返回
  │     └── 未命中 → queryFromDatabase()
  │
  ▼
doQuery()
  ├── StatementHandler (Routing → PreparedStatementHandler)
  ├── ParameterHandler.setParameters()      ← TypeHandler 设置 ? 参数
  ├── PreparedStatement.execute()
  └── ResultSetHandler.handleResultSets()   ← 结果映射 → List<T>
```

### 4.2 Executor 执行器体系

```
                     Executor (接口)
                    /          \
        BaseExecutor(抽象类)  CachingExecutor(装饰器)
        /    |     \              │
   Simple  Reuse  Batch        delegate
```

- **SimpleExecutor**（默认）：每次执行创建新 Statement，用完即关
- **ReuseExecutor**：`Map<String, Statement>` 缓存，同一 SQL 复用 Statement
- **BatchExecutor**：批量 INSERT/UPDATE/DELETE，`stmt.addBatch()` 累积，`flushStatements()` 批量提交
- **CachingExecutor**：装饰器模式，在 BaseExecutor 外包裹二级缓存逻辑

```java
// CachingExecutor.query() — 二级缓存装饰
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds,
        ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            List<E> list = (List<E>) cache.getObject(key);
            if (list == null) {
                list = delegate.query(ms, parameter, rowBounds, resultHandler, key, boundSql);
                cache.putObject(key, list);
            }
            return list;
        }
    }
    return delegate.query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

### 4.3 StatementHandler 体系

```
StatementHandler (接口)
    │
RoutingStatementHandler (路由器)
   /        |          \
Simple    Prepared    Callable    ← 按 statementType 路由
```

`PreparedStatementHandler` 是最常用实现：

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.handleResultSets(ps);  // ★ 结果处理
}
```

### 4.4 ParameterHandler 与 TypeHandler

`DefaultParameterHandler.setParameters()` 遍历 `BoundSql` 中的 `ParameterMapping` 列表，通过 `TypeHandler` 将 Java 对象转换为 JDBC 参数：

```java
public void setParameters(PreparedStatement ps) {
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping mapping = parameterMappings.get(i);
        Object value;
        // 简单类型直接使用，复杂对象通过 MetaObject 反射取值
        if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
        } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(mapping.getProperty());
        }
        mapping.getTypeHandler().setParameter(ps, i + 1, value, mapping.getJdbcType());
    }
}
```

**TypeHandler** 是 Java 类型 ↔ JDBC 类型的桥梁，可自定义枚举映射：

```java
public class SexEnumTypeHandler extends BaseTypeHandler<SexEnum> {
    public void setNonNullParameter(PreparedStatement ps, int i, SexEnum param, JdbcType jt)
            throws SQLException { ps.setInt(i, param.getCode()); }
    public SexEnum getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return SexEnum.getEnumByCode(rs.getInt(columnName));
    }
}
```

### 4.5 ResultSetHandler 结果映射

`DefaultResultSetHandler` 是 MyBatis 最复杂的类（主方法超 1000 行）：

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    List<Object> multipleResults = new ArrayList<>();
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    int resultSetCount = 0;
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    while (rsw != null && resultMaps.size() > resultSetCount) {
        handleResultSet(rsw, resultMaps.get(resultSetCount), multipleResults, null);
        rsw = getNextResultSet(stmt);
    }
    return collapseSingleResultList(multipleResults);
}
```

**嵌套查询 vs 嵌套结果**：

```xml
<!-- 嵌套结果: 一次 JOIN, 内存组装, 无 N+1 -->
<resultMap id="userWithOrders" type="User">
    <id property="id" column="id"/>
    <collection property="orders" ofType="Order">
        <id property="id" column="order_id"/>
        <result property="amount" column="amount"/>
    </collection>
</resultMap>

<!-- 嵌套查询: N+1 问题！每行主数据执行一次子查询 -->
<resultMap id="userWithOrders" type="User">
    <collection property="orders" column="id"
                select="com.example.OrderMapper.selectByUserId"/>
</resultMap>
```

懒加载（`lazyLoadingEnabled=true`）使用 CGLIB 代理，关联对象首次访问才触发子查询，缓解 N+1 但未根除。

---

## 五、插件机制

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

## 六、缓存机制

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

## 七、动态 SQL

### 7.1 OGNL 引擎与 SqlNode 树

动态 SQL 的条件判断（`<if test="...">`）使用 OGNL 表达式引擎，`OgnlCache` 缓存解析后的表达式树：

```java
public class IfSqlNode implements SqlNode {
    private final String test;  // "name != null and name != ''"
    private final SqlNode contents;

    public boolean apply(DynamicContext context) {
        if (evaluator.evaluateBoolean(test, context.getBindings())) {
            contents.apply(context);  // 拼接 SQL 片段
            return true;
        }
        return false;
    }
}
```

动态标签→SqlNode 映射：

| 标签 | SqlNode |
|------|---------|
| `<if>` | IfSqlNode |
| `<choose><when><otherwise>` | ChooseSqlNode |
| `<trim>`/`<where>`/`<set>` | TrimSqlNode / 子类 |
| `<foreach>` | ForEachSqlNode |
| `<bind>` | VarDeclSqlNode |
| 文本/组合 | StaticTextSqlNode / MixedSqlNode |

`<where>` 和 `<set>` 继承自 `TrimSqlNode`：

```java
public class WhereSqlNode extends TrimSqlNode {
    private static List<String> prefixList = Arrays.asList("AND ","OR ",...);
    public WhereSqlNode(Configuration conf, SqlNode contents) {
        super(conf, contents, "WHERE", prefixList, null, null);  // 去子句前导 AND/OR
    }
}

public class SetSqlNode extends TrimSqlNode {
    public SetSqlNode(Configuration conf, SqlNode contents) {
        super(conf, contents, "SET", null, Arrays.asList(","), null);  // 去子句末尾逗号
    }
}
```

`<foreach>` 最复杂：遍历集合元素，为每个元素绑定变量名并生成唯一占位符（`#{__frch_item_0}`），通过 open/close/separator 控制 SQL 拼接。

### 7.2 `#{}` vs `${}` 深度解析

**`#{}` 预编译占位符** — `GenericTokenParser` 将 `#{xxx}` 替换为 `?`，参数通过 `PreparedStatement.setXxx()` 设置，无 SQL 注入风险：

```
SQL: SELECT * FROM user WHERE id = #{id} AND name = #{name}
  → SELECT * FROM user WHERE id = ? AND name = ?
  → ps.setInt(1, id); ps.setString(2, name);
```

**`${}` 字符串直接替换** — XML 解析阶段直接拼入 SQL，不做转义：

```
SQL: SELECT * FROM ${tableName}
  → SELECT * FROM user (假设 tableName="user")
```

> **安全铁律**：用户输入一律用 `#{}`。`${}` 仅用于白名单校验后的内部场景（动态 ORDER BY 列名、动态表名）。

---

## 八、Mapper 接口代理

### 8.1 为什么不需要实现类？

MyBatis 利用 JDK 动态代理在运行时生成接口实现：

```
userMapper.selectById(1L)
  → MapperProxy.invoke() 拦截
    → 标识 MappedStatement: "com.example.UserMapper.selectById"
    → 从 Configuration 获取 MappedStatement
    → 委托 SqlSession 执行 → 返回结果
```

### 8.2 MapperProxyFactory + MapperProxy

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

### 8.3 MapperMethod.execute() 分发

`MapperMethod` 包含两个子对象：`SqlCommand`（持有 SQL id 和命令类型）和 `MethodSignature`（持有返回类型和参数信息）。

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object param = method.convertArgsToSqlCommandParam(args);
    switch (command.getType()) {
        case INSERT: return rowCountResult(sqlSession.insert(command.getName(), param));
        case UPDATE: return rowCountResult(sqlSession.update(command.getName(), param));
        case DELETE: return rowCountResult(sqlSession.delete(command.getName(), param));
        case SELECT:
            if (method.returnsMany())       return executeForMany(sqlSession, args);
            else if (method.returnsMap())   return executeForMap(sqlSession, args);
            else if (method.returnsCursor()) return executeForCursor(sqlSession, args);
            else return sqlSession.selectOne(command.getName(), param);
    }
}
```

---

## 九、Spring Boot 集成

### 9.1 @MapperScan 原理

`@MapperScan` 通过 `@Import(MapperScannerRegistrar.class)` 注册 `ClassPathMapperScanner`，扫描指定包下接口，为每个接口生成 `MapperFactoryBean`（一个 FactoryBean）：

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    public T getObject() { return getSqlSession().getMapper(this.mapperInterface); }
    public Class<T> getObjectType() { return this.mapperInterface; }
    public boolean isSingleton() { return true; }
}
```

### 9.2 MybatisAutoConfiguration

```java
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnBean(DataSource.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration {

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        factory.setMapperLocations(properties.resolveMapperLocations());
        factory.setTypeAliasesPackage(properties.getTypeAliasesPackage());
        return factory.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory factory) {
        return new SqlSessionTemplate(factory);
    }
}
```

### 9.3 SqlSessionTemplate 线程安全

`SqlSessionTemplate` 是一个 JDK 动态代理，每次方法调用由 `SqlSessionInterceptor` 从 `TransactionSynchronizationManager` 获取当前事务绑定的 SqlSession：

```java
private class SqlSessionInterceptor implements InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        SqlSession sqlSession = SqlSessionUtils.getSqlSession(  // ★ 从事务管理器获取
            sqlSessionFactory, executorType, exceptionTranslator);
        try {
            Object result = method.invoke(sqlSession, args);
            if (!isSqlSessionTransactional(sqlSession, sqlSessionFactory)) {
                sqlSession.commit(true);  // 非事务环境自动提交
            }
            return result;
        } finally {
            SqlSessionUtils.closeSqlSession(sqlSession, sqlSessionFactory);
        }
    }
}
```

**同一事务复用 SqlSession** → 一级缓存在事务范围内有效。不同事务互不干扰。

### 9.4 与 @Transactional 协作

```
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    accountMapper.decrease(from, amount);   // SqlSession-A
    accountMapper.increase(to, amount);     // SqlSession-A (同一事务)
}
```

`DataSourceTransactionManager` 获取 Connection 并关闭自动提交 → MyBatis 通过 `SqlSessionUtils` 获取 SqlSession 并注册到 `TransactionSynchronizationManager` → 同一事务内 Mapper 调用复用 SqlSession → 事务提交时 `SqlSessionUtils` 清理缓存。

---

## 十、生产实践

### 10.1 MyBatis-Plus 增强

```java
public interface UserMapper extends BaseMapper<User> {
    // 继承: insert/deleteById/updateById/selectById/selectList/selectPage
}

userMapper.selectList(new LambdaQueryWrapper<User>()
    .eq(User::getStatus, 1)
    .gt(User::getAge, 18)
    .orderByDesc(User::getCreateTime));
```

内部原理：`MybatisSqlSessionFactoryBuilder` 启动时自动注入 CRUD 的 `MappedStatement`。`LambdaQueryWrapper` 利用 `SerializedLambda` 获取字段名，避免字符串魔法值。

> 复杂查询和多表 JOIN 仍建议手写 SQL，不要为使用 Wrapper 牺牲可读性。

### 10.2 常见问题

**N+1**：优先嵌套结果映射（一条 JOIN）。必需懒加载时注意 `lazyLoadTriggerMethods` 配置，避免 `toString()` 意外触发。

**大字段映射**：列表查询用不含大字段的 resultMap，详情查询用 extends 扩展的 resultMap 按需加载。

**游标查询防 OOM**：

```java
@Options(fetchSize = 1000)
@Select("SELECT * FROM large_table")
Cursor<LargeRecord> scanAll();

try (Cursor<LargeRecord> cursor = mapper.scanAll()) {
    cursor.forEach(record -> exportToFile(record));  // 流式处理
}
```

`fetchSize(Integer.MIN_VALUE)` 在 MySQL 下触发流式查询，ResultSet 逐行读取而非一次性加载。

### 10.3 多参数处理

多个参数必须使用 `@Param` 显式命名。`ParamNameResolver` 源码逻辑：JDK 编译默认不保留参数名，通过 `@Param` 命名或启用 `-parameters` 编译选项可获取参数名。超过 3 个参数建议封装 DTO。

```java
// 错误：找不到 name 和 age
User selectByNameAndAge(String name, Integer age);
// 正确
User selectByNameAndAge(@Param("name") String name, @Param("age") Integer age);
```

### 10.4 MyBatis Generator

MBG 根据数据库表自动生成实体类、Mapper 接口和 XML。建议 `overwrite="false"` 避免覆盖手写代码。MyBatis-Plus 的 `AutoGenerator` 提供了支持自定义模板引擎的更现代化替代方案。

---

### 参考

- [MyBatis 官方文档](https://mybatis.org/mybatis-3/)
- [MyBatis-Spring 官方文档](https://mybatis.org/spring/)
- [MyBatis-Plus 文档](https://baomidou.com/)
- [MyBatis 3 源码](https://github.com/mybatis/mybatis-3)
- [PageHelper 分页插件](https://github.com/pagehelper/Mybatis-PageHelper)
