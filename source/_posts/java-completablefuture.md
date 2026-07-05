---
title: CompletableFuture 详解
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - CompletableFuture
  - 异步编程
categories:
  - Java
  - 异步
---

首先，CompletableFuture 是 Future，意味着它支持任务异步执行。在此基础上，它还支持异步任务之间的依赖关系、异步任务的回调等。对于日常业务逻辑，使用 CompletableFuture 能够通过将业务逻辑异步化，来提升 cpu 利用率、缩短响应时间，达到提升性能的目的。

那么 CompletableFuture 和响应式编程的区别是什么呢？

关键区别在于 **Stream** 这个概念上，CompletableFuture 描述的是单次执行的结果，尽管可以通过异步任务之间的依赖关系构建一串异步任务组成的执行图，本质上面向的是单次操作，例如一次 rpc 请求的处理[^1]。

<img src="/java/rpc-async/image-blog-body-completablefuture2.jpg" alt="Java CompletableFuture API Example Invoice Path" style="zoom:67%;" />

响应式编程则面向的是 Stream，或者说叫"流"，也即是数据或事件的序列，有点类似 JAVA 的 Stream API。

当然用 CompletableFuture 也能够实现对数据/事件流的处理，但是有一个关键的特性是响应式编程具备而 CompletableFuture 则需要重新实现的，并且实现起来很复杂，那就是 **back pressure**。back pressure 就是在下游无法承载上游的压力时，采取一些措施。

至于 Java 中提供的 CallBack、Future 机制在实现响应式编程中的问题和缺点，例如"callback hell"、不支持 lazy computation 等，可以从 https://projectreactor.io/docs/core/release/reference/#getting-started 中找到答案。

所以，我们需要一套框架或者说类库来更好的实现响应式编程，需要具备以下特性：

- 支持异步任务的封装及组装，需要 API 来对异步任务进行包装，并且需要很多种算子来对异步任务进行组装，包括过滤、组装、异常处理等算子
- 减少异步任务的嵌套，减少代码的复杂性，增加可读性，避免"callback hell"等复杂度很高的代码
- 支持背压 back pressure，也就是下游和上游之间能协商数据流的速度
