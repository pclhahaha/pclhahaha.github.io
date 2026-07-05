---
title: 堆 (Heap) 与 TopK
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - 堆
  - TopK
  - PriorityQueue
categories:
  - 数据结构
---

优先队列不是 FIFO——每次取出的都是**优先级最高**的元素，而"优先级最高"由比较器定义。底层通常是**堆 (Heap)** 实现。

**定时任务调度：** Java `ScheduledThreadPoolExecutor` 内部使用 `DelayedWorkQueue`（小顶堆实现），堆顶永远是最近需要执行的任务。每次 `take()` 时检查堆顶任务的到期时间：
- 如果已到期 → 立即执行
- 如果未到期 → `awaitNanos(delay)` 等待

但是存在一个问题：如果在等待期间插入了一个更早的任务怎么办？新插入的任务触发 `siftUp`，如果它比当前等待的堆顶更早，会调用 `leader` 线程的 `interrupt()` 重新排队。

**时间轮 (HashedWheelTimer)：** Netty 使用时间轮实现高性能定时器，解决大量定时任务场景下堆的 O(log n) 插入复杂度问题。时间轮把时间分成多个格子（bucket），每个格子是一个任务链表，指针按 tick 转动。插入 O(1)，但精度受 tick 间隔限制。

**Dijkstra 最短路径：** 用优先队列（最小堆）维护"当前已知最短距离的节点"，每次弹出最小距离节点进行松弛操作——这是在链路追踪、路由选择中的核心算法，详见第 7 章。

---
