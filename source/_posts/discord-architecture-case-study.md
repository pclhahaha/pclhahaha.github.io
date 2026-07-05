---
title: Discord 单服务器1500万用户架构
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 系统设计
  - Discord
  - WebSocket
  - 实时通信
categories:
  - 系统设计
---

Discord 的核心卖点是"低延迟的语音+文字聊天"。在游戏场景下，1500 万并发用户、数百万个语音频道、亚 100ms 延迟的实时语音传输——这些需求对服务器的并发处理能力提出了极高要求。

## 一、Elixir 的选择

Discord 最初使用 Erlang（WhatsApp 同款语言），后来迁移到 Elixir——一种运行在 Erlang VM (BEAM) 上的现代函数式语言。

选择 BEAM 虚拟机的核心理由：

- **轻量级进程**：每个 Discord 用户连接对应一个 BEAM 进程，约 2KB 内存
- **抢占式调度**：不依赖操作系统线程调度，BEAM 自己调度进程——防止单个慢进程阻塞整体系统
- **OTP**：内置的容错框架（Supervisor 树）
- **热更新**：无需重启服务即可部署新代码（在游戏进行中尤为重要）

## 二、Guild 与 Channel 模型

Discord 的核心数据结构是 Server（Guild）和 Channel：

```
Guild (服务器)
  ├── Channel A (文字频道)
  │      ├── Message 1
  │      └── Message 2
  ├── Channel B (语音频道)
  │      └── User Connections
  └── Channel C (公告频道)
```

关键设计决策——Discord 使用 **Monolith 架构**：同一台服务器同时处理用户连接、消息路由、语音传输。这与微服务哲学截然相反——为什么？

**低延迟要求**：语音+文字必须在亚 100ms 内交互。如果是微服务，每个消息需要跨越多个服务的网络调用（REST/gRPC），延迟翻倍。单体架构中，所有功能运行在同一 BEAM 实例中，消息传递在内存中完成（微秒级），远比网络调用（毫秒级）快。

## 三、WebSocket 连接管理

每个在线用户的 WebSocket 连接对应一个 Erlang 进程（约 2KB）：

```
连接建立：
  客户端 → WebSocket 握手 → Discord Gateway Server
      ↓
  创建 Gateway 进程 (2KB)
      ├─ 鉴权 token
      ├─ 拉取用户 guild 列表
      └─ 订阅相关的 guild 事件

消息投递：
  发送消息 → 接收方在同一个 guild → 直接广播给该 guild 的所有订阅用户
```

**扇出问题**：一个拥有 50 万成员的大 guild，发一条消息需要通知 50 万个进程。Discord 的优化：

1. 使用 ETS（Erlang Term Storage，内存中的共享表）存储 guild 成员关系
2. 通过 `pg2` 模块（进程组）订阅事件广播——多个进程共享同一事件流
3. 语音和文字的事件通道分离，避免语音流量阻塞文字推送

## 四、语音传输

Discord 的语音通话是实时传输，延迟要求极高（< 200ms）。关键设计：

| 技术选择 | 理由 |
|----------|------|
| **WebRTC** | 浏览器原生支持，已优化的实时音视频协议 |
| **Opus 编码** | 低延迟的语音编码格式，40ms 帧大小 |
| **SFU (Selective Forwarding Unit)** | 服务器只转发流，不解码/重编码——降低 CPU 开销 |

语音流量不经过业务逻辑层，直接通过 WebRTC 的媒体通道传输，降低中间跳数。

## 五、消息存储

Discord 不使用关系型数据库存储消息。消息是海量的、不可变的（不修改不删除）、按时间排序的——这些特征更匹配 Cassandra。

```
消息数据模型：
  Partition Key: channel_id
  Clustering Key: message_id (Snowflake, 时间排序)
  Columns: author_id, content, timestamp, attachments...
```

Cassandra 的宽列模型天然适合"按频道查询消息时间线"。读路径：`SELECT * FROM messages WHERE channel_id = ? ORDER BY message_id DESC LIMIT 50`。

## 六、How Discord Serves 15M Users on One Server

Discord 经常提到一个令人印象深刻的数字：**一台服务器就能承载 1500 万并发用户**。这不是虚构——BEAM 的进程模型使这成为可能。

计算如下：
```
单台服务器：
  - 1500万 Erlang 进程 × 2KB/进程 = 30GB 内存
  - 加上应用程序内存和缓冲区：~50-100GB 总内存
  - 单台服务器配 128-256GB RAM 完全可行
```

当然，这只是"连接数"的极限。在实际架构中，Discord 并没有真正把所有用户放到一台机器上——出于容错和负载均衡，他们会将用户分布到多个 BEAM 集群上。但这个数字说明了 BEAM 在处理高并发方面的天然优势。

## 七、总结

Discord 的架构选择体现了"反范式"思维：在微服务流行的时代，选择单体架构来满足实时通信的低延迟要求；在 NoSQL 流行的时代，选择 Cassandra 来存储海量消息；在主流语言之外，选择小众的 Elixir/Erlang 来发挥 BEAM 虚拟机的并发优势。这些选择共同构成了支持数千万用户的实时通信平台。
