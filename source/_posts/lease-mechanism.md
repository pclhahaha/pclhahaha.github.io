---
title: Lease 租约机制
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 分布式
  - Lease
  - GFS
  - 分布式锁
categories:
  - 分布式
---

Lease 是一种带超时时间的授权机制，核心语义是"在 T 时间内，我承诺不把你持有的权限给别人"。与锁不同，Lease 天然容忍持有者故障——过期自动释放，不需要显式的解锁操作。

## 一、核心机制

Lease 的四个要素：

- **颁发**：授权方颁发一个带有效期（TTL）的授权
- **续约**：持约方在 Lease 过期之前可以申请续约
- **过期自动失效**：授权方在 Lease 过期后自动收回权限，无需通信
- **到期前承诺**：授权方在 Lease 有效期内不会将相同权限授予第三方

```
时间线：
  颁发 ←────────────── TTL (如 60s) ──────────────→ 过期
        持约方可在到期前续约     授权方承诺不转授
```

## 二、与锁的区别

| 维度 | 锁 | Lease |
|------|-----|-------|
| 持有方式 | 显式申请和释放 | 颁发 + 自动过期 |
| 占有者故障 | 死锁（需超时检测才能释放） | 过期自动释放，无死锁 |
| 续约 | 通常不可续 | 到期前可申请续约 |
| 通信开销 | 每次加解锁都要通信 | 只在续约时偶尔通信 |
| 语义 | 互斥访问 | 时间边界内的独占承诺 |

**Lease 的核心优势**：持有者宕机后不需要任何消息来释放资源——TTL 到期，权限自动归还。在分布式系统中，"靠超时兜底"比"靠故障检测"更可靠。

### 实现示例

```java
public class LeaseManager {
    private final ConcurrentHashMap<String, Lease> leases = new ConcurrentHashMap<>();
    
    // 颁发 Lease
    public Lease grant(String resourceId, String holderId, long ttlMs) {
        if (leases.containsKey(resourceId)) {
            Lease existing = leases.get(resourceId);
            if (!existing.isExpired()) {
                throw new LeaseConflictException("lease held by " + existing.holderId);
            }
        }
        Lease lease = new Lease(resourceId, holderId, System.currentTimeMillis() + ttlMs);
        leases.put(resourceId, lease);
        return lease;
    }
    
    // 续约
    public boolean renew(String resourceId, String holderId, long extendMs) {
        Lease lease = leases.get(resourceId);
        if (lease == null || !lease.holderId.equals(holderId)) return false;
        lease.expireAt = Math.max(lease.expireAt, System.currentTimeMillis() + extendMs);
        return true;
    }
    
    // 检查
    public boolean isValid(String resourceId) {
        Lease lease = leases.get(resourceId);
        return lease != null && !lease.isExpired();
    }
}
```

## 三、GFS Chunk Lease——经典应用

GFS 论文中，Master 为每个 Chunk 授予其中一个 Chunkserver "主副本 Lease"（初始 60 秒），该主副本负责对该 Chunk 的所有写操作定序：

```
1. Client 请求 Master：我要写 Chunk A
2. Master 返回：主副本在 CS-1，Lease 剩余 45s
3. Client 向 CS-1 发送数据
4. CS-1 定序写入顺序 → 通知 CS-2、CS-3 写入
5. 全部完成 → 返回 Client
```

**为什么用 Lease 而不是永久授权？** Master 可能宕机重启，如果 Chunkserver 持有永久授权，新 Master 不知道哪些 Chunk 被谁"锁定"。Lease 过期后 Master 可以安全地重新分配主副本——不需要知道旧持有者是否还活着。

## 四、其他应用场景

| 场景 | Lease 的作用 |
|------|-------------|
| **分布式缓存 TTL** | 缓存数据的有效期本质上是应用层 Lease——过期后从源重新拉取 |
| **Leader 选举** | Leader 持有 Lease，Follower 在 Lease 期内不发起选举；Leader 宕机后 Lease 过期，Follower 重新选举 |
| **分布式锁替代** | 当操作可以在 Lease 边界内完成时，Lease 比锁更简洁——无死锁风险，无解锁开销 |
| **服务注册健康检查** | 注册中心为服务实例颁发 Lease，实例定期续约；实例宕机后 Lease 过期，自动摘除 |

## 五、小结

Lease 的核心思想是"用时间承诺替代显式协调"——这是一种比锁更轻量、更有弹性的分布式协同机制。GFS 的 Chunk Lease 是这一思想的经典实践，它用 60 秒的简单承诺替代了复杂的 Chunk 所有权协商，让系统在 Master 故障恢复时能安全地重新分配资源。
