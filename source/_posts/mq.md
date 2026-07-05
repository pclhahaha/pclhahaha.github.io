---
title: 消息队列深度解析
date: 2025-01-01 00:00:00
updated: 2025-01-01 00:00:00
tags:
  - 消息队列
  - Kafka
  - RocketMQ
  - RabbitMQ
categories:
  - 分布式
---

[TOC]

## 一、为什么需要消息队列

### 1.1 三大核心价值

消息队列不是银弹，但在以下三个场景中价值极高：

**解耦**：生产者和消费者不再直接调用，仅通过 MQ 交换消息。上游服务不需要知道下游有哪些服务、各自的地址和接口，新增消费者时上游代码零改动。

```
同步调用（耦合）:                      异步消息（解耦）:
  OrderService                         OrderService
    ├─► InventoryService                   │
    ├─► CouponService                   [发布订单事件]
    └─► NotificationService                 │
                                         MQ Broker
                                       ↙    ↓    ↘
                               Inventory Coupon Notification
```

**异步**：非核心链路从同步调用改为异步消息，显著降低接口响应时间。下单时优惠券、积分、短信等逻辑从关键路径剥离。

```
同步:  下单(50ms) → 扣库存(20ms) → 发优惠券(30ms) → 发短信(100ms) → 返回(总耗时200ms)
异步:  下单(50ms) → 扣库存(20ms) → 返回(70ms)
                   [异步] → 发优惠券 + 发短信
```

**削峰**：突发流量先存入 MQ，后端按自身处理能力匀速消费，避免流量尖峰击垮服务。

```
秒杀场景:  前端流量 100000 QPS → MQ 积压 → 后端匀速消费 5000 QPS
```

没有 MQ，后端需要按峰值扩容，资源利用率极低（日常可能只用 10%）。有了 MQ 削峰填谷，按平均水位配置即可。

### 1.2 使用场景决策树

```
需要消息队列吗?
├─ 需要多个下游服务同时消费同一事件? → 是 → 用 MQ (发布订阅)
├─ 非核心逻辑拖慢了主流程响应时间? → 是 → 用 MQ (异步解耦)
├─ 流量有尖峰, 后端处理能力有限? → 是 → 用 MQ (削峰填谷)
├─ 需要跨系统/跨语言通信? → 是 → 用 MQ
├─ 需要保证最终一致性(分布式事务)? → 是 → 用事务消息
├─ 单服务内部异步处理, 不需要持久化? → 直接用线程池/协程
├─ 数据量大(日志/埋点), 对可靠性要求不高? → Kafka
├─ 金融/电商场景, 事务消息刚需? → RocketMQ
└─ 灵活路由, 低延迟, 管理方便? → RabbitMQ
```

### 1.3 引入 MQ 的代价

任何技术选型都是 trade-off：

| 代价 | 说明 |
|------|------|
| 系统复杂度增加 | 多了 Broker 集群需要部署、监控、运维 |
| 一致性问题 | 消息可能丢失、重复,变成最终一致性 |
| 延迟增加 | 多了一跳转发的网络开销 |
| 调试困难 | 链路追踪变复杂, 需要关联消息 ID |
| 数据一致性 | 生产者/消费者事务需要额外设计 |

> **生产经验**：能用同步解决的问题不要强上 MQ。先问自己：真的需要异步吗？真的需要广播给多消费者吗？MQ 引入的是复杂度，不要为了"架构好看"而引入。

---

## 二、Kafka

Kafka 最初由 LinkedIn 开发并开源，2011年成为 Apache 顶级项目。定位**分布式流平台**，核心能力是超高吞吐的顺序读写。LinkedIn 单集群日处理万亿级消息。

### 2.1 架构概览

```
              ┌────────────────────────────────┐
              │          ZooKeeper/KRaft        │
              │     (元数据/Controller选举)       │
              └────┬──────────────────┬─────────┘
                   │                  │
    ┌──────────────┼──────────────────┼──────────────┐
    │              ▼                  ▼              │
    │         Controller          ~~~~~~~~~~         │
    │   (管理分区Leader选举)        Replicas          │
    │                                               │
    │  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
    │  │ Broker1 │  │ Broker2 │  │ Broker3 │       │
    │  │ T0P0(L) │  │ T0P0(F) │  │ T0P1(L) │ ...   │
    │  │ T0P1(F) │  │ T0P2(L) │  │ T0P2(F) │       │
    │  └─────────┘  └─────────┘  └─────────┘       │
    └──────────────────┬───────────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │Producer │    │Consumer │    │Consumer │
  │  App1   │    │ Group A │    │ Group B │
  └─────────┘    └─────────┘    └─────────┘
```

**核心组件**：

| 组件 | 职责 |
|------|------|
| Broker | Kafka 服务进程，一个 Kafka 集群由多个 Broker 组成 |
| Topic | 逻辑消息分类，生产者按 Topic 发送，消费者按 Topic 订阅 |
| Partition | Topic 物理分区，每个 Partition 是一个有序、不可变的消息序列，持久化在磁盘 |
| Producer | 消息生产者，负责将消息推送到指定 Topic 的指定 Partition |
| Consumer | 消息消费者，以 Pull 模式从 Broker 拉取消息 |
| Consumer Group | 消费组，组内消费者共享 Partitions，实现负载均衡 |
| Controller | 集群控制器（最先启动的 Broker），负责 Partition Leader 选举 |
| Coordinator | 为每个 Consumer Group 分配一个 Broker 作为 GroupCoordinator，管理 offset 提交和再平衡 |

**核心设计思想**：

Kafka 的吞吐量来自于**全面拥抱顺序 IO**。Broker 收到消息后直接追加到 Partition 日志末尾，不做任何索引更新（仅在内存中维护 offset-position 映射）。Consumer 拉取消息时，Broker 按 offset 读取，整个链路全是顺序读写。

### 2.2 Topic、Partition 与 Segment

**Partition 为什么关键**：

一个 Topic 可以有 N 个 Partition，分布在不同的 Broker 上，这是 Kafka 水平扩展的基石。Partition 内部消息严格有序，但不同 Partition 之间无法保证顺序。

```
Topic: order-events (3 partitions)

Partition 0: [msg0] [msg1] [msg2] [msg3] → 写入/读取顺序有序
Partition 1: [msg0] [msg1] [msg2]        → 与 P0 之间无序
Partition 2: [msg0] [msg1]               → 多 Partition = 并发读写
```

**Partition 路由**：Producer 发送消息时通过 `key.hashCode() % partitionCount` 确定目标 Partition。不指定 key 则轮询或随机。

```java
// Producer 发送代码
ProducerRecord<String, String> record = new ProducerRecord<>("order-events", "orderId123", "message body");
// 指定了 key("orderId123") → 相同 key 进入同一 Partition → 保证该 key 的消息有序
```

**Segment**：每个 Partition 物理上由多个 Segment 文件组成。Segment 是 Kafka 存储的基本单元。

```
Partition 0 目录:
/var/kafka-logs/order-events-0/
├── 00000000000000000000.log       ← 第一个 Segment 数据文件
├── 00000000000000000000.index     ← 稀疏索引
├── 00000000000000000000.timeindex ← 时间索引
├── 00000000000000500000.log       ← 第二个 Segment(offset >= 500000)
├── 00000000000000500000.index
├── leader-epoch-checkpoint        ← Leader Epoch 文件
└── partition.metadata
```

Segment 文件名为基准 offset（该 Segment 第一条消息的 offset），如 `00000000000000500000.log` 表示该文件存储从 offset=500000 开始的消息。Segment 有两个配置参数：

- `log.segment.bytes`：Segment 文件大小上限（默认 1GB），达到后自动滚动创建新 Segment
- `log.retention.hours`：消息保留时间（默认 168 小时/7 天），超时的 Segment 直接删除

### 2.3 消息存储与零拷贝

**消息格式 (v2)**：

Kafka 使用 Record Batch 作为最小读写单元，一个批次包含多条消息：

