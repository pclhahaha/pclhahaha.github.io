---
title: MySQL 深度解析
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - MySQL
  - InnoDB
  - 索引
  - 事务
  - 锁
  - 高可用
categories:
  - 存储
---

[TOC]

## 一、InnoDB 存储引擎架构

InnoDB 是 MySQL 5.5 之后的默认存储引擎，也是目前互联网公司使用最广泛的引擎。它支持事务（ACID）、行级锁、MVCC、外键，并且基于聚簇索引组织数据。

### 1.1 Buffer Pool（缓冲池）

Buffer Pool 是 InnoDB 最核心的内存结构，用于缓存磁盘上的数据页和索引页。它的本质是一块连续的内存区域，InnoDB 通过变种的 LRU 算法管理页面淘汰。

**为什么需要 Buffer Pool？** 磁盘 I/O 速度比内存慢数个数量级。通过将热点数据保留在内存中，可以大幅减少磁盘访问次数。

**LRU 变种算法：** InnoDB 将 LRU 链表分为两个区域——Young（热端，5/8）和 Old（冷端，3/8）。新读入的页先插入 Old 区头部，如果该页在 Old 区被再次访问且停留超过 `innodb_old_blocks_time`（默认 1000ms），才会移到 Young 区。这个设计防止了全表扫描把真正的热点数据冲刷掉——这个现象叫“缓冲池污染”。

**关键参数：**

```sql
-- 查看 buffer pool 大小
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';  -- 建议设为物理内存的 50%-70%
SHOW VARIABLES LIKE 'innodb_buffer_pool_instances'; -- 多实例减少并发竞争
SHOW STATUS LIKE 'Innodb_buffer_pool_read_requests';  -- 逻辑读次数
SHOW STATUS LIKE 'Innodb_buffer_pool_reads';          -- 物理读次数
-- 缓冲池命中率 = 1 - (reads / read_requests)，应 > 99%
```

**生产经验：** 如果命中率低于 99%，优先考虑增加 `innodb_buffer_pool_size`。8.0 版本支持在线调整（`SET GLOBAL`）。

### 1.2 Change Buffer（写缓冲）

Change Buffer 是 InnoDB 的一种特殊缓存，用来暂存对非唯一二级索引的修改操作（INSERT/UPDATE/DELETE）。

**工作原理：** 当要修改的二级索引页不在 Buffer Pool 中时，InnoDB 不会立即将磁盘页读入内存，而是将修改操作缓存在 Change Buffer 中，等待后续有其他读操作把该页加载到 Buffer Pool 时再合并（merge）应用。这大大减少了随机 I/O。

**适用场景：** 写多读少的场景收益最大。如果系统中有大量写操作且涉及多个二级索引，Change Buffer 可以显著提升写入性能。MySQL 8.0 中 Change Buffer 默认支持 INSERT/UPDATE/DELETE 操作。

```sql
SHOW VARIABLES LIKE 'innodb_change_buffering';  -- 默认 all
SHOW VARIABLES LIKE 'innodb_change_buffer_max_size'; -- 默认占 buffer pool 25%
```

### 1.3 Adaptive Hash Index（自适应哈希索引）

InnoDB 会监控索引页的访问模式，对于频繁被等值查询访问的索引页，自动在内存中建立哈希索引，加速等值查询。

**特点：**
- 完全由 InnoDB 自动维护，DBA 无法手动干预
- 只对等值查询有效，范围查询无法使用
- 哈希索引本身有内存开销，高并发下可能有锁竞争

```sql
SHOW VARIABLES LIKE 'innodb_adaptive_hash_index'; -- 默认 ON
SHOW ENGINE INNODB STATUS\G  -- 可查看 AHI 使用情况
```

**生产经验：** 大部分场景保持默认开启即可。如果遇到 AHI 相关的 `btr0sea.cc` 竞争，或者负载以范围查询和写入为主，可以考虑关闭。

### 1.4 Log Buffer（日志缓冲）

Log Buffer 是 Redo Log 的内存缓冲区。事务执行过程中生成的 Redo Log 先写入 Log Buffer，随后在合适的时机刷到磁盘（Redo Log File）。

**刷盘时机：**
- 事务提交时（受 `innodb_flush_log_at_trx_commit` 控制）
- Log Buffer 空间不足（达到 `innodb_log_buffer_size` 的一半）
- Master Thread 每秒刷新一次
- 脏页刷盘前（WAL 要求先写日志再写数据）

```sql
SHOW VARIABLES LIKE 'innodb_log_buffer_size'; -- 建议 16MB-64MB
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit'; -- 1: 每次提交都刷盘, 2: 每秒刷盘, 0: 每秒刷但不保证
```

**innodb_flush_log_at_trx_commit 的权衡：**

| 值 | 含义 | 安全性 | 性能 | 推荐场景 |
|---|------|--------|------|---------|
| 0 | 每秒刷，事务提交不触发 | 可能丢失 1s 数据 | 最高 | 允许丢数据的日志类业务 |
| 1 | 每次提交刷盘 | 不丢数据 | 最低 | 金融、支付等核心业务 |
| 2 | 每次提交写 OS cache，每秒刷 | 丢最后一次提交 | 中等 | 普通业务折中方案 |

**生产经验：** 线上核心库务必设为 1。备库可以设为 2 以提升同步性能。开启后必须配合 `innodb_flush_method=O_DIRECT` 防止双重缓存。

### 1.5 Doublewrite Buffer（双写缓冲）

Doublewrite Buffer 是为了解决**部分写失效（Partial Page Write）**问题。InnoDB 的页大小通常是 16KB，而操作系统磁盘原子写单位通常是 512B 或 4KB。如果写 16KB 时发生宕机，可能出现一个页中只有部分被写入的情况，导致页损坏且 Redo Log 无法恢复（Redo Log 假设页是完整的）。

**工作流程：**
1. 将脏页先顺序写入 Doublewrite Buffer（磁盘上一段连续空间，2MB）
2. 写入成功后再随机写入实际的数据文件
3. 如果第二步期间宕机，恢复时从 Doublewrite Buffer 找到完整副本恢复

```sql
SHOW VARIABLES LIKE 'innodb_doublewrite'; -- 默认 ON
```

**生产经验：** 使用原子写（Atomic Write）能力的 SSD（如 Fusion-io），或者云厂商提供的块存储，可以关闭 Doublewrite Buffer 以提升性能。

### 1.6 整体架构与读写流程

InnoDB 读写流程可概括为：

**读流程：**
1. 首先查询 Buffer Pool，命中则直接返回
2. 未命中则将磁盘页加载到 Buffer Pool
3. Change Buffer 中有该页的修改则先合并

**写流程：**
1. 先将修改记录写入 Log Buffer（Redo Log）
2. 修改 Buffer Pool 中的页（变成脏页）
3. 对非唯一二级索引，若目标页不在 Buffer Pool，写 Change Buffer
4. 脏页由后台线程异步刷盘（Page Cleaner Thread）
5. Redo Log 按策略刷盘

这种“先写日志再写数据”的机制就是 **WAL（Write-Ahead Logging）**，是 InnoDB 高性能写入的关键。

---

## 二、B+Tree 索引

### 2.1 为什么选择 B+Tree

B+Tree 是 B-Tree 的变体，主要有以下区别：

**B-Tree vs B+Tree：**

| 特性 | B-Tree | B+Tree |
|------|--------|--------|
| 数据存储 | 所有节点都存储数据 | 只有叶子节点存储完整数据 |
| 叶子节点 | 不维护链表 | 叶子节点通过双向链表串联 |
| 范围查询 | 需要中序遍历，可能需要回溯 | 只需遍历叶子链表，效率极高 |
| 非叶子节点 | 也存数据，扇出更小 | 只存索引键，扇出更大，树更矮 |

**为什么不用红黑树？** 红黑树是二叉树，IO 次数与数据量 O(log₂N) 相关。当数据量很大时，树高度很大，每次比较都要一次磁盘 IO，完全不现实。B+Tree 的一个节点可以存几百上千个 key，树高通常只有 2-4 层，IO 次数极低。

**为什么不用跳表？** 跳表本质也是二叉树变体，层级更多，且节点分散存储，对磁盘随机 IO 的利用不如 B+Tree 页式存储友好。跳表更适合内存场景（如 Redis ZSet 对应 skiplist）。

**为什么不用 Hash？** Hash 索引等值查询 O(1) 确实很快，但它不支持范围查询、排序、模糊匹配。实际业务中范围查询（`WHERE id > 100`、`BETWEEN`）远比纯等值查询常见。InnoDB 的 Adaptive Hash Index 就是折中方案——B+Tree 为主，热点等值查询自动建立哈希索引。

### 2.2 页（Page）结构详解

InnoDB 中最小的数据管理单位是**页**（Page），默认大小为 16KB。每种页类型有固定的结构：

```text
File Header    (38 bytes)   — 页号、类型、前后页指针等
Page Header    (56 bytes)   — 槽数量、记录数、空闲空间等
Infimum+Supremum  (26 bytes) — 最小虚拟记录和最大虚拟记录
User Records                 — 实际的用户数据行
Free Space                   — 空闲空间
Page Directory               — 页目录（槽数组），用于二分查找
File Trailer   (8 bytes)     — 校验和、LSN，用于完整性校验
```

**行格式（Row Format）：** InnoDB 支持多种行格式，MySQL 8.0 默认使用 `DYNAMIC`。

| 行格式 | 特点 |
|--------|------|
| REDUNDANT | 最老格式，很少使用 |
| COMPACT | 紧凑排列，MySQL 5.1 引入 |
| DYNAMIC | 溢出列完全存在单独的页，类似 COMPACT |
| COMPRESSED | 支持页级别压缩 |

