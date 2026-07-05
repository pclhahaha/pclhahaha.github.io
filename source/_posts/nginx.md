---
title: Nginx 深度解析
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - Nginx
  - 反向代理
  - 负载均衡
  - 性能优化
categories:
  - 基础设施
---

[TOC]

## Nginx 概述

### 为什么需要 Nginx

Nginx（读作 "Engine X"）由 Igor Sysoev 于 2004 年发布，初衷是解决 [C10K 问题](http://www.kegel.com/c10k.html)——即单台服务器如何同时处理 10,000 个并发连接。经过近二十年的发展，Nginx 已成为全球最流行的 Web 服务器之一，根据 Netcraft 的统计，全球前 100 万站点中超过 30% 使用 Nginx。

在高并发场景下，Nginx 主要承担以下三种角色：

**反向代理（Reverse Proxy）**：作为客户端与后端服务器之间的中间层，Nginx 接收客户端请求后将请求转发给内部服务器，再把响应返回给客户端。这样做的好处是：隐藏后端服务器细节、统一入口进行安全控制、SSL 卸载等。

**负载均衡（Load Balancer）**：将请求按照特定算法分发到多台后端服务器，避免单点过载，提升系统整体吞吐量和可用性。Nginx 支持多种负载均衡算法，从简单的轮询到基于一致性哈希的会话保持。

**静态资源服务（Static File Server）**：直接处理静态文件（HTML/CSS/JS/图片/视频等）的请求，单机即可达到数万 QPS，无需经过后端应用服务器（Tomcat/Gunicorn/Node.js 等），极大节省计算资源。

一个小型站点的典型架构如下：

```
客户端 ---> Nginx (反向代理/静态资源) ---> 上游服务器集群 (应用服务器)
                |
                +--- 静态文件 (本地磁盘/CDN)
```

### Nginx vs Apache

Apache 自 1995 年诞生以来，一直是互联网基础设施的中坚力量。它采用 **进程驱动（Process-Driven）** 模型，主要有两种 MPM：

- **prefork 模式**：每个连接一个独立进程，稳定但内存开销极大，面对数千并发连接时内存耗尽。
- **worker 模式**：每个连接一个线程，内存消耗有所改善，但线程切换开销随并发数线性增长。

Nginx 采用 **事件驱动（Event-Driven）** 模型，使用少量的 worker 进程，每个进程通过异步非阻塞 I/O 同时处理数万个连接。以下是对比：

| 特性 | Nginx | Apache |
|------|-------|--------|
| 并发模型 | 异步非阻塞，事件驱动 | 进程/线程，阻塞 I/O |
| 内存消耗 | 极低，数 MB 即可服务数千连接 | prefork 模式下每个进程数 MB |
| 静态文件性能 | 极高（直接 sendfile） | 一般 |
| 动态内容处理 | 需反向代理到后端 | 可通过 mod_php/mod_perl 直接处理 |
| 配置灵活性 | 简洁，逻辑清晰 | 支持 .htaccess 目录级配置 |
| 模块系统 | 编译时静态链接 | 可动态加载 |
| SSL 性能 | 优秀 | 一般 |

**适用场景选择**：
- 静态内容、反向代理、高并发 → Nginx
- 共享主机（需要 .htaccess）、内置 PHP 处理 → Apache
- 复杂 URL 重写、需要大量第三方模块 → Apache 更灵活
- 混合部署：Nginx 前置处理静态资源 + 反向代理，Apache 处理动态逻辑

### 事件驱动架构详解

Nginx 的核心架构可以概括为 **master-worker 模型 + 异步非阻塞 I/O**：

```
                  +-----------+
                  |  Master   |
                  |  Process  |
                  +-----+-----+
                        |
          +-------------+-------------+
          |             |             |
     +----v----+   +----v----+   +----v----+
     | Worker 0|   | Worker 1|   | Worker N|
     +----+----+   +----+----+   +----+----+
          |             |             |
     (epoll_wait)   (epoll_wait)   (epoll_wait)
```

**Master 进程**负责：
- 读取并验证配置文件（`nginx -t`）
- 创建、绑定并监听共享 socket
- 启动、停止、热重载 worker 进程
- 处理信号（`SIGHUP` 热重载，`SIGUSR1` 日志重开，`SIGQUIT` 优雅退出）

**Worker 进程**负责：
- 实际处理客户端连接和请求
- 执行事件循环（epoll/kqueue）
- 读取、解析请求，生成响应
- 与上游服务器通信（作为反向代理时）
- 缓存读写

所有 worker 进程共享监听同一个 socket，由内核通过带锁的 accept 机制分发连接。每个 worker 是单线程的，内部使用异步非阻塞模型处理所有 I/O 操作——这正是 Nginx 能支撑数万并发连接的秘密。

为什么单线程能处理这么多连接？核心在于：**一个线程不需要阻塞等待任何一个 I/O 操作**。当读操作无法立即完成时，Nginx 不会挂起线程，而是将这个连接的读事件注册到 epoll 中继续处理下一个连接。待数据就绪后，epoll 通知该线程回来继续处理。这种模式将 CPU 利用率大幅提升，避免了线程切换和内核态拷贝的开销。

## 核心架构

### Master 进程

Master 进程本身不处理任何客户端请求，它的全部工作围绕"管理"展开。以下是一个典型的 Nginx master 进程（PID = 1234）的信号处理逻辑：

**SIGHUP — 热重载配置**：
Master 收到 SIGHUP 后，会重新解析配置文件，创建新的一批 worker 进程，然后通知旧 worker 进程优雅退出——即让旧 worker 处理完当前请求后关闭。整个过程不丢失连接，实现"无中断重启"。

```bash
# 热重载 Nginx 配置
nginx -s reload
# 或
kill -HUP $(cat /var/run/nginx.pid)
```

**SIGUSR1 — 重开日志**：
日志切割时使用。logrotate 轮转日志文件后，需要通知 Nginx 重新打开日志文件。

```bash
nginx -s reopen
```

**SIGQUIT — 优雅关闭**：
不再接收新连接，等待现有连接处理完毕后退出。

```bash
nginx -s quit
```

**SIGTERM / SIGINT — 快速关闭**：
立即关闭所有连接并退出。

Master 进程的一个重要细节是 **CPU 亲和性（affinity）**：通过 `worker_cpu_affinity` 指令，可以将每个 worker 进程绑定到特定 CPU 核心，减少 CPU 缓存失效，进一步提升性能。

```nginx
# 4 核 CPU，将 4 个 worker 分别绑定到 core 0~3
worker_processes 4;
worker_cpu_affinity 0001 0010 0100 1000;
```

### Worker 进程与 epoll 事件循环

每个 worker 进程的核心是**事件循环（Event Loop）**，这是 Nginx 高性能的基石。简化的事件循环逻辑如下：

```
while (true) {
    // 1. 调用操作系统 I/O 多路复用器，以超时方式阻塞等待事件
    events = epoll_wait(epfd, events_list, timeout);

    // 2. 遍历就绪事件
    for (event in events) {
        // 3. 根据事件类型分发处理
        if (event 是新连接) {
            accept 并注册读事件;
        } else if (event 是可读) {
            读取数据，解析 HTTP 请求;
            如果数据完整则生成响应;
        } else if (event 是可写) {
            发送响应数据;
            如果发送完成则清理连接;
        }
    }

    // 4. 处理定时器事件（超时连接清理）
    process_timers();
}
```

**epoll 的工作原理**：

传统的 `select`/`poll` 每次调用都需要将整个 fd 集合从用户态拷贝到内核态，且内核采用 O(n) 的轮询方式检查就绪 fd。当并发连接达到数万时，select/poll 的性能急剧下降。

epoll 则完全不同：
- **epoll_create**：在内核中创建 eventpoll 对象，维护一棵红黑树和就绪链表。
- **epoll_ctl**：将 fd 注册到红黑树中，只需一次拷贝。
- **epoll_wait**：直接返回就绪链表中的 fd，时间复杂度 O(1)。

这就是 Nginx 选择 epoll（Linux）/ kqueue（FreeBSD）/ IOCP（Windows）等高性能 I/O 多路复用机制的原因。

**惊群效应（Thundering Herd）与 accept_mutex**：

当新的 TCP 连接到达时，所有阻塞在 `epoll_wait` 上的 worker 进程都会被内核唤醒，但实际上只有一个 worker 能成功 `accept` 这个连接。其余被唤醒的 worker 发现无事可做就会重新睡眠——这种现象称为"惊群效应"，会浪费 CPU 资源。

Nginx 的解决方案是 **accept_mutex**（接受锁）：在调用 accept 之前，worker 必须先获取这把互斥锁。只有拿到锁的 worker 才能接受新连接，其他 worker 继续等待自己的事件。

```nginx
events {
    accept_mutex on;       # 开启 accept 互斥锁
    accept_mutex_delay 500ms;  # 重试间隔
}
```

Linux 内核 4.5+ 提供了 `SO_REUSEPORT` socket 选项，允许每个 worker 绑定到独立的 listen socket。当新连接到达时，内核通过哈希算法将其分配到特定的 socket 上，从根本上解决惊群问题。但 Nginx 早期就已引入 `accept_mutex`，因此 `SO_REUSEPORT` 的支持是相对晚近的事。

### 模块化设计

Nginx 的模块系统是其灵活性的来源。模块化设计严格遵循分层架构，每个请求按顺序经过一系列模块处理。核心模块类型包括：

**Handler 模块（处理模块）**：
负责处理特定类型的请求并生成响应。例如：
- `ngx_http_static_module`：处理静态文件请求
- `ngx_http_proxy_module`：将请求转发到上游服务器
- `ngx_http_fastcgi_module`：与 FastCGI 后端通信
- `ngx_http_uwsgi_module`：与 uWSGI 后端通信
- `ngx_http_grpc_module`：与 gRPC 后端通信

**Filter 模块（过滤模块）**：
对 Handler 生成的响应进行后处理。Filter 以链式方式组织——响应数据像穿洋葱一样顺序经过各层 filter：
- `ngx_http_gzip_filter_module`：Gzip 压缩
- `ngx_http_range_header_filter_module`：Range 请求处理
- `ngx_http_charset_filter_module`：字符集转换
- `ngx_http_headers_filter_module`：响应头修改

**Upstream 模块（上游模块）**：
管理与上游（后端）服务器的连接池和负载均衡，是实现 `proxy_pass`/`fastcgi_pass` 等指令的基础。

**Load-Balancer 模块（负载均衡模块）**：
实现具体的负载均衡算法，如轮询、IP 哈希、最小连接数等。用户可通过 `upstream` 块配置。

请求处理流程中的模块调用顺序（Nginx 的 11 个处理阶段）：

```
POST_READ → SERVER_REWRITE → FIND_CONFIG → REWRITE → POST_REWRITE
  → PREACCESS → ACCESS → POST_ACCESS → PRECONTENT → CONTENT → LOG
```

理解这四个模块的分工后，再看一段简单的配置就能明白其背后各个模块的工作：

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_set_header X-Real-IP $remote_addr;
    gzip on;
}
```

- `proxy_pass` 触发了 `ngx_http_proxy_module`（Handler 模块）
- `proxy_set_header` 在此 Handler 内部使用
- `gzip on` 触发了 `ngx_http_gzip_filter_module`（Filter 模块），响应在发送前被压缩

### 事件驱动模型与高性能原理

回到核心问题：**Nginx 为什么快？**

1. **进程模型轻量**：固定数量的 worker 进程（通常等于 CPU 核心数），不会随并发连接增长而创建新进程/线程。

2. **异步非阻塞 I/O**：所有 I/O 操作（网络读写、文件读写、代理转发）全部非阻塞，线程不会在任何 I/O 操作上挂起等待。

3. **内存池（Memory Pool）**：Nginx 为每个连接/请求分配独立的内存池，结束时整体释放。这种设计避免频繁的 malloc/free 操作，同时消除内存泄漏隐患。

4. **零拷贝（Zero-Copy）**：通过 Linux `sendfile` 系统调用，数据从磁盘到网络不经用户态拷贝，从内核缓存直接发送到 socket。

5. **事件驱动架构**：基于 epoll/kqueue 等高性能 I/O 多路复用器，事件通知从源头避免无效轮询。

6. **模块化与可扩展性**：过滤器和处理器解耦，各司其职，新功能通过新模块加入，不影响核心流程。

7. **极致优化的代码**：C 语言编写，处处考虑性能。例如 HTTP 解析器使用自动机而非正则表达式；数据结构（红黑树/基数树/队列）全部内联实现。

## 反向代理

### proxy_pass 原理与配置

反向代理是 Nginx 最常见的应用场景之一。`proxy_pass` 指令将请求转发到指定的上游服务器——Nginx 先接收客户端请求，建立与后端的连接，转发请求，接收后端响应，再返回给客户端。

```nginx
location /api/ {
    proxy_pass http://backend_server;
}
```

**路径替换规则**（这是配置中最容易出错的地方）：

`proxy_pass` 的 URI 部分如果是 `/` 结尾，则 location 匹配到的前缀部分会被替换为 `proxy_pass` 的 URI。

```nginx
# location 前缀:     /api/
# proxy_pass URI:    http://backend/v1/
# 请求: /api/users → http://backend/v1/users  (api/ 被替换为 v1/)