```
Record Batch:
┌──────────────────────────────────────┐
│ baseOffset: int64                    │ ← 批次起始 offset
│ length: int32                        │ ← 批次总长度
│ partitionLeaderEpoch: int32          │
│ magic: int8 (=2)                     │ ← V2 版本
│ crc: int32                           │
│ attributes: int16 (压缩类型)          │
│ lastOffsetDelta: int32               │
│ firstTimestamp: int64                │
│ lastTimestamp: int64                 │
│ producerId: int64 (幂等)             │
│ producerEpoch: int16                 │
│ baseSequence: int32                  │
│ records: Record[]                    │ ← 消息数组(可变部分)
└──────────────────────────────────────┘
```

V2 版本的 Record Batch 复用一条通用头部，减少了空间开销。对比 V1 每条消息独立头部，V2 的批处理压缩率更高。

**稀疏索引**：Kafka 并不为每条消息建立索引，而是每隔 `log.index.interval.bytes`（默认 4KB）在 `.index` 文件中记录一条 offset→position 映射。查找时二分定位到最近的索引条目，再顺序扫描剩余部分——用极小的内存和磁盘代价换来 O(logN) 的查找。

```
查找 offset=500000 的消息:
  .index 文件二分定位 → 最近索引条目 (offset=499990, pos=2GB)
  → 从 .log 文件的 2GB 位置顺序扫描 → 找到 offset=500000
```

**零拷贝 sendfile**：

这是 Kafka 极高吞吐的核心技术。传统网络发送过程：

```
传统 read+write（4 次拷贝，2 次系统调用）:
  DMA copy: 磁盘 → kernel buffer (read buffer)
  CPU copy: kernel buffer → user buffer (应用程序)
  CPU copy: user buffer → socket buffer
  DMA copy: socket buffer → 网卡

sendfile 零拷贝（2 次拷贝，1 次系统调用）:
  DMA copy: 磁盘 → kernel buffer (pagecache)
  DMA copy: kernel buffer → 网卡
```

在 Linux 中 Kafka 使用 `FileChannel.transferTo()`，底层调用 `sendfile()` 系统调用。数据从 pagecache 直接 DMA 到网卡，完全绕过用户态，CPU 零参与。这是 Kafka 单机能达到百万 QPS 的关键。

**pagecache 加速**：Kafka 不完全依赖零拷贝。OS 的页缓存会缓存活跃 Segment。消费者通常追最新消息，这些数据大概率在 pagecache 中，此时读取完全不走磁盘 IO，等同于内存读取。

> **生产经验**：Kafka 不推荐使用超大堆内存（通常 5-6GB 即可），把剩余内存留给 OS 的 pagecache。Kafka 重度依赖 pagecache 做读缓存和写缓冲。JVM 堆反而越分配越浪费（GC 压力）。

### 2.4 分区分配策略

Consumer Group 内消费者如何分配 Partition 的消费权，由 `partition.assignment.strategy` 控制。各策略对比：

**RangeAssignor（默认，已不推荐）**：

每个 Topic 独立分配，按 Partition 序号连续分段发给 Group 内消费者。

```
Topic-A(6分区) + 3个消费者:
  C0 → P0, P1
  C1 → P2, P3
  C2 → P4, P5
```

问题：如果 Group 订阅了多个 Topic，部分消费者会"持续过载"。假设 Topic-A 6 分区，Topic-B 3 分区，3 个消费者：
- C0 → Topic-A P0P1 + Topic-B P0 **= 3 个分区**
- C1 → Topic-A P2P3 + Topic-B P1 = 3 个分区
- C2 → Topic-A P4P5 + Topic-B P2 = 3 个分区

但如果 Topic-B 只有 1 个分区：
- C0 → Topic-A P0P1 + Topic-B P0 = 3 个分区
- C1 → Topic-A P2P3 = 2 个分区
- C2 → Topic-A P4P5 = 2 个分区

这还不严重。更严重的是当消费者数量变化触发再平衡时，Range 策略导致**数据倾斜的概率极高**。

**RoundRobinAssignor**：

将所有订阅的 Topic 的 Partitions 统一排序后轮询分配。解决了 Range 在单 Topic 下的倾斜问题，但再平衡时大面积重新分配。

**StickyAssignor**：

在 RoundRobin 基础上，记录上次分配方案，再平衡时**尽可能保持原有分配不变**。减少不必要的 Partition 迁移，降低再平衡开销。

```
初始分配(3消费者, 6分区):
  C0 → P0, P1   C1 → P2, P3   C2 → P4, P5

C2 退出, Sticky 策略:
  C0 → P0, P1, P4     ← 新增 P4(来自 C2)
  C1 → P2, P3, P5     ← 新增 P5(来自 C2)
  (原有 P0-P3 分配未动——"粘性")
```

非 Sticky 策略(如 RoundRobin)可能把所有 6 个分区重新打散到 C0、C1，全部重新分配。

**CooperativeStickyAssignor（推荐）**：

Kafka 2.4+ 引入，是 StickyAssignor 的升级版。传统 Eager 协议下再平衡会**先撤销所有消费者 → 再重新分配**，期间所有消费暂停（Stop-The-World）。Cooperative Sticky 基于 `CooperativeRebalanceProtocol`，允许消费者**分批逐步**释放和分配 Partition，再平衡期间未迁移的分区继续消费——大幅减少短暂消费中断。

```
Eager 协议:   全部 Revoke → STOP → 全部分配
Cooperative:  C2退出→ C2的P4→慢慢迁移给C0, P5→慢慢迁移给C1
              迁移期间 C0(P0P1) 和 C1(P2P3) 照常消费
```

```properties
# 生产推荐配置(Kafka 2.4+)
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

### 2.5 副本机制

**Replica 分类**：

每个 Partition 有 N 个副本（默认 `replication.factor=3`），其中只有 Leader 负责读写，Follower 只从 Leader 同步数据。

```
Partition 0:  Broker1(Leader)  Broker2(Follower)  Broker3(Follower)
                ↑ 读写           ↑ 同步             ↑ 同步
```

所有读**写**请求只发给 Leader，Follower 的唯一职责是同步——这是典型的 Leader-Follower（主从）同步模型，牺牲读的横向扩展换取强一致语义。Kafka 不提供主从读写分离。

**ISR (In-Sync Replicas)**：

Leader 维护一个 ISR 列表，记录与 Leader 同步"足够快"的 Follower 集合。判断标准由两个参数控制：
- `replica.lag.time.max.ms`（默认 30s）：Follower 超过此时间未发起 fetch 请求则踢出 ISR
- `replica.lag.max.messages`（Kafka 0.9+ 已移除此参数）：原按消息条数判断，现已废弃，因为消息大小差异大时缺乏统一标尺

```
ISR 变化示例:
  初始: ISR = [B1(Leader), B2, B3]
  B2 网络抖动, 30s 未同步 → ISR = [B1, B3]
  B2 恢复追上 → ISR = [B1, B3, B2]
```

**HW (High Watermark) 和 LEO (Log End Offset)**：

| 缩写 | 全称 | 含义 |
|------|------|------|
| LEO | Log End Offset | 当前日志最后一条消息的 offset+1（下一写入位置） |
| HW | High Watermark | 所有 ISR 副本都已确认的最小 LEO，消费者只能读到 HW 之前的消息 |

```
B1(Leader):  [msg0] [msg1] [msg2] [msg3] [   ]
              ↑                        ↑
              HW=2                    LEO=4

B2(Follower): [msg0] [msg1] [   ]
              ↑          ↑
              HW=2      LEO=2

B3(Follower): [msg0] [msg1] [msg2] [   ]
              ↑                    ↑
              HW=2                LEO=3

HW = min(LEO_B1, LEO_B2, LEO_B3) = min(4, 2, 3) = 2
```

消费者最多读到 offset=1（即 msg1）。即使 Leader 上有 msg2、msg3，但只要有一个 ISR 副本未同步，这些消息就不可见——这就是 Kafka 如何用 HW 提供**已提交(committed)**语义。

**Leader 故障恢复流程**：

1. Controller 监听到 Leader Broker 失联（ZooKeeper watch / KRaft heartbeat）
2. Controller 从 ISR 中选出新 Leader（优选 AR 列表中靠前的）
3. Controller 向所有 Broker 广播新的 LeaderAndIsr 请求
4. 各 Follower 收到后，将自己的 HW 之上的消息截断（truncate），与新 Leader 对齐

**unclean.leader.election.enable**：

当 ISR 为空（所有 ISR 副本宕机）时：
- `true`：允许从 OSR（Out-of-Sync Replica，非同步副本）中选举 Leader → 可能会丢消息，但服务可用
- `false`（默认）：宁可服务不可用也不丢消息

```
CAP 权衡:
  unclean.leader.election.enable=true  → 牺牲 C（一致性）, 保 A（可用性）
  unclean.leader.election.enable=false → 牺牲 A, 保 C
