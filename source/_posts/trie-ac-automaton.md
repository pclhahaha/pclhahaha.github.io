---
title: Trie 字典树与 AC 自动机
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - Trie
  - AC自动机
  - ES
categories:
  - 数据结构
---

Trie（字典树/前缀树）按字符串的每个字符逐层展开，共享公共前缀，适合前缀匹配场景。

```
Trie 示例 (存储 "abc", "abd", "bce", "bcf"):
                    [root]
                   /      \
                 a        b
                /          \
               b            c
              / \          / \
             c   d        e   f
            (*)  (*)     (*)  (*)    ← * 表示是一个完整单词的结束
```

**ES Completion Suggester 中的 Trie：**

Elasticsearch 的 `completion` 类型底层使用 Trie（实际上是 FST，见下一节）实现搜索建议。用户在搜索框输入 "spr"，ES 快速返回 "spring", "spring boot", "spring cloud" 等建议。

**敏感词过滤：**

Trie 的一个经典应用是 Aho-Corasick (AC) 自动机——基于 Trie + 失配指针实现多模式串同时匹配。常用于：
- 内容审核系统（敏感词过滤）
- 入侵检测系统（Snort 的规则匹配）
- 搜索引擎的 Query 理解

在实际的敏感词过滤系统中，一个包含 10 万个敏感词的 Trie，内存约占 10-50MB，可以实现在一篇 5000 字的文章中在几毫秒内找出所有敏感词。
