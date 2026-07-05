---
title: LSM-Tree 存储引擎
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - LSM-Tree
  - LevelDB
  - RocksDB
  - Cassandra
categories:
  - 数据结构
---

B+Tree 擅长点查和范围查，但在**写入密集型**场景下有痛点：
- 每次写入需要多次随机 I/O（更新 B+Tree 页、页分裂、维护索引）
- 页分裂时需要移动大量数据
- 机械磁盘随机写性能极差（~100 IOPS）

LSM-Tree 的核心思想：**把随机写变成顺序写**。

```
LSM-Tree 核心架构:
                    ┌──────────────────────┐
                    │    Write              │
                    │     ↓                 │
                    │  [WAL (预写日志)]      │  ← 崩溃恢复
                    │     ↓                 │
                    │  [MemTable (内存树)]   │  ← 一般是 SkipList/红黑树
                    │     ↓ (满了 flush)    │
                    └──────────────────────┘
                              │
────────── 磁盘分层的 SSTable ──────────
      ┌─────────────────────────┐
      │  Level 0: [SSTable] [SSTable] [SSTable] │  ← 各 SSTable 之间可能有重叠
      │  Level 1: [SSTable────SSTable────SSTable] │  ← 同一层内 key 不重叠
      │  Level 2: [──────────────────────]       │
      │  ...                                    │
      │  Level N: [──────────────────────]       │
      └─────────────────────────┘
```

**三层结构：**

1. **MemTable（内存表）**：接收写入请求，数据在内存中排序（跳表/红黑树实现），同时为崩溃恢复先写 WAL（Write-Ahead Log）。写操作只是 Append WAL + 插入内存树，速度极快。
2. **SSTable（Sorted String Table）**：MemTable 满了之后刷到磁盘成为 SSTable——一个不可变的、按键排序的文件。Level 0 的 SSTable 可能相互重叠（因为是多个 MemTable 直接刷下来的），但 Level 1+ 的 SSTable 在同一层内是不重叠的。
3. **Compaction（合并压缩）**：后台线程不断将低层 SSTable 与高层 SSTable 合并，消除冗余数据（同 key 的旧版本、已删除的标记），保持查询性能。

**写放大 vs 读放大 vs 空间放大：**

| 放大类型 | B+Tree | LSM-Tree |
|----------|--------|----------|
| 写放大 | ~1.1x（就地更新） | 10x-30x（WAL + 多次 Compaction 重复写入） |
| 读放大 | 1（直接定位） | 10x+（需要查 MemTable + 多层 SSTable + BloomFilter） |
| 空间放大 | ~1.1x | ~1.1x（Compaction 及时清理可保持很低） |

LSM-Tree 适合写多读少的场景（时序数据、日志、消息队列），因为写入极快但读取需要访问多层 SSTable。RocksDB 通过 BloomFilter 在每个 SSTable 上做快速排除来降低读放大。

**实际产品映射：**
- **LevelDB**：Google 开源的 LSM-Tree 实现，MemTable 用跳表
- **RocksDB**：Facebook 基于 LevelDB 的增强版，MySQL MyRocks 存储引擎的底层
- **Cassandra**：LSM-Tree + 去中心化分布，MemTable + SSTable + Compaction
- **HBase**：基于 HDFS 的 LSM-Tree 变体
