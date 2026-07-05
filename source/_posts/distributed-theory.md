---
title: 分布式系统理论基础
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 分布式
  - CAP
  - 一致性
  - 共识算法
  - 分布式事务
categories:
  - 分布式
---

[TOC]

## 一、CAP 定理

### 1.1 定义与历史

CAP 定理由 Eric Brewer 于 2000 年提出，后经 Seth Gilbert 和 Nancy Lynch 在 2002 年形式化证明，是分布式系统领域最基础也是最著名的定理之一。CAP 是以下三个属性的首字母缩写：

- **Consistency（一致性）**：所有节点在同一时刻看到相同的数据。注意这里的"一致性"与 ACID 中的 C 不同——CAP 中的 C 近似于"强一致性"或"线性一致性"，即写操作完成后，所有后续的读操作都能读到最新写入的值。
- **Availability（可用性）**：每个非故障节点在合理时间内都能返回一个非错误的响应。注意这并不意味着返回的数据是"最新"的，只要求服务能响应。
- **Partition Tolerance（分区容错性）**：当网络分区（节点之间通信中断）发生时，系统仍然能够继续运行。

### 1.2 为什么不能同时满足

理解 CAP 的关键在于：**网络分区是不可避免的**。在分布式系统中，节点间的网络通信可能随时中断，这不是一个可选项——它会发生。因此 P（分区容错性）在实际系统中是一个必须满足的条件。

当网络分区发生时，系统面临一个关键抉择：是选择一致性（C），阻塞或拒绝请求直到分区恢复；还是选择可用性（A），允许各分区独立响应请求，接受数据不一致？

以双节点主从复制为例：主节点负责写，从节点同步数据。当网络断开后：
- 如果继续让从节点处理读请求 → 牺牲一致性（从节点数据可能过期）
- 如果拒绝从节点的读请求 → 牺牲可用性（从节点不可用，尽管自身正常运行）

### 1.3 CA / CP / AP 取舍

| 类型 | 选择 | 典型场景 | 代表系统 |
|------|------|----------|----------|
| CP | 一致性 + 分区容错 | 金融转账、库存扣减 | ZooKeeper, etcd, Consul |
| AP | 可用性 + 分区容错 | 社交动态、用户信息 | Eureka, Cassandra, DynamoDB |
| CA | 一致性 + 可用性 | 无网络分区（单机） | 传统关系型数据库（单机） |

**关键的认知转变**：早期很多系统声称自己是 CA 系统，但这其实是一个伪命题。在真实分布式环境中，CA 意味着系统在分区发生时直接不可用——因为既不选择 C 也不选择 A。实际上，CA 只在单机系统或局域网无故障的理想条件下才成立。真正生产级的分布式系统必须在 CP 和 AP 之间做权衡。

### 1.4 实际系统中的权衡

**Eureka 的 CP → AP 演化**：Spring Cloud 生态中，Eureka 最初设计强调分区容错和可用性，牺牲了强一致性。当 Eureka Server 集群出现网络分区时，各节点仍然可以接受服务注册和查询请求，即使数据存在短暂不一致。这是因为服务发现的容忍度较高——"宁可返回一个可能宕机的服务节点，也不拒绝所有请求"。

**Nacos 的 AP + CP 混合模式**：Nacos 默认采用 AP 模式，通过 Distro 协议实现最终一致性服务注册；同时支持切换到 CP 模式（基于 Raft），适用于对一致性要求较高的配置管理场景。这种"同一系统、两种模式"的设计打破了 CAP 二选一的刻板印象，实际上 CAP 指的是在分区发生时的一致性/可用性选择，而非整个系统只能取其一。

**现实中的 CAP 并非二元对立**：
- 系统可以在正常情况下同时提供 C 和 A
- 仅在网络分区时才需要在 C 和 A 之间做取舍
- 不同业务模块可以有不同选择（Nacos 配置中心选 CP，注册中心选 AP）
- 可以通过"概率性 CAP"理解：将延迟超过某个阈值视为一种轻度分区，此时系统仍需做选择

### 1.5 PACELC 扩展

PACELC 是对 CAP 的重要补充，由 Daniel J. Abadi 在 2010 年提出：

```
如果发生网络分区 (P)：
  如何在 A 和 C 之间选择 (A/C)
否则 (Else)：
  如何在延迟 (L) 和一致性 (C) 之间选择 (L/C)
```

PACELC 指出，即使在正常运行（无分区）的情况下，系统设计也面临延迟与一致性的权衡，因为强一致性（如同步复制）必然带来更高的响应延迟。

| 系统 | 有分区时 | 无分区时 | 分类 |
|------|----------|----------|------|
| ZooKeeper | C | C | PC/EC |
| Cassandra | A | L | PA/EL |
| DynamoDB | A | L | PA/EL |
| MongoDB（默认） | C | L | PC/EL |
| CockroachDB | C | L | PC/EL |

理解 PACELC 有助于我们在设计系统时不仅考虑极端的网络故障场景，也要考虑日常运行中"低延迟"与"强一致"的平衡。

## 二、BASE 理论

### 2.1 定义

BASE 理论由 eBay 架构师 Dan Pritchett 提出，是 AP 系统的设计哲学，与 ACID 相对：

- **Basically Available（基本可用）**：系统出现故障时允许损失部分可用性——响应时间可能变长，或部分非核心功能降级——但保证核心功能仍然可用。
- **Soft State（软状态）**：系统中的数据允许存在中间状态，且该中间状态不会影响系统的整体可用性。数据在不同副本之间可以有一段时间的不一致。
- **Eventually Consistent（最终一致性）**：系统中的数据副本在经过一段时间的同步后，最终能够达到一致的状态。

### 2.2 BASE 与 ACID 对比

| 维度 | ACID | BASE |
|------|------|------|
| 核心思想 | 强一致性优先 | 可用性优先 |
| 一致性模型 | 强一致性 | 最终一致性 |
| 典型场景 | 金融交易、订单扣款 | 社交动态、商品评论、库存缓存 |
| 系统复杂度 | 事务管理复杂 | 需设计冲突检测和补偿逻辑 |
| 可扩展性 | 难以横向扩展 | 易于横向扩展 |
| 响应延迟 | 较高（锁等待、同步） | 较低 |

**工程实践中的融合**：现代大型系统几乎不会在全局层面严格遵守 ACID。通常的做法是：**在需要强一致性的核心链路使用 ACID 事务（如订单创建），在非核心链路使用 BASE 模式（如库存缓存同步、社交动态推送）**。这种融合被称为"柔性事务"或"最终一致性事务"，我们会在分布式事务章节深入讨论。

## 三、一致性模型

### 3.1 前言与分层

一致性模型定义了系统对读写操作的可见性保证，是程序员理解分布式系统行为的核心工具。需要注意区分两个不同层面的"一致性"：

| 层面 | 含义 | 常见术语 |
|------|------|----------|
| 数据一致性 | 副本之间的数据是否相同 | MySQL 主从、Redis Cluster |
| 事务隔离性 | 并发事务之间的可见性 | ACID 中的 I（Isolation） |
| 一致性模型 | 对读写操作顺序的承诺 | 线性一致、因果一致 |

本章讨论的是第三层——读写操作顺序的承诺。

### 3.2 强一致性 / 弱一致性 / 最终一致性

**强一致性（Strong Consistency / Linearizability）**：又称线性一致性，是所有一致性模型中最严格的一种。它要求：
- 任意时刻，所有节点看到的数据完全相同
- 写操作完成后，所有后续的读操作都必须能看到该写入（或更新的写入）
- 所有操作看起来就像按照某种全局顺序执行

线性一致性对系统性能影响极大，通常需要同步复制和全局时钟。Paxos/Raft 等共识算法是实现线性一致性的核心技术，详见本页第七章「分布式一致性/共识算法」。

**弱一致性（Weak Consistency）**：不保证读操作能立即看到最新写入，系统只保证"最终"会一致。弱一致性没有严格定义，实际上它是除强一致性之外所有模型的统称。