location /api/ {
    proxy_pass http://backend/v1/;
}

# 如果 proxy_pass 不含 URI（没有路径部分）
# 请求原样转发: /api/users → http://backend/api/users

location /api/ {
    proxy_pass http://backend;
}
```

**请求转发流程**：

```
客户端 ──→ [Nginx Worker]
              │  1. 解析 HTTP 请求
              │  2. 匹配 location
              │  3. proxy_pass 建立与后端的 TCP 连接
              │  4. 向后端发送 HTTP 请求
              │  5. 接收后端响应
              │  6. 将响应返回客户端
              └──────────────────────────────→ 后端服务器
```

### proxy_set_header 头部管理

请求经过反向代理后，后端服务器看到的连接信息会发生变化——源 IP 变为了 Nginx 的 IP。若需要获取客户端真实信息，必须显式传递这些头：

```nginx
location /api/ {
    proxy_pass http://backend;

    # 传递真实客户端 IP
    proxy_set_header X-Real-IP $remote_addr;

    # 传递代理链 IP 列表
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # 传递原始 Host 头
    proxy_set_header Host $host;

    # 传递原始协议（HTTP/HTTPS）
    proxy_set_header X-Forwarded-Proto $scheme;

    # 传递原始端口
    proxy_set_header X-Forwarded-Port $server_port;
}
```

**X-Real-IP vs X-Forwarded-For**：

- `X-Real-IP`：只包含直接客户端的 IP（即 Nginx 看到的远端 IP），常被简单场景使用。
- `X-Forwarded-For`：记录了完整的代理链 IP。格式为 `client, proxy1, proxy2, ...`。通过 `$proxy_add_x_forwarded_for` 变量，Nginx 会将当前客户端 IP 追加到已有值的末尾。

当多个代理串联时，`X-Forwarded-For` 能完整追踪一个请求穿越了哪些代理节点，这对安全审计和问题排查极为重要。

### proxy_buffering 缓冲机制

反向代理中，缓冲机制直接影响响应性能和用户体验。Nginx 作为中间层，两端网络速度可能不对等——例如后端很快但客户端很慢。缓冲能解耦这二者，让后端将完整响应快速交给 Nginx 后去处理下一个请求，Nginx 再按客户端速度慢慢发送。

**缓冲开启（默认）**：

```nginx
proxy_buffering on;                        # 开启缓冲（默认）
proxy_buffer_size 4k;                      # 响应头缓冲区大小
proxy_buffers 8 64k;                       # 缓冲区的数量和大小（8 个各 64k）
proxy_busy_buffers_size 128k;              # 客户端忙碌时可用的缓冲总大小
proxy_max_temp_file_size 256m;             # 临时文件最大大小
proxy_temp_file_write_size 256k;           # 一次写入临时文件的数据量
```

缓冲工作方式：
1. Nginx 从后端接收响应，首先填充 `proxy_buffer_size` 指定的头缓冲区。
2. 响应体则存储在 `proxy_buffers` 定义的缓冲池（8×64k = 512k）中。
3. 若响应体超过缓冲池总大小，多余部分写入临时文件（`proxy_temp_path`）。
4. 客户端读取时，Nginx 从缓冲中取出数据发送。只要缓冲未满，后端就可继续向 Nginx 发送数据。

**关闭缓冲（适用于流式/实时场景）**：

```nginx
location /stream/ {
    proxy_pass http://backend;
    proxy_buffering off;            # 关闭缓冲，实时转发
    proxy_cache off;                # 不可与缓冲同时关闭
}
```

关闭缓冲意味着 Nginx 收到后端的每个数据块立即同步发送给客户端，适用于：
- Server-Sent Events (SSE)
- 大文件下载（避免耗尽 Nginx 内存）
- 实时日志流
- 长轮询（Long Polling）

**缓冲大小调优建议**：

| 场景 | proxy_buffer_size | proxy_buffers | 说明 |
|------|------------------|---------------|------|
| 一般 API 响应 | 4k | 8 16k | 响应小，少量缓冲即可 |
| 中等页面响应 | 8k | 16 32k | 页面数十 KB |
| 大文件下载 | 64k | 32 128k | 但建议关闭缓冲 |
| 请关闭缓冲 | — | — | 流式、SSE、长轮询 |

### 代理超时配置

反向代理必须设置合理的超时时间。默认超时设置比较保守（60 秒），生产环境中需根据场景精细调控。

```nginx
location /api/ {
    proxy_pass http://backend;

    # 与后端建立 TCP 连接的超时时间
    # 默认 60s，一般配置 5~10s
    proxy_connect_timeout 5s;

    # 向后端发送请求的超时时间
    # 默认 60s，一般业务不会超过此值
    proxy_send_timeout 30s;

    # 从后端读取响应的超时时间
    # 注意：这不是整个响应的超时，而是两次连续的读操作之间的间隔
    # 默认 60s
    proxy_read_timeout 30s;

    # 读取后端响应头的超时时间
    # 默认 60s
    proxy_read_header_timeout 30s;

    # 长连接保持时间，对后端启用 keepalive 连接复用
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

**重要**：`proxy_read_timeout` 是连续两次成功读操作之间的最大间隔。如果后端持续稳定地发送数据（即便很慢），此超时也不会触发。若连接不是长时处理的，建议设为 30 秒以内，以便快速释放资源。

### WebSocket 代理

WebSocket 通过 HTTP 升级握手后保持长连接进行双向通信。Nginx 必须显式配置支持 WebSocket 协议升级：

```nginx
location /ws/ {
    proxy_pass http://websocket_backend;

    # HTTP 1.1 是 WebSocket 的前提条件
    proxy_http_version 1.1;

    # 传递升级请求头
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # WebSocket 连接不传递 Host 头会导致部分后端拒绝
    proxy_set_header Host $host;

    # 长连接不设置超时（或设置极大值）
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;

    # 关闭缓冲——否则 Nginx 会等待后端发送完数据才转发
    proxy_buffering off;
}
```

**踩坑提示**：如果返回 101 Switching Protocols 之后的 WebSocket 连接秒断，90% 的情况是：
1. 未配置 `proxy_http_version 1.1`（默认 HTTP/1.0）
2. 未配置 `proxy_set_header Connection "upgrade"`
3. `proxy_read_timeout` 过短导到长连接被关闭

### gRPC 代理

gRPC 基于 HTTP/2，要求端到端的 HTTP/2 支持。Nginx 提供了专门的 `grpc_pass` 指令：

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location / {
        grpc_pass grpc://grpc_backend;
        # 或者 TLS 到后端
        # grpc_pass grpcs://grpc_backend;

        grpc_set_header Host $host;

        # gRPC 错误处理
        error_page 502 = /grpc_502;
        error_page 503 = /grpc_503;
        error_page 504 = /grpc_504;
    }

    location = /grpc_502 {
        internal;
        default_type application/grpc;
        add_header grpc-status 14;          # UNAVAILABLE
        add_header grpc-message "service unavailable";
        return 200;
    }
}
```

gRPC 代理同样支持负载均衡、健康检查等功能，`upstream` 块中使用 `grpc://` 协议前缀即可。

