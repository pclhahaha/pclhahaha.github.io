---
title: Netty 线程模型 (EventLoop)
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - Netty
  - EventLoop
  - 线程模型
  - boss
  - worker
categories:
  - Java
  - 网络
---

Netty 的线程模型是其高性能的核心——基于 Reactor 模式，使用少量的 EventLoop 线程处理海量连接的 IO 事件。

## 一、Reactor 模式

Reactor 是一种事件驱动的 IO 处理模式，核心思想是将 IO 事件（连接、读、写）分发给少量线程处理，而不是为每个连接分配独立线程。

```
传统 BIO 模型（每连接一线程）：
  Client → [Thread-1]
  Client → [Thread-2]
  Client → [Thread-3]
  100万连接 = 100万线程 ≈ 1TB内存

Reactor 模型（Netty）：
  100万连接 → [EventLoop-1 (1个线程)]
            → [EventLoop-2 (1个线程)]
            → ...
            → [EventLoop-N (1个线程)]
  N = CPU核数 × 2，总共几个线程处理百万连接
```

## 二、BossGroup 与 WorkerGroup

Netty 使用主从 Reactor 模式：

```
               [BossGroup]
             (通常只有 1 个 EventLoop)
              │       │
         accept         accept
              │                │
         [WorkerGroup]
     (多个 EventLoop，默认 CPU核数×2)
        │      │      │
      处理读   处理写   处理业务
```

- **BossGroup**：负责监听端口、接受新连接，将连接注册到 WorkerGroup
- **WorkerGroup**：负责已建立连接的所有 IO 读写和业务逻辑处理

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);      // 1 个 Boss
EventLoopGroup workerGroup = new NioEventLoopGroup();     // 默认 CPU核数×2

ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline()
                   .addLast(new StringDecoder())
                   .addLast(new StringEncoder())
                   .addLast(new BusinessHandler());  // 业务处理也在 EventLoop 中
             }
         });

ChannelFuture future = bootstrap.bind(8080).sync();
```

## 三、EventLoop 核心原理

每个 EventLoop 内部是一个**单线程的事件循环**：

```java
// EventLoop.run() 简化版本
while (!shutdown) {
    // 1. 调用 Selector.select() 等待 IO 事件
    int ready = selector.select(timeoutMillis);
    
    // 2. 处理就绪的 IO 事件
    if (ready > 0) {
        processSelectedKeys(selector.selectedKeys());
    }
    
    // 3. 执行任务队列中的任务
    runAllTasks();
}
```

**关键约束**：一个 Channel 在其生命周期内只会绑定到一个 EventLoop 线程上。这意味着同一个 Channel 的所有 IO 操作（读、写、编解码、业务处理）都由同一个线程执行——天然线程安全，无需同步。

## 四、线程绑定

Channel 到 EventLoop 的绑定在注册时完成：

```java
// 新的连接到来时，BossGroup 选择一个 Worker EventLoop
// 选择一个 EventLoop 的方式是轮询
int idx = Math.abs(executorIdx.getAndIncrement() % workerGroup.size());
EventLoop worker = workerGroup.next();

// 将 Channel 注册到这个 Worker 的 Selector 上
worker.register(channel);
// 此后这个 Channel 的所有事件都由这个 worker 线程处理
```

## 五、避免 EventLoop 阻塞

由于业务逻辑也运行在 EventLoop 中，耗时操作会阻塞整个 EventLoop 上的所有 Channel：

```java
// ❌ 错误：在 EventLoop 中执行耗时操作
ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        String result = heavyDatabaseQuery(msg.toString());  // 阻塞！
        ctx.writeAndFlush(result);
    }
});

// ✅ 正确：将耗时操作交给业务线程池
ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        executor.submit(() -> {
            String result = heavyDatabaseQuery(msg.toString());
            ctx.channel().eventLoop().execute(() -> {  // 回到 EventLoop 写回
                ctx.writeAndFlush(result);
            });
        });
    }
});
```

使用 `DefaultEventExecutorGroup` 可以为特定 Handler 分配独立的线程池：

```java
EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(16);

ch.pipeline()
  .addLast(new StringDecoder())
  .addLast(new StringEncoder())
  .addLast(businessGroup, new BusinessHandler());  // BusinessHandler 在独立线程池执行
```

## 六、小结

Netty 的线程模型精髓在于：用极少量的线程处理海量连接，每个连接的所有操作绑定到同一个线程，天然无锁+高效。理解 BossGroup/WorkerGroup 的分工和 EventLoop 的单线程约束，是写出高性能 Netty 应用的基础。
