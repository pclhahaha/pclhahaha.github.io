---
title: 责任链模式 — Filter/Interceptor/Pipeline
date: 2026-07-05
updated: 2026-07-05
tags:
  - 设计模式
  - 责任链
  - Servlet Filter
  - Netty Pipeline
categories:
  - 设计模式
---

**意图**：将多个处理对象连成一条链，沿着这条链传递请求，直到有对象处理它为止。

**UML 简述**：多个 Handler 组成链表，每个持有后继引用，请求沿链传递直到被处理。

**Servlet Filter —— 责任链的标准实现**

```java
public class LoggingFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp,
                         FilterChain chain) throws IOException, ServletException {
        System.out.println(">>> 请求到达: " + req.getRemoteAddr());
        chain.doFilter(req, resp);  // 传递给下一个 Filter
        System.out.println("<<< 响应返回");
    }
}
```

Tomcat 中的 `ApplicationFilterChain` 持有 `Filter[]` 数组和位置指针 `pos`，逐个调用 filter 并递归传递 `this`（FilterChain）到链尾，最终调用 `servlet.service()`。

**Spring Interceptor**：层次在 Filter 之后、Controller 之前，可以实现 `preHandle`（返回 false 终止链）、`postHandle`、`afterCompletion`。比 Filter 更精细——可以拿到 Handler 对象且天然支持 Spring Bean。

**Filter vs Interceptor 对比**

| 维度 | Servlet Filter | Spring Interceptor |
|------|---------------|-------------------|
| 容器 | Servlet 容器（Tomcat） | Spring IoC 容器 |
| 粒度 | URL 级别 | Handler 方法级别 |
| 能力 | 只能拿到 Request/Response | 可以拿到 Handler 对象 |
| 能否访问 Spring Bean | 需要额外处理 | 天然支持 |

**Netty ChannelPipeline —— 双向责任链**：`ChannelPipeline` 是 `ChannelHandlerContext` 组成的双向链表，同时支持 Inbound（Decoder → Handler）和 Outbound（Encoder 逆序），将责任链发挥到了极致。

---
