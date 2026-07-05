---
title: gRPC 与 Protocol Buffers
date: 2026-07-05
updated: 2026-07-05
tags:
  - gRPC
  - Protobuf
  - HTTP/2
  - RPC
categories:
  - Java
  - RPC
---

### 一、Protocol Buffers 基础

Protocol Buffers 的介绍文档：

https://developers.google.com/protocol-buffers/docs/overview

#### 1.1 proto3

gRPC 使用的是 Protocol Buffers 的 proto3 版本。

https://developers.google.com/protocol-buffers/docs/proto3

Java 语言使用 Protocol Buffers 的教程如下：

https://developers.google.com/protocol-buffers/docs/javatutorial#compiling-your-protocol-buffers

### 二、gRPC 服务定义与四种 RPC 类型

#### 2.1 服务定义

```protobuf
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

#### 2.2 四种 RPC 类型

- **Unary RPC**：一元调用，客户端发送一个请求，服务端返回一个响应。

  ```protobuf
  rpc SayHello(HelloRequest) returns (HelloResponse) {
  }
  ```

- **Server streaming RPC**：服务端流式 RPC，客户端发送一个请求，服务端返回一个流。

  ```proto
  rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse) {
  }
  ```

- **Client streaming RPC**：客户端流式 RPC，客户端发送一个流，服务端返回一个响应。

  ```proto
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) {
  }
  ```

- **Bidirectional streaming RPC**：双向流式 RPC，客户端和服务端都可以发送流。

  ```proto
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse) {
  }
  ```

### 三、gRPC 认证

- **Credentials**：channel credentials / call credentials
- **SSL/TLS**
- **Token-based authentication with Google**

### 四、gRPC 核心特性

使用场景：

- 微服务架构
- 边缘计算：边缘设备连接到后端服务，移动设备、浏览器、IoT
- 生成高效的客户端

核心特性：

- Idiomatic client libraries in 10 languages
- Highly efficient on wire and with a simple service definition framework
- Bi-directional streaming with HTTP/2 based transport
- Pluggable auth, tracing, load balancing and health checking

#### 4.1 HTTP/2 与双向 streaming

gRPC 基于 HTTP/2 传输，天然支持双向流式通信。HTTP/2 的多路复用特性使得单个连接上可以同时进行多个请求和响应，避免了 HTTP/1.1 的 head-of-line blocking 问题。这为 gRPC 的四种 RPC 类型提供了高效的底层传输能力。

---
