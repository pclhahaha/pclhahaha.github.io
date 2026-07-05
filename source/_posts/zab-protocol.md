---
title: ZAB 协议 (ZooKeeper)
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 分布式
  - ZAB
  - ZooKeeper
  - 原子广播
categories:
  - 分布式
---

ZAB（ZooKeeper Atomic Broadcast）是 ZooKeeper 内部使用的共识协议，专门为 ZooKeeper 的"主从架构 + 高性能写入"场景设计。与 Raft 和 Paxos 同为共识协议，但 ZAB 更侧重写入性能和顺序保证。

## 一、协议概述

ZAB 协议包括两个模式：

| 模式 | 说明 | 触发条件 |
|------|------|----------|
| **崩溃恢复** (Leader Election) | 选举新 Leader | Leader 宕机或集群启动 |
| **消息广播** (Atomic Broadcast) | Leader 将写请求原子广播给所有 Follower | 正常运行期间 |

ZooKeeper 将所有写请求转发给 Leader，Leader 通过 ZAB 协议将写操作作为事务（Proposal）原子广播给所有 Follower。

## 二、消息广播（正常运行）

消息广播是 ZAB 的稳态模式，流程如下：

```
Client 写请求 → Leader → 生成 Proposal → 广播给所有 Follower

Leader    Follower1   Follower2
  │           │           │
  ├─Propose──→│           │     1. Leader 生成 Proposal 并广播
  │           │           │
  │←──ACK────┤           │     2. Follower 将 Proposal 写入磁盘并返回 ACK
  │           ├──ACK─────→│
  │           │           │
  ├─Commit───→│           │     3. Leader 收到超过半数的 ACK
  │           │           │       向所有 Follower 广播 Commit
  │           ├──Commit──→│
```

- **Proposal**：Leader 为每个写请求生成一个全局唯一、单调递增的 `zxid`（ZooKeeper Transaction ID）。zxid 的高 32 位是 epoch（Leader 任期号），低 32 位是递增计数器。
- **ACK**：Follower 接收到 Proposal 后写入本地事务日志，然后回复 ACK。
- **Commit**：Leader 收到超过半数的 ACK 后，向所有 Follower 发送 Commit 消息，Follower 将事务应用到内存数据中。

## 三、崩溃恢复（Leader 选举）

当 Leader 宕机时，Follower 进入崩溃恢复模式：

### 3.1 选举流程

```
1. 所有 Follower 转变为 LOOKING 状态
2. 每个节点投票给自己（投给 zxid 最大的节点）
3. 收到其他人的投票后，与自己的投票比较：
   └→ 如果对方 zxid 更大 → 改投对方
   └→ 如果 zxid 相同 → 投 myid 较大的
4. 某个节点获得超过半数选票 → 成为新 Leader
5. 新 Leader 通知所有 Follower 进入消息广播模式
```

### 3.2 恢复阶段

新 Leader 当选后，需要确保所有 Follower 的数据一致：

```
Leader 找出所有 Follower 中最新的 zxid（记为 max_zxid）
Leader 向所有 Follower 发起同步：
  ├→ Follower A: 我最后提交到 zxid=100
  ├→ Follower B: 我最后提交到 zxid=98
  └→ Leader: 把 101~max_zxid 的事务发给 B，补全差距

只有当集群中超过半数节点完成同步后，Leader 才开始处理新的写请求
```

## 四、ZAB 与 Raft 的对比

ZAB 和 Raft 都实现了主从复制 + 原子广播，但设计侧重点不同：

| 维度 | ZAB | Raft |
|------|-----|------|
| **设计目标** | ZooKeeper 专用 | 通用共识协议 |
| **写入模式** | Leader 处理所有写操作 | Leader 处理所有写操作 |
| **Leader 选举** | zxid 最大 + myid 最大 | Term 最大 + 日志最新 |
| **Follower 落后处理** | Leader 主动同步 | Leader 发送 AppendEntries |
| **读操作** | 默认从 Follower 读（可能旧数据） | 默认从 Leader 读（强一致性） |
| **提交条件** | 过半 ACK | 过半 ACK |

ZAB 的选举条件（优先事务最新提交最多的节点，其次选择 ID 大的节点）与 Raft 有微妙差异——ZAB 更侧重"数据最新"，而 Raft 则综合考量"Term 最新 + 日志长度"，对安全性提供更强的理论保障。

## 五、典型应用场景

由于 ZAB 专为 ZooKeeper 设计，它的所有应用都通过 ZooKeeper 暴露：

| 场景 | 说明 |
|------|------|
| **分布式锁** | 利用 ZK 的临时顺序节点实现互斥锁 |
| **服务注册与发现** | 利用 ZK 的 Watcher 机制通知服务上下线 |
| **配置中心** | 利用 ZK 的统一配置存储 |
| **Leader 选举** | 利用 ZK 的临时节点争夺 |

## 六、小结

ZAB 是专为 ZooKeeper 场景优化的共识协议。相比 Raft 的通用性，ZAB 在选举效率和读写分离方面有针对性优化。理解 ZAB 的两个模式（消息广播和崩溃恢复）就理解了 ZooKeeper 如何在高吞吐写入和强一致性之间取得平衡。
