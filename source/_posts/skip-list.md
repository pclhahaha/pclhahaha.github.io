---
title: 跳表 (SkipList)
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - 跳表
  - SkipList
  - Redis
categories:
  - 数据结构
---

跳表是多层的链表——每层的节点指向同层的下一个节点，上层节点跨越多层，实现"跳跃"查找。

```
跳表结构 (Redis ZSet 实现):
Level 3:  [head] ───────────────────────────→ [30] ──────────→ NULL
Level 2:  [head] ──────────→ [15] ──────────→ [30] ──────────→ NULL
Level 1:  [head] ──→ [5] ──→ [15] ──→ [21] ──→ [30] ──→ [42] → NULL
Level 0:  [head]→[2]→[5]→[12]→[15]→[18]→[21]→[27]→[30]→[35]→[42]→NULL
                          ↑ 底层包含全部元素
```

**查找过程 (找 key=21)：** 从最高层 Level 3 开始 → 30 > 21，退到 Level 2 → 15 < 21 → 跳到 15 → 30 > 21，退到 Level 1 → 从 15 走到 21 → 命中！整个过程跳过了 Level 0 的多个节点，复杂度 O(log n)。

**时间复杂度分析：**
- 查找：O(log n)（概率期望，与红黑树相同）
- 插入：O(log n)（随机生成层数，期望 O(log n)）
- 删除：O(log n)
- 空间复杂度：O(n)（每层节点的期望比例为 1/2，总节点数 ≈ 2n）

**Redis ZSet 为什么选跳表而不是红黑树？**

| 对比维度 | 跳表 | 红黑树 |
|----------|------|--------|
| 实现复杂度 | 简单（300-400 行 C） | 复杂（旋转/变色逻辑繁琐） |
| 范围查询 | O(log n + k)，天然支持链表遍历 | 需要中序遍历（递归/栈） |
| 并发支持 | 容易实现无锁（每层独立的链表） | 几乎不可能无锁（旋转影响多个节点） |
| 内存占用 | 略大（多指针） | 较小（两个子节点指针 + 颜色） |
| 查询性能 | 稍慢（概率均衡，常数因子更大） | 稍快（确定性平衡） |

Redis 的 ZSet 需要频繁做 `ZRANGEBYSCORE` 范围查询——跳表的底层 Level 0 就是一条完整的有序链表，直接遍历即可，而红黑树需要中序遍历做栈或递归。此外，Redis 是单线程的，不需要锁，但跳表的简单性对于 C 代码的可维护性有天然优势。

```c
// Redis 跳表节点 (redis/src/server.h)
typedef struct zskiplistNode {
    sds ele;                       // 成员 (sds 字符串)
    double score;                  // 分值（排序依据）
    struct zskiplistNode *backward; // 后退指针（仅 Level 0 使用）
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;        // 跨度（用于计算 rank）
    } level[];                     // 柔性数组
} zskiplistNode;
```