**最终一致性（Eventual Consistency）**：弱一致性的一种具体形式，保证如果没有新的写操作，最终所有副本都会收敛到相同的值。它并不承诺收敛的时间。DNS 系统是最终一致性的经典实例。

### 3.3 顺序一致性（Sequential Consistency）

顺序一致性由 Lamport 定义：操作结果与所有处理器按某种顺序执行各自主机上的操作一致，且同一处理器上的操作保持程序顺序。顺序一致性强于因果一致性，但弱于线性一致性——它不要求操作的全局顺序与物理时间一致。

```
// 线性一致性与顺序一致性的区别
// 线性一致性：操作顺序 = 物理时间顺序
P1: write(x=1)           write(x=2)
P2:            read(x)→2

// 顺序一致性：允许重排，但保证单处理器的程序顺序
P1: write(x=1)           write(x=2)
P2:            read(x)→1  // 在顺序一致性下可能，在线性一致下不可能
```

### 3.4 因果一致性（Causal Consistency）

因果一致性保证：有因果关系的操作在所有节点上看到的顺序一致，但无因果关系的并发操作可以任意排序。因果关系定义为"happens-before"关系——如果操作 A 可能影响操作 B，则 A happens-before B。

微信朋友圈的评论回复就是典型的因果一致性场景：回复必须出现在被回复的评论之后。但两个互不相关的评论（没有因果关系）可以在不同用户的手机上以不同顺序显示。

实现因果一致性通常需要**向量时钟**来追踪操作之间的 happens-before 关系。

### 3.5 读己之写（Read Your Writes）

**读己之写**：保证用户总能看到自己之前的写入。这是一种非常实用的一致性保证——用户更新了头像后立即刷新页面，必须看到新头像，不能看到旧头像。

实现方式：
- 将写入操作路由到主节点，读取也路由到同一节点
- 使用 version 或 timestamp 确保从库追上了写操作的位点
- 对同一个用户的操作路由到同一个分区/节点

### 3.6 单调读（Monotonic Reads）

**单调读**：保证客户端不会读到比之前更旧的数据。即如果已经读到了版本 N，后续读取的版本必须 ≥ N。

典型问题场景：用户看了朋友圈，切换到后台后再切回——如果不保证单调读，可能出现"一条动态消失了"的现象。这是因为两次读请求被路由到了不同步程度不同的节点。

实现方式：基于用户 ID 做会话粘滞，将同一用户的路由到同一节点。

### 3.7 应用场景速查

| 一致性模型 | 严格度 | 典型应用 |
|------------|--------|----------|
| 线性一致性 | 最高 | 分布式锁、选主、交易扣款 |
| 顺序一致性 | 高 | 分布式文件系统元数据 |
| 因果一致性 | 中 | 社交评论、协同编辑 |
| 读己之写 | 中 | 用户个人信息修改 |
| 单调读 | 中低 | 时间线类应用 |
| 最终一致性 | 低 | DNS、CDN、缓存同步 |

## 四、分布式事务

分布式事务泛指跨多个节点的事务操作，目标是保证跨节点的数据一致性。传统单机数据库的 ACID 事务在分布式环境下无法直接使用，需要方案升级。

### 4.1 2PC（两阶段提交）

#### 流程

2PC 引入协调者（Coordinator），将事务分为两个阶段：

**阶段一 Prepare（投票阶段）**：
1. 协调者向所有参与者发送 prepare 请求
2. 参与者执行事务操作（但不提交），记录 undo 和 redo 日志
3. 参与者返回 YES（可以提交）或 NO（需要回滚）

**阶段二 Commit（提交阶段）**：
1. 若所有参与者返回 YES → 协调者发 commit 请求，参与者提交事务并释放锁
2. 若任一参与者返回 NO 或超时 → 协调者发 rollback 请求，参与者回滚并释放锁

```java
// 2PC 协调者伪代码
public boolean twoPhaseCommit(List<Participant> participants) {
    // Phase 1: Prepare
    for (Participant p : participants) {
        boolean ok = p.prepare();
        if (!ok) {
            rollbackAll(participants);
            return false;
        }
    }
    // Phase 2: Commit
    for (Participant p : participants) {
        p.commit();
    }
    return true;
}
```

#### 缺陷

| 缺陷 | 描述 |
|------|------|
| **同步阻塞** | Prepare 阶段参与者锁定资源后等待协调者指令，期间其他事务无法访问这些资源 |
| **单点故障** | 协调者宕机后，参与者无法知道应该 commit 还是 rollback，陷入"阻塞等待" |
| **脑裂** | commit/rollback 阶段，部分参与者收到指令、部分未收到，数据不一致 |
| **性能差** | 所有参与者必须串行等待，任一慢节点拖慢整个事务 |

### 4.2 3PC（三阶段提交）

3PC 在 2PC 基础上增加了一个 CanCommit 阶段，并为参与者和协调者分别引入超时机制：

1. **CanCommit**：协调者询问参与者"是否可以提交"（不执行任何操作，不持锁）
2. **PreCommit**：协调者发送预提交，参与者执行事务但不提交
3. **DoCommit**：协调者发送提交，参与者提交或超时自动提交

#### 改进
- 引入超时机制：参与者等待超时后**自动提交**（假设协调者已同意），减少阻塞时间
- CanCommit 阶段降低不必要持有的锁时间

#### 缺陷
- **网络分区问题未根本解决**：如果 PreCommit 后发生网络分区，部分参与者因超时自动提交、部分收到回滚——数据依然不一致
- 多一轮通信开销，在高可用系统中性能反而不如 2PC
- 在现实中几乎不被使用，更像一个学术上的改进尝试

### 4.3 TCC（Try-Confirm-Cancel）

TCC 是二阶段提交在业务层的变体，将事务拆分为三个操作：

- **Try**：预留资源（锁定库存、冻结余额）
- **Confirm**：确认提交（扣减库存、扣款）
- **Cancel**：取消释放（释放库存、解冻余额）

```java
// TCC 接口示例
public interface TccService {
    // Try: 冻结库存
    boolean tryFreezeInventory(String orderId, String skuId, int count);
    
    // Confirm: 确认扣减
    boolean confirmDeductInventory(String orderId, String skuId);
    
    // Cancel: 释放冻结
    boolean cancelReleaseInventory(String orderId, String skuId);
}
```

#### 关键挑战

**空回滚（Empty Rollback）**：Try 阶段因网络超时未到达服务端，但协调者认为需要 Cancel，此时服务端未执行过 Try。Cancel 操作要能识别并返回成功（幂等处理）。

**悬挂（Suspension）**：Try 请求因网络延迟在 Cancel 之后才到达。此时业务应拒绝该 Try 请求（资源已释放），需要保留 Cancel 记录判断。

**幂等（Idempotence）**：Confirm 和 Cancel 可能因网络重试被多次调用，必须保证重复调用不会重复扣减或重复释放。

```java
// 使用事务控制表处理悬挂的简单思路
public boolean tryFreeze(String orderId) {
    // 检查是否有 Cancel 先到达的记录
    if (isCancelled(orderId)) return false;
    // 幂等检查
    if (isTried(orderId)) return true;
    // 执行业务并记录状态
    return freezeInventory(orderId);
}
```

### 4.4 Saga

Saga 由 Hector Garcia-Molina 在 1987 年的论文中提出，核心思想是将长事务拆分为一组有序的本地事务，每个本地事务都有对应的补偿事务。

#### 编排（Choreography）vs 协调（Orchestration）

| 模式 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **编排式** | 各服务通过事件驱动相互调用，无中心协调者 | 松耦合、天然分布 | 复杂流程难以追踪，循环依赖风险 |
| **协调式** | 中央 Saga 协调器控制事务执行和补偿流程 | 流程清晰、易于监控 | 协调器成为耦合点和单点 |

