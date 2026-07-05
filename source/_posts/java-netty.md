---
title: Java NIO 与 Netty
date: 2025-01-01 00:00:00
updated: 2025-01-01 00:00:00
tags:
  - Java
  - NIO
  - Netty
  - 网络编程
categories:
  - Java
---

# Java NIO 与 Netty
## 一、IO模型

### 1.1 传统 BIO Socket 编程

在 Java 早期版本中，网络编程主要基于 BIO（Blocking I/O）模型，即一个连接对应一个线程。以下是一份简单的服务端示例代码，利用 Java 的 Socket 实现对端口的监听。

在此模型下，每个客户端连接都需要分配一个独立的线程进行处理，当并发连接数较大时，线程资源消耗严重，系统伸缩性受限。

### 1.2 IO模型概述

操作系统层面的 I/O 模型定义了应用程序与内核之间进行数据交互的方式，主要分为以下五种：

- **阻塞 I/O（Blocking I/O）**：应用进程发起 I/O 请求后被阻塞，直到数据就绪并完成拷贝。
- **非阻塞 I/O（Non-blocking I/O）**：应用进程发起 I/O 请求后立即返回，通过轮询检查数据是否就绪。
- **I/O 多路复用（I/O Multiplexing）**：通过 select/poll/epoll 等系统调用同时监听多个文件描述符，任一就绪时通知应用进程。
- **信号驱动 I/O（Signal-driven I/O）**：应用进程通过 sigaction 系统调用注册信号处理函数，数据就绪时内核发送 SIGIO 信号通知。
- **异步 I/O（Asynchronous I/O）**：应用进程发起 I/O 请求后立即返回，内核完成全部数据拷贝后通知应用进程。


| epoll  | Linux 2.6+ 引入，基于红黑树和就绪链表，通过回调机制实现 O(1) 就绪事件获取，支持 ET/LT 模式 |

## 二、Java NIO 深入解析

### 2.1 核心概念

Java NIO（New I/O / Non-blocking I/O）在 JDK 1.4 中引入，提供了面向块（Buffer）和基于通道（Channel）的 I/O 操作方式，配合选择器（Selector）实现多路复用。


| ShortBuffer   |      |

#### Selectors

![Java NIO: Selectors](/java/netty/overview-selectors.png)

Selector 类似上面提到的 select/poll/epoll，允许一个线程同时监听多个 Channel，包括 Channel 的建立连接、数据到达等事件。通过将多个 Channel 注册到同一个 Selector 上，单个线程即可管理成百上千个连接，极大地提升了并发处理能力。

### 2.2 NIO 源码分析

#### Selector, SelectorProvider

Selector 针对 SelectableChannel 实现多路复用。可以通过 `Selector.open()` 方法创建，该方法调用系统默认的 SelectorProvider 来创建 Selector，也可以通过定制化的 SelectorProvider 来创建。

SelectorProvider 默认先加载系统属性指定的 Selector，若不存在，则通过 SPI 方式加载，若不存在则取默认 Selector，这个随系统而区分，例如，在 macOS 上是 `KQueueSelectorImpl`。

Selector 通过 SelectionKey 来表示 SelectableChannel 在其上的注册，并且维护了 3 个 SelectionKey 的集合：

1. **key set**：包含了当前所有注册的 channel

2. **selected-key set**：包含了在上一轮 selection 中检测到的就绪的 key 集合，该集合中的 key 在其感兴趣的操作中至少一项已经就绪

3. **cancelled-key set**：包含了已经 cancel 但还没有被 deregister 的 key 集合，会在 selector 的下一轮 selection 时从 key set 中移除

selection 操作包含 3 个步骤：

1. 清空 cancelled-key set，对应的 channel 进行 deregister，其他 set 中包含的 cancelled-key set 中的 key 也去除

2. 请求底层操作系统对剩余的 channel 的就绪状态进行更新，当检测到 channel 就绪时分为两种情况：若 channel 对应的 key 不在 selected-key set 中，则将其添加到 selected-key set 中并更新其 ready-operation set，丢弃之前的 ready-operation set 信息；若 key 已经在 selected-key set 中，则将之前的 ready-operation set 和新的 ready-operation set 去并集。

`KQueueSelectorImpl` 的 select 操作源码如下所示：

