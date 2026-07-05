---
title: 排行榜系统设计
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 系统设计
  - 排行榜
  - Redis ZSet
  - TopK
categories:
  - 系统设计
---

排行榜是游戏、社交、电商中最常见的功能之一。从手游的战力榜到直播的礼物榜，从社区签到榜到电商销量榜——排行榜的设计看似简单，但一旦加上"实时更新""千万用户""多维度""分周期"等真实约束，就变成了一个有深度的系统设计问题。

## 一、需求分析

### 1.1 功能需求

| 需求 | 说明 | 示例 |
|------|------|------|
| **更新分数** | 用户分数变化时更新排名 | 玩家完成一场对局 +50 分 |
| **查询排名** | 查询指定用户的当前排名 | "我是第 328 名" |
| **Top N 查询** | 查询排行榜前 N 名 | 全服战力榜前 100 |
| **周边排名** | 查询用户周围的名次 | "比我高 3 名和低 3 名的玩家" |
| **分周期** | 日榜/周榜/月榜/总榜 | 每周一重置周榜 |

### 1.2 非功能需求

- **实时性**：分数更新后 1 秒内反映在排名中
- **高并发**：支持 1000 万用户同时在线，排行榜查询 QPS 10 万+
- **低延迟**：排名查询 < 5ms
- **可扩展**：排行榜玩家数量可达数千万

## 二、方案选型

### 2.1 为什么不用 MySQL

最直观的做法是用 MySQL 存储分数，然后 `SELECT COUNT(*) FROM leaderboard WHERE score > ?` 来计算排名：

```sql
-- 查询用户排名（暴力方案）
SELECT COUNT(*) + 1 AS rank 
FROM leaderboard 
WHERE score > (SELECT score FROM leaderboard WHERE user_id = 1001);
```

问题：千万用户排行榜，这条 SQL 需要全表扫描。即使按 score 建索引，高频率的更新会导致索引维护开销巨大。

### 2.2 用 Redis ZSet

Redis 的 Sorted Set（ZSet）将**插入和排名查询都控制在 O(log N)** 的时间复杂度。

```bash
# 更新分数
ZADD leaderboard:total 1500 user:1001

# 查询排名（ZREVRANK 返回降序排名，0-based）
ZREVRANK leaderboard:total user:1001
# → 327（排名第 328）

# 查询 Top 10
ZREVRANGE leaderboard:total 0 9 WITHSCORES

# 查询分数
ZSCORE leaderboard:total user:1001
# → "1500"

# 查询用户周围排名（前后各 3 名）
ZREVRANGEBYSCORE leaderboard:total +inf -inf WITHSCORES LIMIT 324 7
```

### 2.3 复杂度分析

| 操作 | Redis ZSet | MySQL |
|------|-----------|-------|
| 插入/更新分数 | O(log N) | O(log N)（索引更新） |
| 查询排名 | O(log N) | O(N) 或 O(log N)（需要额外优化） |
| Top N 查询 | O(log N + N) | O(log N + N) |
| 批量查询 | O(M log N) | O(M log N) |

Redis ZSet 的各项操作在同量级上比 MySQL 快 10-100 倍（纯内存 vs 磁盘 IO），且 API 天然适合排行榜场景。

## 三、数据结构设计

### 3.1 ZSet 的底层实现

Redis ZSet 底层是 **skiplist + dict** 的双结构：

```
ZADD leaderboard 1500 user:1001

skiplist 结构（按 score 排序）:
  {user:1003, 2000}
  {user:1007, 1850}
  {user:1001, 1500}
  {user:1012, 1200}
  ...

dict 结构（O(1) 按成员查找）:
  "user:1001" → score = 1500
  "user:1003" → score = 2000
```

- skiplist 负责范围查询和排序（ZREVRANGE、ZREVRANK）
- dict 负责 O(1) 按成员查分数（ZSCORE）

两者通过共享同一份成员数据保持一致性。

### 3.2 内存估算

ZSet 的内存占用：

- 每个成员（member）：约 50-80 字节（包含 skiplist 节点、dict entry、sds 字符串）
- 1000 万玩家 → 约 500-800 MB
- 加上分数（double: 8 字节）和指针开销，总计约 1-1.5 GB

单机 Redis 可以轻松承载，超过这个规模可以考虑 Redis Cluster 分片。

## 四、实时更新策略

### 4.1 积分变动 → 排名更新延时

```
玩家完成对局 → 应用服务器更新数据库 → 更新 Redis ZADD → 排名立即生效
```

一次积分更新包括：