```
// 编排式 Saga（事件驱动）
订单服务 → 库存服务 → 支付服务
  ↓ ↓ ↓  事件驱动，各服务监听上游事件并执行下一步

// 协调式 Saga（中心协调）
订单Saga协调器 → 创建订单 → 锁定库存 → 执行支付
                   ↓失败         ↓失败
                补偿订单     释放库存 ← 补偿
```

#### 前向恢复与后向恢复

- **前向恢复（Forward Recovery）**：重试失败的事务步骤，直到成功。适用于幂等操作、确定性成功概率高的场景。
- **后向恢复（Backward Recovery）**：执行补偿事务回滚已完成的步骤。适用于业务上需要原子性、失败后"恢复原状"的场景。

### 4.5 本地消息表 + 最大努力通知

这是一种通过消息中间件实现最终一致性的方案：

**核心思路**：
1. 业务操作和消息记录在**同一个本地事务**中写入（本地消息表与业务表同库）
2. 后台任务轮询消息表，将消息投递到消息队列
3. 消费者处理消息，通过 ACK 确认消费
4. 对于需要通知结果的场景，生产方提供查询接口，消费方处理后回调通知

```sql
-- 本地消息表结构
CREATE TABLE local_message (
    id          BIGINT PRIMARY KEY,
    business_id VARCHAR(64),
    topic       VARCHAR(128),
    payload     TEXT,
    status      TINYINT DEFAULT 0,  -- 0待发送 1已发送 2已消费
    retry_count INT DEFAULT 0,
    create_time DATETIME,
    update_time DATETIME
);
```

```java
// 业务操作 + 消息写入在同一事务中
@Transactional
public void placeOrder(Order order) {
    orderMapper.insert(order);                    // 业务操作
    messageMapper.insert(buildMessage(order));    // 消息记录
}
```

**最大努力通知**：投递方尽力通知，不保证 100% 到达；如果通知失败，依赖消费方的查询机制来获取最终结果。适用于对一致性要求不是 100% 的场景（如支付结果回调）。

### 4.6 AT 模式（Seata 原理）

Seata 的 AT 模式是一种无侵入的分布式事务方案，核心机制包括：

1. **全局事务**：由 TM（Transaction Manager）发起，TC（Transaction Coordinator）协调
2. **分支事务**：每个参与的 RM（Resource Manager）注册分支事务
3. **两阶段提交**：
   - **一阶段**：各 RM 执行 SQL 并提交本地事务，同时记录 **undo_log（前镜像 + 后镜像）**
   - **二阶段-提交**：TC 通知全局提交，各 RM 异步删除 undo_log
   - **二阶段-回滚**：TC 通知全局回滚，各 RM 根据 undo_log 生成反向 SQL 并执行

```sql
-- Seata undo_log 表结构
CREATE TABLE undo_log (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    branch_id     BIGINT,
    xid           VARCHAR(100),
    context       VARCHAR(128),
    rollback_info LONGBLOB,
    log_status    INT,
    log_created   DATETIME,
    log_modified  DATETIME
);
```

**关键设计**：
- undo_log 与业务操作在同一个本地事务中提交，保证原子性
- 使用全局锁防止二阶段回滚期间数据被修改
- 一阶段本地事务的提交释放了数据库锁，极大减少了 2PC 的锁持有时间
- 前后镜像对比生成反向 SQL，不需要业务方编写补偿逻辑

### 4.7 各方案对比表

| 方案 | 一致性 | 性能 | 侵入性 | 适用场景 |
|------|--------|------|--------|----------|
| 2PC | 强一致 | 低 | 低 | 短事务、跨库操作 |
| 3PC | 强一致 | 低 | 低 | 理论模型，生产极少使用 |
| TCC | 最终一致 | 中 | 高（需实现Try/Confirm/Cancel） | 资金扣减、库存锁定 |
| Saga 编排 | 最终一致 | 中高 | 中 | 长事务、跨服务流程 |
| Saga 协调 | 最终一致 | 中 | 中 | 流程可控的长事务 |
| 本地消息表 | 最终一致 | 高 | 低 | 异步解耦、非实时一致性 |
| 最大努力通知 | 最终一致 | 高 | 低 | 回调通知、弱依赖 |
| Seata AT | 弱一致/最终一致 | 中高 | 低（无侵入） | 希望低改造成本的场景 |

### 4.8 生产实践要点

1. **不要在所有场景使用分布式事务**——优先考虑业务重构（如合并服务减少跨库操作），实在不行再上分布式事务
2. **幂等性是分布式事务的基石**——所有 RPC 调用、消息消费、补偿操作都必须设计为幂等
3. **事务反查机制**：TCC 应提供反查接口，Saga 的协调器需要能查询分支事务状态
4. **监控与告警**：对所有分布式事务建立全链路追踪和异常告警，及时发现悬挂、空回滚、超时等问题
5. **避免长事务**：事务跨度越长，锁定资源越久，失败概率越大——长事务应拆分为 Saga

## 五、分布式锁

在分布式系统中，多个进程可能同时操作共享资源。分布式锁是保证互斥访问的基础原语。

### 5.1 基于数据库

#### 实现方式

利用数据库的唯一索引特性实现锁的排他性：

```sql
-- 尝试获取锁
INSERT INTO distributed_lock (lock_key, holder, expire_time) 
VALUES ('order_lock', 'node-1', NOW() + INTERVAL 30 SECOND);

-- 释放锁
DELETE FROM distributed_lock 
WHERE lock_key = 'order_lock' AND holder = 'node-1';
```

#### 要点

- 唯一索引保证同一时刻只有一个节点持有锁
- `expire_time` 防止持锁节点宕机后锁永远不被释放
- 后台心跳线程定期更新 `expire_time` 延长锁
- 释放时验证 `holder`，防止误删其他节点的锁

#### 优缺点

| 优点 | 缺点 |
|------|------|
| 方案简单，依赖现有基础设施 | 数据库是单点，性能瓶颈 |
| 事务支持，实现可靠 | 锁释放需要额外的超时检测 |
| 不需要额外组件 | 高并发场景下数据库压力大 |

### 5.2 基于 Redis

#### SET NX PX

```java
// 获取锁 (Jedis)
String result = jedis.set("lock_key", "unique_value", 
    SetParams.setParams().nx().px(30000));  // NX: 不存在才设置, PX: 过期30s

// 释放锁（Lua 脚本保证原子性）
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "   return redis.call('del', KEYS[1]) " +
                "else return 0 end";
jedis.eval(script, Collections.singletonList("lock_key"), 
    Collections.singletonList("unique_value"));
```

**关键点**：
- `unique_value`（UUID + 线程ID）保证释放时是锁的持有者，防止误删
- Lua 脚本保证 get + del 的原子性
- 过期时间需要合理设置——太短可能业务未完成锁已释放，太长可能长期阻塞

#### Redlock 算法（Redis 官方分布式锁）

在 Redis 主从架构中，主节点宕机后从节点可能尚未同步锁信息，导致两个客户端同时持有锁。Redlock 通过多实例规避此问题：

1. 获取当前时间（毫秒）
2. 依次向 N 个独立的 Redis 实例请求锁（SET NX PX），使用相同的 key 和随机 value
3. 获取锁的时间 = 当前时间 - 步骤1 的时间。只有当获取到超过半数（N/2+1）实例的锁，且总耗时 < 锁的有效时间，才算成功
4. 若获取失败，向所有实例发送释放请求

#### Redlock 争议

Martin Kleppmann 对 Redlock 提出了著名批评，核心观点：
- Redlock 依赖非单调的时钟假设——GC 停顿、时钟跳跃可能导致锁提前过期
- 分布式锁不是安全的——即使 Redlock 也无法保证 100% 互斥
- 推荐使用 **fencing token**（单调递增的序列号）来保证正确性

#### Redisson

Redisson 是 Java 生态最流行的 Redis 客户端（提供分布式锁），关键实现：

