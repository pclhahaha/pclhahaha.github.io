---
title: 渐进式 Rehash (Redis dict)
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - 哈希表
  - Rehash
  - Redis
categories:
  - 数据结构
---

Redis 的核心数据结构 `dict` 在以下场景广为使用：全局 key 空间、Hash 类型底层、Set 类型（小集合用 ziplist，大集合用 dict）。

```c
// Redis dict 结构 (redis/src/dict.h)
typedef struct dict {
    dictType *type;
    dictEntry **ht_table[2];    // 两个哈希表：ht[0] 正常使用，ht[1] 用于 rehash
    unsigned long ht_used[2];
    long rehashidx;             // -1 表示无 rehash 进行中，≥0 表示正在 rehash
    int iterators;
} dict;
```

**渐进式 Rehash：** Redis 是单线程的，如果一次性把 `ht[0]` 所有 bucket 拷贝到 `ht[1]`，大 key 场景下会长时间阻塞。Redis 的做法是分步搬迁：

1. 创建 `ht[1]`（容量为 `ht[0]` 的 2 倍）
2. 设置 `rehashidx = 0`
3. 每次对 dict 的**增删改查**操作时，顺手把 `rehashidx` 指向的那个 bucket 从 `ht[0]` 搬迁到 `ht[1]`，然后 `rehashidx++`
4. 所有 bucket 搬迁完毕，`ht[1]` 变成新的 `ht[0]`，销毁旧的

在渐进式 rehash 期间，新增操作只写到 `ht[1]`，查询/删除/更新需要同时查两个表。

两个触发 Rehash 的条件（负载因子 = `used / size`）：
- **扩容**：负载因子 ≥ 1 且无 `BGSAVE`/`BGREWRITEAOF` 子进程；或负载因子 > 5（此时不管有没有子进程，强制扩容）
- **缩容**：负载因子 < 0.1

> 更详细的 Redis 内部结构分析参见 [Redis 深度解析](..\data\redis.md)。
