---
title: Raft 共识算法详解
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - Raft
  - 共识算法
  - etcd
  - Consul
categories:
  - 分布式
---

Raft 是 Diego Ongaro 和 John Ousterhout 于 2013 年提出的共识算法，核心设计目标是功能与 Paxos 等价但更易于理解和工程实现。Raft 将共识问题分解为三个相对独立的子问题：**领导者选举（Leader Election）**、**日志复制（Log Replication）**、**安全性（Safety）**。

#### 7.2.1 基础概念

- **节点状态**：每个节点在任何时刻处于三种状态之一——**Leader**（最多一个，处理所有客户端请求）、**Follower**（被动响应 Leader 和 Candidate 的 RPC）、**Candidate**（竞选 Leader 的中间状态）
- **Term（任期）**：逻辑时钟概念，用连续递增的整数表示。每个 Term 最多产生一个 Leader，如果选举失败（选票分散），该 Term 以无 Leader 结束，进入下一个 Term
- **心跳机制**：Leader 周期性向所有 Follower 发送 AppendEntries RPC（即使没有新的日志要复制），Follower 收到心跳后重置选举超时计时器——心跳压制了 Follower 发起选举的冲动

#### 7.2.2 Leader Election（领导者选举）

1. 所有节点以 **Follower** 身份启动
2. Follower 在 **election timeout**（随机 150–300ms）内未收到 Leader 的心跳 → 自身变为 **Candidate**，将当前 Term +1
3. Candidate 先给自己投一票，然后并行向所有其他节点发送 **RequestVote RPC**
4. 三种可能的结果：
   - **赢得选举**：在一个 Term 内获得超过半数选票 → 成为 Leader，立即向所有节点发送心跳（空的 AppendEntries）建立权威，阻止新的选举
   - **其他节点成为 Leader**：收到来自新 Leader 的心跳，且心跳中的 Term ≥ 自身 Term → 承认新 Leader，退回 Follower 状态
   - **选票分散（Split Vote）**：无人获得多数票 → 超时后 Term +1，重新发起选举。随机超时机制（每个 Candidate 的选举超时独立随机）极大降低了反复分散的概率

#### 7.2.3 Log Replication（日志复制）

1. Leader 接收客户端写请求，将命令封装为日志条目追加到本地日志中
2. Leader 并行向所有 Follower 发送 **AppendEntries RPC**，携带需要复制的日志条目
3. 当某条日志条目被**超过半数**的节点安全写入后，Leader 将该条目标记为 **committed**，然后应用到状态机（执行命令），并将结果返回客户端
4. Leader 在后续的 AppendEntries 中携带 **leaderCommit** 索引，通知各 Follower 哪些条目已被提交——Follower 据此将对应条目应用到自己的状态机

Raft 的日志复制保证了 Log Matching Property（日志匹配特性）：如果不同日志中两条条目拥有相同的 Index 和 Term，则它们存储的命令相同、且该 Index 之前的所有条目也完全一致。这是 Raft 安全性保证的基石。

#### 7.2.4 安全性保证

- **Election Restriction**：Candidate 的日志必须至少和大多数节点一样"新"（比较最后一个日志条目的 Term 和 Index），否则不会被选为 Leader。这保证了已提交的日志不会在新 Leader 上任后被覆盖
- **Commitment Rule**：Leader 只能通过计数副本数来提交**当前 Term** 的日志条目，历史 Term 的条目通过"连带提交"（Leader 提交了一条当前 Term 的日志后，所有之前的日志也随之提交）
- **Configuration Change**：使用 Joint Consensus（联合共识）在两阶段中安全地变更集群成员

实现案例：etcd、Consul、TiKV、CockroachDB

推荐阅读：Raft 论文（《In Search of an Understandable Consensus Algorithm》）、Raft 官网的可视化动画、MIT 6.824 课程的 Raft 实验
