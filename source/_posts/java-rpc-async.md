---
title: RPC 与异步编程
date: 2020-03-08 12:00:00
updated: 2020-03-08 12:00:00
tags:
  - Java
  - RPC
  - gRPC
  - CompletableFuture
  - RxJava
  - Reactor
  - 异步编程
categories:
  - Java
---

RPC（Remote Procedure Call）和异步编程是后端开发中两个独立又相互交织的领域。RPC 解决的是"如何跨网络调用远程服务"，异步编程解决的是"如何在等待时不让线程空闲"。当两者结合——异步 RPC——就构成了现代高性能微服务通信的基础。

## 一、RPC：跨网络的函数调用

RPC 的核心理念是让远程调用看起来像本地调用。你调 `userService.getUserById(100)`，而不关心这个服务在哪个 IP 的哪个端口上。

### 1.1 调用流程

```
客户端                            服务端
  │                                │
  ├─ 1. 序列化请求 (protobuf)       │
  ├─ 2. 发送到网络                  │
  │                                ├─ 3. 反序列化请求
  │                                ├─ 4. 执行业务逻辑
  │                                ├─ 5. 序列化响应
  │                                ├─ 6. 发送回网络
  ├─ 7. 反序列化响应               │
  ├─ 8. 返回结果给调用方            │
```

### 1.2 gRPC 与 Protocol Buffers

gRPC 是 Google 开源的高性能 RPC 框架，基于 HTTP/2 协议，使用 Protocol Buffers 作为接口定义语言和序列化协议。

```protobuf
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);  // 服务端流
  rpc UpdateUsers (stream User) returns (UpdateResult);     // 客户端流
  rpc Chat (stream Message) returns (stream Message);       // 双向流
}

message GetUserRequest {
  int64 user_id = 1;
}
```

gRPC 支持四种调用模式——一元调用（传统请求-响应）、服务端流、客户端流、双向流。详细内容参见 [gRPC 与 Protocol Buffers](/grpc-protobuf/)。

## 二、异步编程：不让线程闲着

### 2.1 为什么需要异步

在传统的同步阻塞模型中，一个请求对应一个线程：

```
Request → [Thread-1] → 查询数据库（阻塞 100ms）→ 等待 Redis（阻塞 20ms） → 返回
```

整个等待期间线程完全空闲，但占用了内存（每个线程约 1MB 栈）。1000 个并发请求 = 1000 个线程 = 1GB 内存。如果请求延迟增加，线程池打满，系统崩溃。

异步编程的核心思想：**在等待 I/O 时不阻塞线程，去处理其他请求**。

### 2.2 CompletableFuture

Java 8 引入的 CompletableFuture 是异步编程的基础工具：

```java
// 异步查询用户 + 异步查询订单 → 合并结果
CompletableFuture<User> userFuture = 
    CompletableFuture.supplyAsync(() -> userService.getUser(id));

CompletableFuture<List<Order>> ordersFuture = 
    CompletableFuture.supplyAsync(() -> orderService.getOrders(id));

// 等待两者都完成，然后合并
String result = userFuture
    .thenCombine(ordersFuture, (user, orders) -> 
        user.getName() + " has " + orders.size() + " orders")
    .exceptionally(ex -> "Error: " + ex.getMessage())
    .join();
```

CompletableFuture 的核心方法：

| 方法 | 作用 |
|------|------|
| `thenApply()` | 转换结果（同步） |
| `thenCompose()` | 串联异步操作 |
| `thenCombine()` | 合并两个异步结果 |
| `allOf()` | 等待所有完成 |
| `exceptionally()` | 异常处理 |

详细内容参见 [CompletableFuture 详解](/java-completablefuture/)。

### 2.3 响应式编程：从 Future 到 Stream

CompletableFuture 处理单次异步结果，但遇到流式数据（如 WebSocket 实时消息、Kafka 消息流）就不够了。响应式编程将"数据流 + 操作符"的组合抽象提升到一等公民。

Reactive Streams 规范定义了四个核心接口：

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}
public interface Subscription {
    void request(long n);  // 背压：下游告诉上游我能处理多少
    void cancel();
}
```

Project Reactor（Spring WebFlux 的基础）提供了 `Flux`（0-N 个元素）和 `Mono`（0-1 个元素）：

```java
Flux<User> users = userRepository.findAll()
    .filter(u -> u.getAge() > 18)
    .map(User::getName)
    .take(10);
```

详细内容参见 [响应式编程 — RxJava / Reactor](/reactive-programming/)。

## 三、异步 RPC：两者结合的实践

gRPC 原生支持异步调用和流式调用——这正是 RPC 和异步编程的结合点：

```java
// 异步 RPC 调用
stub.getUser(request, new StreamObserver<User>() {
    @Override
    public void onNext(User user) {
        // 收到响应
    }
    @Override
    public void onError(Throwable t) {
        // 错误处理
    }
    @Override
    public void onCompleted() {
        // 调用完成
    }
});
```

## 四、小结

RPC 解决的是分布式通信，异步编程解决的是并发效率。两者结合——异步 RPC——是现代微服务通信的主流范式。gRPC + CompletableFuture/Reactor 构成了 Spring 生态中从 RPC 到异步编程的完整技术栈。