```

> **生产经验**：金融/支付场景必须设为 `false`。日志/监控场景可以设为 `true`（丢几条日志可接受，但不能中断）。

### 2.6 Leader Epoch：为什么 HW 不够

**HW 截断的数据丢失问题**：

```
场景1(consumer 读完 follower 后 leader 宕机):
  Leader LEO=2, HW=2   Follower LEO=2, HW=2
  消费者从 Follower 读到 msg0, msg1 (HW=2)
  Leader 收到 msg2(LEO=3), Follower 未同步
  Leader 宕机 → Follower 当选 Leader → 截断到 HW=2
  结果: msg2 丢失但 msg0-msg1 安全

场景2(HW 截断数据差异问题):
  Leader LEO=2, HW=2   Follower LEO=1, HW=1
  Leader 宕机 → Follower 当选 Leader(LEO=1)
  之前 Leader 的 msg1 是 leader epoch=0 产出的
  新 Leader 产出 msg1'(leader epoch=1)
  旧 Leader 恢复, 发现 HW 差异 → 截断高位数据
  但 epoch 语义下, 旧 Leader 的 msg1 vs 新 Leader 的 msg1' 需要能区分
```

**Leader Epoch 方案**（Kafka 0.11+）：

Leader Epoch 本质是每个 Leader 任期内分配一个单调递增的序号。每个 Broker 维护 `leader-epoch-checkpoint` 文件：

```
Leader Epoch  checkpoint:
0  offset=0    ← 第一个 Leader 任期起始 offset
1  offset=100  ← 第二个 Leader 任期起始 offset
2  offset=250  ← 第三个 Leader 任期起始 offset
```

当 Follower 从新 Leader 同步时，先发送自己的 Leader Epoch → 新 Leader 返回对应的 offset → Follower 据此决定截断点，而不是仅凭 HW。

```
Follower 恢复步骤:
  1. Follower 向新 Leader 发送 LeaderEpochRequest(epoch=1)
  2. Leader 返回 LeaderEpochEndOffset(epoch=1, endOffset=100)
  3. Follower 截断到 offset=100
  4. 从 offset=100 开始同步

这样避免了仅凭 HW 截断导致的 "少截"(数据歧义) 或 "多截"(丢失已提交数据) 问题。
```

### 2.7 幂等生产者与事务

**幂等生产者**（Exactly Once, 单 Partition）：

原理：每个 Producer 分配 PID（Producer ID），每条消息携带 `<PID, Partition, SequenceNumber>` 三元组。Broker 维护每个 PID 的最近 5 个序列号，检测到重复 SequenceNumber 时直接丢弃。

```
启用幂等: enable.idempotence=true

Producer 发送 msg0(seq=0) → Broker 写入成功, 记录 PID:seq=0
Producer 重试 msg0(seq=0) → Broker 检测 seq=0 已存在 → 丢弃, 但返回成功 ACK
Producer 发送 msg1(seq=1) → Broker 写入成功, 记录 PID:seq=1
```

> 注意：幂等只能保证**单分区内**的 Exactly Once，跨分区无法保证。另外 Producer 重启后 PID 会变，不能跨会话去重。

**事务**（Exactly Once，跨分区）：

Kafka 事务提供原子性地写入多个 Topic+Partition。实现基于两阶段提交：

```
beginTransaction()
  send T0P0 msg-A
  send T1P0 msg-B
  send T2P0 msg-C
  // 消费-处理-生产 (consume-transform-produce)
commitTransaction()
// 要么 A+B+C 全部提交, 要么全部回滚
```

事务实现中 Kafka 引入了 Transaction Coordinator（事务协调器），原理：
1. Producer 向 TC 注册，获得 transactionalId（对应 PID 和 epoch）
2. TC 在 `__transaction_state` 内部 Topic 中记录事务状态（Ongoing → PrepareCommit → CompleteCommit）
3. 提交时 TC 向涉及的各 Partition Leader 发送 Marker 消息（Commit/Abort），Consumer 自动跳过未提交或已回滚的数据
4. 失败恢复时 TC 扫描 `__transaction_state` 完成待定事务

**消费者事务隔离**：`isolation.level=read_committed`（默认 `read_uncommitted`）。设置为 `read_committed` 后，消费者只读取已提交的消息，事务中未提交的消息不可见。

```java
// 事务 Producer 配置
props.put("transactional.id", "tx-order-001");
props.put("enable.idempotence", true);  // 事务依赖幂等

