---
title: 拓扑排序 (Kahn's Algorithm)
date: 2026-07-05
updated: 2026-07-05
tags:
  - 算法
  - 拓扑排序
  - DAG
  - Maven
  - Spring Boot
categories:
  - 数据结构
---

**场景：Maven/Gradle 构建依赖解析**

```
项目依赖:
module-A 依赖 [common]
module-B 依赖 [common]
module-C 依赖 [module-A, module-B]

依赖 DAG:
        [common]      ← 入度 = 0
          /    \
   [module-A] [module-B]  ← 入度 = 1
          \    /
        [module-C]  ← 入度 = 2

拓扑排序输出: common → A → B → C (或 common → B → A → C)
```

Maven 根据 POM 依赖关系构建有向无环图 (DAG)，然后用拓扑排序确定编译顺序。如果存在循环依赖（A 依赖 B，B 依赖 A），拓扑排序会失败，构建直接报错——这比等到运行时发现 Class 找不到要好得多。

**拓扑排序算法 (Kahn's Algorithm)：**

```
1. 计算每个节点的入度 (in-degree)
2. 所有入度为 0 的节点入队
3. while 队列非空:
     出队 v → 加入结果
     for v 的每个邻居 w:
         w 的入度减 1
         if w 入度 == 0 → w 入队
4. 如果结果数 < 节点数 → 存在环
```

Spring Boot 的自动配置加载、Gradle 的 Task 执行计划、AirFlow 的 DAG 调度，底层都有拓扑排序的影子。


时间复杂度 O((V+E) log V)。对于网络延迟这种边权重非负的场景是最优的。