## 负载均衡

### 四层 vs 七层负载均衡

理解负载均衡在 OSI 模型中的工作层次至关重要：

**四层负载均衡（传输层）**：工作在 TCP/UDP 层，根据 IP 地址和端口进行转发。Nginx 的 `stream` 模块提供四层负载均衡能力。

```nginx
stream {
    upstream mysql_replicas {
        server 10.0.0.1:3306 weight=1;
        server 10.0.0.2:3306 weight=1;
        server 10.0.0.3:3306 backup;
    }

    server {
        listen 3306;
        proxy_pass mysql_replicas;
        proxy_connect_timeout 5s;
    }
}
```

**七层负载均衡（应用层）**：工作在 HTTP 层，可以解析 HTTP 协议内容，根据 URL、Header、Cookie 等做精细化路由，支持会话保持、缓存、压缩等高级功能。

| 对比维度 | 四层 (stream) | 七层 (http) |
|---------|---------------|-------------|
| 协议支持 | TCP/UDP 通用 | HTTP/HTTPS/HTTP2/gRPC |
| 精细路由 | 不支持 | 支持根据 URL/Header 等 |
| 性能 | 极快（不进应用层） | 略慢（需解析 HTTP） |
| 会话保持 | 基于源 IP | Cookie/Header 等方式 |
| 缓存/压缩 | 不支持 | 支持 |
| SSL 终结 | 透明转发 | 可终结 |
| 适用场景 | 数据库/MQ/非 HTTP 应用 | Web 应用 |