- **Watch Dog 看门狗**：默认锁过期时间 30s，后台每 10s 续期一次（`internalLockLeaseTime / 3`），进程存活期间锁永不过期
- **可重入锁**：通过 hash 结构存储锁持有计数（key → threadId → count），加锁时 count++，解锁时 count--
- **公平锁**：基于 Redis 队列 + Pub/Sub 实现等待线程排队唤醒
- **红锁**：`RedissonRedLock` 封装 Redlock 算法，组合多个 `RLock` 实例

```java
// Redisson 使用示例
RLock lock = redisson.getLock("order_lock");
try {
    // 尝试加锁，最多等待 10s，锁有效期 30s
    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
        // 业务逻辑
    }
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 5.3 基于 ZooKeeper / etcd

#### ZK 临时顺序节点

核心原理：
1. 所有客户端在同一父节点下创建**临时顺序节点**（EPHEMERAL + SEQUENTIAL）
2. 客户端获取所有子节点列表，若自己的节点序号最小则获得锁
3. 若不是最小，则对前一个节点注册 **Watch**，当前序节点删除（释放锁或客户端断开）时收到通知
4. 临时节点的特性：客户端 session 断开时节点自动删除，天然防止死锁

```
/locks/order/
    ├── _c_0000000001  ← 持锁（序号最小）
    ├── _c_0000000002  ← 等待，watch _c_0000000001
    └── _c_0000000003  ← 等待，watch _c_0000000002
```

#### etcd 实现

etcd 基于 Raft 实现 CP 模型的分布式锁，使用 `lease` + 事务实现：

```go
// etcd 分布式锁 (Go 伪代码)
txn := client.Txn(ctx).
    If(clientv3.Compare(clientv3.CreateRevision(key), "=", 0)).
    Then(clientv3.OpPut(key, val, clientv3.WithLease(leaseId))).
    Else(clientv3.OpGet(key))

resp, _ := txn.Commit()
if !resp.Succeeded {
    // 锁已被其他客户端持有
}
```

### 5.4 三种方案对比 + 选型

| 维度 | 数据库 | Redis | ZooKeeper/etcd |
|------|--------|-------|-----------------|
| 可靠性 | 中 | 中低（单实例）/ 中（Redlock） | 高（CP 系统） |
| 性能 | 低 | 极高 | 中 |
| 实现复杂度 | 低 | 低（单实例）/ 中（Redlock） | 中 |
| 死锁风险 | 需手动超时处理 | 需设过期 + 看门狗 | 临时节点自动处理 |
| 客户端阻塞等待 | 需要轮询 | Pub/Sub 或轮询 | Watch 机制 |
| 适用场景 | 低并发、已有 DB | 高并发、可接受低概率失效 | 强一致性要求 |

**选型建议**：
- **Redis**：高并发、可接受极低概率的锁失效（如防重复提交、缓存更新串行化）
- **ZK/etcd**：对一致性有严格要求的场景（如选主、任务调度唯一执行）
- **数据库**：没有 Redis/ZK 基础设施的小团队、低并发场景

## 六、分布式 ID

### 6.1 为什么需要分布式 ID

在单库系统中，数据库自增主键可以满足 ID 生成的唯一性需求。但分库分表后，多个数据库独立增长会导致 ID 冲突；而业务 ID 通常需要全局唯一、趋势递增（利于索引），因此需要专门的分布式 ID 生成方案。

### 6.2 UUID

```
550e8400-e29b-41d4-a716-446655440000
```

| 优点 | 缺点 |
|------|------|
| 本地生成，零网络开销 | 非自增、字符串占空间大（36字符） |
| 全局唯一，无单点 | 作为 InnoDB 聚簇索引时页分裂严重 |
| 实现简单 | 不包含时间信息，不可排序 |

适用场景：非主键的唯一标识（如日志 traceId、临时 token）。

### 6.3 数据库自增ID

通过独立的 ID 生成表（或号段表）获取全局自增 ID：

```sql
CREATE TABLE id_generator (
    biz_tag VARCHAR(32) PRIMARY KEY,
    max_id  BIGINT NOT NULL,
    step    INT DEFAULT 1000
);

-- 批量获取号段（原子操作）
UPDATE id_generator SET max_id = max_id + step WHERE biz_tag = 'order';
SELECT max_id FROM id_generator WHERE biz_tag = 'order';
```

**问题**：DB 成为单点瓶颈。一次 DB 交互只能获取 `step` 个 ID，用完后再请求。

### 6.4 号段模式（Leaf-segment）

美团 Leaf 的号段模式是对数据库方案的升级：

- 客户端先批量获取一个号段（如 [0, 1000]），缓存在本地
- 本地内存中基于号段分配 ID，无需每次访问 DB
- **双 buffer 优化**：当前号段消费到 10% 时，异步预加载下一个号段到备用 buffer，号段切换时无缝衔接

```java
// 双 buffer 号段模式核心
public class SegmentBuffer {
    private Segment[] segments = new Segment[2];  // 双 buffer
    private volatile int currentPos;              // 当前使用的 segment 索引
    
