---
title: Dijkstra 最短路径算法
date: 2026-07-05
updated: 2026-07-05
tags:
  - 算法
  - Dijkstra
  - 图论
  - 服务网格
categories:
  - 数据结构
---

**场景 1 - 服务网格/API 网关的路由选择：**

```
服务间调用延迟图:
    [Service-A]
    5ms/  \3ms
       /    \
[Service-B] [Service-C]  ← 问：A → D 的最快路径？
    2ms\    /4ms
    [Service-D]
```

最短路径：A → C(3ms) → D(4ms) = 7ms，而 A → B(5ms) → D(2ms) = 7ms 也是最短。Dijkstra 可以求出所有节点到 A 的最短延迟路径。

**场景 2 - 分布式链路追踪的时间分析：** 在一个分布式链路中，请求经过多层服务 (Gateway → Service A → Service B → Service C)。如果某条链路延迟很高，可以通过分析拓扑图 + 边权重（每段耗时）定位瓶颈是哪一跳。

**Dijkstra 核心逻辑（用小顶堆优化）：**

```
dist[source] = 0，其余 = ∞
小顶堆 pq 放入 (0, source)

while pq 非空:
    (d, u) = pq.poll()
    if d > dist[u]: continue  // 过时记录跳过
    for u 的每个邻居 v (边权重 w):
        if dist[u] + w < dist[v]:
            dist[v] = dist[u] + w
            pq.offer((dist[v], v))
```

时间复杂度 O((V+E) log V)。对于网络延迟这种边权重非负的场景是最优的。
