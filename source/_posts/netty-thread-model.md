---
title: Netty 线程模型 (EventLoop)
date: 2026-07-05
updated: 2026-07-05
tags:
  - Netty
  - EventLoop
  - 线程模型
categories:
  - Java
  - 网络
---

EventLoopGroup 是 Netty 的线程池模型，负责管理一组 EventLoop。每个 EventLoop 绑定一个单独的线程，服务多个 Channel。通过这种线程绑定机制，Netty 实现了无锁化的串行处理，保证了线程安全的同时避免了锁竞争。

EventLoopGroup 通常分为两类：

- **bossGroup**（通常 1 个线程）：负责接收客户端连接请求，将新连接注册到 workerGroup
- **workerGroup**（通常 CPU 核数 × 2 个线程）：负责处理已建立连接的 I/O 读写操作

![NioEventLoopGroup](/java/netty/NioEventLoopGroup.png)

![NioEventLoop](/java/netty/NioEventLoop.png)