**页内查找过程：** 页内通过 Page Directory（槽）做二分查找。Page Directory 每 4-8 条记录划分一个槽，槽指向该组中最大的记录。先在槽中二分定位到组，再在组内顺序遍历。时间复杂度约 O(log N)。

### 2.3 聚簇索引 vs 二级索引

**聚簇索引（Clustered Index）：**
- InnoDB 的主键索引就是聚簇索引
- 叶子节点存储的是**完整的行数据**
- 一张表只有一个聚簇索引
- 如果没有显式定义主键，InnoDB 会选择第一个唯一非空索引；若都没有，则创建一个隐藏的 6 字节 ROW_ID

**二级索引（Secondary Index）：**
- 非主键索引都是二级索引
- 叶子节点存储的是**索引键 + 主键值**
- 通过二级索引查找完整行时，需要两步：先在二级索引找到主键，再到聚簇索引读取整行。这个过程叫**回表**

```sql
-- 示例：表结构
CREATE TABLE users (
    id INT PRIMARY KEY,           -- 聚簇索引
    name VARCHAR(50),
    age INT,
    INDEX idx_name (name)         -- 二级索引
);

-- 回表现象
SELECT * FROM users WHERE name = 'zhangsan';
-- 先在 idx_name 中找到 name='zhangsan' 对应的 id
-- 再用 id 去聚簇索引中读取完整的行数据
```

**生产经验：** 设计表时建议使用自增 ID 作为主键。自增 ID 可以保证数据顺序插入，减少页分裂和随机 IO。使用 UUID 作为主键会导致插入点随机分布在 B+Tree 各处，造成大量页分裂和页空间浪费。

### 2.4 覆盖索引（Covering Index）

如果查询的列都可以在二级索引中找到，就不需要回表。这就是**覆盖索引**。

```sql
-- 普通查询：需要回表
SELECT * FROM users WHERE name = 'zhangsan';  -- * 包含所有列

-- 覆盖索引：不需要回表
SELECT id, name FROM users WHERE name = 'zhangsan';  -- id 和 name 都在 idx_name 中
```

EXPLAIN 中 `Extra` 列显示 `Using index` 即表示使用了覆盖索引。

**生产经验：** 高频查询应尽量设计覆盖索引。缺点是索引字段过多会增大索引维护成本和空间占用。另外注意 `SELECT *` 几乎永远无法使用覆盖索引。

### 2.5 索引下推（Index Condition Pushdown, ICP）

MySQL 5.6 引入的特性。核心思想：把部分 WHERE 条件过滤下推到存储引擎层，减少回表和返回 SQL 层的行数。

```sql
-- 假设有联合索引 idx(name, age)
SELECT * FROM users WHERE name LIKE '王%' AND age = 25;

-- 没有 ICP 时：
--   存储引擎先通过 idx 找到所有 name LIKE '王%' 的记录
--   逐条回表查出完整行，返回 SQL 层
--   SQL 层再过滤 age = 25

-- 有 ICP 时：
--   存储引擎在索引层面就直接过滤 age = 25
--   只有同时满足两个条件的记录才回表
--   大幅减少回表次数
```

EXPLAIN 中 `Extra` 列显示 `Using index condition` 表示使用了 ICP。

### 2.6 MRR（Multi-Range Read）

MRR 是 MySQL 5.6 引入的优化。核心思想：将二级索引查到的主键 ID 先排序，再按顺序回表，减少随机 IO。

这是因为二级索引的物理顺序和主键的物理顺序通常不一致。通过二级索引查到的主键通常是乱序的，直接回表会产生大量随机 IO。MRR 将主键排序后再按顺序回表，将随机 IO 转化为顺序 IO。

```sql
-- 在没有 idx(name, age) 但分别有 idx(name) 和 idx(age) 时
SELECT * FROM users WHERE name = 'zhangsan' AND age = 25;
-- MRR 将两次索引扫描的结果合并去重排序后批量回表
```

EXPLAIN 中 `Extra` 列显示 `Using MRR` 表示使用了 MRR。

### 2.7 前缀索引

对于 TEXT、VARCHAR 等长字符串列，可以只索引前 N 个字符来减少索引大小。

```sql
ALTER TABLE users ADD INDEX idx_email_prefix (email(8));
-- 只对 email 的前 8 个字符做索引
```

**选择前缀长度的原则：** 在索引大小和区分度之间取平衡。一般通过计算不同前缀长度的区分度来确定：

```sql
-- 计算不同前缀长度的区分度
SELECT COUNT(DISTINCT LEFT(email, 4)) / COUNT(*) AS sel4,
       COUNT(DISTINCT LEFT(email, 6)) / COUNT(*) AS sel6,
       COUNT(DISTINCT LEFT(email, 8)) / COUNT(*) AS sel8
FROM users;
```

**缺点：** 前缀索引无法用于 ORDER BY 和 GROUP BY，也无法用于覆盖索引。

### 2.8 索引设计建议

1. **高频查询优先建立索引**，按 WHERE + JOIN + ORDER BY 来考虑
2. **选择区分度高的列**，区分度 = `COUNT(DISTINCT col) / COUNT(*)`，一般大于 0.1
3. **联合索引遵循最左前缀原则**，把区分度高的列放在前面
4. **字符串列尽量使用前缀索引**，但注意前缀索引的局限性
5. **避免在索引列上做函数运算**，任何运算都会导致索引失效
6. **避免建立过多索引**，索引会降低写入性能（INSERT/UPDATE/DELETE 需要维护索引）
7. **利用覆盖索引**减少回表查询
8. **定期分析索引使用情况**，通过 `pt-duplicate-key-checker` 找到冗余索引

---

## 三、事务与 MVCC

### 3.1 事务四大特性（ACID）

| 特性 | 说明 | InnoDB 实现方式 |
|------|------|----------------|
| 原子性（Atomicity） | 事务的所有操作要么全部成功，要们全部失败 | Undo Log（回滚机制） |
| 一致性（Consistency） | 事务执行前后，数据从一个一致状态变为另一个一致状态 | 由其他三者共同保证 |
| 隔离性（Isolation） | 并发事务之间互不干扰 | MVCC + 锁 |
| 持久性（Durability） | 事务提交后数据永久保存 | Redo Log（WAL 机制） |

### 3.2 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|----------|------|---------|
| READ UNCOMMITTED | ✓ | ✓ | ✓ | 无锁，直接读最新数据 |
| READ COMMITTED | ✗ | ✓ | ✓ | 每次查询生成新 ReadView |
| REPEATABLE READ（默认） | ✗ | ✗ | 部分解决 | 事务开始时生成一个 ReadView |
| SERIALIZABLE | ✗ | ✗ | ✗ | 读加共享锁，全部串行化 |

```sql
-- 查看和设置隔离级别
SHOW VARIABLES LIKE 'transaction_isolation';
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

**脏读：** 读取到其他事务未提交的数据。
**不可重复读：** 同一事务内两次读取同一行数据，结果不同（其他事务提交了 UPDATE）。
**幻读：** 同一事务内两次查询，结果集的行数不同（其他事务插入了新行）。

### 3.3 MVCC 原理

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 实现高性能并发读写的基础。它的核心思想是：**读操作不阻塞写操作，写操作不阻塞读操作**。

InnoDB 通过在每行记录上维护两个隐藏字段来实现 MVCC：

| 隐藏字段 | 大小 | 说明 |
|---------|------|------|
| DB_TRX_ID | 6 bytes | 最近修改该行的事务 ID |
| DB_ROLL_PTR | 7 bytes | 回滚指针，指向 Undo Log 中的上一个版本 |
| DB_ROW_ID | 6 bytes | 隐藏主键（当没有显式主键时） |

**版本链：** 每次修改一行记录时，InnoDB 会将修改前的版本写入 Undo Log，新版本通过 `DB_ROLL_PTR` 指针指向旧版本，形成一个版本链。每个版本都带有当时修改该行的事务 ID（`DB_TRX_ID`）。

### 3.4 ReadView 与可见性判断

ReadView（读视图）是 MVCC 可见性判断的核心数据结构。ReadView 包含以下关键信息：

```text
m_ids:        创建 ReadView 时，系统中所有活跃（未提交）的读写事务 ID 集合
min_trx_id:   m_ids 中的最小值（或者 m_ids 为空时取当前最大事务 ID + 1）
max_trx_id:   创建 ReadView 时，系统下一个要分配的事务 ID（即当前最大事务 ID + 1）
creator_trx_id: 创建该 ReadView 的事务 ID
```

**可见性判断规则（针对版本链上的某个版本）：**

1. 如果 `版本DB_TRX_ID == creator_trx_id`：该版本由当前事务自己修改，**可见**
2. 如果 `版本DB_TRX_ID < min_trx_id`：说明修改该版本的事务在 ReadView 创建前已提交，**可见**
3. 如果 `版本DB_TRX_ID >= max_trx_id`：说明修改该版本的事务在 ReadView 创建后才开始，**不可见**
4. 如果 `min_trx_id <= 版本DB_TRX_ID < max_trx_id`：需要判断事务 ID 是否在 `m_ids` 中
   - 如果在 `m_ids` 中：该事务在 ReadView 创建时尚未提交，**不可见**
   - 如果不在 `m_ids` 中：该事务在 ReadView 创建时已提交，**可见**

对不可见的版本，沿着 `DB_ROLL_PTR` 向前查找上一个版本，直到找到第一个可见的版本为止。

### 3.5 可重复读 vs 读已提交的 ReadView 生成时机差异

这是面试高频问题，也是理解 MVCC 的关键。

**READ COMMITTED —— 每次快照读生成新的 ReadView：**
每次 SELECT 语句执行时，都重新生成一个 ReadView。这意味着如果其他事务在两次 SELECT 之间提交了修改，第二次 SELECT 就可以看到这些修改。这会导致不可重复读，但避免了读取未提交的数据（脏读）。

```sql
-- 时间线示例：RC 隔离级别
-- 事务 A                          事务 B
START TRANSACTION;
SELECT age FROM t WHERE id=1;  -- 读到 age=20
                                  UPDATE t SET age=25 WHERE id=1;
                                  COMMIT; -- 事务 B 提交