// 消费-处理-生产示例
producer.initTransactions();
try {
    producer.beginTransaction();
    for (ConsumerRecord<String, String> record : consumer.poll(100)) {
        // 处理消息
        producer.send(new ProducerRecord<>("output-topic", result));
        // 提交 offset (通过 Producer 而非 consumer.commitSync)
        producer.sendOffsetsToTransaction(offsetMap, "consumer-group-id");
    }
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

### 2.8 消费者与再平衡

**消费者拉模式**：Kafka 消费者采用 Pull（拉）模型，而非 Push（推）。原因：
- Pull 允许消费者按自身处理能力拉取，天然支持流控
- 如果推模式，Broker 不知道消费者处理速度，要么消费过慢积压、要么推送过快 OOM

```java
// 消费者核心循环
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
    // 手动提交 offset
    consumer.commitSync();
}
```

拉模式的实现：`poll()` 并非一次网络请求，而是预取+Fetcher 缓存。Consumer 在后台异步拉取消息放入内部队列，`poll()` 从队列获取。

**自动提交 vs 手动提交**：

```
enable.auto.commit=false（生产环境必须 false）

自动提交危险场景:
  poll 获取 100 条消息
  处理到第 50 条时, 自动提交间隔触发 → 提交 offset=100
  随后消费者宕机 → 剩余 50 条消息丢失(offset 已提交,重启后从 101 开始)

手动提交:
  for each record:
    process(record)          // 先处理
    consumer.commitSync()    // 逐条提交(不推荐,性能差)
  // 或者:
  processBatch(records)      // 整批处理
  consumer.commitSync()      // 批量提交(容易重复消费,但 at-least-once)
```

**再平衡(ReBalance)**：

Consumer Group 发生以下情况时触发再平衡：
- 新消费者加入 Group
- 消费者离开 Group（心跳超时）
- 订阅的 Topic 的 Partition 数量变化
- 消费者取消订阅

再平衡过程（Eager 协议）：
1. GroupCoordinator 通知所有消费者 Revoke（撤销所有分区）
2. 消费者提交当前 offset → 释放分区
3. Coordinator 重新分配分区方案
4. 通知消费者 Assign（分配新分区）
5. 消费者从上次提交的 offset 恢复消费

过程中 Step1-Step4 期间所有消费者暂停消费，是 `Stop-The-World` 式的。

```
Kafka 再平衡优化:
  max.poll.interval.ms=300000     (poll 间隔上限, 超时触发再平衡)
  session.timeout.ms=45000        (会话超时)
  heartbeat.interval.ms=3000      (心跳间隔, 建议 < session.timeout.ms/3)
  group.initial.rebalance.delay.ms=3000  (初始再平衡延迟, 避免短暂加入就退出)
```

**重复消费与漏消费**：

| 场景 | 原因 | 结果 |
|------|------|------|
| `at-most-once` | 先提交 offset 再处理 | 可能漏消费（处理失败但 offset 已提交） |
| `at-least-once` | 先处理再提交 offset | 可能重复消费（提交失败，重启后重复处理） |
| `exactly-once` | 消费+处理+提交在事务中 | 精确一次 |

```java
// 推荐的手动提交代码(封装)
for (ConsumerRecord<String, String> record : records) {
    String msgId = record.value();  // 解析业务 ID
    if (redis.setnx("dedup:" + msgId, "1", 86400)) {
        // 首次处理(幂等去重)
        process(record);
    }
}
// 记录处理成功的最大 offset, 批量提交
```

Kafka 的 exactly-once 消费只能通过事务模式实现（`consumer-producer-transaction`），即 consume-transform-produce 场景。

### 2.9 Kafka 性能优化

**为什么 Kafka 这么快**：

```
单机百万 QPS 的秘密:
  1. 顺序写磁盘 (追加, 无磁头寻道, 600MB/s vs 随机写 100KB/s)
  2. pagecache (写入到 OS 缓存即返回, OS 异步刷盘)
  3. 零拷贝 sendfile (消费端读取不经过用户态)
  4. 批量处理 (Producer 攒批, Broker 批量刷盘, Consumer 批量拉取)
  5. 分区并行 (多分区 = 多线程并发读写)
  6. 无状态 Broker (Broker 不记录消费状态, metadata 全部在 Consumer 端)
```

**Producer 端优化参数**：

```properties
# 批次大小 - 调整到吞吐 vs 延迟的甜蜜点
batch.size=16384                     # 默认 16KB, 可调到 64KB~512KB
linger.ms=5                          # 攒批等待时间, 默认 0, 建议 1-5ms

# 压缩 - 以 CPU 换吞吐
compression.type=lz4                 # lz4 在 CPU 和压缩率之间平衡最好
# gzip: 压缩率高, CPU 开销大; snappy: CPU 低, 压缩率一般; lz4: 最佳折中

# 内存管理
buffer.memory=33554432              # Producer 缓冲总大小(默认 32MB)
max.in.flight.requests.per.connection=5  # 同时发送的最大未确认请求数

# 可靠性
acks=all                            # 所有 ISR 副本确认后才返回
retries=3                           # 重试次数(已启用幂等的场景, 默认 Integer.MAX_VALUE)
enable.idempotence=true             # 开启幂等(自动配置 acks=all, retries=MAX, max.in.flight=5)
```

**Broker 端优化参数**：

```properties
# --- 日志刷盘 ---
log.flush.interval.messages=10000   # 多少条消息刷盘(默认 Long.MAX, 几乎不主动刷, 信任 OS)
log.flush.interval.ms=1000          # 刷盘间隔
# Kafka 的持久化信条: 相信 OS 的 pagecache, 尽量不过早强制刷盘

# --- 网络 ---
num.network.threads=8               # 网络线程数(接收请求、发送响应)
num.io.threads=16                   # IO 线程数(处理实际读写)
socket.send.buffer.bytes=102400     # SO_SNDBUF
socket.receive.buffer.bytes=102400  # SO_RCVBUF

# --- 内存 ---
# Kafka 的堆内存 = 6GB 够用了, 关键是让 OS 有更多内存做 pagecache
# 假设机器 64GB 物理内存: heap=6GB, 剩余 58GB 给 OS(pagecache+其他)

# --- 分区 ---
num.partitions=3                    # 单 Topic 默认分区数, 根据吞吐和集群规模设定
# 分区不是越多越好: 分区越多 → Controller 负载越大 → 故障恢复时间越长
```

**Consumer 端优化参数**：

```properties
# 拉取参数
fetch.min.bytes=1024                # 最少拉取字节数(累计到此才返回)
fetch.max.wait.ms=500               # 最长等待时间(即使未满 fetch.min.bytes 也返回)
max.partition.fetch.bytes=1048576   # 单次从单分区拉取最大值(1MB)
fetch.max.bytes=52428800            # 单次 poll 总拉取最大值(50MB)

# 消费吞吐优化
max.poll.records=500                # 单次 poll 最大记录数
```

**压缩对比测试数据（参考）**：

| 压缩算法 | 压缩率 | CPU 开销 | 吞吐量 | 网络带宽节省 | 推荐场景 |
|----------|--------|----------|--------|-------------|----------|
| none | 100% | 0 | 最高 | 0 | 内网高带宽 |
| lz4 | ~50% | 低 | 略降 5% | ~50% | **通用推荐** |
| snappy | ~55% | 低 | 略降 5% | ~45% | 计算弱场景 |
| gzip | ~30% | 高 | 降 30%+ | ~70% | 跨机房/公网 |
| zstd | ~35% | 中 | 降 15%+ | ~65% | Kafka 2.1+ 推荐 |

### 2.10 Kafka vs 其他 MQ

Kafka 本质是**分布式日志存储系统**，而不是传统消息队列。理解这一点才能正确选型：

| 对比维度 | Kafka | RocketMQ | RabbitMQ |
|----------|-------|----------|----------|
| 定位 | 分布式流平台/日志系统 | 金融级业务消息 | 通用消息代理(AMQP) |
| 吞吐量 | 极高(百万QPS) | 高(十万QPS) | 中(万级QPS) |
| 延迟 | ms 级 | ms 级 | us 级(微秒级) |
| 消息优先级 | 不支持 | 不支持 | 支持 |
| 事务消息 | 支持(0.11+) | 原生支持 | 不支持 |
| 延迟消息 | 无原生支持 | 支持(18级) | 通过 DLX 模拟 |
| 顺序消息 | 单分区内有序 | 同一队列内有序 | 单队列内有序 |
| 消息回查 | 不支持 | 支持 | 不支持 |
| 运维复杂度 | 高(需 ZooKeeper/KRaft) | 高(多独立组件) | 低(Erlang 稳定) |
| 社区活跃度 | 极高 | 较高(阿里) | 高(VMware) |

---

## 三、RocketMQ

RocketMQ 源自阿里巴巴自研的消息中间件 MetaQ，2016年捐赠给 Apache 基金会。淘宝双十一交易链路使用 RocketMQ 承载核心业务消息。

### 3.1 架构

```
                        ┌──────────────┐
                        │  NameServer  │  无状态、纯内存元数据路由
                        │  (Cluster)   │  各节点独立, 无数据同步
                        └──────┬───────┘
              注册/心跳↑        ↑ 路由发现
   ┌─────────────────┼────────┼─────────────────┐
   ▼                 │        │                  ▼
┌─────────┐     ┌─────────┐   ┌─────────┐   ┌─────────┐
│Producer │     │ Broker  │   │ Broker  │   │Consumer │
│ Group   │     │ Master  │   │ Master  │   │ Group   │
│         │────►│   ↓     │   │         │◄──│         │
│         │     │ 同步    │   │         │   │         │
│         │     │ Slave   │   │ Slave   │   │         │
└─────────┘     └─────────┘   └─────────┘   └─────────┘
```

**组件职责**：

| 组件 | 职责 |
|------|------|
| NameServer | 轻量级路由注册中心，Broker 注册地址和 Topic 信息。各 NameServer 节点独立无状态，不互相通信 |
| Broker | 消息存储和转发核心。区分 Master 和 Slave，Master 提供读写，Slave 从 Master 同步 |
| Producer | 生产者。通过 NameServer 发现 Broker，按负载均衡发送 |
| Consumer | 消费者。Pull 模式，支持广播消费和集群消费两种模式 |

**为什么 NameServer 不用 ZooKeeper**：

NameServer 设计极为简洁——CAP 理论下选择 AP（可用性+分区容错），牺牲了强一致性。Broker 注册通过超时心跳（默认 30s），NameServer 节点之间不要求数据一致。如果某 NameServer 数据陈旧，Broker 和 Client 直接重试下一个——代价极低。

> 设计哲学：简单即可靠。NameServer 只有几百行代码，几乎不出 bug。ZooKeeper 生态复杂，运维成本高，对于路由发现这类场景属于过度设计。

### 3.2 消息存储

RocketMQ 存储采用**单一 CommitLog + 多个 ConsumeQueue** 的架构，与 Kafka 的「一个 Partition 一个日志」完全不同：

```
磁盘目录结构:
/root/store/
├── commitlog/
│   ├── 00000000000000000000     ← 所有 Topic 的消息统一写入此 CommitLog
│   └── 00000000001073741824
├── consumequeue/
│   ├── TopicA/0/
│   │   └── 00000000000000000000 ← ConsumeQueue(每个文件约 600 万条目)
│   └── TopicA/1/
│       └── 00000000000000000000
├── index/                       ← 消息 Key/时间索引
│   └── ...
└── config/
    └── consumerOffset.json       ← 消费进度
```

**CommitLog**：所有 Topic 的所有消息顺序追加到一个 CommitLog。每个条目固定 20 字节大小：

```
CommitLog 条目（固定 20 字节）:
┌──────────────────────────────┐
│ offset:   long (8B)          │  ← 消息在 CommitLog 中的物理偏移
│ size:     int  (4B)          │  ← 消息大小
│ tagsCode: long (8B)          │  ← Tag 的 HashCode
└──────────────────────────────┘
```

**ConsumeQueue**：Topic 按 Partition（Queue）生成 ConsumeQueue，每个 ConsumeQueue 是该 Topic/Queue 对应的消息索引列表。这不是消息体副本，而是指向 CommitLog 的指针。

```
ConsumeQueue 条目（固定 20 字节）:
offset(8B) + size(4B) + tagsCode(8B)
```

通过 ConsumeQueue 条目可以快速定位到 CommitLog 中的具体消息。因为 ConsumeQueue 是固定大小条目，消费时随机访问效率极高。

```
消费流程:
  1. Consumer 根据 QueueId + offset → 定位 ConsumeQueue 索引条目
  2. 解析出 CommitLog offset + size
  3. 从 CommitLog 读取完整消息体
```

**对比 Kafka 存储**：

| 维度 | Kafka | RocketMQ |
|------|-------|----------|
| 底层文件 | 一个 Partition = 一组 Segment | 全局 CommitLog + 多个 ConsumeQueue |
| 写入 | 不同 Partition 方向分散写 | 所有 Topic 顺序写入同一个 CommitLog |
| 读取 | 按 Partition 直接读 | 先读 ConsumeQueue(索引) → 再读 CommitLog(数据) |
| 磁盘 IO | 写入分散、读取集中 | 写入集中、读取两次（索引+数据） |
| 适合场景 | 海量日志、流式处理 | Topic 数量多(轻量级 Queue) |

RocketMQ 的设计取舍：因为只有顺序追加，所有 Topic 共享同一份顺序写——磁盘写入效率极高。但读取时多了一次索引查找。此设计使 RocketMQ 支持海量 Topic（数万个），每个 Topic 的队列开销极低（仅 ConsumeQueue 索引文件）。

### 3.3 事务消息

RocketMQ 的事务消息是分布式事务的一种实现（最终一致性），原理图：

```
Producer          Broker              本地事务执行
  │                 │                    (如订单DB)
  │── Half Msg ────►│
  │                 │ (存储半消息, 对消费者不可见)
  │◄─── OK ────────│
  │                 │
  │ 执行本地事务 ◄──┘
  │ (如: Insert order)
  │                 │
  │── Commit ──────►│  成功 → 半消息变为可见
  │   或 Rollback   │  失败 → 半消息标记删除
  │                 │
  │  如果 Commit     │
  │  超时未收到      │
  │                 │
  │◄── 回查 ───────│  Broker 回调 Producer checkLocalTransaction()
  │── Commit ──────►│  Producer 返回事务状态
```

**半消息（Half Msg）**：发送给 Broker 但不会被消费者拉取到的消息。原理：
- Broker 将半消息先写入 CommitLog，标记 consumeQueueOffset=0
- 不写入 ConsumeQueue（因此对消费者不可见）
- 收到 Commit 后更新 OP Topic，写 ConsumeQueue → 消费者可见

**回查机制**：

Producer 发送半消息后，可能因网络、宕机导致 Commit/Rollback 丢失。Broker 定时扫描未确认的半消息，调用 Producer 注册的 `checkLocalTransaction()` 接口：

```java
// Producer 实现事务回查
public class OrderTransactionListener implements TransactionListener {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // Step1: 执行本地事务
        try {
            orderService.createOrder((Order)arg);
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // Step2: Broker 回查时调用
        String orderId = msg.getKeys();
        Order order = orderService.getOrder(orderId);
        if (order != null) {
            return LocalTransactionState.COMMIT_MESSAGE;
        }
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
}
```

> **生产经验**：回查接口必须幂等且快速。建议直接查数据库确认状态，不要依赖缓存（可能过期）。设置合理超时时间（默认 6s），超时后 Broker 会重试回查最多 15 次。

### 3.4 延迟消息

RocketMQ 不支持"任意时间的延迟", 而是 **18 个固定的延迟级别**：

```java
// messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"
//                   1   2   3    4   5  6  7  8  9  10 11 12 13 14  15  16  17 18

Message msg = new Message("TopicTest", "TagA", "Hello".getBytes());
msg.setDelayTimeLevel(3);  // 延迟 10s
```

**实现原理**：Broker 内部有一个 `SCHEDULE_TOPIC_XXXX` 系统 Topic，按延迟级别分为 18 个队列 (delayLevel=1→Queue0, delayLevel=2→Queue1, ...)。延迟流程：
1. Broker 收到延迟消息后不写入目标 ConsumeQueue，而是写入 `SCHEDULE_TOPIC_XXXX` 对应队列
2. 一个内部 Timer 线程轮询每个延迟队列，到期后将消息从 `SCHEDULE_TOPIC_XXXX` 移动到目标 Topic 的 ConsumeQueue
3. Consumer 正常消费

```
延迟消息流程:
  Producer → Broker(CommitLog)
              → 写入 ScheduleTopic Queue[delayLevel]
              → Timer 扫描到期的延迟消息
              → 写入目标 Topic ConsumeQueue
              → Consumer 正常消费
```

如果业务需要超过 2 小时的延迟，可以将 `messageDelayLevel` 配置扩展为更多级别。但注意级别过多会增加 Timer 扫描开销。

### 3.5 顺序消息

RocketMQ 实现全局严格顺序和分区顺序两种方案：

**分区顺序（生产推荐）**：

相同 Sharding Key（如订单 ID）的消息发往同一 MessageQueue，保证局部顺序：

```java
// 生产者: 指定选择算法
producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Long orderId = (Long) arg;
        int index = (int) (orderId % mqs.size());
        return mqs.get(index);
    }
}, orderId);

// 消费者: 同一 Queue 的消息用单线程处理
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext ctx) {
        for (MessageExt msg : msgs) {
            process(msg);  // 这些消息来自同一 Queue, 顺序有保证
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

消费者端 `MessageListenerOrderly` 内部加了一把 **Broker 级别的锁**（对 Queue 加锁，不是全局锁）。同一 Queue 同一时刻只有一个线程消费，但不同 Queue 并行消费。这就是分区有序的核心实现。

**全局顺序（谨慎使用）**：同一 Topic 的所有消息只发往一个 Queue。此时吞吐降到单机水平，失去了 MQ 的扩展能力。极少场景使用（如数据库 Binlog 同步）。

### 3.6 高可用与 DLedger

**主从同步**（旧方案）：

- 同步双写：Master 写入后等待 Slave 刷盘确认 → 可靠性高，延迟大
- 异步复制：Master 完成后异步推给 Slave → 延迟低，可能丢消息

Master 故障时需要人工干预切换到 Slave，自动化切换（如依赖 controllers）存在脑裂风险。

**DLedger（Raft 协议实现）**：

RocketMQ 4.5+ 引入基于 Raft 协议的 DLedger 替代传统主从同步。每个 DLedger Group 内通过 Raft 选举 Leader：

```
DLedger Group:
  Broker1 (Leader)         Broker2 (Follower)       Broker3 (Follower)
  接收写入                   同步日志                  同步日志
  提交到多数派后返回

Raft 保证:
  - 强 Leader 选举, 防止脑裂
  - 日志必须复制到多数节点后才标记为 committed
  - 故障自动切换, 秒级恢复
```

DLedger 的核心优势：**自动故障转移**、**无单点**、**强一致日志复制**。代价是写操作需要多数派确认（N=3 时写入延迟增加），吞吐下降约 20-30%。

```properties
# DLedger 配置
enableDLegerCommitLog=true
dLegerGroup=broker-a
dLegerPeers=n0-192.168.1.1:40911;n1-192.168.1.2:40911;n2-192.168.1.3:40911
dLegerSelfId=n0
```

---

## 四、RabbitMQ

RabbitMQ 是 AMQP 0-9-1 协议的经典实现，由 Erlang 语言编写。定位**通用消息代理**，以其灵活的路由能力和丰富的协议支持闻名。

### 4.1 架构

```
                   RabbitMQ Broker
┌─────────────────────────────────────────────────┐
│                    VHost                         │
│  ┌─────────────────┐                            │
│  │     Exchange    │ ← 生产者发消息到 Exchange   │
│  │  (Direct/Topic  │                            │
│  │   Fanout/Header)│                            │
│  └────┬──────┬─────┘                            │
│       │      │                                   │
│  Binding   Binding    (路由 key 匹配)            │
│       │      │                                   │
│  ┌────▼──────▼─────┐                            │
│  │    Queue        │ ← 消息队列(持久化/绑定)      │
│  └────┬────────────┘                            │
│       │ Consumer 消费                             │
└───────┼──────────────────────────────────────────┘
        ▼
    Consumer
```

**核心模型**：

| 概念 | 说明 |
|------|------|
| Broker | RabbitMQ 服务实例 |
| VHost | 虚拟主机，逻辑隔离（类似 namespace）。每个 VHost 有独立的 Exchange/Queue/权限 |
| Exchange | 交换机，收到消息后按路由规则投递到 Queue |
| Queue | 消息队列，消息持久化存储。消费者从 Queue 取消息 |
| Binding | 绑定关系，Exchange 和 Queue 之间的路由规则 |
| Routing Key | 路由键，Exchange 根据此 key 和 Binding 规则决定消息投递到哪个 Queue |
| Channel | 轻量级连接（复用 TCP 连接），一个 Connection 内可以创建多个 Channel |

**RabbitMQ 内存模型**：VMware 官方实测单 Queue 可达 20000 条/s。但在 Erlang 内存模型和 GC 机制下，消息堆积千万级以上时会性能断崖式下跌。设计时请用大量 Queue（如数百个）分散积压，而非少量 Queue 深度堆积。

### 4.2 Exchange 类型

**Direct Exchange**：精确匹配 Routing Key。

```
Binding:  QueueA ← routing_key="order"   Exchange(direct)
          QueueB ← routing_key="payment" Exchange(direct)

Producer → Exchange, routing_key="order"  → QueueA (精确匹配)
Producer → Exchange, routing_key="payment" → QueueB
Producer → Exchange, routing_key="stock"  → 丢弃 (无匹配)
```

```
       Producer
          │
          ▼
    ┌─────────────┐
    │ Direct       │
    │ Exchange     │
    └───┬─────┬────┘
        │     │
 key=order  key=payment
        │     │
   ┌────▼┐ ┌──▼──────┐
   │QueueA│ │QueueB   │
   └──────┘ └─────────┘
```

**Topic Exchange**：按通配符模式匹配。

```
Binding: QueueA ← routing_key="order.*"     (匹配 order.create, order.pay)
         QueueB ← routing_key="order.#"     (匹配 order.* 和 order.a.b.c 等任意多级)
         QueueC ← routing_key="*.create"    (匹配 order.create, payment.create)

*  : 匹配一级 (用 . 分隔)
#  : 匹配零级或多级
```

**Fanout Exchange**：广播——忽略 Routing Key，消息投递到所有绑定的 Queue。性能最高（无路由匹配开销），用于广播场景（如配置刷新、缓存失效）。

**Headers Exchange**：按 Header 属性路由（不再依赖 Routing Key）。绑定时指定 `x-match=all|any` 和多个 header 匹配条件。极少使用，因性能较差。

```java
// Spring AMQP 声明示例
@Bean
public DirectExchange orderExchange() {
    return new DirectExchange("order.exchange", true, false);
}

@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
        .withArgument("x-dead-letter-exchange", "dlx.exchange")
        .withArgument("x-dead-letter-routing-key", "dlx.order")
        .build();
}