    public long getId() {
        long id = segments[currentPos].nextId();
        if (segments[currentPos].usageRate() > 0.9) {
            // 异步预加载备用 buffer
            asyncLoadSegment(1 - currentPos);
        }
        if (segments[currentPos].isExhausted()) {
            currentPos = 1 - currentPos;  // 切换 buffer
        }
        return id;
    }
}
```

优点：解决了 DB 的性能瓶颈，可实现 10w+ QPS；缺点：ID 不是严格全局自增（多实例各自消耗号段，时间线可能交错）。

### 6.5 Snowflake（雪花算法）

Twitter 开源的经典方案，ID 为 64 位长整型：

```
┌─┬───────────────────────────────────────┬──────────┬────────────────┐
│0│           41-bit timestamp            │10-bit    │   12-bit       │
│ │           (毫秒，约69年)              │ workerId │  sequence      │
└─┴───────────────────────────────────────┴──────────┴────────────────┘
```

每毫秒单机可生成 4096 个 ID，整体趋势递增。

#### 时钟回拨问题及解决方案

Snowflake 严重依赖机器时钟。时钟回拨会导致 ID 重复：

**方案一：抛异常**（Leaf-snowflake 默认）：
- 检测到回拨后拒绝服务，等待时钟追上

**方案二：使用历史时间戳**：
- 回拨发生时，沿用之前的时间戳生成 sequence，直到时钟追上

**方案三：扩展位 + 防重**：
- 在 ID 中增加 1-bit clock-back 标识，记录回拨事件

**方案四：借用未来时间**（百度 UidGenerator）：
- 基于 RingBuffer 预生成 ID，缓存最近一段时间的时间戳，回拨时直接使用缓存

```java
// 时钟回拨检测核心逻辑
public synchronized long nextId() {
    long currentTime = System.currentTimeMillis();
    if (currentTime < lastTimestamp) {
        long offset = lastTimestamp - currentTime;
        if (offset <= MAX_BACKWARD_MS) {  // 容忍小范围回拨（< 5ms）
            // 等待时钟追上
            Thread.sleep(offset);
            currentTime = System.currentTimeMillis();
        } else {
            throw new ClockBackwardsException("Clock moved backwards: " + offset);
        }
    }
    if (currentTime == lastTimestamp) {
        sequence = (sequence + 1) & SEQUENCE_MASK;
        if (sequence == 0) {
            currentTime = waitNextMillis(lastTimestamp);
        }
    } else {
        sequence = 0;
    }
    lastTimestamp = currentTime;
    return (currentTime - EPOCH) << TIMESTAMP_SHIFT
         | workerId << WORKER_ID_SHIFT
         | sequence;
}
```

### 6.6 Redis 自增

利用 Redis 单线程特性与 `INCR` / `INCRBY` 命令：

```
INCR order_id      → 返回 1001
INCRBY order_id 10 → 返回 1011（预取号段）
```

| 优点 | 缺点 |
|------|------|
| 高性能、天然递增 | 依赖 Redis 持久化（RDB/AOF），宕机可能丢失 |
| 实现极简 | 非严格趋势递增（主从切换后可能回退） |
| 可横向扩展（分段 key） | 需额外维护 Redis 集群 |

### 6.7 各方案对比

| 方案 | 趋势递增 | 高可用 | 性能 | 依赖 | 适用场景 |
|------|----------|--------|------|------|----------|
| UUID | 否 | 极高 | 极高 | 无 | 日志 traceId |
| DB 自增 | 是 | 低 | 低 | DB | 小规模 |
| Leaf-segment | 近似 | 高 | 极高 | DB + 自有服务 | 大规模分库分表 |
| Snowflake | 近似（时间序） | 高 | 极高 | 无（本地生成） | 通用高并发 |
| Redis | 是 | 中 | 高 | Redis | 已有 Redis 的中小规模 |

### 6.8 美团 Leaf 与百度 UidGenerator

**美团 Leaf**：双模式架构——号段模式 + Snowflake 模式，通过 ZooKeeper 注册 workerId 并做心跳续约。

**百度 UidGenerator**：
- **DefaultUidGenerator**：标准 Snowflake 实现，基于数据库表分配 workerId
- **CachedUidGenerator**：基于 RingBuffer（Disruptor 思想），提前批量生成 ID 并缓存在环形队列中，消除 Snowflake 的 synchronized 竞争；支持容忍时钟回拨

## 七、分布式一致性/共识算法

共识算法是实现 RSM（Replicated State Machine）的基础，可以说 WAL 和 RSM 是现代分布式系统的基石。共识算法主要分为主从协议、全序广播协议、Quorum 协议等类型。

### 7.1 Paxos

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

### 7.2 Raft

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

### 7.3 拜占庭将军问题

拜占庭将军问题是 Leslie Lamport 等人在 1982 年提出的经典问题，描述的是在存在**恶意节点**（拜占庭节点，可能发送任意虚假信息、主动破坏共识）的情况下，系统中的诚实节点如何达成一致。这是比一般的 crash-stop 或 crash-recovery 故障更为严苛的容错模型——拜占庭节点不仅可能"不响应"，还可能"说谎"。

#### PBFT（Practical Byzantine Fault Tolerance）

PBFT 是 Miguel Castro 和 Barbara Liskov 在 1999 年提出的第一个实用的拜占庭容错共识算法，能够在恶意节点不超过总数 1/3 的情况下保证系统的安全性和活性。PBFT 使用三阶段协议：

1. **Pre-prepare**：主节点（Primary）收到客户端请求后，分配一个序列号，并向所有副本节点广播 Pre-prepare 消息
2. **Prepare**：每个副本节点收到 Pre-prepare 消息并验证通过后，广播 Prepare 消息，表示已接收并认可该请求在当前序列号上的顺序
3. **Commit**：当副本节点收集到 2f+1 条匹配的 Prepare 消息（含自身）后，广播 Commit 消息；当收集到 2f+1 条 Commit 消息后，执行请求并响应客户端

系统总节点数 N ≥ 3f + 1，其中 f 为可容忍的拜占庭节点数。

其他拜占庭容错协议：
- **Ripple 协议**：基于信任节点列表（UNL）的拜占庭容错共识
- **Tendermint 协议**：Cosmos 生态使用的 BFT 共识引擎，结合了 PoS 和 PBFT
- **Stellar 协议**：基于联邦拜占庭协议（FBA）的共识机制

### 7.4 Gossip 协议

Gossip 是一种去中心化的信息传播协议，节点随机选择邻居进行信息交换，最终使信息在整个集群中传播。其本质模仿人类社会中的流言传播（epidemic protocol），适用于**最终一致性**场景。

Gossip 的核心特点：
- **去中心化**：没有中心协调节点，所有节点地位对等，任何节点都可以发起信息交换
- **容错性极强**：即使部分节点故障或网络丢包，信息通过多条路径最终仍能到达所有健康节点
- **可扩展**：节点数增加时传播延迟呈对数增长，不存在单点瓶颈
- **冗余通信**：信息会通过多个路径重复传播，带来一定的带宽冗余——这是去中心化的代价

单次 Gossip 通信通常包含三种模式：
- **Push**：节点 A 将自身数据推送给节点 B
- **Pull**：节点 A 从节点 B 拉取数据
- **Push-Pull**：节点 A 和 B 互相交换数据（效率最高，收敛最快）

典型实现：Cassandra 集群节点状态检测（Gossip 协议维护成员列表）、Consul 的 Serf 成员管理、Redis Cluster 的节点间通信

推荐阅读：[Gossip 协议：从流行病说起](https://www.nosuchfield.com/2019/03/12/Gossip-protocol-from-epidemics/)

### 7.5 区块链共识

区块链使用共识算法在去中心化的点对点网络中确定交易顺序，是在完全开放、无需许可的环境下达成共识的机制：

**Proof-of-Work（PoW）**：
工作量证明，节点通过解决计算难题（反复尝试 nonce 值使区块哈希满足目标难度）来竞争记账权，最先解决的节点获得出块奖励。比特币采用该机制——安全性由算力投入保证（攻击者需要掌握超过 51% 的算力），但能耗巨大，吞吐量低（约 7 TPS）。

**Proof-of-Stake（PoS）**：
权益证明，节点根据持有的代币数量和时间（币龄）来竞争记账权，持有更多代币的节点有更高概率被选为出块者。相比 PoW 更节能环保，以太坊已于 2022 年通过 The Merge 从 PoW 过渡到 PoS。

### 7.6 ZAB（ZooKeeper Atomic Broadcast）

ZAB 是 ZooKeeper 内部使用的原子广播协议，与 Raft 设计理念相似但实现不同：

- **Leader Election**：类似 Raft，基于 epoch（对应 Raft 的 term）选举 Leader
- **Broadcast（广播）**：Leader 将事务提议广播给所有 Follower，等待超过半数 ACK 后提交
- **Recovery（恢复）**：Leader 选举完成后，新 Leader 与 Follower 进行状态同步，确保所有已提交的事务都被复制

**与 Raft 的关键区别**：
- ZAB 按 **epoch** 划分阶段，每个 epoch 有且仅有一个 Leader
- ZAB 的 recovery 阶段更加复杂，需要处理前一个 Leader 未完成的事务
- ZK 通过 FIFO 的顺序保证写操作的顺序，但不保证跨客户端的线性一致性

### 7.7 EPaxos

EPaxos（Egalitarian Paxos）是对 Multi-Paxos 的重要优化，由 CMU 研究团队在 2013 年提出：

- **无需 Leader**：任何节点都可以发起提议，消除 Leader 瓶颈
- **依赖追踪**：通过记录提议之间的依赖关系来判断冲突，而非依赖全局顺序
- **快速路径**：无冲突的并发命令可以直接通过一轮通信达成一致（无需经过 Leader）
- **适合广域网（WAN）**：在跨地域部署场景下，去 Leader 化显著减少延迟

EPaxos 在理论上非常优美，但工程实现复杂，认知门槛较高。TiKV 等新一代分布式数据库已开始尝试引入 EPaxos 概念优化多 Region 场景下的共识效率。

## 八、租约（Lease）机制

### 8.1 原理

Lease 是一种带超时时间的授权机制，本质是"在 T 时间内我保证不把你持有的权限给别人"。与锁不同——锁倾向于抢占和阻塞，Lease 倾向于时间边界内的承诺。

Lease 的核心要素：
- **颁发**：授权方颁发一个带有效期（TTL）的授权
- **续约**：持约方在 Lease 过期之前可以申请续约
- **过期自动失效**：授权方在 Lease 过期后自动收回权限
- **到期前承诺**：授权方在 Lease 有效期内不会将相同权限授予第三方

### 8.2 与锁的区别

| 维度 | 锁 | Lease |
|------|-----|-------|
| 持有方式 | 显式申请和释放 | 颁发 + 自动过期 |
| 占有者故障 | 死锁（需超时检测） | 过期自动释放 |
| 续约 | 通常不可续 | 可以在到期前续约 |
| 语义 | 互斥访问 | 时间边界内的承诺 |
| 通信开销 | 每次加锁解锁 | 续约时偶尔通信 |

### 8.3 应用场景

- **GFS 的 Chunk Lease**：Master 授予 Chunkserver 主副本 Lease（初始 60s），主副本负责定序写操作，Lease 过期后 Master 可重新分配
- **分布式缓存**：缓存数据的有效期可视为一种 Lease，过期后从源重新拉取
- **Leader 选举**：Leader 持有 Lease，Follower 在 Lease 期内不发起选举
- **分布式锁的替代**：当操作可以在 Lease 边界内完成时，Lease 比锁更简洁

## 九、版本向量与时钟

在分布式系统中，**没有全局一致物理时钟**，各个节点的时钟可能不同步。因此需要逻辑时钟来定义事件的先后关系。

### 9.1 Lamport 逻辑时钟

Lamport 在 1978 年的论文《Time, Clocks, and the Ordering of Events》中提出了逻辑时钟（Logical Clock）：

**规则**：
1. 每个进程维护一个计数器 `C`，初始为 0
2. 每发生一个本地事件，`C = C + 1`
3. 发送消息时，将 `C` 值附带在消息中
4. 接收消息时，`C = max(本地C, 消息中的C) + 1`

**Happens-Before 关系**（记作 `a → b`）：
- 同一进程内：若 a 在 b 之前发生，则 `a → b`
- 跨进程：若 a 是发送事件，b 是对应的接收事件，则 `a → b`
- 传递性：若 `a → b` 且 `b → c`，则 `a → c`

```
进程 P1:  [a:1] → [b:2] ──────── [e:5] → [f:6]
                  ↓  msg(2)        ↑  msg(4)
