---
title: IO 多路复用 — select/poll/epoll
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - IO
  - 多路复用
  - select
  - poll
  - epoll
categories:
  - Java
  - 网络
---

IO 多路复用是高性能网络编程的基石。从 select 到 poll 再到 epoll，内核的发展史就是一部"如何用更少的资源管理更多连接"的优化史。

## 一、为什么需要多路复用

在传统的阻塞 IO 模型中，`accept()` 和 `read()` 都是阻塞的——一个连接的处理会阻塞整个线程：

```c
// 传统阻塞模型：一个连接一个线程
while (1) {
    int client = accept(server_fd, ...);  // 阻塞等待新连接
    pthread_create(..., handle_client, client);  // 为每个连接创建线程
}
```

问题：1000 个连接 = 1000 个线程。每个线程占用 ~1MB 栈空间 + 上下文切换开销。多路复用让一个线程同时监视成百上千个连接。

## 二、select

```c
fd_set read_fds;
FD_ZERO(&read_fds);
FD_SET(socket1, &read_fds);  // 将 socket1 加入监视集合
FD_SET(socket2, &read_fds);

// 阻塞等待任意一个 socket 可读
int ready = select(max_fd + 1, &read_fds, NULL, NULL, NULL);

for (int i = 0; i <= max_fd; i++) {
    if (FD_ISSET(i, &read_fds)) {
        // socket i 可读了
    }
}
```

**特点**：

| 特性 | 说明 |
|------|------|
| 最大连接数 | 1024（FD_SETSIZE 限制，可重编译但效率下降） |
| 传入传出 | 每次调用都要把整个 fd_set 从用户态复制到内核态 |
| 查找就绪 | O(N) 遍历所有 fd 检查 `FD_ISSET` |
| 触发模式 | 仅水平触发（Level Triggered） |

## 三、poll

poll 解决了 select 的 1024 限制，但本质还是轮询：

```c
struct pollfd fds[10000];
fds[0].fd = socket1; fds[0].events = POLLIN;
fds[1].fd = socket2; fds[1].events = POLLIN;

int ready = poll(fds, nfds, timeout);

for (int i = 0; i < nfds; i++) {
    if (fds[i].revents & POLLIN) {
        // fds[i].fd 可读了
    }
}
```

**相比 select 的改进**：

- 使用链表存储 fd，不再有 1024 限制
- 事件和返回用不同字段（`events` 传入，`revents` 传出）

**未解决的问题**：仍然需要 O(N) 遍历所有 fd 来确定哪些就绪。

## 四、epoll

epoll 是 Linux 上的终极 IO 多路复用方案，通过"回调通知"彻底告别 O(N) 遍历。

### 4.1 核心 API

```c
int epfd = epoll_create(1);  // 创建 epoll 实例

struct epoll_event ev;
ev.events = EPOLLIN;  // 监视可读事件
ev.data.fd = socket1;
epoll_ctl(epfd, EPOLL_CTL_ADD, socket1, &ev);  // 注册

// 等待就绪事件
struct epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);

// events[0..n-1] 都是就绪的 fd，直接处理
for (int i = 0; i < n; i++) {
    int fd = events[i].data.fd;
    // 处理 fd
}
```

**核心差异**：`events[]` 返回的都是真正就绪的 fd，不需要遍历所有注册的 fd。

### 4.2 触发模式

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| **LT（水平触发）** | 只要缓冲区有数据就不停通知 | 简单可靠，潜在通知风暴 |
| **ET（边缘触发）** | 只在状态变化时通知一次 | 高性能，但需要非阻塞 IO + 一次读完 |

Netty 默认使用 ET 模式以减少不必要的通知。

### 4.3 底层实现

epoll 在内核中用**红黑树**存储所有注册的 fd，用**就绪链表**存储就绪的 fd：

```
epoll_create → 创建 eventpoll 对象
                ├─ 红黑树（rbr）：存储所有监视的 fd
                ├─ 就绪链表（rdllist）：存储就绪的 fd
                └─ 等待队列（wq）：阻塞在 epoll_wait 的进程

epoll_ctl(ADD) → 将 fd 加入红黑树
                  → 注册回调：当 fd 就绪时，将其加入就绪链表

epoll_wait → 检查就绪链表
             ├─ 有就绪 → 立即返回就绪事件列表
             └─ 无就绪 → 进程睡眠，等待回调唤醒
```

当网卡收到数据 → 触发中断 → 内核协议栈处理 → 调用 epoll 注册的回调 → 将 fd 加入就绪链表 → 唤醒等待 epoll_wait 的进程。

## 五、对比总结

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| 性能（大量空闲连接） | O(N) 线性下降 | O(N) 线性下降 | O(1)，仅活跃连接 |
| 最大连接数 | 1024 | 无限制 | 无限制（受系统内存限制） |
| 内核态/用户态拷贝 | 每次完整拷贝 | 每次完整拷贝 | 仅拷贝就绪事件 |
| 触发模式 | 仅 LT | 仅 LT | LT + ET |
| 适用场景 | 连接数少 | 连接数中 | **高并发首选** |

## 六、在 Netty 中的应用

Netty 在 Linux 上自动选择 epoll，通过 `EpollEventLoop` 实现：

```java
// Netty 自动选择最优传输
NioEventLoopGroup group = new NioEventLoopGroup();
// 如果系统支持 epoll，实际创建 EpollEventLoop（更高效）

// 或显式指定
EpollEventLoopGroup epollGroup = new EpollEventLoopGroup();
```

## 七、小结

select/poll 靠轮询找到就绪的 fd（O(N)），epoll 靠回调通知直接拿到就绪列表（O(1)）。这个 O(N) 到 O(1) 的跨越，让单台服务器可以承载数十万甚至百万级的长连接。Java NIO 的 Selector 在 Linux 上底层就是 epoll。
