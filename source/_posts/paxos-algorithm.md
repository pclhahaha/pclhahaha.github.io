---
title: Paxos 算法
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - Paxos
  - 共识算法
categories:
  - 分布式
---

Paxos 是 Leslie Lamport 提出的经典共识算法，是分布式一致性理论的基础。Paxos 包含三个角色：

- **Proposer（提议者）**：提出提案，推动共识达成
- **Acceptor（接受者）**：对提案进行投票，决定是否接受
- **Learner（学习者）**：学习最终被选中的值，不参与投票

**Basic Paxos 两阶段协议**：

**Phase 1 — Prepare（准备阶段）**：
1. Proposer 选择一个全局唯一的提案编号 N
2. Proposer 向超过半数的 Acceptor 发送 Prepare(N) 请求
3. Acceptor 收到 Prepare(N) 请求后：如果 N 大于之前见过的所有编号，则做出两个承诺——不再接受编号小于 N 的任何 Prepare 请求，不再接受编号小于 N 的任何 Accept 请求——同时返回已经接受过的最大编号的提案值（如果有的话）

**Phase 2 — Accept（接受阶段）**：
1. 如果 Proposer 收到了超过半数的 Acceptor 对 Prepare 的响应，则向这些 Acceptor 发送 Accept(N, V) 请求，其中 V 是响应中编号最大的提案的值（如果所有响应都没有已接受的提案，Proposer 可以自由决定 V）
2. Acceptor 收到 Accept 请求后，只要自己遵守了对 N 的承诺（即没有见过比 N 更大的 Prepare 编号），就接受该提案
3. 当提案被多数派 Acceptor 接受后，该提案被选中（Chosen），Learner 学习该值

**Multi-Paxos**：Basic Paxos 每次达成共识都需要两轮通信（Prepare + Accept），Multi-Paxos 通过选举一个稳定的 Leader 来优化——一旦 Leader 被选举出来，后续提案可以跳过 Prepare 阶段，直接进入 Accept 阶段，将复杂度从两轮降为一轮。

Paxos 相关变种包括：
- **Zab**：ZooKeeper 使用的原子广播协议，基于 Paxos 思想设计
- **Raft**：以可理解性为核心设计目标的共识算法
- **EPaxos**：无 Leader 的 Paxos 变体，适合广域网部署

实现案例：ZooKeeper（基于 Zab 协议）


> 📖 独立文章：[Raft 共识算法详解](/raft-algorithm/)