进程 P2:       [c:3] → [d:4] ────┘
                  Lamport Time: C(d) < C(e)，但 d 和 e 无因果关系！
```

**局限性**：Lamport Clock 只能判断"如果 `a → b` 则 `C(a) < C(b)`"，但**不能反向推断**——`C(a) < C(b)` 不能得出 `a → b`（它们可能只是并发）。

### 9.2 Vector Clock（向量时钟）

向量时钟解决了 Lamport Clock 无法判断并发的问题：

- 集群中 N 个节点，每个节点维护一个长度为 N 的向量 `[v1, v2, ..., vN]`
- 节点 i 发生本地事件时，`v[i]++`
- 发送消息时附带完整的向量
- 接收消息时，对自己的向量逐元素取 max，然后 `v[自己]++`

**因果判断**：
```
A = [2, 1, 0]
B = [1, 1, 0]
→ A 和 B 有因果关系吗？
   A[i] >= B[i] for all i  → A 发生在 B 之后
   存在 A[i] < B[i] 且 B[j] < A[j] → A 和 B 是并发的
```

当 `A > B` 且 `B > A` 都不成立时，A 和 B 是**并发更新**——需要冲突解决策略（如 CRDT、Last-Write-Wins）。

**实际应用**：Amazon Dynamo 使用向量时钟追踪不同版本的因果历史，在读取时检测冲突并交由客户端或应用层解决。

### 9.3 Hybrid Logical Clock（HLC）

HLC（混合逻辑时钟）结合了**物理时钟**（贴近真实时间）和**逻辑时钟**（保证 happens-before 关系）：

```
HL Clock = (physical_time, logical_counter)

更新规则：
1. 本地事件：先取 max(本地物理时钟, 当前HLC的物理时间)
2. 逻辑计数器在小的时间窗口内递增
3. 接收消息：取 max(本地物理时钟, 消息中的物理时间)
```

HLC 的核心价值：
- 与物理时间接近，可读性强，便于调试和审计
- 保证 happens-before 因果一致性
- 被 CockroachDB、YugabyteDB 等新一代分布式数据库采用作为事务时间戳

## 十、限流算法

在分布式系统中，限流是保护服务不被突发流量击垮的关键手段。核心目标是：在保证系统承载能力的前提下，尽可能多地处理请求。

### 10.1 计数器（固定窗口）

最简单的方式：在固定的时间窗口内计数，超过阈值则拒绝。

```
窗口: [12:00:00 - 12:00:01)
计数器: 0 → 1 → 2 → ... → 100 → 拒绝
12:00:01: 计数器重置为 0
```

**临界问题**：在窗口边界处（如 12:00:00.900 - 12:00:01.100），两个窗口各允许 100 次，实际通过了 200 次——是期望值的两倍。

### 10.2 滑动窗口

固定窗口的改进版，将大窗口切分为多个小格子：

```
将 1 秒窗口切分为 10 个格子（每个 100ms）
当前时间: 12:00:01.350
计算范围: [12:00:00.350, 12:00:01.350] 内命中 10 个格子中的计数
```

滑动窗口更平滑，基本消除了固定窗口的临界突变问题，是生产环境的常用方案。

### 10.3 漏桶（Leaky Bucket）

漏桶算法的核心：

- 请求以任意速率到达，先放入**固定容量的桶**（队列）
- 桶以**恒定速率**漏出（处理）请求
- 桶满后，新请求被丢弃

**特点**：强制输出速率恒定，能有效应对突发流量（突发请求由桶容量缓存），但峰值处理能力被严格限制。适合需要平滑输出的场景（如消息队列生产速率控制）。

### 10.4 令牌桶（Token Bucket）

令牌桶是漏桶的"逆操作"，也是 Guava RateLimiter 和 Sentinel 限流的基础：

- 系统以**固定速率**向桶中放入令牌
- 桶有最大容量（`maxTokens`），满后丢弃多余令牌
- 请求到达时从桶中取令牌，取到则通过，否则拒绝或等待

```java
// Guava RateLimiter 令牌桶实现
RateLimiter limiter = RateLimiter.create(10.0);  // 每秒 10 个许可

public void handleRequest() {
    if (limiter.tryAcquire(100, TimeUnit.MILLISECONDS)) {
        // 处理请求
    } else {
        // 限流
        throw new RateLimitException("too many requests");
    }
}

// 预热模式：启动时低速率，逐步增加到稳定速率
RateLimiter warmupLimiter = RateLimiter.create(10.0, 5, TimeUnit.SECONDS);
```

#### SmoothBursty vs SmoothWarmingUp

Guava RateLimiter 提供两种令牌获取模式：

- **SmoothBursty**：默认模式，允许突发流量——桶中的令牌可以一次性取完，然后进入等待
- **SmoothWarmingUp**：预热模式——系统刚启动时令牌生成速率较低，逐步增加到稳定值，适合需要预热的系统（如缓存冷启动）

### 10.5 Sentinel

Sentinel 是阿里巴巴开源的流量治理组件，以"流量为切入点"提供：

- **限流**：支持 QPS/并发线程数/关联资源限流，基于滑动窗口统计
- **熔断**：基于响应时间/异常比例/异常数熔断
- **热点参数限流**：对"热点"参数值（如爆款商品 ID）进行更细粒度的限流
- **系统自适应限流**：根据系统 Load、CPU 使用率、平均 RT 等自动调整阈值

Sentinel 的限流算法基于改进的滑动窗口（LeapArray），使用环形数组存储各时间窗口的统计数据，滑动时只更新头尾两个窗口，时间复杂度 O(1)。

```java
// Sentinel 限流规则配置
FlowRule rule = new FlowRule();
rule.setResource("orderService");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);  // QPS 上限 100
FlowRuleManager.loadRules(Collections.singletonList(rule));