```python
def update_score(user_id: str, delta: int):
    # 1. 更新数据库（持久化）
    db.execute("UPDATE users SET score = score + ? WHERE id = ?", delta, user_id)
    
    # 2. 更新 Redis（实时排名）
    redis.zincrby("leaderboard:total", delta, f"user:{user_id}")
```

### 4.2 批量更新优化

当有大量玩家同时完成对局（如大型赛事结束），可以批量更新：

```python
# 使用 Redis Pipeline 批量更新
pipe = redis.pipeline()
for user_id, delta in score_changes:
    pipe.zincrby("leaderboard:total", delta, f"user:{user_id}")
pipe.execute()
```

## 五、多周期排行榜（日/周/月/总榜）

### 5.1 方案对比

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| **独立 Key** | 每个周期一个 ZSet | 实现简单、查询快 | 内存 x 4 倍（日/周/月/总） |
| **按日期分 Key** | 每天一个 ZSet，查询时合并 | 内存可控 | 查询需合并计算，复杂 |
| **定时快照** | 总榜实时更新，周期榜定时从总榜快照 | 节省内存 | 周期榜有延迟（最高 1 小时） |

### 5.2 推荐方案：独立 Key

对于千万级用户，用独立 Key 存储四个周期榜：

```
leaderboard:daily:20260705   ← 日榜（每天 0 点重置）
leaderboard:weekly:202627    ← 周榜（每周一 0 点重置）
leaderboard:monthly:202607   ← 月榜（每月 1 日 0 点重置）
leaderboard:total            ← 总榜（永不重置）
```

### 5.3 定时重置

使用 Lua 脚本原子性地删除旧 Key 并建立新空榜：

```lua
-- 重置日榜
redis.call("DEL", "leaderboard:daily:" .. new_date)
-- 将过去 24 小时内活跃用户的分数复制到新榜（可选，需要从其他存储拉取）
```

或者直接新建空榜即可——用户在当日第一次获得分数时自动加入日榜。

## 六、分段排行榜

当用户量极大（如数亿），单榜排名会有问题：

- **大 V 效应**：前 100 名被头部玩家垄断，普通用户看不到自己的进步
- **更新频率**：千万用户的排行榜，每次都更新整个 ZSet 压力大

方案：按段位分榜。

```
leaderboard:bronze    ← 0-999 分段
leaderboard:silver    ← 1000-1999 分段
leaderboard:gold      ← 2000-2999 分段
leaderboard:platinum  ← 3000-3999 分段
leaderboard:diamond   ← 4000+ 分段
```

玩家只在当前段位内排名，段位提升时转移到新榜。每个段位的排行榜玩家数在百万级别，查询更高效。

## 七、与历史数据的关系

排行榜只是"当前快照"，真正的积分明细存储在 MySQL 中：

```sql
CREATE TABLE score_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    score_change INT NOT NULL,   -- 正数为加分，负数为减分
    reason VARCHAR(50),          -- 'game_win', 'daily_bonus', 'admin_adjust'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_time (user_id, created_at)
);
```

排行榜依赖于 Redis ZSet（实时查询），积分明细依赖于 MySQL（持久化、审计、恢复）。如果 Redis 数据丢失（如宕机），可以从 MySQL 全量重建排行榜：

```python
# 从 MySQL 重建排行榜
users = db.query("SELECT user_id, score FROM users ORDER BY score DESC")
pipe = redis.pipeline()
for user_id, score in users:
    pipe.zadd("leaderboard:total", {f"user:{user_id}": score})
pipe.execute()
```

## 八、容量估算

假设 5000 万用户，人均分数变动 10 次/天：

- Redis 内存：5000 万 × 80 字节 ≈ 4GB（四榜 ≈ 16GB）
- 写入 QPS：5000 万 × 10 / 86400 ≈ 5800 QPS（Redis 轻松应对）
- 读取 QPS：假设 10 万 QPS（排行榜高频查看），Redis 单机 10 万 QPS（多个 key 查询）

如果超过单机 Redis 容量，使用 Redis Cluster 按用户 ID 哈希分片——但注意 ZSet 的成员不可拆分，所以分片只适用于多榜场景（每个榜独立），不适用于拆分单个大榜。

## 九、小结

排行榜系统的核心选择是 Redis ZSet——它的 skiplist + dict 双结构天然适配"更新分数 O(log N) + 查询排名 O(log N)"的需求。在工程实践中，关键权衡点在于：多周期的内存开销、分段榜的玩家体验优化、以及 Redis 和 MySQL 之间的一致性保证。对于绝大多数排行榜场景（千万级用户、万级 QPS），单机 Redis + 多 Key 方案已经足够。
