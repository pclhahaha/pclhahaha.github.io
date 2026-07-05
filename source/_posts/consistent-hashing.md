---
title: 一致性哈希
date: 2026-07-05
updated: 2026-07-05
tags:
  - 数据结构
  - 一致性哈希
  - 分布式缓存
  - Redis
categories:
  - 数据结构
---

**问题背景：** 分布式缓存集群（如 Redis Cluster）在节点增减时，如果使用 `hash(key) % N` 取模路由，N 变化时几乎所有 key 的映射关系都变了，缓存大面积失效（**缓存雪崩**）。

**一致性哈希解法：** 把节点和 key 都映射到一个环形空间（0 ~ 2^32-1 的圆环）。每个节点负责环上的一段区间，增减节点时只影响相邻节点上的数据。

```
一致性哈希环:
                    0
                    │
             ┌──────┼──────┐
    Node1    │      │      │    (Node1, Node2, Node3 把环分成三段)
             │      │      │
    ─────────┼──────┼──────┼──── Node2
             │      │      │
             └──────┼──────┘
                    │
                  2^32-1

Key 映射规则: hash(key) 落在哪个区间，就归属哪个 Node 管理
```

**虚拟节点：** 如果节点数太少，环上分布不均会导致数据倾斜——某些节点负载远高于其他节点。解法是**虚拟节点**——每个物理节点映射为多个虚拟节点分散在环上，均匀化数据分布。Dubbo 的一致性哈希负载均衡也采用了这个方案。

> 一致性哈希在 Redis Cluster 和客户端分区（如 Jedis Sharded）中的实现细节，参见 [分布式系统](/distributed-theory/)。

## 三、Java 实现

`TreeMap` 底层是红黑树，`ceilingEntry(key)` 可以 O(log N) 找到哈希环上 >= key 的最小节点——天然适合一致性哈希的环查找。

```java
import java.util.*;

public class ConsistentHash<T> {
    private final int virtualReplicas;
    private final TreeMap<Integer, T> ring = new TreeMap<>();

    public ConsistentHash(int virtualReplicas) {
        this.virtualReplicas = virtualReplicas;
    }

    public void addNode(T node) {
        for (int i = 0; i < virtualReplicas; i++) {
            ring.put(hash(node + "#" + i), node);
        }
    }

    public void removeNode(T node) {
        for (int i = 0; i < virtualReplicas; i++) {
            ring.remove(hash(node + "#" + i));
        }
    }

    public T getNode(String key) {
        if (ring.isEmpty()) return null;
        int h = hash(key);
        Map.Entry<Integer, T> entry = ring.ceilingEntry(h);
        if (entry == null) entry = ring.firstEntry(); // 超出环尾则绕回起点
        return entry.getValue();
    }

    private int hash(String s) {
        return Math.abs(s.hashCode()); // 生产环境用 MurmurHash3
    }
}
```

**核心查找逻辑只有 4 行**：hash → ceilingEntry → null 检查 → firstEntry 兜底。虚拟节点建议 150-200 个/物理节点，标准差可控制在 10% 以内。TreeMap + MurmurHash 在 1 万虚拟节点下，单次查找约 4-6μs。

## 四、小结

一致性哈希是分布式路由的基础算法——从 Redis Cluster 的 16384 slot 到 CDN 的节点选择，再到 Dubbo 的负载均衡，都能看到它的身影。实现上只需要一个 TreeMap + 对 ceilingEntry 的调用，但背后"虚拟节点""哈希环"的思想影响了整个分布式系统设计。

---