**实际架构中通常同时用两层**：四层（LVS/HAProxy）在最外层做 TCP 级别的高性能和简单分发，七层（Nginx/Envoy）在内层做细粒度的 HTTP 路由。

### 负载均衡算法

Nginx 的 `upstream` 块定义一组后端服务器，并可通过不同算法指定请求分发策略：

```nginx
upstream backend {
    # 负载均衡算法（可选，默认轮询）
    # least_conn;
    # ip_hash;
    # hash $request_uri consistent;

    server 192.168.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 weight=1 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;                         # 备份服务器
    server 192.168.1.13:8080 down;                           # 永久下线
}
```

**轮询（Round Robin，默认）**：
请求按顺序轮流分发给每台上游服务器，配合 `weight` 实现加权轮询。

```nginx
# 加权轮询：每 4 个请求中，3 个到 server1，1 个到 server2
upstream backend {
    server 192.168.1.10 weight=3;
    server 192.168.1.11 weight=1;
}
```

**ip_hash**：
根据客户端 IP 的哈希值分配服务器。同一 IP 的请求始终打到同一台后端。适用于需要会话保持的场景，但不适合有大量 NAT 或代理的环境。

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10;
    server 192.168.1.11;
}
```

**least_conn（最少连接数）**：
将请求发送到当前活跃连接数最少的服务器。适用于长连接或处理时长不均匀的场景。

```nginx
upstream backend {
    least_conn;
    server 192.168.1.10 weight=3;
    server 192.168.1.11 weight=1;
}
```

**hash（哈希）**：
根据自定义键的哈希值选择服务器，常用于基于 URL 或请求参数的一致性哈希路由。添加 `consistent` 后使用一致性哈希环，服务器增删时仅重定向部分请求，适合缓存场景。

```nginx
upstream backend {
    hash $request_uri consistent;
    server 192.168.1.10;
    server 192.168.1.11;
    server 192.168.1.12;
}
```

**random（随机）**：
`random` 模块提供随机算法，支持 `two`（随机选两台，选连接数少的）等方法。

```nginx
upstream backend {
    random two least_conn;
    server 192.168.1.10;
    server 192.168.1.11;
}
```

### upstream 健康检查

健康检查机制保障只有健康的后端才接收流量。Nginx 提供被动和主动两种方式。

**被动健康检查（默认，开源版内置）**：

```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
}
```

- `max_fails`：在 `fail_timeout` 时间内允许的最大失败次数。
- `fail_timeout`：服务器被标记为不可用后，在此时间段内不再尝试向其转发请求。超过此时间后，Nginx 仁慈地重新尝试一次——若成功就恢复标记为可用。
- 默认值：`max_fails=1`，`fail_timeout=10s`。

**主动健康检查（需 `nginx_upstream_check_module` 模块）**：

商业版 Nginx Plus 或通过 `nginx_upstream_check_module` 补丁编译的社区版支持主动探测。配置示例如下：

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    # 每 5 秒主动探测一次 /health 接口
    check interval=5000 rise=2 fall=3 timeout=3000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

- `interval`：探测间隔（毫秒）
- `rise`：连续成功多少次后标记为可用
- `fall`：连续失败多少次后标记为不可用
- `timeout`：探测超时
- `type`：探测协议类型（http/tcp/ssl_hello/mysql/ajp）

**通用健康检查接口建议**：后端应用应暴露独立的 `/health` 或 `/healthz` 端点，该接口仅检查应用本身是否正常工作（不依赖外部服务），避免因依赖服务故障导致健康检查失败引发的雪崩。

### 会话保持

在多服务器集群中，若应用有状态（如 Session），需确保同一用户的请求路由到同一台服务器。

**ip_hash（最简单）**：

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10;
    server 192.168.1.11;
}
```

ip_hash 的局限性：NAT 网络下大量用户共享同一出口 IP，导致流量不均；客户端 IP 变化（如移动网络切换）会导致会话丢失。

**sticky cookie（Nginx Plus / nginx-sticky-module）**：

```nginx
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    server 192.168.1.10;
    server 192.168.1.11;
}
```

工作原理：第一个请求到达时 Nginx 分配一个后端，在响应中添加 `Set-Cookie: srv_id=...`。后续请求携带此 Cookie，Nginx 据此路由到同一后端。这种方法解决了 ip_hash 的问题。

**通用做法**：将会话状态抽取到共享存储（Redis/Memcached）或使用无状态 Token（JWT），从根本上解决会话保持的需求。

### 动态 upstream

传统的 upstream 变更需要修改配置文件 + `nginx -s reload`，这条命令实际上执行的是优雅重启——启动新 worker，关闭旧 worker。对于大规模集群，频繁重载会增加运维负担。

**ngx_http_dyups_module** 是阿里开源的动态 upstream 模块，允许通过 HTTP API 动态添加/删除上游服务器：

```nginx
# dyups 模块配置示例
dyups_upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

server {
    listen 8081;
    location /upstream {
        dyups_interface;
    }
}

server {
    listen 80;
    location / {
        dyups_backend backend;
    }
}
```

通过 HTTP 接口管理：

```bash
# 查看所有 upstream
curl http://127.0.0.1:8081/upstream

# 动态更新 upstream
curl -d "server 192.168.1.12:8080;" http://127.0.0.1:8081/upstream/backend
```

**现代替代方案**：在 Kubernetes 等容器编排环境中，upstream 的动态变更可由 Ingress Controller + Service 自动处理，无需在 Nginx 层面手动管理。

## 静态资源与缓存

### 静态文件服务

Nginx 配合操作系统零拷贝机制，能以极低开销处理静态文件。

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    location / {
        try_files $uri $uri/ =404;
    }

    # 特定文件类型缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

**sendfile / tcp_nopush / tcp_nodelay** 是静态文件高性能传输的三板斧：

```nginx
http {
    # 开启 sendfile 系统调用
    # 数据从磁盘到内核态再到网络协议栈以零拷贝方式传输，完全绕过用户态
    sendfile on;

    # 与 sendfile 配合，在一个 TCP 包中尽量发送完整 HTTP 响应
    # 避免出现多个小包单独发送（仅对 sendfile 生效）
    tcp_nopush on;

    # 对 keepalive 连接，禁用 Nagle 算法，允许小包立即发送
    # 与 tcp_nopush 互斥，各管各的场景
    tcp_nodelay on;
}
```

三者配合的逻辑：
- **sendfile**：用户态零拷贝，直接内核对内核传输。
- **tcp_nopush**：当使用 sendfile 发送文件时，积累数据直到达到 MSS（最大报文段大小）再发送，减少网络包数量。
- **tcp_nodelay**：对 keepalive 的读-写往返禁用 Nagle 算法，适用于需要低延迟的小数据交互。

### 浏览器缓存