@Bean
public Binding orderBinding() {
    return BindingBuilder.bind(orderQueue()).to(orderExchange()).with("order.created");
}
```

### 4.3 消息确认机制

RabbitMQ 提供生产端和消费端的双重确认：

**Publisher Confirm（生产者确认）**：

```
Broker 收到消息并持久化后返回 ACK:
  Spring AMQP:
    rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
        if (ack) {
            // 消息已到达 Broker (Exchange 确认收到)
        } else {
            log.error("消息发送失败: " + cause);
            // 需要补偿: 重试、落库、告警
        }
    });
```

> 注意：Publisher Confirm 只确认消息到达 Exchange，不保证到达 Queue（因为 Routing Key 不匹配时静默丢弃）。需要同时设置 `mandatory=true` 配合 ReturnCallback 感知无法路由的消息。

```java
// ReturnCallback: 消息无法路由到 Queue 时的回调
rabbitTemplate.setMandatory(true);
rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
    log.error("消息无法路由: exchange={}, routingKey={}, reply={}", exchange, routingKey, replyText);
});
```

**Consumer Ack（消费者确认）**：

```
确认模式:
  - auto: 方法正常返回 → ACK; 抛异常 → NACK 并 requeue
  - manual: 消费者显式调用 basicAck / basicNack / basicReject
  - none: 自动 ACK (绝不推荐——消息可能静默丢失)

