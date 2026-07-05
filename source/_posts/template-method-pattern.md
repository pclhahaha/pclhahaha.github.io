---
title: 模板方法模式
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 模板方法
  - Spring refresh()
  - JdbcTemplate
categories:
  - 设计模式
---

**意图**：在一个方法中定义算法的骨架，将某些步骤延迟到子类中实现。

模板方法是我个人认为后端开发中最被低估的模式——它的价值在于**把不变的部分固化在父类，把可变的部分留给子类**，避免了重复代码。

**UML 简述**：抽象类定义 templateMethod（final，不可覆盖），调用一系列步骤方法——其中抽象方法由子类实现，钩子方法可选覆盖。

**AbstractApplicationContext.refresh() —— 史上最强的模板方法**

Spring 容器的启动过程是模板方法模式最宏大的展现：

```java
// AbstractApplicationContext 的 refresh() 是模板方法（final 修饰）
public final void refresh() throws BeansException {
    prepareRefresh();                    // 1. 准备刷新
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  // 2. 获取 BeanFactory
    prepareBeanFactory(beanFactory);     // 3. 准备 BeanFactory
    postProcessBeanFactory(beanFactory); // 4. BeanFactory 后置处理（子类扩展点）
    invokeBeanFactoryPostProcessors(beanFactory); // 5. 调用 BeanFactoryPostProcessor
    registerBeanPostProcessors(beanFactory);      // 6. 注册 BeanPostProcessor
    initMessageSource();                 // 7. 初始化消息源
    initApplicationEventMulticaster();   // 8. 初始化事件广播器
    onRefresh();                         // 9. 子类扩展点（钩子方法）
    registerListeners();                 // 10. 注册监听器
    finishBeanFactoryInitialization(beanFactory); // 11. 实例化所有单例 Bean
    finishRefresh();                     // 12. 完成刷新
}
```

`refresh()` 定义了容器启动的完整骨架，其中 `postProcessBeanFactory()` 和 `onRefresh()` 是留给子类的钩子：

```java
// Spring Boot 中的扩展
public class AnnotationConfigServletWebServerApplicationContext
       extends ServletWebServerApplicationContext {
    @Override
    protected void onRefresh() {
        // 启动内嵌 Web 服务器（Tomcat/Jetty）
        createWebServer();
    }
}
```

**JdbcTemplate —— 模板方法在数据访问中的典范**

```java
public <T> T query(String sql, RowMapper<T> rowMapper) {
    Connection conn = null; PreparedStatement ps = null; ResultSet rs = null;
    try {
        conn = dataSource.getConnection();
        ps = conn.prepareStatement(sql);
        rs = ps.executeQuery();
        return rowMapper.mapRow(rs, 1);  // 只有这一步是变化的
    } catch (SQLException e) {
        throw new DataAccessException(e);
    } finally {
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(ps);
        JdbcUtils.closeConnection(conn);
    }
}
```

JdbcTemplate 将 JDBC 的 try-catch-finally 样板代码全部封装，开发者只需提供 SQL 和 `RowMapper`——"**回调 + 模板方法**"的混合模式。

**InputStream 中的模板方法**：JDK `InputStream.read(byte[])` 的默认实现循环调用子类的 `read()`，父类定义"按字节填充数组"的骨架，子类只需实现单字节读取。

---
`
