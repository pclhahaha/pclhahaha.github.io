---
title: 时间轮 (Timing Wheel)
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - 时间轮
  - Netty
  - Kafka
categories:
  - 数据结构
---

传统定时任务用堆 (PriorityQueue)，每次取出最近任务，复杂度 O(log n)。在连接数极高的场景（如 Netty 管理百万级连接的超时检测），堆的 log n 插入成为瓶颈。

**时间轮**把时间维度映射到数组桶：

```
时间轮 (tick = 1s, wheelSize = 8):
          ┌───┐
          │ 0 │ → [taskA] → [taskB]  ← 第 0 个 bucket
          ├───┤
          │ 1 │ → [taskC]
   ┌──────┼───┼──────┐
   │      │ 2 │      │ ← 指针 (cursor) 在每个 tick 前进一格
   │      ├───┤      │
   │ 7    │ 3 │      │
   │      ├───┤      │
   │      │ 4 │      │
   └──────┼───┼──────┘
          │ 5 │ → [taskD]
          ├───┤
          │ 6 │ → [taskE] → [taskF]
          └───┘
```

- **插入**：O(1)——直接挂到对应 bucket 的链表
- **删除**：O(1)——从链表移除
- **触发**：指针走到 bucket 时，取出所有该 bucket 的任务并执行

**多层时间轮**处理跨度很大的延迟——比如 3600 秒后的任务，放在"小时轮"里，等快到时间了再下沉到"分钟轮"。Netty 的 `HashedWheelTimer` 和 Kafka 的 `TimingWheel` 都采用类似设计。

---
