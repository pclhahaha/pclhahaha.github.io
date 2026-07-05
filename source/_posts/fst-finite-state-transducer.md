---
title: FST 有限状态转换器
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - FST
  - Lucene
  - ES
categories:
  - 数据结构
---

**FST 是 Elasticsearch/Lucene 引擎的基石数据结构。** 需要先理解为什么有它：

```
ES 数据流向:
原始文档
  ↓ 分词 (Analyzer)
Term 字典 (所有不重复的词)  ← 可能数百万甚至数亿个
  ↓ 每个 Term → 倒排列表 (Posting List, 哪些文档包含这个词)
倒排索引
```

**问题：** 数亿个 Term，怎么快速找到某个词对应的倒排列表？最直接的办法是哈希表 (O(1))，但哈希表没法前缀查询。B-Tree 可以，但字符比较的存储效率低。

**FST 的解法：** FST 本质上是一个**有向无环图 (Minimal Acyclic DFA)**，将共享前缀和后缀的字符串压缩到极致。

```
普通 Trie 存储 "mon", "tues", "thurs":
        root
       / | \
      m  t  t            (多个 t 重复)
      |  |  |
      o  u  h
      |  |  |
      n  e  u
         |  |
         s  r
            |
            s

FST 存储 (共享前缀 + 后缀):
    [root]
      ├── m → o → n [$]
      └── t ─┬─ u → e → s [$]
             └─ h → u → r → s [$]
```

FST 共享了 `t` 前缀和 `s` 后缀，而且每个节点上可以挂载数值（如 Term 在倒排文件中的偏移量），既是**字典**又是**映射**。

**Lucene 的 BlockTree Terms Index：**

```
Lucene 内部术语索引:
                    [Term Index (FST)]
                           │
                    ┌──────┼──────┐
                    │      │      │
            term_a ─┘ term_m ─┘ term_z ─┘
                    │      │      │
                    ▼      ▼      ▼
              [Term Dictionary (BlockTree - 有序前缀块)]
                    │      │      │
                    ▼      ▼      ▼
            [Posting Lists (跳表/FOR编码/RBM位图)]
```

- **Term Index**：用 FST 存储所有 Term 的前缀，快速定位到 Term Dictionary 对应的 block
- **Term Dictionary**：用 BlockTree 存储，每块包含一些按字典序排序的 Term，块内二分查找
- **Posting List**：倒排列表，存储文档 ID 列表，用 FOR 编码/RBM 位图压缩

FST 极大地压缩了 Term Index 的内存占用——通常可以把一个 10 亿 Term 的索引前缀压缩到几 GB 甚至几百 MB，**全部加载到内存中**，实现微秒级的 Term 定位。

---
