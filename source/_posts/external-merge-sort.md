---
title: 分治思想与外部归并排序
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 算法
  - 归并排序
  - MapReduce
  - 分治
categories:
  - 数据结构
---

**问题：** 要对 100GB 数据排序，但内存只有 4GB。怎么办？

**解法：外部归并排序 (External Merge Sort)**——这就是 MapReduce 和 GFS 的排序基础：

```
外部归并排序 (100GB, 内存 4GB):
Phase 1 (分片排序):
  100GB → split → [4GB] [4GB] [4GB] ... [4GB]   (25 个分片)
  每个 4GB 分片在内存中排序（快排/堆排）
  输出 25 个有序小文件到磁盘

Phase 2 (归并):
  每个文件读一小块到输入 buffer (堆/败者树维护最小值)
          ↓
  K 路归并 → 输出最终有序大文件
```

**MapReduce 的 Shuffle 阶段**就是这个思路的分布式版本：Map 每个输出分区内排序，Reduce 拉取多个 Map 的分区数据做归并排序 (`merge sort`)。

**分治思想在后端的更多体现：**
- **数据库分库分表**：把数据按分片键拆分到多个库/表 → 查询方汇总合并
- **聚合查询下推**：`COUNT(*)`/`SUM()` 先在各分片执行，结果汇总到协调者
- **GFS 的 chunk 设计**：大文件被切分成 64MB chunks 分布式存储在 chunkserver 上

> 更详细的 GFS 设计参见 [分布式系统](distributed.md)。