手动确认:
  channel.basicAck(deliveryTag, false);     // 确认单条
  channel.basicNack(deliveryTag, false, requeue); // 拒绝单条, 可选 requeue
  channel.basicReject(deliveryTag, requeue);      // 拒绝单条, 不能批量

  // basicNack 第三个参数 multiple=true 批量 Nack(该 deliveryTag 及之前所有)
```

**ACK 超时与 Consumer 恢复**：RabbitMQ 本身没有 consumer ack timeout 机制。消费者处理消息时间过长不 ACK，消息一直处于 unacked 状态。需要通过 `consumer_timeout`（RabbitMQ 3.12+）配置。

### 4.4 死信队列与延迟队列

**死信队列（DLX: Dead Letter Exchange）**：消息在以下三种情况下成为死信：
1. 被消费者 Reject 且 `requeue=false`
2. 消息 TTL 到期（Queue 级别的 `x-message-ttl`）
3. Queue 满（`x-max-length` 或 `x-max-length-bytes` 超出限制）

```
正常 Queue:
  ┌─────────────┐
  │ order.queue │──► Consumer
  └──────┬──────┘
         │ (死信条件触发)
         ▼
  ┌─────────────┐
  │ dlx.exchange │→ DLQ Queue → 死信消费者(告警/记录/补偿)
  └─────────────┘
```

**延迟队列**：RabbitMQ 没有原生延迟队列，通过 TTL + DLX 组合实现。

```
延迟队列实现:
  ┌────────────────────┐
  │ delay.queue         │    ← x-message-ttl=10000  (10秒)
  │ x-dead-letter-      │    ← 绑定 DLX
  │   exchange=target   │
  └─────────┬──────────┘
            │ (消息过期, 自动成为死信)
            ▼
  ┌────────────────────┐
  │ target.queue        │    ← 真正的消费队列
  └─────────┬──────────┘
            ▼
        Consumer
```

但这种方式有**严重缺陷**：Queue 级别的 TTL 作用于整个队列，一旦设置了不同 TTL 的消息需要不同队列。而且 RabbitMQ 只在消息到达 Queue 头部时才会检测 TTL（惰性过期）——即使退后一条消息的 TTL 到了，只要它不在队首就不会被投递到 DLX。

**RabbitMQ 3.8+ 延迟插件**：通过 `rabbitmq_delayed_message_exchange` 插件获得原生延迟能力。安装插件后，Exchange 类型包含 `x-delayed-message`，消息携带 `x-delay` 头指定延迟毫秒数。

### 4.5 镜像队列与仲裁队列

**镜像队列（Mirrored Queues, 经典队列高可用）**：

采用 Active-Active 架构，所有节点都能接收读写，但背后节点同步遵循 Master 模式：

```
镜像队列(ha-mode=exactly, ha-params=2, 即 1 Master + 1 Mirror):
  Node1(Master)          Node2(Mirror)
  ┌────────────┐         ┌────────────┐
  │ Queue V3   │←─同步──│ Queue V3   │
  └────────────┘         └────────────┘
      ↑ 读写                 ↑ (故障时可接管)
  Consumer/Producer      Consumer/Producer