```java
protected int doSelect(long var1) throws IOException {
    boolean var3 = false;
    if (this.closed) {
        throw new ClosedSelectorException();
    } else {
        this.processDeregisterQueue();

        int var7;
        try {
            this.begin();
            var7 = this.kqueueWrapper.poll(var1);
        } finally {
            this.end();
        }

        this.processDeregisterQueue();
        return this.updateSelectedKeys(var7);
    }
}
```

#### SelectableChannel, ServerSocketChannel, SocketChannel

`SelectableChannel` 可以通过 `Selector` 实现多路复用。

1. 为了和 `Selector` 一起使用，首先需要通过该类的 `register` 方法注册，返回的 SelectionKey 表示注册结果，可以通过 SelectionKey 的 `cancel()` 方法 deregister，实际取消注册的时间是 Selector 下一次进行 selection 操作的时间，channel 关闭时取消注册的时机也是这样。selector 关闭时会立刻将注册的 channel 取消注册，没有延时。

    ```java
    public abstract SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException;
    ```

2. 一个 channel 最多只能在一个 selector 上注册一次。

3. selectable channel 是线程安全的。

4. selectable channel 有两种模式：阻塞/非阻塞。阻塞模式下，I/O 调用会阻塞；非阻塞模式下，I/O 调用不会阻塞。创建后默认处于阻塞模式，但在注册到 selector 之前必须设置为非阻塞模式，直到注册解除后才能改为阻塞模式。

`ServerSocketChannel` 是 SelectableChannel 的服务端 channel 的实现。

`SocketChannel` 是 SelectableChannel 的一般的 channel 实现。

#### 示例：NIO 服务端实现

```java
public void serve(int port) throws IOException {
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    serverChannel.configureBlocking(false);
    serverChannel.bind(new InetSocketAddress(port));

    Selector selector = Selector.open();
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);
    final ByteBuffer msg = ByteBuffer.wrap("Hi!\r\n".getBytes());
    for (; ; ) {
        try {
            selector.select();
        } catch (IOException ex) {
            ex.printStackTrace();
            // handle exception
            break;
        }
        Set<SelectionKey> readyKeys = selector.selectedKeys();
        Iterator<SelectionKey> iterator = readyKeys.iterator();
        while (iterator.hasNext()) {
            SelectionKey key = iterator.next();
            iterator.remove();
            try {
                if (key.isAcceptable()) {
                    ServerSocketChannel server =
                            (ServerSocketChannel) key.channel();
                    SocketChannel client = server.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_WRITE |
                            SelectionKey.OP_READ, msg.duplicate());
                    System.out.println(
                            "Accepted connection from " + client);
                }
                if (key.isWritable()) {
                    SocketChannel client =
                            (SocketChannel) key.channel();
                    ByteBuffer buffer =
                            (ByteBuffer) key.attachment();
                    while (buffer.hasRemaining()) {
                        if (client.write(buffer) == 0) {
                            break;
                        }
                    }
                    client.close();
                }
            } catch (IOException ex) {
                key.cancel();
                try {
                    key.channel().close();
                } catch (IOException cex) {
                    // 在关闭时忽略
                }
            }
        }
    }
}
```

## 三、Netty 框架

### 3.1 ServerBootstrap

ServerBootstrap 是 Netty 中用于引导服务端程序的启动类，继承自 AbstractBootstrap，负责配置 EventLoopGroup、Channel 类型、ChannelHandler 等核心组件。

```java
public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ServerBootstrap.class);

    private final Map<ChannelOption<?>, Object> childOptions = new ConcurrentHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> childAttrs = new ConcurrentHashMap<AttributeKey<?>, Object>();
    private final ServerBootstrapConfig config = new ServerBootstrapConfig(this);
    private volatile EventLoopGroup childGroup;
    private volatile ChannelHandler childHandler;
}
```