通过 `expires` 指令设置缓存失效时间，Nginx 自动生成 `Cache-Control` 头。对版本号或哈希管理的静态资源（如 Webpack 的带哈希文件名），可设置极长缓存时间。

```nginx
location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# HTML 不缓存（总是请求最新版本）
location ~* \.html$ {
    expires -1;
    add_header Cache-Control "no-cache, must-revalidate";
}
```

**ETag**：Nginx 默认生成 ETag（基于文件最后修改时间和内容长度）。对于多服务器集群，各机器返回的 ETag 可能不同（inode 等因素），可关闭 ETag 统一使用 Last-Modified：

```nginx
location / {
    etag off;
    if_modified_since before;
}
```

### 代理缓存

代理缓存是反向代理最强大的功能之一——Nginx 缓存上游服务器的响应，直接返回缓存的副本，绕过后端请求。

```nginx
http {
    # 定义缓存路径和规则
    # levels: 缓存文件两级目录结构
    # keys_zone: 共享内存区域名称和大小（1m 可存约 8000 条 key）
    # max_size: 缓存总大小上限
    # inactive: 缓存内容在多久未被访问后淘汰
    proxy_cache_path /var/cache/nginx
                     levels=1:2
                     keys_zone=my_cache:100m
                     max_size=10g
                     inactive=60m
                     use_temp_path=off;

    server {
        location /api/ {
            proxy_pass http://backend;

            # 启用缓存
            proxy_cache my_cache;

            # 缓存键（决定哪些请求共享同一缓存）
            proxy_cache_key "$scheme$request_method$host$request_uri";

            # 哪些状态码的响应可以缓存
            proxy_cache_valid 200 302 60m;
            proxy_cache_valid 404 1m;

            # 哪些请求不被缓存（跳过缓存直接回源）
            proxy_cache_bypass $http_cache_control $cookie_nocache;

            # 添加响应头标记是否命中缓存
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

**`$upstream_cache_status` 可能的值**：
- `MISS`：缓存未命中（第一次请求，或缓存过期）
- `HIT`：缓存命中
- `EXPIRED`：缓存过期，正向源验证（返回 304 则复用旧缓存）
- `STALE`：后端不可用，返回了过期缓存（需开启 `proxy_cache_use_stale`）
- `BYPASS`：请求被 `proxy_cache_bypass` 条件跳过
- `REVALIDATED`：源返回 304 Not Modified，旧缓存复用
- `UPDATING`：旧缓存正被更新

**缓存失效策略**：

```nginx
# 后端出错时返回过期缓存（确保可用性）
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;

# 后台异步更新缓存（一个请求触发更新，其余请求返回旧缓存）
proxy_cache_background_update on;
```

**缓存清理**：Nginx 本身不带缓存内容删除功能。商业化 Nginx Plus 有 purge API；开源版可以搭配 `ngx_cache_purge` 模块实现：

```nginx
location ~ /purge(/.*) {
    allow 127.0.0.1;
    deny all;
    proxy_cache_purge my_cache $scheme$request_method$host$1;
}
```

### Gzip 压缩

Gzip 可将文本传输量降至原来的 20%~30%，代价是少量 CPU 开销。生产环境推荐配置：

```nginx
http {
    gzip on;
    gzip_min_length 256;               # 低于此大小的响应不压缩
    gzip_comp_level 5;                 # 压缩级别 1-9，5 是较优平衡点
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        application/rss+xml
        image/svg+xml
        font/ttf;
    gzip_vary on;                      # 添加 Vary: Accept-Encoding
    gzip_proxied any;                  # 对代理响应也启用压缩
    gzip_disable "msie6";              # 禁用 IE6 的 Gzip（旧配置，可移除）
}
```

**gzip_static**：当磁盘上存在预压缩的 `.gz` 文件时，直接发送该文件而无需实时压缩。使用 Webpack/Nginx 等构建工具预生成 gzip 文件，发送时完全省去压缩的 CPU 消耗。

```nginx
location /assets/ {
    gzip_static on;
    expires 30d;
}
```

## HTTPS 配置

### SSL/TLS 配置

生产环境的 HTTPS 配置需兼顾安全性与兼容性。以下是现代安全的推荐配置：

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # 证书文件（完整证书链：服务器证书 + 中间证书）
    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # TLS 协议版本（禁用不安全的 SSLv3、TLSv1、TLSv1.1）
    ssl_protocols TLSv1.2 TLSv1.3;

    # 加密套件（以现代加密算法优先）
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';

    # 优先使用服务器端定义的加密套件顺序（而非客户端偏好）
    ssl_prefer_server_ciphers on;

    # DH 参数文件（用于增强 DH 密钥交换的安全性）
    ssl_dhparam /etc/nginx/certs/dhparam.pem;

    # OCSP Stapling 开启（减少客户端验证证书的时间）
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/certs/fullchain.pem;

    # DNS 解析器（用于 OCSP Stapling 验证）
    resolver 8.8.8.8 114.114.114.114 valid=300s;
    resolver_timeout 5s;
}
```

**生成 DH 参数文件**：

```bash
openssl dhparam -out /etc/nginx/certs/dhparam.pem 2048
```

### HTTP/2 配置

HTTP/2 通过二进制分帧、多路复用、服务器推送等特性大幅提升 Web 性能。在 Nginx 中开启 HTTP/2 只需在 `listen` 指令后添加 `http2`：

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # HTTP/2 推送（推送关键资源）
    location / {
        http2_push /css/main.css;
        http2_push /js/main.js;
    }

    # 推送上限（每个连接最大推送数）
    http2_max_concurrent_pushes 10;

    # 推送所需的最小流（默认，可降低以更早推送）
    http2_push_preload on;
}
```

使用 HTTP/2 Server Push 的另一种方式是通过后端在响应头中添加 `Link` 头，Nginx 配合 `http2_push_preload on` 自动识别并推送：

```
Link: </css/main.css>; rel=preload; as=style
```

> **注意**：Chrome 106+ 已弃用 HTTP/2 Server Push。推荐使用 `<link rel="preload">` 替代。HTTP/2 最大的价值在于多路复用和头部压缩，Push 已非核心特性。

### HSTS 配置

HTTP Strict Transport Security (HSTS) 告知浏览器只能通过 HTTPS 访问该域名，杜绝 SSL 剥离攻击：

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # 有效期 2 年（秒），同时应用到子域名，允许提交至浏览器预加载列表
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}
```

**同时配置 HTTP→HTTPS 强制跳转**：

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### SSL 会话缓存

SSL/TLS 握手涉及非对称加密运算，消耗较大。SSL 会话缓存可将握手参数复用，将后续握手的开销降低近 90%。

```nginx
http {
    # 共享 SSL 会话缓存
    # builtin: 每个 worker 内的缓存
    # shared: 所有 worker 共享的缓存
    # 10m 约可存 80,000 个 session
    ssl_session_cache shared:SSL:10m;

    # 会话超时时间
    ssl_session_timeout 1d;

    # Session Ticket（简化了缓存但降低前向安全性，谨慎）
    ssl_session_tickets on;
    ssl_session_ticket_key /etc/nginx/certs/ticket.key;
}
```

## 性能优化

### worker 配置

Nginx worker 的关键参数直接影响并发处理能力：