```

所有客户端只与 Master 交互，Mirror 节点是被动的。Master 执行任何操作（写入、ACK）前，先将操作发送到所有 Mirror → 收到确认 → 然后 Master 执行。同步复制，强一致。

**镜像队列的缺陷**：

- 所有消息完全复制到 Mirror → 每个节点存储整个 Queue，磁盘使用效率低
- 同步复制导致性能下降明显（瓶颈在同步最慢的节点）
- 不支持自动故障恢复和集群规模扩展（加节点不提升吞吐，只增加副本）

**仲裁队列（Quorum Queues, RabbitMQ 3.8+）**：

基于 Raft 共识协议的 Queue 类型，替代镜像队列的新方案：

| 特性 | 镜像队列 | 仲裁队列 |
|------|----------|----------|
| 共识协议 | 类同步复制(设计不规范) | Raft(成熟共识算法) |
| 数据复制 | 全量复制到所有 Mirror | Raft 日志复制(多数派确认) |
| 故障恢复 | 人工/MQ 策略切换 | 自动 Leader 选举秒级 |
| 持久性 | 可选(durable/transient) | 强制持久化(数据无法脱离磁盘) |
| 消息顺序 | 保证 | 保证 |
| 重复消费处理 | 无 | 自动检测+过滤重复消息 |
| 内存占用 | 小 | 较大(Raft 日志元数据) |
| 吞吐量 | 较高 | 略低于镜像队列(写在 Raft 日志) |

仲裁队列关键配置：
```java
// 声明仲裁队列
Queue quorumQueue = QueueBuilder.durable("quorum.order.queue")
    .quorum()  // 标记为仲裁队列
    .withArgument("x-quorum-initial-group-size", 3)  // 初始节点数
    .build();
```

> **生产经验**：新项目直接使用仲裁队列，不要引入镜像队列。仲裁队列基于 Raft 的设计更健壮，且支持自动恢复和扩展。镜像队列已进入维护模式。

---

## 五、通用主题

### 5.1 消息可靠性保证

消息从生产到消费，经历三个阶段，每个阶段都可能丢失消息：

```
阶段1: 生产可靠        阶段2: 存储可靠          阶段3: 消费可靠
Producer → Broker    Broker 存储、副本        Broker → Consumer
```

**阶段1：生产可靠性**

| MQ | 保证方式 | 关键参数/API |
|----|---------|-------------|
| Kafka | 同步发送 + acks=all | `acks=all`, `enable.idempotence=true`, 同步 `Future.get()` |
| RocketMQ | 同步发送 + 刷盘策略 | `SYNC_FLUSH` 刷盘, `sendResult.getSendStatus()` 检查 |
| RabbitMQ | Publisher Confirm + mandatory | `setConfirmCallback()`, `setReturnCallback()`, 消息持久化 `deliveryMode=2` |

```java
// Kafka 可靠发送示例
Future<RecordMetadata> future = producer.send(record);
RecordMetadata meta = future.get(); // 同步等待, 阻塞获取结果
// 或用回调:
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // 重试或落库
    }
});
```

**阶段2：存储可靠性**

- Kafka：设置 `replication.factor=3`, `min.insync.replicas=2`, `unclean.leader.election.enable=false`
- RocketMQ：同步刷盘 + 主从同步复制（或 DLedger）
- RabbitMQ：`durable=true` + 镜像队列或仲裁队列

**阶段3：消费可靠性**

| MQ | 可靠性保证 | 具体机制 |
|----|---------|---------|
| Kafka | 手动提交 offset | `enable.auto.commit=false`, `consumer.commitSync()` 处理完再提交 |
| RocketMQ | 消费状态返回 | `ConsumeConcurrentlyStatus.CONSUME_SUCCESS` 或 `RECONSUME_LATER` |
| RabbitMQ | 手动 ACK | `channel.basicAck()` 确认, `channel.basicNack()` 处理失败 requeue |

> **核心原则**：只要有一环缺失，消息就可能丢失。一条穿越全链路的消息 = 生产者确认 + Broker 副本 + 消费者确认。三缺一，可靠性不作数。

### 5.2 消息幂等性

MQ 的语义保证是 at-least-once（至少一次），这意味着每条消息可能被重复消费。消费端必须实现幂等。

**方案1：业务唯一键**（最推荐）

每条消息携带业务唯一 ID（如数据库主键、订单号等），消费端利用数据库唯一约束保证幂等：

```java
// 数据库唯一键去重
public void consume(OrderEvent event) {
    try {
        orderDao.insert(event.toOrder());  // order_id 有唯一索引
    } catch (DuplicateKeyException e) {
        // 已处理过, 跳过
    }
    // 幂等保证: 相同 order_id 只会插入成功一次
}
```

**方案2：去重表**（通用方案）

维护一张 `message_consumer_log` 表，记录 `msg_id` 和处理状态：

```sql
CREATE TABLE message_consumer_log (
    msg_id VARCHAR(128) PRIMARY KEY,
    status VARCHAR(20),    -- PROCESSING, CONSUMED
    create_time DATETIME,
    update_time DATETIME
);

-- 消费逻辑
INSERT INTO message_consumer_log (msg_id, status) VALUES (?, 'PROCESSING');
-- 成功返回 → 执行业务逻辑
-- 失败(重复键) → 跳过
```

```java
public void consume(Message msg) {
    String msgId = msg.getKey();
    // 尝试插入去重记录(唯一键冲突则跳过)
    boolean inserted = messageLogDao.insertIfAbsent(msgId);
    if (!inserted) return; // 已消费, 跳过

    try {
        businessLogic(msg);
        messageLogDao.updateStatus(msgId, "CONSUMED");
    } catch (Exception e) {
        // 处理失败, 状态保持 PROCESSING
        // 支持后续重试(根据状态决定是否重试)
        throw e;
    }
}
```

**方案3：Redis SetNX**（适合低延迟场景）

```java
String dedupKey = "msg:dedup:" + msgId;
Boolean firstTime = redisTemplate.opsForValue()
    .setIfAbsent(dedupKey, "1", Duration.ofHours(24));
if (Boolean.TRUE.equals(firstTime)) {
    process(msg);
}
```

**三种方案对比**：

| 方案 | 可靠性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|----------|
| 业务唯一键 | 最高 | 高(单次查) | 低 | **优先推荐** |
| 去重表 | 高 | 中(写两次 DB) | 中 | 通用 |
| Redis SetNX | 中(过期后失去幂等) | 极高 | 低 | 允许 24h 窗口内去重 |

> **生产经验**：业务唯一键是第一选择。例如订单号天生唯一，不需要额外去重表。只有在消息没有自然唯一键时才用去重表或 Redis 方案。

### 5.3 消息积压处理

消息积压是 MQ 系统最常见的生产事故。原因通常是消费速度低于生产速度，或消费者全部宕机。

**积压程度分级**：

| 级别 | 描述 | 应急方案 |
|------|------|----------|
| 轻度 (< 百万) | 消费慢 | 扩容消费者实例 |
| 中度 (百万~千万) | 临时消费速度不够 | 紧急增加消费者，跳过非核心处理 |
| 重度 (>千万) | 磁盘空间告急 | 先止损(扩容磁盘/新建 Topic 转移)，再处理积压 |

**应急处理流程**：

```bash
# 步骤1: 止损 - 判断积压原因
# 消费者宕机 → 重启消费者
# 消费逻辑慢 → 扩容消费者实例(不能超过分区数)

# 步骤2: 临时修复 - 跳过非核心逻辑
# 快速消费积压的代码(跳过复杂计算、第三方调用):
public void emergencyConsume(Message msg) {
    // 只做核心落库, 跳过通知/统计
    orderDao.insert(msg.toOrder());
    // 不做: smsService.send(), analyticsService.report()
}
```

**步骤3：消息转发**（重度积压）

如果积压量大到快影响磁盘空间，把积压的 Topic 消息快速消费后转发到新 Topic：

```
原 Topic(积压 1 亿条) → 紧急消费者(只做转发) → 新 Topic
→ 新消费者组(加速消费)
```

**步骤4：批量扩容消费者**（如果消费者实例 < 分区数）

Kafka 的消费并发上限 = Partition 数量。如果 8 个分区但只有 4 个消费者，扩容到 8 个即可线性提升消费速度。

```
特别注意: Kafka 消费者实例数 > 分区数时, 多余实例空闲(不会参与消费)
RocketMQ 同理: 消费者数 > 消息队列数(默认 readQueueNums)
```

**预防措施**：

```java
// 监控消费滞后 (Kafka)
consumer.metrics().forEach((name, metric) -> {
    if (name.name().equals("records-lag")) {
        gauge.set(metric.value());  // Prometheus 指标上报
    }
});

