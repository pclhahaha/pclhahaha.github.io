---
title: Gossip 协议
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - Gossip
  - Cassandra
  - Redis Cluster
categories:
  - 分布式
---

Gossip 是一种去中心化的信息传播协议，节点随机选择邻居进行信息交换，最终使信息在整个集群中传播。其本质模仿人类社会中的流言传播（epidemic protocol），适用于**最终一致性**场景。

Gossip 的核心特点：
- **去中心化**：没有中心协调节点，所有节点地位对等，任何节点都可以发起信息交换
- **容错性极强**：即使部分节点故障或网络丢包，信息通过多条路径最终仍能到达所有健康节点
- **可扩展**：节点数增加时传播延迟呈对数增长，不存在单点瓶颈
- **冗余通信**：信息会通过多个路径重复传播，带来一定的带宽冗余——这是去中心化的代价

单次 Gossip 通信通常包含三种模式：
- **Push**：节点 A 将自身数据推送给节点 B
- **Pull**：节点 A 从节点 B 拉取数据
- **Push-Pull**：节点 A 和 B 互相交换数据（效率最高，收敛最快）

典型实现：Cassandra 集群节点状态检测（Gossip 协议维护成员列表）、Consul 的 Serf 成员管理、Redis Cluster 的节点间通信

推荐阅读：[Gossip 协议：从流行病说起](https://www.nosuchfield.com/2019/03/12/Gossip-protocol-from-epidemics/)