SELECT age FROM t WHERE id=1;  -- 读到 age=25（不可重复读！）
COMMIT;
```

**REPEATABLE READ —— 第一次快照读时生成 ReadView，此后不变：**
在事务开始后第一次执行 SELECT（快照读）时生成一个 ReadView，整个事务期间都使用这个 ReadView。因此即使其他事务提交了修改，当前事务也看不到。

```sql
-- 时间线示例：RR 隔离级别
-- 事务 A                          事务 B
START TRANSACTION;
SELECT age FROM t WHERE id=1;  -- 读到 age=20，生成了 ReadView
                                  UPDATE t SET age=25 WHERE id=1;
                                  COMMIT;
SELECT age FROM t WHERE id=1;  -- 仍然读到 age=20（可重复读！）
COMMIT;
```

**注意：** RR 隔离级别下，如果事务一开始执行的是 UPDATE/DELETE/SELECT FOR UPDATE 等加锁语句，也会立即生成 ReadView。

### 3.6 Undo Log 版本链

Undo Log 不仅用于事务回滚，也是 MVCC 版本链的基础。

```text
聚簇索引记录：

id=1, name='zhangsan', age=20
DB_TRX_ID=100
DB_ROLL_PTR ──→ Undo Log 记录 (age=18, DB_TRX_ID=90, roll_ptr ──→ ...)
```

每次 UPDATE 操作：
1. 先将修改前的旧值写入 Undo Log
2. 用当前事务的 `DB_TRX_ID` 更新记录
3. 将 `DB_ROLL_PTR` 指向刚写入的 Undo Log
4. 这样就形成了一个从新到旧的版本链

**Undo Log 的清理：** Undo Log 不能无限制增长。InnoDB 有一个后台线程（Purge Thread），当确定某个 Undo Log 版本不会再被任何活跃事务需要时，就会将其删除。Purge 的依据是系统中的最小活跃事务 ID（如果所有需要这个版本的事务都已经提交了，这个版本就可以清理了）。

### 3.7 MVCC 能解决幻读吗？

**在 REPEATABLE READ 隔离级别下，MVCC 解决了快照读的幻读问题，但不能完全解决当前读的幻读问题。**

快照读（Snapshot Read）——普通的 SELECT 语句，通过 MVCC 版本链保证同一事务内看到的数据一致，不会出现幻读。

当前读（Current Read）——`SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE`、`UPDATE`、`DELETE` 等，读取的是数据的最新版本，并且会加锁。对于当前读，InnoDB 通过 **Next-Key Lock**（临键锁）来解决幻读。

```sql
-- 幻读示例（快照读，RR下不会出现）
START TRANSACTION;
SELECT * FROM t WHERE id > 10; -- 假设返回 3 行
-- 此时事务 B 插入了 id=15 并提交
SELECT * FROM t WHERE id > 10; -- RR下仍返回 3 行，MVCC 避免了幻读

-- 当前读场景（RR下通过 Next-Key Lock 防止幻读）
START TRANSACTION;
SELECT * FROM t WHERE id > 10 FOR UPDATE; -- 加 Next-Key Lock，阻止其他事务插入
-- 此时事务 B 尝试 INSERT INTO t VALUES(15) 会被阻塞
```

实际上，RR 级别下事务 B 在第一种快照读场景中也能插入成功（快照读不加锁），只是事务 A 的第二次 SELECT 仍然是看到 3 行。所以严格来说，RR 解决了快照读的幻读，而 SERIALIZABLE 才能彻底解决包括当前读在内的幻读。不过在实际业务中，我们说的幻读通常指快照读层面的。

---

## 四、锁机制

### 4.1 锁的类型概览

InnoDB 的锁体系可以按不同维度分类：

**按粒度：**
- 行级锁（Row-level Lock）—— InnoDB 默认，粒度最细
- 表级锁（Table-level Lock）—— 如 `LOCK TABLES`，一般不用
- 页级锁（Page-level Lock）—— BDB 引擎使用

**按模式：**
- 共享锁（S Lock）
- 排他锁（X Lock）
- 意向共享锁（IS Lock）
- 意向排他锁（IX Lock）

**按算法（行锁的实现方式）：**
- 记录锁（Record Lock）
- 间隙锁（Gap Lock）
- 临键锁（Next-Key Lock）

### 4.2 共享锁（S Lock）与排他锁（X Lock）

**共享锁（Shared Lock）：** 允许持有锁的事务读取一行数据。多个事务可以同时对同一行持有共享锁。

```sql
SELECT ... LOCK IN SHARE MODE;   -- MySQL 8.0 之前
SELECT ... FOR SHARE;            -- MySQL 8.0 推荐写法
```

**排他锁（Exclusive Lock）：** 允许持有锁的事务更新或删除一行数据。排他锁与其他任何锁都不兼容。

```sql
-- 自动加排他锁
SELECT ... FOR UPDATE;
UPDATE ... WHERE ...;
DELETE FROM ... WHERE ...;
INSERT INTO ...;  -- 插入操作会自动加排他锁
```

**S 锁与 X 锁的兼容矩阵：**

|     | S   | X   |
|-----|-----|-----|
| S   | ✓   | ✗   |
| X   | ✗   | ✗   |

### 4.3 记录锁（Record Lock）

记录锁锁住的是**单个索引记录**，不是整行数据。InnoDB 的行锁实际上是加在索引上的。

```sql
-- 假设 id 是主键
SELECT * FROM t WHERE id = 5 FOR UPDATE;
-- 对 id=5 的聚簇索引记录加 X 锁
```

**重要：** 如果查询没有走索引，InnoDB 会对所有扫描到的行加锁！例如 `WHERE name = 'zhangsan'` 如果没有索引，会锁住全表所有记录。

```sql
-- 危险的查询！如果没有索引会导致全表锁
SELECT * FROM t WHERE unindexed_col = 'value' FOR UPDATE;
-- 实际效果类似于锁表
```

### 4.4 间隙锁（Gap Lock）

间隙锁锁住的是**索引记录之间的间隙**，这个间隙中不允许插入新的记录。间隙锁之间不冲突——多个事务可以在同一个间隙上持有间隙锁（这是为了防止幻读时对同一段间隙加锁）。

```sql
-- 假设 t 表 id 值有 1, 3, 5, 7
START TRANSACTION;
SELECT * FROM t WHERE id > 3 AND id < 7 FOR UPDATE;
-- 间隙锁加在 (3, 5) 和 (5, 7) 两个区间
-- 此时其他事务不能在这些间隙中插入记录（如 id=4, id=6）
```

间隙锁**最大的影响**是它可能锁住一个很宽的范围，导致并发性能下降。

**生产经验：** 间隙锁只在 REPEATABLE READ 及以上隔离级别生效。如果将隔离级别降为 READ COMMITTED，间隙锁会被禁用，可以提升并发插入性能，但代价是可能出现幻读。如果业务上使用了唯一约束来防止重复，可以考虑在 RC 级别运行。

### 4.5 临键锁（Next-Key Lock）

Next-Key Lock = Record Lock + Gap Lock，即锁住一条记录及其前面的间隙。它是 InnoDB 默认的行锁实现方式。

```text
索引记录（主键值）：1    5    10    15