```nginx
# 自动等于 CPU 核心数
worker_processes auto;

# 设置 worker 最大打开文件数限制
# 每个连接至少需要一个文件描述符，代理场景需两个（客户端+后端）
worker_rlimit_nofile 65535;

events {
    # 单个 worker 进程的最大并发连接数
    # 受系统限制（ulimit -n 和 worker_rlimit_nofile）
    worker_connections 4096;

    # 多路复用模型（Nginx 会自动选择最优，通常无需显式指定）
    use epoll;

    # 允许单个 worker 同时 accept 多个新连接
    multi_accept on;
}
```

**最大并发连接数计算公式**：

```
max_clients = worker_processes × worker_connections
```

以 8 核 CPU、`worker_connections 4096` 为例：
- 作为反向代理：`max_clients = 8 × 4096 / 2 = 16384`（因每个客户端连接同时占用一个后端连接）
- 作为静态服务器：`max_clients = 8 × 4096 = 32768`

**worker_connections 设置多少合适？**

不是越大越好。每个连接都会申请一个约 128~256 字节的接收缓冲区。100,000 个并发连接需要约 20~25 MB 的内存，十万级连接内存还远未到瓶颈，但要注意系统级别的限制：

```bash
# 查看当前系统限制
ulimit -n
cat /proc/sys/fs/file-max
```

### keepalive 优化

HTTP keepalive 允许单个 TCP 连接复用多个 HTTP 请求，避免频繁握手的开销：

```nginx
http {
    # 客户端 keepalive 连接的超时时间
    # 过长浪费连接，过短无法复用
    keepalive_timeout 65;

    # 单个 keepalive 连接上允许的最大请求数
    # 达到上限后服务器端主动关闭连接
    keepalive_requests 100;

    # upstream keepalive（与后端的连接复用）
    # 这是性能优化中被严重低估的配置
    upstream backend {
        server 192.168.1.10:8080;
        keepalive 32;              # 每个 worker 与后端保持的空闲连接数
        keepalive_timeout 60s;     # 后端连接超时（Nginx 1.19.10+）
        keepalive_requests 1000;   # 每个后端连接的最大请求数
    }
}
```

**upstream keepalive 的重要性**：反向代理场景下，客户端到 Nginx 和 Nginx 到后端都有连接管理的开销。开启 upstream keepalive 后，Nginx 与后端之间的连接得以复用，省去了每次代理请求都建立新连接的开销（TCP 三次握手、TLS 握手等）。

### 内核参数优化

系统内核参数瓶颈通常在十万级并发时显现。调整 Linux 内核参数：

```bash
# /etc/sysctl.conf 或 /etc/sysctl.d/99-nginx.conf

# 监听队列最大长度（即 accept 队列）
# 高并发下默认 128 太小
net.core.somaxconn = 65535

# 未完成连接的 SYN 队列长度
net.ipv4.tcp_max_syn_backlog = 65535

# 接收缓冲区最大值
net.core.rmem_max = 16777216

# 发送缓冲区最大值
net.core.wmem_max = 16777216

# 允许将 TIME_WAIT 状态的 socket 用于新连接
net.ipv4.tcp_tw_reuse = 1

# 减少系统的 FIN_WAIT2 等待时间
net.ipv4.tcp_fin_timeout = 10

# TCP keepalive 探测间隔
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# 本地端口范围（Nginx 作为反向代理大量连后端时）
net.ipv4.ip_local_port_range = 1024 65000
```

应用配置：

```bash
sysctl -p /etc/sysctl.d/99-nginx.conf
```

### 日志优化

作为反向代理，Nginx 的访问日志是排查问题的重要依据，但同时也是性能损耗的来源。优化策略：

**日志缓冲**：批量写入而非每条单写。

```nginx
http {
    access_log /var/log/nginx/access.log combined buffer=64k flush=5s;
}
# buffer: 缓冲区大小，写满后一次性刷盘
# flush: 至多在此时间内必须刷盘一次（确保日志不丢过久）
# 这两项组合可大幅减少 write 系统调用和磁盘 I/O
```

**日志压缩**：

```nginx
# Nginx 1.7.6+ 支持直接输出 gzip 压缩日志
access_log /var/log/nginx/access.log combined gzip buffer=64k flush=5s;
```

**条件日志**：过滤不需要记录的请求（如健康检查探测、静态资源 200），避免日志膨胀：

```nginx
map $request_uri $loggable {
    ~/healthz  0;
    ~\.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ 0;
    default 1;
}

access_log /var/log/nginx/access.log combined if=$loggable;
```

**异步日志（syslog）**：将日志发送到 syslog，由专用日志服务异步处理：

```nginx
access_log syslog:server=unix:/dev/log combined;
```

## 常用配置示例

### 静态文件服务器

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /data/www/static;

    sendfile on;
    tcp_nopush on;

    charset utf-8;

    # 安全：禁止访问隐藏文件（.git, .env 等）
    location ~ /\. {
        deny all;
    }

    # 跨域资源共享
    location / {
        add_header Access-Control-Allow-Origin *;
        add_header Timing-Allow-Origin *;

        # 开启目录浏览（按需）
        # autoindex on;
    }

    # 带哈希的静态资源——永久缓存
    location ~* \.[a-f0-9]{8,}\.(js|css|png|jpg|jpeg|gif|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
}
```

### 反向代理 + 负载均衡

```nginx
upstream app_servers {
    least_conn;
    keepalive 32;

    server 10.0.0.1:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 10.0.0.4:8080 backup;
}

server {
    listen 80;
    server_name api.example.com;

    # 添加真实 IP 和安全头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location /api/v1/ {
        proxy_pass http://app_servers/;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # 超时配置
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;

        # 代理缓存
        proxy_cache api_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_key "$scheme$request_uri";
        # 仅缓存可缓存的请求
        proxy_cache_methods GET HEAD;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://app_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
        proxy_buffering off;
    }
}
```

### HTTPS 站点配置（完整）

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header X-XSS-Protection "1; mode=block" always;

    root /var/www/html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 限流配置

Nginx 提供两种限流机制：**请求速率限流**（`limit_req`）和**并发连接数限流**（`limit_conn`）。

**limit_req（请求速率限制）**：

```nginx
http {
    # 定义限流共享内存区：10m 存储约 160,000 个 key，速率 10r/s
    # 以客户端 IP 为 key
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

    # 基于请求 URI 的限流（API 频率限制）
    limit_req_zone $binary_remote_addr$request_uri zone=api_limit:10m rate=30r/m;

    server {
        location /login/ {
            # burst=20: 允许 20 个请求排队（突发）
            # nodelay: 排队请求不延迟（立即响应 503），无此参数则请求排队等待处理
            limit_req zone=mylimit burst=20 nodelay;
            limit_req_status 429;

            proxy_pass http://backend;
        }
    }
}
```

**limit_conn（并发连接数限制）**：

```nginx
http {
    # 定义连接数限制 zone
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /download/ {
            # 每个 IP 最多同时 5 个连接
            limit_conn conn_limit 5;
            limit_conn_status 429;

            # 限速下载（每秒 300k）
            limit_rate 300k;

            root /data/downloads;
        }
    }
}
```

**使用 `$binary_remote_addr` 而非 `$remote_addr` 的理由**：IP 地址 "192.168.1.1" 作为字符串存储需 12~15 个字节，而二进制的 `$binary_remote_addr` 仅需 4 个字节（IPv4）或 16 个字节（IPv6），内存占用大幅降低。

### CORS 配置

跨域资源共享（CORS）是现代 Web 开发的必备配置：

```nginx
server {
    location /api/ {
        # 允许的来源（生产环境不使用 *，限制为具体域名）
        add_header Access-Control-Allow-Origin $http_origin always;

        # 允许的请求方法
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;

        # 允许的请求头
        add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Requested-With" always;

        # 允许携带 Cookie
        add_header Access-Control-Allow-Credentials "true" always;

        # 预检请求缓存时间（OPTIONS 请求）
        add_header Access-Control-Max-Age 86400 always;

        # 预检请求快速响应
        if ($request_method = OPTIONS) {
            return 204;
        }

        proxy_pass http://backend;
    }
}
```

**注意**：当使用 `Access-Control-Allow-Credentials true` 时，`Access-Control-Allow-Origin` 不能设为 `*`，必须指定具体域名。

### 防盗链

防止其他网站直接引用本站图片/资源（盗链），节省带宽：

```nginx
location ~* \.(png|jpg|jpeg|gif|svg|mp4|webm)$ {
    # 允许的引用来源
    valid_referers none blocked
                  example.com
                  *.example.com
                  ~\.google\. ~\.baidu\. ~\.bing\.;

    if ($invalid_referer) {
        # 返回 403 或替换为提示图片
        # return 403;
        rewrite ^/ /images/hotlink-denied.png break;
    }

    expires 30d;
}
```

### 重定向

**return（简单重定向）**：

```nginx
# 301 永久重定向
server {
    listen 80;
    server_name old.example.com;
    return 301 $scheme://new.example.com$request_uri;
}