ServerBootstrap 继承 AbstractBootstrap：

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {

    volatile EventLoopGroup group;
    @SuppressWarnings("deprecation")
    private volatile ChannelFactory<? extends C> channelFactory;
    private volatile SocketAddress localAddress;
    private final Map<ChannelOption<?>, Object> options = new ConcurrentHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> attrs = new ConcurrentHashMap<AttributeKey<?>, Object>();
    private volatile ChannelHandler handler;
}
```

其中 ServerBootstrap 相较于 AbstractBootstrap 增加了 `childGroup`、`childHandler`、`childOptions` 和 `childAttrs`，分别用于配置新连接（子 Channel）的 EventLoopGroup、处理器和参数。ServerBootstrapConfig 用于对外暴露 ServerBootstrap 的配置信息。

### 3.2 Channel 与 ChannelPipeline

#### Channel

Channel 表示一个 socket 或者一个支持 read、write、connect、bind 的 I/O 操作的组件，具备以下功能：

- channel 当前的状态（是否 open，是否 connected）
- channel 的配置参数 ChannelConfig（例如：接受缓冲区大小）
- 支持的 I/O 操作（read、write、connect、bind）
- ChannelPipeline 处理所有的 I/O 事件，以及关联这个 channel 的请求

核心特性：

1. 所有 I/O 操作是异步的，I/O 调用不阻塞立刻返回一个 ChannelFuture
2. channel 是分等级的，可以有 parent，也可以创建共享一个 socket 连接的 sub-channel
3. 释放资源：调用 `close()` 或 `close(ChannelPromise)`

#### ChannelPipeline

```java
*                                                 I/O Request
*                                            via {@link Channel} or
*                                        {@link ChannelHandlerContext}
*                                                      |
*  +---------------------------------------------------+---------------+
*  |                           ChannelPipeline         |               |
*  |                                                  \|/              |
*  |    +---------------------+            +-----------+----------+    |
*  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  .               |
*  |               .                                   .               |
*  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
*  |        [ method call]                       [method call]         |
*  |               .                                   .               |
*  |               .                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  +---------------+-----------------------------------+---------------+
*                  |                                  \|/
*  +---------------+-----------------------------------+---------------+
*  |               |                                   |               |
*  |       [ Socket.read() ]                    [ Socket.write() ]     |
*  |                                                                   |
*  |  Netty Internal I/O Threads (Transport Implementation)            |
*  +-------------------------------------------------------------------+
```

1. ChannelPipeline 包含一组 ChannelHandler，ChannelHandler 处理或者拦截 channel 的 inbound 事件或者 outbound 操作，ChannelPipeline 相当于实现了一种高级形式的拦截过滤器，让用户控制如何处理事件或者控制 ChannelHandler 之间的交互。

2. 当一个 channel 被创建时，会自动创建其对应的 ChannelPipeline

ChannelPipeline 中的事件传播遵循责任链模式：
- **Inbound 事件**：从 Pipeline 头部向尾部传播（Head → Tail），通过 `ChannelHandlerContext.fireIN_EVT()` 触发
- **Outbound 操作**：从 Pipeline 尾部向头部传播（Tail → Head），通过 `ChannelHandlerContext.OUT_EVT()` 触发

#### ChannelHandler

ChannelHandler 是 Netty 中处理 I/O 事件和拦截 I/O 操作的核心接口。主要分为两类：

- **ChannelInboundHandler**：处理入站 I/O 事件（如 channelActive、channelRead、exceptionCaught 等）
- **ChannelOutboundHandler**：处理出站 I/O 操作（如 connect、write、flush、close 等）

通过 ChannelInitializer 可以在 Channel 初始化时动态添加 ChannelHandler 到 ChannelPipeline 中。


> 📖 独立文章：[Netty 线程模型 (EventLoop)](/netty-thread-model/)

## 参考文献

- [Java NIO Tutorial — Jenkov](http://tutorials.jenkov.com/java-nio/overview.html)
- [美团技术团队 — Java NIO 浅析](https://tech.meituan.com/2016/11/04/nio.html)
- [InfoQ — Netty 高性能之道](https://www.infoq.cn/article/netty-high-performance/#anch111813)
- [Netty 源码分析系列 — 简书](https://www.jianshu.com/p/b9f3f6a16911)
- [Netty 源码解析 — SegmentFault](https://segmentfault.com/a/1190000003063859)
- [Netty 服务端启动过程分析 — 掘金](https://juejin.im/post/5bdaf8ea6fb9a0227b02275a)
- [Netty 线程模型 — 知乎](https://zhuanlan.zhihu.com/p/22834126)
- [select, poll, epoll 比较 — ChinaUnix](http://blog.chinaunix.net/uid-26339466-id-3292595.html)
- [深入理解 Netty — 知乎](https://zhuanlan.zhihu.com/p/27441342)
- [Netty 核心组件分析 — 掘金](https://juejin.im/post/5bd584bc518825292865395d)
- [Netty 源码编译 — w3xue](https://www.w3xue.com/exp/article/201812/14152.html)