// 告警规则(示例):
// records-lag > 100000 持续 5min → P2 告警
// records-lag > 1000000 持续 2min → P1 告警
```

### 5.4 顺序消息的实现

**为什么要全局顺序**：RDBMS binlog 同步、状态机复制等场景要求严格顺序。

**单 Partition 单线程消费**（Kafka + RocketMQ 通用模式）：

```
生产者: 相同 key → 路由到同一 Partition
消费者: 对 Partition 加锁, 单线程消费

Kafka:
  // Producer: 指定 key
  new ProducerRecord<>("topic", "orderId123", messageBody);

  // Consumer: 按分区顺序处理
  // 每个 Partition 只会分配给 Group 内的一个消费者
  // 这个消费者内部按顺序 poll 和 process

RocketMQ:
  // Producer: MessageQueueSelector
  // Consumer: MessageListenerOrderly
```

**全局顺序**：Topic 只有 1 个 Partition/MessageQueue——所有消息严格有序，但吞吐上限为单机。建议只在写 Binlog 或其他极少数场景下采用。

### 5.5 MQ 选型参考表

| 选型维度 | Kafka | RocketMQ | RabbitMQ |
|----------|-------|----------|----------|
| **吞吐量** | ★★★★★ 百万级 | ★★★★☆ 十万级 | ★★★☆☆ 万级 |
| **延迟** | ms 级 | ms 级 | μs 级 |
| **可靠性** | ★★★★☆ at-least-once | ★★★★★ 金融级 | ★★★★☆ 高可靠性 |
| **事务消息** | 支持(0.11+), 但复杂 | ★★★★★ 原生半消息+回查 | 不支持 |
| **延迟消息** | 需自建 | ★★★★★ 18 个级别 | 需 DLX 模拟/插件 |
| **消息查询/回溯** | 按 offset 回溯 | ★★★★★ 按 Key/时间查询 | 不支持回溯 |
| **多语言支持** | Java 最佳, 多语言 | Java 最佳 | ★★★★★ 所有语言 |
| **运维复杂度** | ★★★☆☆ 需 ZooKeeper/KRaft | ★★☆☆☆ 多个独立组件 | ★★★★★ 简单部署 |
| **社区活跃** | ★★★★★ Apache 顶级 | ★★★★☆ 阿里主导 | ★★★★☆ VMware 维护 |
| **云原生** | ★★★★★ K8s Operator | ★★★☆☆ 容器化一般 | ★★★★☆ K8s 支持好 |
| **典型场景** | 日志采集、流计算、事件溯源、Clickstream | 交易消息、电商订单、分布式事务 | 业务解耦、RPC 异步化、服务间通信 |
| **不适合场景** | 低延迟(ms以下)、大量 Topic | 海量 Topic (数万+) 有压力 | 高吞吐(10w+)、消息大量积压 |

**选型决策规则**：

```
# 1. 日处理量 > 百亿消息

→ Kafka (别无选择)
# 2. 金融/支付/电商核心交易链路

→ RocketMQ (事务消息 + 回查 + 刷盘保证)
# 3. 企业内服务间异步通信，路由灵活，请求-响应模式

→ RabbitMQ (AMQP 标准、Exchange 灵活路由)
# 4. 日志、埋点、流式 ETL

→ Kafka (极致吞吐、直连存储、Kafka Streams)
# 5. 物联网（千万设备连接，低功耗协议）

→ RabbitMQ (MQTT 插件) 或 EMQX
# 6. 中小型项目、少量消息、不想运维复杂

→ RabbitMQ (单节点就够用)
```

### 5.6 面试常见问题

**1. Kafka 为什么能实现百万 QPS？**

顺序写磁盘、零拷贝 sendfile、pagecache、批量处理、分区并发、无状态 Broker——这些是 Kafka 性能优势的六大支柱。顺序写 SSD 可达 600MB/s，传统随机写磁盘仅 ~100KB/s，差 6000 倍。sendfile 将网络传输的 CPU 复制从 4 次降到 2 次 DMA，且全程绕过用户态。pagecache 使读操作和写缓冲几乎都在内存中完成。批量处理将大量小 IO 汇聚为大 IO。

**2. RocketMQ 的事务消息和 Kafka 的事务有何区别？**

RocketMQ 事务消息通过半消息（Half Msg）+ 回查机制，解决分布式事务中 MQ 与 DB 的一致性问题。Kafka 事务是保证生产者跨分区原子写入（更接近数据库事务语义）。RocketMQ 的事务消息设计更贴近业务——引入"反查"确保事务最终一致——而 Kafka 需要 EOS（Exactly Once Semantics）实现跨分区事务，但应用层更多用于流处理而不是业务事务。

**3. 如何保证消息不丢失？**

从生产端到 Broker 到消费端三阶段不可缺一。生产端：同步确认（`acks=all`、Publisher Confirm）。Broker 端：多副本，`min.insync.replicas >= 2`，拒绝 unclean 选举。消费端：禁用自动提交，处理完毕再提交 offset 或 ACK。另外，幂等去重保证重复消息不会产生数据问题。三阶段任一缺失都会导致消息可靠性失效。

**4. ISR 为什么缩小？**

网络抖动或 Follower 节点 GC 停顿，导致 Follower 超过 `replica.lag.time.max.ms`（默认 30s）未与 Leader 同步，被踢出 ISR。这时 `min.insync.replicas` 可能不满足，生产者收到 NOT_ENOUGH_REPLICAS 异常。优化方向：增大 `replica.lag.time.max.ms`、增加 `num.replica.fetchers` 线程数、增加 Follower 节点资源。

**5. 消息重复消费了怎么办？**

MQ 的保证是 at-least-once，重复消费是必然的（而非 bug）。解决方案就是消费端幂等——业务唯一键、去重表、Redis SetNX。设计消费逻辑时必须假定每条消息至少会收到一次。

**6. 顺序消息消费慢怎么办？**

这是顺序消息的天然瓶颈——单线程消费意味着无法水平扩展。如果消费端是 IO 密集型，把核心逻辑拆为两部分：先消费存入本地 B+Tree / 有序队列，再用多线程处理，最终按 key 聚合。也可以队列内加速：将计算密集型操作异步化，只保留顺序的聚合步骤。

**7. RabbitMQ 的 Channel 为什么是线程不安全的？**

Channel 是 Erlang 进程映射到客户端的轻量级对象，其底层 Erlang 进程是单线程 actor 模型（不是锁保护）。因此多个线程并发操作同一个 Channel 会破坏 Erlang 的消息不变性。方案：每个线程分配独立 Channel，或线程间通过队列传递操作。

**8. 如何实现一个延时任意时间的延时消息？**

原生 RocketMQ 只支持 18 个级别，RabbitMQ 需 DLX 模拟。通用方案是：维护一个基于时间轮的调度器（如 Netty 的 `HashedWheelTimer`），生产者把消息的时间写入数据库，调度器定时轮询到期消息并通过 MQ 投递出去。时间轮的优点是 O(1) 插入和删除，适合海量定时消息场景。

---

## 六、总结

消息队列的选择不是看哪个技术"最新最好"，而是匹配业务形态：

- **高吞吐流处理** → Kafka：顺序读写、零拷贝、pagecache，单机百万 QPS
- **核心交易链路** → RocketMQ：事务消息、半消息回查、分布式最终一致性
- **灵活路由通信** → RabbitMQ：Direct/Topic/Fanout 路由、AMQP 标准、低延迟

三个 MQ 的存储模型截然不同——Kafka 的 Partition Segment、RocketMQ 的 CommitLog+ConsumeQueue、RabbitMQ 的 Queue 索引——反映了不同的设计哲学：

- Kafka 是"分布式日志"——消息分层为有序 Segment，适合大规模顺序消费
- RocketMQ 是"数据库的事务日志"——所有 Topic 共用一个 CommitLog，轻量级 Topic
- RabbitMQ 是"消息路由器"——Exchange 路由 → Queue 存储，灵活与可靠兼得

理解存储模型和共识机制（ISR、Raft、主从同步）的区别，比掌握 API 重要得多。MQ 的难点不在"发送和接收"，而在于**消息可靠性、幂等性、积压处理、顺序保证**这些生产问题的解决。

选好 MQ，用好 MQ，才能让异步消息从"有问题"变成"看不见"——这才是高级工程师的分水岭。
