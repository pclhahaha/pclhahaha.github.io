---
title: ZAB 协议 (ZooKeeper)
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - ZAB
  - ZooKeeper
  - 原子广播
categories:
  - 分布式
---

ZAB 是 ZooKeeper 内部使用的原子广播协议，与 Raft 设计理念相似但实现不同：

- **Leader Election**：类似 Raft，基于 epoch（对应 Raft 的 term）选举 Leader
- **Broadcast（广播）**：Leader 将事务提议广播给所有 Follower，等待超过半数 ACK 后提交
- **Recovery（恢复）**：Leader 选举完成后，新 Leader 与 Follower 进行状态同步，确保所有已提交的事务都被复制

**与 Raft 的关键区别**：
- ZAB 按 **epoch** 划分阶段，每个 epoch 有且仅有一个 Leader
- ZAB 的 recovery 阶段更加复杂，需要处理前一个 Leader 未完成的事务
- ZK 通过 FIFO 的顺序保证写操作的顺序，但不保证跨客户端的线性一致性