# 特定路径 302 临时重定向
location /old-page {
    return 302 /new-page;
}
```

**rewrite（复杂重写）**：

```nginx
# URL 重写为 SEO 友好的格式
# /article?id=123 → /article/123.html
location / {
    rewrite ^/article/(\d+)\.html$ /article?id=$1 last;
}

# 旧 API 路径迁移
# 仅对 /api/v1/* 的请求修改路径，不影响其他
location /api/v1/ {
    rewrite ^/api/v1/(.*)$ /v2/$1 break;
    proxy_pass http://backend;
}
```

**return vs rewrite 的选择**：`return` 比 `rewrite` 效率更高，因为 `rewrite` 在 Server 和 Location 之间会引起额外的匹配循环。只要简单重定向就应使用 `return`。

## OpenResty 简介

### OpenResty vs Nginx

OpenResty 是基于 Nginx 核心和 LuaJIT 构建的高性能 Web 平台，由章亦春（agentzh）创建。它与原生 Nginx 的关系像浏览器与操作系统的关系——Nginx 提供了底层事件驱动引擎，OpenResty 提供了丰富的即用层。本质上是 Nginx 的超级增强版。

```bash
# 原生 Nginx 架构
Nginx Core + 标准模块 + 可选第三方模块

# OpenResty 架构
Nginx Core + LuaJIT + 丰富的 Lua 库 (lua-resty-*)
    + Redis / MySQL / Memcached / DNS / Upstream 的客户端
    + 模板引擎 / 正则 / JSON 等
```

**核心区别**：

| 特性 | Nginx | OpenResty |
|------|-------|-----------|
| 编程模型 | 纯配置驱动 | Lua 脚本 + 配置 |
| 动态路由 | 有限（Rewrite 模块） | 完全动态（Lua 访问数据库/Redis） |
| 扩展方式 | C 模块（编译时链接） | Lua 脚本（运行时加载） |
| 学习曲线 | 中等 | 较高 |
| 性能 | 极高 | 接近原生（LuaJIT 优化） |
| 应用场景 | 反向代理/静态服务 | API 网关/WAF/动态路由 |

### Lua 脚本与请求阶段

OpenResty 将 Lua 脚本注入了 Nginx 请求处理的每个阶段，极其灵活：

```nginx
server {
    listen 80;
    server_name api.example.com;

    # 访问阶段：鉴权、限流、WAF
    location /api/ {
        access_by_lua_block {
            local redis = require "resty.redis"
            local red = redis:new()
            red:connect("127.0.0.1", 6379)

            local key = "rate:" .. ngx.var.binary_remote_addr
            local count, err = red:incr(key)
            if not count then
                ngx.log(ngx.ERR, "redis error: ", err)
                return ngx.exit(500)
            end

            red:expire(key, 60)
            if count > 100 then
                return ngx.exit(429)
            end
        }

        proxy_pass http://backend;
    }

    # 日志阶段：自定义日志格式、审计
    log_by_lua_block {
        local cjson = require "cjson"
        local log_data = cjson.encode({
            time = ngx.time(),
            ip = ngx.var.remote_addr,
            uri = ngx.var.uri,
            status = ngx.var.status,
            rt = ngx.var.request_time,
        })
        -- 发送到 Kafka / 日志系统
        local sock = ngx.socket.tcp()
        sock:connect("10.0.0.10", 9090)
        sock:send(log_data)
    }
}
```

**与 Nginx 11 个阶段的对应**：

| Nginx 阶段 | OpenResty 指令 | 典型用途 |
|-----------|---------------|----------|
| POST_READ | `set_by_lua` | 变量初始化 |
| REWRITE | `rewrite_by_lua` | URL 重写、重定向 |
| ACCESS | `access_by_lua` | 鉴权、限流、IP 黑白名单 |
| CONTENT | `content_by_lua` | 完全动态生成响应 |
| LOG | `log_by_lua` | 自定义日志、审计上报 |

### 应用场景

**动态路由**：根据请求参数、User-Agent 等动态选择后端：

```nginx
upstream web_backend { server 10.0.0.1:8080; }
upstream mobile_backend { server 10.0.0.2:8080; }

server {
    location / {
        rewrite_by_lua_block {
            local ua = ngx.var.http_user_agent or ""
            if ua:find("Mobile") or ua:find("Android") then
                ngx.var.target = "mobile_backend"
            else
                ngx.var.target = "web_backend"
            end
        }
        proxy_pass http://$target;
    }
}
```

**API 网关**：OpenResty（尤其是基于它的 Kong、APISIX 等产品）是构建 API 网关的主流选择，支持认证、限流、日志、监控、AB 测试等功能的一站式处理。

**WAF（Web 应用防火墙）**：如著名的 ngx_lua_waf——使用 Lua 规则在 Nginx 层面拦截 SQL 注入、XSS、文件包含等攻击。

## Nginx 与后端架构

### Nginx 在微服务中的角色

在微服务架构中，Nginx 通常扮演 API 网关和入站负载均衡两重角色：

```
Internet
    │
    ▼
┌──────────────┐
│   L4 LB      │  (LVS / F5 / 云负载均衡器)
│  (高可用)    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Nginx 集群  │  (入口层：全局路由、限流、证书管理)
│  (入口 API   │
│   网关)      │
└──────┬───────┘
       │
       ├──────────→ ┌──────────────────┐
       │            │ 微服务 A 集群     │
       │            │ (内部 Nginx 前置) │
       │            └──────────────────┘
       │
       └──────────→ ┌──────────────────┐
                    │ 微服务 B 集群     │
                    │ (内部 Nginx 前置) │
                    └──────────────────┘
