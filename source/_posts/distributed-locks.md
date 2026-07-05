---
title: 分布式锁 — Redis/ZK/DB
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - 分布式锁
  - Redis
  - ZooKeeper
  - Redlock
categories:
  - 分布式
---

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
