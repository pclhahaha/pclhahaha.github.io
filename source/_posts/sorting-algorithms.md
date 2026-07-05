---
title: JDK 排序工程智慧
date: 2026-07-05
updated: 2026-07-05
tags:
  - 算法
  - 排序
  - TimSort
  - QuickSort
  - JDK
categories:
  - 数据结构
---

| 算法 | 平均复杂度 | 最坏复杂度 | 空间 | 稳定 | 核心思路 |
|------|-----------|-----------|------|------|----------|
| 快速排序 | O(n log n) | O(n²) | O(log n) | 否 | 分区 + 分治 |
| 归并排序 | O(n log n) | O(n log n) | O(n) | 是 | 分治 + 合并 |
| 堆排序 | O(n log n) | O(n log n) | O(1) | 否 | 建堆 + 弹出 |
| 插入排序 | O(n²) | O(n²) | O(1) | 是 | 小规模时简单高效 |
| 计数排序 | O(n+k) | O(n+k) | O(k) | 是 | 非比较排序，需要知道范围 |

**JDK 排序的实际选择：**

```java
// JDK 8+ Arrays.sort() 的实际实现 (Dual-Pivot QuickSort):
// 1. 基本类型 (int[], long[], double[]) → Dual-Pivot QuickSort
//    - 选两个基准值 (pivot)，比经典快排减少约 10% 比较
//    - 小数组 (< 47 个元素) 用插入排序
//    - 不稳定排序，但基本类型数组不需要稳定性
//
// 2. 对象类型 (Object[], String[]) → TimSort
//    - 由 Python 的 Tim Peters 发明
//    - 归并排序 + 插入排序的混合体，利用数据中已有的有序片段 (run)
//    - 稳定排序，O(n log n) 最坏
//    - 对部分有序的数据极快（接近 O(n)）
```

**为什么区分基本类型和引用类型？** 基本类型数组排序不需要稳定性——`[5a, 5b]` 和 `[5b, 5a]` 没有区别。引用类型需要稳定性（`Comparator` 可能只比较部分字段），所以用稳定的 TimSort。

**生产建议：** 除非数据确实已经近似有序，否则不要手写排序。JDK 的 `Arrays.sort` 经过了二十年的工程调优（Dual-Pivot + TimSort + 归并 + 插入的混合策略 + 内联 SIMD 优化），自己写的排序几乎不可能比它好。