```

**入口 Nginx 的职责**：
- 全局路由：按域名/路径将请求分发到不同服务
- SSL 终结：TLS 握手在入口层完成，内部走 HTTP
- 统一鉴权：在入口层统一校验 Token/Session
- 全局限流：全局速率限制，保护下游
- 请求/响应日志统一收集

**内部 Nginx 的职责**：
- 服务发现集成（`resolver` + DNS）
- 实例级负载均衡
- 蓝绿/灰度发布

### Nginx + Keepalived 高可用

单台 Nginx 存在单点故障风险。通过 Keepalived 实现双机热备，对外暴露同一个 Virtual IP（VIP）：

```
              VIP: 192.168.1.100

 ┌──────────────────┐    ┌──────────────────┐
 │   Nginx Master   │    │   Nginx Backup   │
 │ 192.168.1.10     │    │ 192.168.1.11     │
 │ (MASTER 状态)    │◄───│ (BACKUP 状态)    │
 │ Keepalived       │VRRP│ Keepalived       │
 └────────┬─────────┘    └────────┬─────────┘
          │                       │
          └───────────┬───────────┘
                      │
              后端服务器集群
```

Keepalived 配置示例（`/etc/keepalived/keepalived.conf`）：

```
vrrp_instance VI_1 {
    state MASTER                    # 主: MASTER，备: BACKUP
    interface eth0
    virtual_router_id 51            # 主备需相同
    priority 100                    # 主: 100，备: 90
    advert_int 1                    # 心跳包间隔（秒）

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        192.168.1.100               # VIP
    }

    track_script {
        chk_nginx                   # 监听 Nginx 健康脚本
    }
}

vrrp_script chk_nginx {
    script "killall -0 nginx"       # 检测 Nginx 进程是否存在
    interval 2
    weight -20                      # 失败时降低优先级
}
```

**典型问题与处理**：Keepalived 主备切换时存在短暂（1~3 秒）的流量中断，对于金融/支付场景，使用云厂商的弹性负载均衡（ELB/ALB/CLB）能获得更高的可用性保证。

### Nginx vs LVS / HAProxy / Envoy

| 维度 | Nginx | LVS | HAProxy | Envoy |
|------|-------|-----|---------|-------|
| 工作层次 | 4/7 层 | 4 层 | 4/7 层 | 4/7 层 |
| 反向代理 | 是、核心功能 | 否 | 是 | 是 |
| 静态服务 | 是 | 否 | 否 | 否 |
| 负载均衡算法 | 5+ 种 | 10 种 | 10+ 种 | 多种 |
| 性能基准 | 极高（~50k 并发/MB 内存） | 最高 | 非常高 | 高 |
| HTTP/2 | Nginx 原生支持 | 否 | HAProxy 2+ | 原生支持 |
| gRPC | 支持 | 不支持 | 支持 | 原生支持（基于 HTTP/2） |
| 动态配置 | 有限 | 有限 | Runtime API | xDS 全动态 |
| 可观测性 | 日志文件 | 有限 | Prometheus | 开箱即用 |
| 云原生 | 一般 | 不支持 | 部分支持 | 完全支持 |
| 适合场景 | Web 应用 | 极简 TCP/UDP 转发 | TCP 高并发 | 微服务、Service Mesh |

**选择建议**：
- 传统 Web 应用、反向代理、静态文件 → **Nginx**
- 最简单高效的四层转发、数据库负载均衡 → **LVS**
- TCP 代理、高并发四层 + 七层 → **HAProxy**
- 微服务、Kubernetes、Service Mesh、全动态配置 → **Envoy**

### 常见问题排查

#### 502 Bad Gateway

**原因**：Nginx 从后端收到了无效响应（例如后端返回的响应不合法、连接已断开、响应头格式错误）。

**常见场景及排查**：

1. **后端服务已挂**：检查后端进程是否运行。
```bash
curl -v http://backend_host:port/health
```

2. **后端超时**：Nginx 的 `proxy_read_timeout` 小于后端的处理时间。
```nginx
proxy_read_timeout 60s;  # 调大该值
```

3. **负载过高**：后端请求处理不过来，连接队列满，连接被拒绝。

4. **后端错误**：PHP-FPM / Gunicorn / Node.js 进程崩溃，内部报错返回了未预期的数据。

5. **header 过大**：后端的响应头超过了 `proxy_buffer_size`。
```nginx
proxy_buffer_size 16k;   # 增大响应头缓冲区
```

**快速诊断**：
```bash
# 查看 Nginx 错误日志（80% 的问题都能在此找到线索）
tail -f /var/log/nginx/error.log

# 直接连到后端测试
curl -H "Host: example.com" http://<后端IP>:<端口>/api/health

# 查看 worker 进程状态
ps aux | grep nginx
```

#### 504 Gateway Timeout

**原因**：Nginx 在 `proxy_read_timeout` 时间内未从后端收到完整响应。

**常见场景**：
1. 后端处理超时（长任务、慢 SQL、外部 API 调用）
2. 后端进程被阻塞（线程池满、死锁）
3. 网络问题（Nginx 与后端的网络不稳定）

**解决方案**：
```nginx
# 按业务场景合理设置超时
location /long-running-task/ {
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
    proxy_pass http://backend;
}
```

> **根本方法**：504 通常是应用层问题。优先检查后端服务日志（慢查询、GC 停顿、线程池满等），而非无限增大超时。

#### 413 Request Entity Too Large

上传文件超过 Nginx 请求体大小限制。

```nginx
client_max_body_size 50m;     # 允许 50MB 的请求体
client_body_buffer_size 10m;  # 请求体缓冲区大小

# 特别是文件上传的 location
location /upload/ {
    client_max_body_size 200m;
    proxy_request_buffering off;  # 关闭请求体缓冲，直接流式传输到后端
}
```

#### upstream 节点反复上线/下线

日志中上游节点反复出现 `connect() failed (111: Connection refused)` 并自动切换到其他节点——这通常是 `max_fails` 过于激进、或者后端响应慢触发了被动健康检查。

**解决**：
```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=5 fail_timeout=60s;  # 放宽阈值
    server 192.168.1.11:8080 max_fails=5 fail_timeout=60s;
}
```

> **排查**：被动健康检查非常容易被个别慢请求触发，表现为一台后端反复被标记为 FAIL。监控后端响应时间分布（P99），设置合理的超时配合 `proxy_next_upstream` 和 `proxy_next_upstream_tries`。

**总结**：

Nginx 由一款轻量 Web 服务器跃升为现代互联网基础设施的核心组件。其事件驱动架构、模块化设计和极其稳定的性能，使其在"单机十万并发"成为现实。掌握 Nginx 不仅意味着能写出可用的配置——更意味着理解了网络协议在高并发下的工作原理，懂得如何设计稳定、可扩展的后端架构。

对于后端工程师而言，Nginx 绝不是锦上添花的运维工具——它是理解现代后端系统中流量入口、请求路由、负载分发这些关键概念的必修课。将 Nginx 用好、用透，是在高流量、高可用系统设计中少走弯路的基础功。