Next-Key Lock 锁范围：
(-∞, 1]  (1, 5]  (5, 10]  (10, 15]  (15, +∞)
```

每一段 Next-Key Lock 锁住**前开后闭**的区间：**前一条记录到当前记录的左开右闭区间**。其中 `(` 对应的 Record Lock，`(` 到该记录之间是 Gap Lock。

**退化为 Record Lock 的场景：** 当查询条件是**等值查询**且索引列是**唯一索引**时，Next-Key Lock 会退化为 Record Lock，不再加 Gap Lock。

```sql
-- 等值查询 + 唯一索引：退化为 Record Lock
SELECT * FROM t WHERE id = 5 FOR UPDATE;  -- 只锁 id=5 这一条

-- 等值查询 + 非唯一索引：Next-Key Lock 完整加锁
SELECT * FROM t WHERE name = 'zhangsan' FOR UPDATE;  -- 不仅锁记录，还锁间隙

-- 范围查询：不论是否为唯一索引，都是 Next-Key Lock
SELECT * FROM t WHERE id > 5 FOR UPDATE;  -- 锁 (5, 10], (10, 15], (15, +∞)
```

### 4.6 意向锁（Intention Lock）

意向锁是**表级锁**，用于协调行锁和表锁之间的关系。

**问题场景：** 假设事务 A 对某几行加了行级 X 锁，事务 B 想要对整张表加 X 锁。事务 B 不知道有没有行级锁，它需要扫描表中每一行来判断。意向锁解决了这个问题——事务 A 加行锁前，先对表加一个意向锁。

- **意向共享锁（IS Lock）：** 事务打算给某些行加共享锁前，先加表级 IS 锁
- **意向排他锁（IX Lock）：** 事务打算给某些行加排他锁前，先加表级 IX 锁

这样一来，事务 B 加表级锁时只需检查表上的意向锁即可，不需要逐行扫描。

**兼容矩阵：**

|     | IS  | IX  | S   | X   |
|-----|-----|-----|-----|-----|
| IS  | ✓   | ✓   | ✓   | ✗   |
| IX  | ✓   | ✓   | ✗   | ✗   |
| S   | ✓   | ✗   | ✓   | ✗   |
| X   | ✗   | ✗   | ✗   | ✗   |

### 4.7 AUTO-INC 锁

AUTO-INC 锁是一种特殊的表级锁，用于保护自增主键的并发安全。

**传统 AUTO-INC 锁：** 在 INSERT 语句持有锁期间，其他 INSERT 必须等待。锁的释放时机不是事务提交，而是 INSERT 语句执行完毕。这意味着同一个事务内的多条 INSERT 不能并发执行。

**轻量级锁（MySQL 5.1.22 后默认）：** `innodb_autoinc_lock_mode=2` 时，InnoDB 使用更轻量级的 mutex 而不是表级锁来分配自增 ID。这样可以支持并发 INSERT。但要注意轻量级锁模式下，`INSERT ... SELECT` 这类批量语句自增 ID 分配是连续的，而简单 INSERT 的 ID 不保证连续。

```sql
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
-- 0: 传统模式, 1: 连续模式（默认）, 2: 交错模式
```

### 4.8 死锁检测与处理

**死锁产生的四个必要条件：**
1. 互斥：资源不能共享
2. 持有并等待：持有资源的同时等待其他资源
3. 不可剥夺：已获得的资源不能被强制夺走
4. 循环等待：事务之间形成环路

**InnoDB 死锁处理：**
- InnoDB 会自动检测死锁（通过等待图 wait-for graph），发现死锁后回滚代价最小的事务
- 回滚后释放锁，其他事务继续执行
- 应用程序收到死锁错误（Error 1213）后，应重试事务

```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G
-- 查找 "LATEST DETECTED DEADLOCK" 段落
```

**避免死锁的实践经验：**

1. **按相同顺序访问资源：** 不同事务以相同顺序对记录加锁，这是最重要的原则
2. **减小事务粒度：** 把大事务拆成小事务，减少持锁时间
3. **尽量使用主键或唯一索引：** 减少加锁范围（唯一索引等值查询只加 Record Lock）
4. **避免在事务中做耗时操作：** 如网络 IO、远程调用，缩短持锁时间
5. **使用 READ COMMITTED 隔离级别：** 如果业务允许，RC 级别没有 Gap Lock，减少锁冲突几率
6. **使用低隔离级别：** 可以在允许脏读的场景使用 RC 以减少锁竞争

---

## 五、日志系统

### 5.1 Redo Log（重做日志）

Redo Log 是 InnoDB 的事务日志，记录了对数据页的物理修改。核心作用是保证事务的**持久性（Durability）**。

**WAL（Write-Ahead Logging）原则：** 在将修改真正写入数据文件之前，必须先保证 Redo Log 写入磁盘。这是 InnoDB 实现快速写入的基石——因为 Redo Log 是顺序写入的，而数据写入是随机的。

**Redo Log 文件：** 通常是固定大小的一组文件，如 `ib_logfile0` 和 `ib_logfile1`，循环写入。可以通过 `innodb_log_files_in_group` 和 `innodb_log_file_size` 配置。

```sql
SHOW VARIABLES LIKE 'innodb_log_file_size';     -- 单个文件大小，默认 48MB
SHOW VARIABLES LIKE 'innodb_log_files_in_group'; -- 文件个数，默认 2
-- 总大小 = innodb_log_file_size * innodb_log_files_in_group
```

**Redo Log 的三种空间状态：**
- **LSN（Log Sequence Number）：** 一个单调递增的 8 字节整数，是 Redo Log 的总写入量（逻辑偏移量）
- **flushed_to_disk_lsn：** 已经刷新到磁盘的 Redo Log 位置
- **last_checkpoint_lsn：** 上一次 Checkpoint 的位置

```sql
-- 查看 LSN
SHOW ENGINE INNODB STATUS\G
-- 查找 "LOG" 段落
```

**Redo Log 的写入与 Checkpoint 机制：**

Redo Log 循环写入，当写满时需要触发 Checkpoint——将内存中的脏页刷到数据文件，然后才能覆盖旧的 Redo Log。如果脏页刷盘速度跟不上 Redo Log 写入速度，用户线程会被阻塞，出现明显的性能抖动。这就是为什么 Redo Log 不能设置得太小的原因。

**生产经验：**
- Redo Log 大小影响写性能。如果 Redo Log 太小，Checkpoint 频繁触发，大量写 IO 引发抖动。建议单文件 1G-4G（视写入量）。
- MySQL 8.0 支持在线调整 `innodb_log_file_size`，无需重启
- 监控 `Innodb_log_waits` 状态，> 0 表示 Redo Log 不够用

### 5.2 Undo Log（回滚日志）

Undo Log 提供了两件事：
1. **事务回滚：** 记录修改前的旧数据，ROLLBACK 时恢复
2. **MVCC：** 提供历史版本供多版本并发控制使用

**Undo Log 的类型：**
- **INSERT Undo Log：** INSERT 操作产生的日志。事务提交后可以直接删除（因为 INSERT 的数据对外部事务不可见）
- **UPDATE Undo Log：** UPDATE/DELETE 操作产生的日志。需要保留到不再有事务需要其历史版本为止

**Undo Tablespace：** MySQL 8.0 中 Undo Log 有独立的 tablespace 文件，可以配置为多个文件，支持在线截断（truncate）回收空间。

```sql
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';  -- 独立 Undo 表空间数量
SHOW VARIABLES LIKE 'innodb_max_undo_log_size'; -- 单个表空间最大大小
SHOW VARIABLES LIKE 'innodb_purge_batch_size';  -- Purge 批处理大小
```

**大事务的风险：** 长事务会导致 Undo Log 无法回收，可能把 Undo 表空间撑得特别大。此外，Undo Log 的 Purge 线程只能清理大于当前所有活跃事务最小 ID 的旧版本，长事务会阻止清理。

**生产经验：** 监控长事务是 DBA 或 SRE 的日常。通过以下查询找到长事务并报警：

```sql
SELECT * FROM information_schema.innodb_trx
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;
```

### 5.3 Binlog（二进制日志）

Binlog 是 MySQL Server 层的日志（不是 InnoDB 的），记录了所有修改数据库的操作。它的三大用途：
1. **主从复制：** Master 的 Binlog 发送到 Slave，Slave 重放
2. **数据恢复：** 基于时间点的恢复（Point-in-Time Recovery）
3. **审计：** 可用于数据变更溯源

**三种 Binlog 格式：**

| 格式 | 记录内容 | 优点 | 缺点 |
|------|---------|------|------|
| STATEMENT | SQL 语句 | 日志量小 | 主从不一致风险（如 NOW(), UUID()） |
| ROW | 行级别的变更（变更前后各列的值） | 安全可靠，不会产生不一致 | 日志量大，尤其是批量操作 |
| MIXED | 默认 STATEMENT，特定情况自动切换 ROW | 两者折中 | 还是存在部分语句不可控 |

```sql
SHOW VARIABLES LIKE 'binlog_format';     -- 推荐 ROW
SHOW VARIABLES LIKE 'sync_binlog';       -- 1: 每次提交刷盘，推荐设为 1
SHOW VARIABLES LIKE 'max_binlog_size';   -- 单个 Binlog 文件大小上限
```

**生产经验：** 线上强烈推荐使用 **ROW 格式**。STATEMENT 格式中 `NOW()`、`UUID()`、`LIMIT` 无 ORDER BY 等都可能导致主从不一致。ROW 格式虽然日志量大，但这是值得的安全性投入。

### 5.4 二阶段提交（Two-Phase Commit）

二阶段提交是为了解决 **Redo Log 和 Binlog 之间的一致性问题**。因为 Redo Log 属于 InnoDB 引擎，而 Binlog 属于 MySQL Server 层，它们是两个独立的日志系统。

**为什么需要二阶段提交？** 如果没有协调机制，可能会出现：
- 先写 Redo Log 后写 Binlog：如果写完 Redo Log 后 MySQL 宕机，Binlog 没写，从库少了这条数据
- 先写 Binlog 后写 Redo Log：如果写完 Binlog 后宕机，Redo Log 没写，主库恢复后发现数据不对

**二阶段提交流程：**

```text
1. PREPARE 阶段：
   ─→ InnoDB 将事务的 Redo Log 刷盘
   ─→ 但事务还未真正提交（处于 PREPARE 状态）
   ─→ 此时 XA 事务 ID 被写入 Redo Log

2. COMMIT 阶段：
   ─→ 写 Binlog 并刷盘
   ─→ InnoDB 提交事务（将 Redo Log 中的 PREPARE 改为 COMMIT）
   ─→ 清理 Undo Log purge 标记
```

**崩溃恢复时如何判断：**

恢复时扫描 Binlog，对于每个 XA 事务：
- 如果 Binlog 中有完整记录 → 事务应该提交，重做 Redo Log
- 如果 Binlog 中没有记录 → 事务处于 PREPARE 状态且 Binlog 未写，回滚

这样就保证了主从数据一致。

### 5.5 崩溃恢复流程

当 MySQL 异常宕机后重启时，恢复流程如下：

1. **加载单页恢复**
   - 检查 Doublewrite Buffer，修复损坏的数据页
   
2. **Redo Log 前滚（重做）**
   - 从上次 Checkpoint 的 LSN 开始，顺序读取 Redo Log
   - 将 Redo Log 中记录的修改逐条应用到数据页
   - 重放完成后，数据库恢复到宕机前的"物理一致"状态
   
3. **Undo Log 回滚（撤销未提交事务）**
   - 扫描 Undo Log，找到 PREPARE 状态的事务
   - 检查 Binlog 中该事务是否完整：
     - 若 Binlog 中有 → 提交该事务（因为从库需要同步）
     - 若 Binlog 中无 → 回滚该事务
   - 对于活跃事务（未 PREPARE），直接回滚
   
4. **正常运行**
   - 完成上述步骤后，MySQL 恢复到一致状态，开始接受连接

### 5.6 LSN 与 Checkpoint

**LSN（Log Sequence Number）** 是 Redo Log 的核心概念。它是一个 8 字节的单调递增整数，代表 Redo Log 写入的总字节数（逻辑上）。

**LSN 值在以下多处出现：**
- 每个数据页的 File Header 中记录该页最后一次被修改的 LSN
- Redo Log 中的每条记录都有自己的 LSN
- Checkpoint 信息记录在日志文件的 Header 中

**Checkpoint（检查点）** 的作用：
1. 缩短崩溃恢复时间——恢复只需从最近 Checkpoint 开始重放 Redo Log
2. 将 Buffer Pool 中的脏页刷盘——当 Redo Log 空间不足时
3. 脏页刷盘——保证脏页不会永远留在内存

**Checkpoint 类型：**
- **Sharp Checkpoint：** 关闭时把所有脏页刷盘。只发生在正常关闭
- **Fuzzy Checkpoint：** 运行时逐步刷脏页。有几种触发方式：
  - Master Thread Checkpoint：每秒或每十秒一次
  - FLUSH_LRU_LIST Checkpoint：保证 LRU 有足够的空闲页
  - Async/Sync Flush Checkpoint：Redo Log 快写满了
  - Dirty Page too much Checkpoint：脏页比例超过阈值

```sql
SHOW VARIABLES LIKE 'innodb_max_dirty_pages_pct';      -- 脏页比例上限，默认 90%
SHOW VARIABLES LIKE 'innodb_io_capacity';               -- 后台 I/O 吞吐量，建议设为 SSD IOPS
SHOW VARIABLES LIKE 'innodb_flush_neighbors';            -- SSD 建议关闭(0)，减少写放大
```

---

## 六、SQL 优化实战

### 6.1 EXPLAIN 详解

EXPLAIN 是 SQL 优化的第一步，它展示了 MySQL 如何执行这条 SQL。

```sql
EXPLAIN SELECT * FROM users WHERE name = 'zhangsan' AND age > 20;
```

**EXPLAIN 输出字段详解：**

| 字段 | 含义 | 关键值 |
|------|------|--------|
| id | SELECT 序号 | 值越大优先执行，相同时从上到下 |
| select_type | 查询类型 | SIMPLE（简单查询）、PRIMARY（最外层）、SUBQUERY、DERIVED（派生表）、UNION |
| table | 访问的表 | — |
| partitions | 命中的分区 | — |
| **type** | **访问类型** | **最重要！从好到坏见下节** |
| possible_keys | 可能使用的索引 | — |
| key | 实际使用的索引 | NULL 表示没用索引 |
| key_len | 使用的索引字节数 | 用于判断使用了联合索引的多少列 |
| ref | 与索引比较的列或常量 | const、某个列名、func |
| rows | 估计需要扫描的行数 | 越小越好 |
| filtered | 返回行数占扫描行数的百分比 | 越高越好，100% 最理想 |
| **Extra** | **额外信息** | **同样重要，见下文** |

**Extra 字段重要值：**

| 值 | 说明 | 建议 |
|----|------|------|
| Using index | 覆盖索引，不需要回表 | 最优 |
| Using index condition | 使用了 ICP | 良好 |
| Using where | MySQL Server 层做了过滤 | 存储引擎返回的数据多 |
| Using temporary | 使用了临时表 | 需要优化（通常由 GROUP BY / DISTINCT 导致） |
| Using filesort | 需要额外排序 | 需要优化，考虑给排序列加索引 |
| Using join buffer | 使用了 join buffer | 需要优化 join 条件或索引 |
| Impossible WHERE | WHERE 条件恒假 | 检查业务逻辑 |

### 6.2 type 字段从最优到最差

```text
system  — 表只有一行（系统表）
const   — 最多匹配一行（主键等值查询或唯一索引等值查询）
eq_ref  — 每个被驱动表和前表匹配时最多返回一行（JOIN 中唯一索引等值查询）
ref     — 非唯一索引等值查询，返回多行
fulltext— 全文索引
ref_or_null — 类似 ref，但包含 NULL 查找
index_merge — 使用了索引合并优化
unique_subquery — IN 子查询中使用主键
index_subquery — IN 子查询中使用非唯一索引
range   — 索引范围扫描（BETWEEN、>、<、IN 等）
index   — 全索引扫描（比 ALL 好，因为索引通常比数据小）
ALL     — 全表扫描（最差！）
```

**生产经验：** 目标至少是 `range`，最好达到 `ref`。出现 `ALL` 就需要立刻警觉并添加索引。

### 6.3 索引失效的常见场景

以下场景会导致索引部分或完全失效（假设使用联合索引 `idx(a, b, c)`）：

**1. 违反最左前缀原则**
```sql
SELECT * FROM t WHERE b = 1 AND c = 2;   -- 跳过 a，索引失效
SELECT * FROM t WHERE c = 3;             -- 跳过 a 和 b，索引失效
```

**2. 索引列上做函数运算或表达式计算**
```sql
SELECT * FROM t WHERE YEAR(create_time) = 2025;  -- 函数破坏索引
SELECT * FROM t WHERE create_time >= '2025-01-01' AND create_time < '2026-01-01'; -- 正确写法
SELECT * FROM t WHERE id + 1 = 10;               -- 表达式破坏索引（MySQL 8.0.13+ 某些情况可优化）
```

**3. 隐式类型转换（类型对索引列）**
```sql
SELECT * FROM t WHERE phone = 13800138000;  -- phone 是 VARCHAR，做了隐式转换 → CAST(phone AS SIGNED)
-- 但反过来不会失效：WHERE id = '123' → CAST('123' AS SIGNED)，索引列没被转换
```

**4. 模糊查询前缀通配符**
```sql
SELECT * FROM t WHERE name LIKE '%zhang';  -- 前缀 %，索引失效
SELECT * FROM t WHERE name LIKE 'zhang%';  -- 后缀 %，索引有效
```

**5. OR 条件中存在非索引列**
```sql
SELECT * FROM t WHERE a = 1 OR b = 2;      -- b 没有索引，整体不使用索引（会用 index_merge 看情况）
SELECT * FROM t WHERE a = 1 OR a = 2;      -- 会使用 a 索引
```

**6. NOT、!=、<> 对索引列**
```sql
SELECT * FROM t WHERE status != 0;   -- 不等于可能不走索引（取决于数据分布）
SELECT * FROM t WHERE NOT a = 1;     -- NOT 同理
```

**7. IS NULL 和 IS NOT NULL（取决于数据分布）**
```sql
SELECT * FROM t WHERE name IS NULL;      -- 可能走也可能不走，取决于 NULL 比例
SELECT * FROM t WHERE name IS NOT NULL;  -- 同理
```

**8. 联合索引中间某列使用范围查询，后续列失效**
```sql
SELECT * FROM t WHERE a = 1 AND b > 10 AND c = 2;
-- a 和 b 走索引，c 不走（范围查询后，后续列索引失效）
```

**9. 参与列对比**
```sql
SELECT * FROM t WHERE a = b;  -- 同表列对比，不走索引
```

**10. ORDER BY 的坑**
```sql
SELECT * FROM t WHERE a = 1 ORDER BY c;  -- 中间跳过了 b，需要 filesort
SELECT * FROM t WHERE a = 1 ORDER BY a, b, c; -- 完美匹配索引顺序
```

**11. 数据量太小，优化器放弃索引**
```sql
-- 表只有几十行时，优化器认为全表扫描更快
```

**12. leading wildcard 导致字符集转换**
```sql
-- 两表 JOIN 时字符集不同，MySQL 会做隐式转换导致索引失效
-- 一个用 utf8，一个用 utf8mb4 就可能触发
```

### 6.4 慢查询日志（Slow Query Log）

慢查询日志记录执行时间超过 `long_query_time` 的 SQL 语句。

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒视为慢查询（秒，可设小数）
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
SET GLOBAL log_slow_admin_statements = ON;      -- 记录 DDL 等管理操作
SET GLOBAL min_examined_row_limit = 1000;       -- 扫描行数大于此值的才记录

SHOW VARIABLES LIKE 'slow_query_log_file';  -- 慢查询日志文件路径
```

**慢查询分析工具：**

- **mysqldumpslow：** MySQL 自带，基础的慢查询统计
  ```bash
  mysqldumpslow -s t -t 10 slow.log   # 按总时间排序，显示前 10 条
  mysqldumpslow -s c -t 10 slow.log   # 按出现次数排序
  ```

- **pt-query-digest（Percona Toolkit）：** 功能强大，业界标准
  ```bash
  pt-query-digest slow.log > report.txt   # 生成分析报告
  pt-query-digest --since='12h' slow.log  # 分析最近 12 小时
  ```

### 6.5 Profiling

MySQL 的 SHOW PROFILE 可以分析单个 SQL 语句的执行耗时分布。

```sql
SET profiling = 1;
SELECT * FROM users WHERE name = 'zhangsan';
SHOW PROFILES;  -- 查看所有已记录的查询
SHOW PROFILE FOR QUERY 1;  -- 查看某条查询的详细耗时

-- 查看 CPU、IO 等细分维度
SHOW PROFILE CPU, BLOCK IO FOR QUERY 1;
```

**主要阶段耗时关注：**
- `Sending data` — 数据读取、过滤、发送的时间，通常是主要优化目标
- `Creating tmp table` — 创建临时表，说明有 GROUP BY / ORDER BY 导致
- `Sorting result` — 排序，filesort 耗时
- `System lock` — 锁等待时间

> 注意：profiling 功能在 MySQL 8.0 中已标记为 deprecated，推荐使用 `performance_schema` 替代。

### 6.6 JOIN 优化

**NLJ（Nested Loop Join，嵌套循环连接）：** 最基本的 JOIN 算法，两层循环。驱动表每取一行，都去被驱动表中找到匹配的行。

```text
for each row in 驱动表:
    for each row in 被驱动表:
        if match: output
```

被驱动表需要**在 JOIN 列上有索引**。驱动表应该选择**结果集较小**的表（小表驱动大表）。

**BNL（Block Nested Loop Join）：** 当被驱动表没有索引时，将驱动表数据加载到 Join Buffer，然后全表扫描被驱动表做匹配。性能较差。

**Hash Join（MySQL 8.0.18+）：** MySQL 8.0.18 引入了 Hash Join 替代 BNL。先构建哈希表（小表作为驱动表），再通过对大表做哈希查找匹配。复杂度 O(N+M)，极大提升了无索引 JOIN 的性能。

**JOIN 优化原则：**
1. 小表驱动大表
2. 被驱动表的 JOIN 列必须要有索引
3. 只 SELECT 需要的列，避免 `SELECT *`
4. 多级 JOIN 时，每一级 JOIN 的中间结果集越小越好

```sql
-- 查看 JOIN 的消耗
EXPLAIN SELECT u.name, o.order_no
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.create_time > '2025-01-01';
-- 注意观察 type 列、ref 列、Extra 列
```

### 6.7 LIMIT 优化

大偏移量分页查询是经典问题。当 `LIMIT offset, size` 中 offset 很大时，MySQL 需要扫描前面所有行再丢弃，效率很低。

```sql
-- 差：需要扫描 100020 行，然后丢弃前 100000 行
SELECT * FROM t ORDER BY id LIMIT 100000, 20;
```

**优化方案一：子查询 + 覆盖索引**
```sql
-- 先在子查询中用覆盖索引快速定位到起始 ID
SELECT * FROM t WHERE id >= (
    SELECT id FROM t ORDER BY id LIMIT 100000, 1
) ORDER BY id LIMIT 20;
```

**优化方案二：使用主键作为游标**
```sql
-- 前提：主键自增，上一页最后一条 id = 99999
SELECT * FROM t WHERE id > 99999 ORDER BY id LIMIT 20;
-- 这种方式也叫"游标翻页"，性能极佳但无法跳页
```

**优化方案三：延迟关联**
```sql
-- 先在覆盖索引中找到主键，再回表
SELECT t1.* FROM t t1
INNER JOIN (SELECT id FROM t ORDER BY id LIMIT 100000, 20) t2
ON t1.id = t2.id;
```

### 6.8 COUNT(*) 优化

**MyISAM 的 COUNT(*)：** MyISAM 把表的总行数记录在磁盘，COUNT(*) 直接返回，O(1)。

**InnoDB 的 COUNT(*)：** InnoDB 没有记录总行数（因为 MVCC 的原因，每行有多个版本），必须遍历索引来计数。

**优化技巧：**

```sql
-- 1. 优先用 COUNT(1) 或 COUNT(*)，性能一样
SELECT COUNT(*) FROM t; -- 和 COUNT(1) 性能相同，MySQL 会做优化

-- 2. 有二级索引时，优化器会选择最小的二级索引扫描
-- 二级索引叶子只存索引列 + 主键，比聚簇索引小得多

-- 3. 带 WHERE 条件的 COUNT：保证 WHERE 列有索引
SELECT COUNT(*) FROM t WHERE status = 1; -- status 需要索引

-- 4. 对于超高频率的分页总数查询，考虑：
--    a) 使用 Redis 计数器
--    b) 使用额外的统计表（用触发器或定时任务汇总）
--    c) ES 中存储统计值（如果已经有 ES）
-- 5. 如果只是判断"是否存在"，用 LIMIT 1
SELECT 1 FROM t WHERE status = 1 LIMIT 1; -- 比 COUNT(*) 快
```

**生产经验：** COUNT(*) 是线上最常见的慢查询之一，尤其是几十亿行的大表。如果业务允许显示近似值（如"约 10000 条数据"），可以用 `EXPLAIN` 预估行数或 `SHOW TABLE STATUS` 的行数估算（但后者不精确，误差可能达到 40-50%）。

### 6.9 其他优化技巧

**避免 SELECT *：** 只查需要的列，这样可以：
- 使用覆盖索引避免回表
- 减少网络传输数据量
- 减少内存占用

**合理使用 IN vs EXISTS：**
```sql
-- IN：子查询结果集小用 IN
SELECT * FROM t1 WHERE id IN (SELECT id FROM t2 WHERE ...);

-- EXISTS：子查询结果集大，外表小用 EXISTS
SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id);
```

**批量操作代替循环：**
```java
// 差：逐条 INSERT
for (User user : users) {
    jdbcTemplate.update("INSERT INTO users VALUES (?, ?)", user.getId(), user.getName());
}

// 好：批量 INSERT
jdbcTemplate.batchUpdate("INSERT INTO users VALUES (?, ?)", users, 1000,
    (ps, user) -> {
        ps.setLong(1, user.getId());
        ps.setString(2, user.getName());
    });
```

**连接池配置建议（HikariCP）：**
```java
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(20);       // 公式：connections = ((core_count * 2) + effective_spindle_count)
config.setMinimumIdle(10);           // 最小空闲链接
config.setConnectionTimeout(30000);  // 等待连接超时
config.setIdleTimeout(600000);       // 空闲连接超过此时间被回收
config.setMaxLifetime(1800000);      // 连接最大存活时间（应小于数据库 wait_timeout）
```

---

## 七、高可用架构

### 7.1 主从复制基本原理

MySQL 主从复制的核心是依靠 Binlog 的同步。基本原理分三步：

```text
Master                                   Slave
┌──────────┐                        ┌──────────────┐
│  写入    │  ①I/O Thread 拉取      │              │
│  产生    │◄───────────────────────│  I/O Thread  │
│  Binlog  │  发送 Binlog Events    │      ↓       │
│          │                        │  Relay Log   │
└──────────┘                        │      ↓       │
                                    │ SQL Thread   │
                                    │   重放 SQL    │
                                    └──────────────┘
```

1. **Master** 把数据变更写入 Binlog
2. **Slave 的 I/O 线程**连接 Master，读取 Binlog 并写入 Relay Log
3. **Slave 的 SQL 线程**读取 Relay Log，重放 SQL 将变更应用到从库

**复制延迟的主要原因：**
- 主库并发写入多，从库只有一个 SQL 线程串行重放（5.6 之前）
- 网络延迟
- 从库硬件配置低于主库
- 大事务导致卡顿

### 7.2 复制格式与选择

和 Binlog 的三种格式对应：

| 格式 | 复制特点 | 推荐场景 |
|------|---------|---------|
| STATEMENT | Slave 执行 SQL，可能有主从不一致 | 不推荐 |
| ROW | 复制的是行变更数据，安全可靠 | **推荐线上使用** |
| MIXED | 混合模式 | 不建议，不确定性大 |

**ROW 格式复制的注意点：**
- `FULL`（默认）：前后镜像都记录，日志量大
- `MINIMAL`：只记录变更的列和标识列，日志量小
- `NOBLOB`：和 FULL 类似，但 TEXT/BLOB 列不记录前镜像

```sql
SHOW VARIABLES LIKE 'binlog_row_image';  -- 推荐 FULL 或 MINIMAL
```

### 7.3 并行复制

**传统单线程复制瓶颈：** MySQL 5.5 及之前，Slave 只有一个 SQL 线程，从库重放速度跟不上主库写入速度，造成主从延迟。

**MySQL 5.6 库级并行复制：** 按 Schema（库）并行。不同库的事务可以并行执行。但如果只有一个库，无济于事。

**MySQL 5.7 基于组提交的并行复制（LOGICAL_CLOCK）：**

核心原理：在主库上，能够在同一个 Binlog Group Commit 中提交的事务之间没有锁冲突，因此在从库上也可以并行执行（因为同一个时间内这些事务之间没有锁冲突，不存在依赖关系）。

```sql
-- 从库开启并行复制
STOP SLAVE SQL_THREAD;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;  -- 并行 worker 数量
START SLAVE SQL_THREAD;
```

**MySQL 8.0 基于 Write-Set 的并行复制：**

8.0 进一步优化，使用 Write-Set 来判断事务之间是否有冲突。Write-Set 是事务所有写操作的行的哈希值集合。如果两个事务的 Write-Set 没有交集，它们就可以并行执行。这种方式比 LOGICAL_CLOCK 粒度更细。

```sql
SET GLOBAL binlog_transaction_dependency_tracking = 'WRITESET';
```

### 7.4 GTID 复制

**GTID（Global Transaction Identifier，全局事务标识符）** 是 MySQL 5.6 引入的，为每个事务分配全局唯一的 ID。

```text
GTID 格式：server_uuid:transaction_id
例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5
表示 server_uuid 为 3E11FA47... 的服务器上的第 1 到 5 个事务
```

**GTID 的优势：**
1. 简化故障切换——不需要指定 Binlog 文件名和位置，自动寻找断点
2. 支持多源复制（多主一从）
3. 从库切换主库更简单——CHANGE MASTER TO MASTER_AUTO_POSITION = 1

```sql
-- 开启 GTID
-- my.cnf 配置：
-- gtid_mode = ON
-- enforce_gtid_consistency = ON
-- 从库配置：
CHANGE MASTER TO MASTER_AUTO_POSITION = 1;
```

**GTID 的限制：**
- 不支持 `CREATE TABLE ... SELECT`（因为它是两个事务：DDL + DML）
- 不支持非事务引擎（如 MyISAM）的临时表
- 事务内不能同时更新事务表和非事务表

### 7.5 半同步复制（Semi-Synchronous Replication）

**异步复制的风险：** 主库提交后立即返回客户端，如果此时主库宕机，未同步到从库的数据永久丢失（因为从库还没来得及拉取 Binlog）。

**半同步复制流程：**
1. 主库提交事务，写 Binlog
2. 主库等待至少一个从库确认已收到 Binlog（不是重放完成，只是收到）
3. 收到 ACK 后，主库才返回客户端

```sql
-- 主库安装插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 10000; -- 等待 ACK 超时时间(ms)

-- 从库安装插件
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
```

**超时退化：** 如果等待 ACK 超时，主库自动降级为异步复制。当从库恢复后，再自动切回半同步。

**无损复制（Lossless Semi-Sync，MySQL 5.7+）：**

传统半同步是 `AFTER_COMMIT` —— 主库先提交，再等 ACK。5.7 引入了 `AFTER_SYNC`（无损复制）：
- AFTER_COMMIT：主库提交 → 等 ACK → 返回客户端。缺点是**其他事务可能已经读到这个数据，但主库宕机后数据丢失**（因为从库没同步到）
- AFTER_SYNC（默认）：主库等着 ACK → 收到 ACK 后提交 → 返回客户端。**保证数据一定在从库有了才提交**

```sql
SHOW VARIABLES LIKE 'rpl_semi_sync_master_wait_point';
-- AFTER_SYNC（推荐）
```

### 7.6 MHA / Orchestrator

**MHA（Master High Availability）：**
- 由 Yoshinori Matsunobu 开发的自动化主从故障切换工具
- 监控主库，发现主库宕机后自动完成 failover
- 核心能力：将挂掉的主库上最新的 Binlog 补齐到从库，确保数据不丢
- 缺点：已停更多年，不再更新

**Orchestrator：**
- GitHub 开源，基于 Go 编写
- Web UI 展示拓扑结构
- 支持自动化故障切换
- 支持多机房、跨数据中心拓扑管理
- 社区活跃，替代了 MHA 的地位

**Orchestrator 的核心能力：**
1. 拓扑发现——自动发现复制关系
2. 健康检查——定期检查主从延迟、连接状态
3. 故障切换——主库宕机时自动选择最优从库提升为主库
4. 恢复（Recovery）——中间主库故障时自动修复拓扑

```bash
# Orchestrator 故障切换流程
# 1. 检测到主库不可达
# 2. 选择最优备选从库（数据最新、延迟最小）
# 3. 补齐 Binlog（尝试从挂掉的主库或同级从库拉取）
# 4. 将选定的从库提升为主库
# 5. 重新配置其他从库指向新主库
# 6. 通知（Webhook / 邮件等）
```

### 7.7 MGR（MySQL Group Replication）

MGR 是 MySQL 5.7.17 引入的官方高可用方案，基于 Paxos 协议。

**MGR 架构：**
```text
       ┌───────┐      ┌───────┐      ┌───────┐
       │ Node1 │◄────►│ Node2 │◄────►│ Node3 │
       └───────┘      └───────┘      └───────┘
           ▲                               ▲
           └─────── Paxos Group ────────────┘
```

**核心特点：**
- 多主模式（Multi-Primary）：所有节点都可读写，冲突检测基于认证
- 单主模式（Single-Primary）：只有一个节点可写，其他只读（推荐生产用此模式）
- 自动故障检测和切换
- 强一致性——事务在大多数节点确认后才提交

**单主模式 vs 多主模式：**
| 特性 | 单主模式 | 多主模式 |
|------|---------|---------|
| 写入节点 | 1 个 | 所有（N 个） |
| 冲突可能 | 无 | 有（需要应用层处理冲突） |
| 性能 | 好 | 存在冲突检测开销 |
| 复杂度 | 低 | 高（需要处理写冲突） |
| 推荐 | 绝大多数场景 | 特殊场景（如多地写入延迟敏感） |

**MGR 的限制：**
- 表必须有主键
- 不支持 SERIALIZABLE 隔离级别
- 不支持外键（尤其级联操作）
- 大事务会阻塞 Group 通信（最好拆小）

```sql
-- 查看 MGR 状态
SELECT * FROM performance_schema.replication_group_members;
SELECT * FROM performance_schema.replication_group_member_stats;
```

**InnoDB Cluster：** MySQL 8.0 中，MGR + MySQL Router + MySQL Shell 组成的完整高可用方案叫 InnoDB Cluster，提供了从部署、监控到故障切换的一站式体验。

---

## 八、分库分表

### 8.1 垂直拆分与水平拆分

数据量增长到单表无法承受时，需要分库分表。但首先要确认是否真的有分库分表的必要：
1. 硬件升级（SSD、更大内存、更快的 CPU）是否还能扛住？
2. 读写分离、缓存是否优化到位？
3. 是否有不合理的架构（如未加索引的大表）？

如果上述手段都穷尽了，再考虑拆分。

**垂直拆分（Vertical Sharding）：**

按业务模块拆分数据库。例如将用户、订单、商品分别放在不同的数据库中。

```text
单体数据库                         垂直拆分后
┌──────────────┐              ┌──────────────┐
│  users       │              │  用户库       │  → users, user_profile
│  orders      │     ═══►     ├──────────────┤
│  products    │              │  订单库       │  → orders, order_items
│  payments    │              ├──────────────┤
└──────────────┘              │  商品库       │  → products, inventory
                              └──────────────┘
```

**优点：**
- 实现简单，模块间解耦
- 不同库可以使用不同的优化策略
- 单库数据量减小

**缺点：**
- 无法 JOIN 跨库的表（需要应用层实现）
- 分布式事务问题
- 数据热点可能依然存在（如果某库数据特别大）

**水平拆分（Horizontal Sharding）：**

按行对表进行拆分，将同一张表的数据分散到多个数据库（或同一库的多个分表）中。

```text
orders 表（1000 万行）            水平拆分后
┌────────────────┐          db_0.orders (0-499万)
│  所有订单数据   │   ═══►   db_1.orders (500-999万)
│                │          db_2.orders (1000-1499万)
└────────────────┘          ...
```

**优点：**
- 解决单表数据量过大的问题
- 理论上可以无限扩展
- 均衡数据分布

**缺点：**
- 跨分片查询复杂（JOIN、ORDER BY、GROUP BY、分页等）
- 分布式事务
- 分布式 ID 生成
- 扩容迁移成本高

### 8.2 分片键选择

分片键（Sharding Key）是水平拆分中最关键的设计决策。一个好的分片键要做到：

**分片键选择原则：**
1. **区分度高：** 数据均匀分布到各个分片，避免热点
2. **查询能带分片键：** 如果一个查询不带分片键，就需要扫描所有分片（全分片扫描），性能极差
3. **避免跨分片事务：** 关联数据尽量在同一个分片内

**常用分片方案：**

| 分片键类型 | 示例 | 优点 | 缺点 |
|-----------|------|------|------|
| 按用户 ID 取模 | `user_id % 64` | 简单均匀 | 扩容时数据需要重新分布 |
| 按时间范围 | `2024-01`, `2024-02` | 容易理解，旧数据可归档 | 热点（新数据总是落在新分片） |
| 一致性哈希 | 环形hash | 扩容影响范围小 | 实现复杂 |

**一致性哈希：** 将分片节点放在哈希环上（0 到 2^32-1），数据按 key 的 hash 落到环上后顺时针找最近的节点。扩容时只需要迁移受影响的节点的部分数据。

**生产经验：**
- 用户强相关的数据（订单、购物车）用 `user_id` 分片，保证同一用户的所有数据在同一个分片内
- 内容类数据（文章、评论）用 `article_id` 分片
- 不要用不太相关或经常变化的字段作为分片键（如 email——用户可能变更邮箱）

### 8.3 分布式 ID 生成

分片后，单库自增 ID 方案不可行（各分片可能产生相同的 ID）。需要全局唯一的 ID 生成方案。

**方案一：UUID**
```java
String id = UUID.randomUUID().toString();  // "550e8400-e29b-41d4-a716-446655440000"
```
- 优点：实现简单，本地生成无网络开销
- 缺点：字符串太长，字符集问题，无序导致 B+Tree 索引分裂严重（InnoDB 不是按 UUID 有序存储的）

**方案二：Snowflake（雪花算法）**

Twitter 开源，生成 64 位长整型 ID。结构如下：

```text
+-------------------------------------------------------------+
| 1bit |   41bit 时间戳   | 10bit 机器ID | 12bit 序列号        |
+-------------------------------------------------------------+
  符号位（0）  相对起始时间的毫秒数   5bit 机房 5bit 机器  同毫秒内递增
```

- 优点：趋势递增（利于 InnoDB 插入），高性能，独立部署
- 缺点：依赖机器时钟，时钟回拨会导致 ID 重复

```java
// 基础 Snowflake 实现示例
public class SnowflakeIdWorker {
    private long workerId;
    private long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;
    
    public synchronized long nextId() {
        long timestamp = timeGen();
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("Clock moved backwards!");
        }
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & 0xFFF;
            if (sequence == 0) timestamp = tilNextMillis(lastTimestamp);
        } else {
            sequence = 0L;
        }
        lastTimestamp = timestamp;
        return ((timestamp - 1288834974657L) << 22)
             | (datacenterId << 17)
             | (workerId << 12)
             | sequence;
    }
    // ...
}
```

**方案三：Leaf（美团开源）**

Leaf 提供两种模式：
- **号段模式（Segment）：** 从数据库批量获取 ID 段，缓存在内存中使用。用完再取新的号段。数据库挂了短时间内 ID 仍可用。
- **Snowflake 模式：** 基于 Snowflake 改进，通过 ZooKeeper 持久顺序节点解决 workerId 分配问题，通过周期性上报时间戳解决时钟回拨问题。

**方案四：数据库号段（号段模式）**

```sql
-- 号段表
CREATE TABLE id_allocator (
    biz_type VARCHAR(32) NOT NULL PRIMARY KEY,
    max_id   BIGINT NOT NULL,
    step     INT NOT NULL DEFAULT 1000
);

-- 用事务保证号段分配不重复
BEGIN;
UPDATE id_allocator SET max_id = max_id + step WHERE biz_type = 'order';
SELECT max_id FROM id_allocator WHERE biz_type = 'order';
COMMIT;
-- 拿到 max_id 后，可分配的 ID 段是 (max_id - step, max_id]
```

**方案五：Redis 自增**
```java
long id = redisTemplate.opsForValue().increment("order:id:seq", 1);
// 优点：高性能，简单
// 缺点：Redis 宕机会丢失当前的计数，需要额外做持久化保障
```

**生产建议：** 大多数中等规模业务推荐使用 Leaf 或自研的号段模式。大型业务（如电商订单）可考虑美团的 Leaf 或自研 Snowflake 集群。

### 8.4 分布式事务

分库分表后，跨分片的操作需要分布式事务保证数据一致性。主流的分布式事务方案：

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|---------|
| 2PC/XA | 强 | 低 | 低 | 对一致性要求极高、低并发 |
| TCC | 最终 | 中 | 高 | 资金、交易结算 |
| Saga | 最终 | 高 | 中 | 长事务、工作流 |
| 本地消息表 | 最终 | 高 | 中 | 普通分布式调用 |
| 事务消息（RocketMQ） | 最终 | 高 | 低 | 异步解耦场景 |
| Best Effort | 弱 | 高 | 低 | 日志、通知等非核心场景 |

**XA/2PC（两阶段提交）：**

MySQL 原生支持 XA 事务。

```sql
-- 协调者
XA START 'order_123';
INSERT INTO db1.orders VALUES (...);
XA END 'order_123';
XA PREPARE 'order_123';

XA START 'inventory_456';
UPDATE db2.inventory SET stock = stock - 1 WHERE ...;
XA END 'inventory_456';
XA PREPARE 'inventory_456';

-- 如果所有参与者都 PREPARE 成功
XA COMMIT 'order_123';
XA COMMIT 'inventory_456';

-- 如果有参与者失败
XA ROLLBACK 'order_123';
XA ROLLBACK 'inventory_456';
```

**XA 的缺点：**
- 同步阻塞，性能差——参与者在 PREPARE 到 COMMIT/ROLLBACK 期间持有锁
- 协调者单点故障——如果协调者在 COMMIT 阶段宕机，参与者可能永久阻塞
- 数据不一致风险——如果协调者部分发送 COMMIT 后宕机，参与者状态不一致

**TCC（Try-Confirm-Cancel）：**

将每个业务操作拆成三个阶段：
```text
Try:     预留资源（冻结库存、冻结资金等）
Confirm: 使用资源（正式扣减库存/资金）
Cancel:  释放资源（回退冻结的资源）
```

例子：账户 A 转账 100 元给账户 B。
```text
Try:    A 账户冻结 100 元
        B 账户准备接收 100 元
Confirm: A 账户扣减 100 元，B 账户增加 100 元
Cancel:  A 账户解冻 100 元
```

TCC 对业务侵入性非常强——每个业务操作都需要实现 Try/Confirm/Cancel 三个接口。

**Saga：**

Saga 将长事务拆分为多个本地短事务，每个本地事务有对应的补偿操作。如果某个步骤失败，则按逆序执行补偿操作。

```text
正向流程：A 扣款 → B 加款 → C 发送通知
补偿流程：C 撤回通知 ← B 撤回加款 ← A 撤回扣款
```

**生产者实践：**
- Saga 适合长事务场景（如复杂业务流程）
- 框架选择：Seata Saga 模式或自研状态机
- 需要注意补偿操作的幂等性

**Seata（阿里巴巴开源）：**

Seata 是目前 Java 生态最主流的分布式事务解决方案，支持 AT、TCC、Saga、XA 四种模式。AT 模式通过全局锁 + Undo Log 的方式对业务零侵入。

```java
// Seata AT 模式示例 —— 业务代码完全不需要处理分布式事务
@GlobalTransactional  // 只需要加一个注解
public void createOrder(OrderDTO orderDTO) {
    orderService.createOrder(orderDTO);     // 本地事务 1
    inventoryService.reduceStock(orderDTO); // 远程调用 → 本地事务 2
    accountService.decrease(orderDTO);      // 远程调用 → 本地事务 3
}
```

### 8.5 数据迁移方案

数据迁移是分库分表最复杂的运维操作之一。

**方案一：停机迁移**
1. 停服，挂维护页面
2. 导出原库数据
3. 用脚本按分片规则写入新库
4. 切换数据源，重启服务
5. 验证通过，恢复服务

适合：允许停机的非核心业务。

**方案二：双写方案（推荐）**
1. 新数据同时写新旧两套库（双写）
2. 后台程序把历史数据批量迁移到新库
3. 数据校验——逐行比对两套库的数据一致性
4. 灰度切换到新库（先 1% 流量验证，再逐步放量）
5. 稳定后下线旧库

**关键注意事项：**
- 双写阶段要用事务保证一致性（或最终一致性）
- 批量迁移要求幂等（支持重复执行）
- 要有回滚预案（切回旧库）
- 数据校验工具（如基于 Binlog 的数据对比）

**方案三：基于 Binlog 同步**
1. 搭建新库
2. 同步工具（Canal/Debezium）监听主库 Binlog，实时同步到新库
3. 后台全量迁移历史数据
4. 数据校验
5. 切换流量到新库
6. 断开同步

### 8.6 常见中间件

**Proxy 模式（服务端代理）：**
- ShardingSphere-Proxy：Apache 项目，功能丰富，社区活跃
- MyCat：老牌中间件，社区活跃度下降
- DBLE：爱可生基于 MyCat 改进版本
- MySQL Router：MySQL 官方，主要用于路由和负载均衡

**Client 模式（客户端代理，JDBC 层）：**
- ShardingSphere-JDBC：Apache 项目，广泛使用，支持分库分表、读写分离、数据脱敏、分布式事务

```java
// ShardingSphere-JDBC 配置示例（application.yml）
spring:
  shardingsphere:
    datasource:
      names: ds0, ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/db_0
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/db_1
    rules:
      sharding:
        tables:
          orders:  # 对 orders 表进行分片
            actualDataNodes: ds$->{0..1}.orders_$->{0..15}
            databaseStrategy:
              standard:
                shardingColumn: user_id
                shardingAlgorithmName: database-inline
            tableStrategy:
              standard:
                shardingColumn: order_id
                shardingAlgorithmName: table-inline
        shardingAlgorithms:
          database-inline:
            type: INLINE
            props:
              algorithm-expression: ds$->{user_id % 2}
          table-inline:
            type: INLINE
            props:
              algorithm-expression: orders_$->{order_id % 16}
    props:
      sql-show: true  # 打印 SQL 方便调试
```

**Proxy vs Client 对比：**

| 维度 | Proxy 模式 | Client 模式 |
|------|-----------|-------------|
| 多语言支持 | ✓（任何语言连接 Proxy） | ✗（需要语言的 SDK）|
| 性能 | 多一层网络跳转 | 进程内执行，无额外网络开销 |
| 运维复杂度 | 需要维护 Proxy 集群 | 应用内配置，随应用发布 |
| 异构数据源 | ✓ | ✗ |
| 数据库连接数 | 由 Proxy 管理 | 应用直连，连接数更多 |

**生产建议：**
- Java 技术栈推荐 ShardingSphere-JDBC（Client 模式），性能最好，配置灵活
- 多语言技术栈推荐 ShardingSphere-Proxy（Proxy 模式）
- 非必须不要分库分表——能用读写分离、缓存、索引优化解决的就不要分

---

## 参考

1. MySQL 官方文档 — https://dev.mysql.com/doc/refman/8.0/en/
2. 《高性能 MySQL（第4版）》— Baron Schwartz 等
3. MySQL 索引原理 — https://tech.meituan.com/2014/06/30/mysql-index.html
4. InnoDB 锁机制 — https://tech.meituan.com/2014/08/20/innodb-lock.html
5. 大众点评订单分库分表实践 — https://tech.meituan.com/2016/11/18/dianping-order-db-sharding.html
6. 分库分表关键步骤 — https://www.infoq.cn/article/key-steps-and-likely-problems-of-split-table
7. Leaf 美团分布式 ID 生成系统 — https://tech.meituan.com/2017/04/21/mt-leaf.html
8. Seata 官方文档 — https://seata.io/zh-cn/docs/overview/what-is-seata.html
9. ShardingSphere 官方文档 — https://shardingsphere.apache.org/document/current/cn/overview/
10. MySQL Group Replication — https://dev.mysql.com/doc/refman/8.0/en/group-replication.html