// 使用
Entry entry = null;
try {
    entry = SphU.entry("orderService");
    // 业务逻辑
} catch (BlockException e) {
    // 被限流或熔断
} finally {
    if (entry != null) entry.exit();
}
```

### 10.6 限流算法对比

| 算法 | 平滑度 | 突发容忍 | 实现复杂度 | 适用场景 |
|------|--------|----------|------------|----------|
| 固定窗口 | 低 | 否 | 极低 | 简单场景 |
| 滑动窗口 | 中 | 否 | 低 | 通用限流 |
| 漏桶 | 高 | 是（由桶容量） | 低 | 流量整型、消息队列 |
| 令牌桶 | 中高 | 是 | 中 | 突发流量场景、网关 |
| Sentinel | 高 | 可配置 | 中 | 生产级流量治理 |

## 十一、经典论文：Google File System (GFS)

GFS 是 Google 于 2003 年发表的经典分布式文件系统论文（《The Google File System》），其系统设计思想深刻影响了后续分布式存储系统的发展（HDFS 即为 GFS 的开源实现）。本节从前提假设、系统架构、一致性模型和系统交互四个方面分析 GFS 的核心设计。

### 11.1 前提与假设

GFS 在设计之初就明确了其与众不同的假设条件，这些假设决定了 GFS 很多与传统文件系统截然不同的设计决策：

- **节点/组件失效是常态**：廉价的商用服务器随时可能故障，因此监控、异常检测、容错和自动恢复是系统必备特性，而非可选增强
- **存储文件尺寸较大，通常是 GB 级别**：IO 操作和 block size 需要针对大文件重新设计
- **写文件主要是 append 操作，随机写几乎不存在；读文件主要是顺序读，随机读较少**：需要关注写的原子性，并针对 append 进行深度性能优化
- **系统设计主要关注高吞吐量（throughput），而不是低响应延迟（latency）**：批量数据处理场景下吞吐量远比单次延迟重要

### 11.2 系统架构

![image-20200127153256454](/distributed/assets/image-20200127153256454.png)

GFS 集群由一个 **Master** 和多个 **Chunkserver** 组成，被多个 **Client** 访问。文件被分割为固定大小的 **chunk**。

- **chunk**：文件存储在固定大小的 chunk 中，每个 chunk 由一个全局唯一的 64 位 chunk handle 标识
- **Master**：维护文件系统的所有元数据，包括 namespace（文件命名空间）、access control 信息、文件到 chunk 的映射、所有 chunk 的当前位置。同时控制系统活动，包括 chunk lease management（租约管理）、孤立 chunk 的垃圾回收、chunkserver 之间的 chunk 迁移。Master 与 chunkserver 之间通过 HeartBeat 心跳信息进行通信，包括下达指令和收集状态
- **Chunkserver**：出于可靠性的考虑，chunk 一般在多个 chunkserver 上进行 3 副本存储。Chunkserver 不缓存文件——因为 Linux 系统的 buffer cache 已经缓存了热点数据，在 chunkserver 上再做一层缓存没有额外收益
- **Client**：客户端不针对文件进行缓存，只对元数据进行缓存。原因：大部分应用需要扫描大文件，太大无法缓存；不做缓存也自然避免了缓存一致性问题

#### 11.2.1 Chunk Size

GFS 的 chunk size 为 **64MB**，远大于典型文件系统的 block size。利用 lazy space allocation（延迟空间分配）避免了内部碎片造成的空间浪费。

较大的 chunk size 提供了以下好处：
- 减少 Client 与 Master 的交互次数（一次交互可获取更大的操作范围）
- Client 与 Chunkserver 保持较长的 TCP 持久连接，减少连接建立开销
- 减小 Master 上存储的元数据体积，使元数据可以全部存储在内存中，带来极快的查找性能

副作用：一个小文件可能仅占用一个 chunk，当多客户端同时访问该文件时，承载该 chunk 的 chunkserver 会成为热点。

#### 11.2.2 元数据（Metadata）

Master 存储三种类型的元数据：
1. 文件和 chunk 的 namespace
2. 文件到 chunk 的映射
3. 每个 chunk 的副本位置

前两种元数据通过 **operation log** 持久化存储在 Master 的本地磁盘，并备份到远程机器上。

##### Chunk Locations

Master 不持久化存储 chunk 位置信息——而是在启动时以及 chunkserver 加入集群时，通过询问 chunkserver 来获取其持有的 chunk 列表。这种设计避免了 Master 与 chunkserver 之间因为保持 chunk 位置信息一致而带来的复杂性（chunkserver 自身就是 chunk 位置的最权威信息源）。

##### Operation Log

operation log 不仅是元数据的唯一持久化存储，也定义了并发操作的逻辑时间线——文件和 chunk 以及它们的版本都可以通过逻辑创建时间来唯一确定。

元数据的变更需要在对客户端可见之前完成持久化（写入本地磁盘并同步到远程机器），Master 通过批量收集日志记录来减少对系统吞吐量的影响。

Master 通过**重放 operation log** 来恢复文件系统状态。为缩短恢复时间，当 operation log 超过特定大小时，Master 会保存一个 **checkpoint**——恢复时只需加载最近的 checkpoint 并重放其后的 log。Checkpoint 以类似 B 树的形式组织，能够直接映射到内存中并且不需要额外解析就能用于 namespace 查找。生成 checkpoint 时，Master 切换到新的 operation log 文件，并在单独线程中异步创建 checkpoint；恢复时会验证 checkpoint 的完整性，跳过不完整的 checkpoint。

### 11.3 一致性模型

#### 11.3.1 保障（Guarantees）

文件 namespace 的修改（例如文件的创建）保证原子性——这些操作全部由 Master 处理：namespace 加锁保证原子性和正确性，operation log 定义了这些操作的全局总序。

文件块在修改之后的状态取决于修改的类型、是否成功、是否是并发操作。下表总结了各种情况下的结果：

![image-20200127205351882](/distributed/assets/image-20200127205351882.png)

其中 **defined** 表示文件块是一致的，并且对其的修改是完整可见的。随机写由于可能覆盖原有数据，因此在并发的情况下不能保证每次操作的可见性（可能出现 **consistent but undefined** 的状态——所有客户端看到相同的内容，但内容可能是多次写入的交错混合）。

GFS 通过以下两点来保证文件块在一系列成功修改之后的可见性，并保证包含最后一次修改：
1. 在其所有副本上按照**相同的顺序**应用修改
2. 使用 **chunk version number** 检测过期副本（版本号不匹配），过期副本会很快被垃圾回收清除

GFS 通过 Master 与所有 chunkserver 之间的定期握手检测失效的 chunkserver，并通过 **checksum** 检测数据损坏。一旦问题被发现，数据会从有效副本中恢复。只有在所有副本在 GFS 响应之前都丢失的情况下数据才算真正丢失（时间窗口通常为几分钟），并且 GFS 不会返回损坏的数据，而是返回明确的错误。

#### 11.3.2 对应用层的启示

GFS 的一致性模型是"弱化版"的——它不保证所有场景下的强一致性，而是将部分责任上移到应用层：

- 尽量使用 **append** 写操作（而非随机写），append 的原子性保证每条记录至少被原子地写入一次
- 应用层采用 **checkpoint 机制**来标记一致的数据点
- 应用层的记录应能 **自我验证（checksum）** 和 **自我识别（unique identifier）**，以检测和去除重复或损坏的数据

这种"放宽文件系统一致性要求、由应用层承担部分责任"的设计哲学，是 GFS 能够在廉价硬件上实现高吞吐的核心原因之一。

### 11.4 系统交互

#### 11.4.1 主副本及写顺序

Master 从一个 chunk 的所有副本中选择一个发放 **lease（租约）**，该副本称为**主副本（Primary）**。主副本对所有对该 chunk 的写操作选择一个序列化顺序，所有副本按此顺序执行写操作。Master 发放给主副本的 lease 初始超时时间为 60s，后续主副本通过 chunkserver 与 Master 之间的心跳信息发送延长租约的请求。

下图展示了一次写操作的控制流与数据流：

![image-20200127213256584](/distributed/assets/image-20200127213256584.png)

- **步骤 1**：Client 向 Master 询问 chunk 的主副本及所有次副本的位置。如当前没有主副本（lease 已过期），Master 选择一个副本下发 lease，使其成为新的主副本
- **步骤 2**：Master 响应主副本及次副本的位置，Client 缓存该信息。此后只有当主副本无法访问或主副本响应为不再持有 lease 时，Client 才需要再次访问 Master
- **步骤 3**：Client 向**所有**副本推送数据（以任意顺序），Chunkserver 将数据缓存在 LRU buffer 中
- **步骤 4**：Client 发送写请求到主副本，主副本为它收到的所有写操作分配一个连续的序列号，按序列号顺序应用到本地
- **步骤 5**：主副本将写请求转发给次副本，次副本按主副本指定的序列号顺序应用
- **步骤 6**：次副本响应主副本写入完成
- **步骤 7**：主副本响应 Client。若有任何副本写入出错，错误上报给 Client，由 Client 发起重试

注意：过大的写操作或跨越 chunk 边界的写操作，GFS Client 会将其拆分为多个独立的写操作——因此并发写操作可能导致文件块中混杂来自不同 Client 的数据片段。

#### 11.4.2 数据流解耦

GFS 一个关键设计是**数据流与控制流的解耦**：控制流从 Client → 主副本 → 次副本传递，而数据流以**数据管道（pipeline）**的形式在一条精心选择的 chunkserver 链路上线性传输。这种设计充分利用了每台机器的网络带宽，避开网络瓶颈和高延迟链路，最大化全双工带宽利用率，最小化推送数据的总延迟。

### 11.5 Master 的操作

#### 11.5.1 Namespace 管理与锁

Master 通过 namespace 锁保证文件操作的原子性和正确性，operation log 定义操作的全局顺序。GFS 的 namespace 是扁平的，没有传统文件系统的目录层级概念——路径只是逻辑上的命名约定，不对应物理目录结构。

### 11.6 容错与诊断

GFS 在容错方面依赖于三个核心机制：

- **Master 高可用**：Master 元数据通过 operation log 持久化到本地磁盘并远程备份，配合 shadow master（只读副本）在主 Master 故障时提供只读服务
- **Chunk 多副本**：chunk 默认 3 副本存储在不同机架的不同 chunkserver 上，任何单点故障不影响数据可用性
- **数据完整性检测**：通过 checksum 检测数据损坏并自动从正常副本恢复；通过 chunk version number 检测并回收过期的陈旧副本

## 十二、学习路线与工程实践

### 12.1 学习 RoadMap

分布式系统的学习需要循序渐进，建议按以下路线深入：

1. **WAL + RSM 基础**：精读 Stonebraker 的《Architecture of a Database System》，理解什么是 WAL（Write-Ahead Log）；阅读 Jim Gray 的 Transaction 经典著作，理解 Replicated State Machine 原理
2. **分布式共识算法**：研究分布式共识算法，这是日志复制协议的基石——全序广播（TOB）协议、Quorum-based 协议（Multi-Paxos、Raft）、Primary-backup 协议
3. **Partitioning & Replication + Local Storage Engine**：掌握 partitioning 和 replication 策略，了解 local storage engine（BDB、InnoDB、LevelDB）
4. **Local Storage Engine 深入**：阅读 BDB、InnoDB、LevelDB 源码；[LevelDB 源码阅读参考](http://www.grakra.com/2017/06/17/Leveldb-RTFSC/)
5. **磁盘和网络优化技术**：理解底层 IO 对分布式系统性能的影响
6. **分布式一致性（隔离性）**：深入理解隔离级别与分布式场景下的 trade-off
7. **分布式事务**：STM、MVCC、乐观锁等并发控制技术
8. **分布式查询优化**：SQL 优化器基础与分布式查询执行

经典学习路径建议：

- 从存储系统入手，Google 的老三篇（GFS、MapReduce、BigTable）入门，结合 MIT 6.824 课程实验
- 推荐阅读：《Distributed Systems for Fun and Profit》
- 从复制协议开始深入：Paxos → Raft → EPaxos
- 研究非 Google 系存储系统：Dynamo、Haystack 等最终一致性系统
- 分布式计算系统：MapReduce、Dremel、Spark、MillWheel、Sawzall
- SQL 优化器基础 + 分布式思维：F1、Spanner、HyPer、Impala、Presto、Kudu、AsterixDB

### 12.2 工程实现编程技巧

在分布式系统研发实践中，以下工程能力至关重要：

1. **Pure / Impure 分离**：像 Haskell 那样将纯逻辑与副作用分离，提高代码的可测试性和可推理能力
2. **幂等设计**：让部分有状态的模块成为 metal unit，配合 fail-stop 设计有效规避 Byzantine Fault
3. **元编程能力**：降低代码的冗余和耦合，使代码更适合扩展和组合
4. **超高系统编程能力**：对内存管理、并发控制、网络 IO 有深刻理解和精准控制能力
5. **单测与 Mock 测试设计能力**：能够在分布式环境下模拟各类故障场景
6. **漂亮的日志输出**：可观测性是分布式系统的生命线——高质量的日志是问题定位的第一手资料
7. **性能分析能力**：[Brendan Gregg 博客](http://www.brendangregg.com/) 是性能分析的经典参考
8. **使用 Docker 加速开发效率**：一致性的开发和测试环境是分布式系统团队的基础设施

### 12.3 周边知识体系

1. **精通 MySQL sharding**：不知道单库分表的痛，就不知道为什么需要坚定不移地搞分布式关系数据库
2. **精通大数据 BI 解决方案**：不知道传统数仓的痛，就不知道为什么需要全新的分布式列式数据库
3. **区块链、分布式计算引擎**：了解这些相关领域可以拓宽对去中心化和大规模计算的认知视野

## 十三、总结

分布式系统理论的核心问题是：在**网络不可靠、时钟不可信、节点会故障**的前提下，如何构建**正确、可靠、高性能**的系统。CAP 定理给了我们选择的方向，BASE 理论给了我们务实的哲学，一致性模型帮我们量化了"正确"的程度，而分布式事务、分布式锁、分布式 ID 则是将理论落地到工程的具体武器。

理解这些理论的"为什么"，比知道"怎么做"更为重要——因为技术方案总是在变化，但背后的取舍逻辑是永恒的。

## 参考资料

- Gilbert & Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services", 2002
- Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design", 2012 (PACELC)
- Pritchett, "BASE: An Acid Alternative", ACM Queue, 2008
- Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System", 1978
- Lamport, "The Part-Time Parliament", ACM TOCS, 1998 (Paxos)
- Ghemawat, Gobioff & Leung, "The Google File System", SOSP, 2003
- Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm", USENIX ATC, 2014 (Raft)
- Castro & Liskov, "Practical Byzantine Fault Tolerance", OSDI, 1999 (PBFT)
- Hector Garcia-Molina, "Sagas", 1987
- [Redlock 争议 - Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [美团 Leaf 分布式 ID 生成服务](https://tech.meituan.com/2017/04/21/mt-leaf.html)
- [百度 UidGenerator](https://github.com/baidu/uid-generator)
- [Seata AT 模式文档](https://seata.io/zh-cn/docs/dev/mode/at-mode.html)
- [Sentinel 官方文档](https://sentinelguard.io/zh-cn/)
- [分布式系统技术系列 - 选主算法](http://www.distorage.com/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E6%8A%80%E6%9C%AF%E7%B3%BB%E5%88%97-%E9%80%89%E4%B8%BB%E7%AE%97%E6%B3%95/)
- [分布式系统技术系列 - Gossip 算法](http://www.distorage.com/%e5%88%86%e5%b8%83%e5%bc%8f%e7%b3%bb%e7%bb%9f%e6%8a%80%e6%9c%af%e7%b3%bb%e5%88%97-gossip%e7%ae%97%e6%b3%95/)
- [Gossip 协议：从流行病说起](https://www.nosuchfield.com/2019/03/12/Gossip-protocol-from-epidemics/)
- [Paxos 算法详解](https://zhuanlan.zhihu.com/p/27335748)
- [知乎：分布式系统学习讨论](https://www.zhihu.com/question/62464757/answer/206613596)
- [Brendan Gregg 性能分析](http://www.brendangregg.com/)
