---
title: JVM 调优实战
date: 2026-07-05
updated: 2026-07-05
tags:
  - JVM
  - 调优
  - GC日志
  - Arthas
  - MAT
categories:
  - 性能优化
---

JVM 的性能瓶颈 80% 以上与 GC 有关。理解 GC 行为是后端工程师的基本功。

### 1.1 GC 日志分析

**开启 GC 日志（JDK 11+ 统一参数）**：
```bash
-Xlog:gc*,gc+age=trace:file=/var/log/app/gc-%t.log:time,level,tags:filecount=10,filesize=50M
```

**关键日志解读（G1 示例）**：
```
[2026-07-05T10:30:15.123+0800][gc,start     ] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[2026-07-05T10:30:15.156+0800][gc,phases    ] GC(42)   Pre Evacuate Collection Set: 0.2ms
[2026-07-05T10:30:15.157+0800][gc,phases    ] GC(42)   Evacuate Collection Set: 30.5ms
[2026-07-05T10:30:15.158+0800][gc,phases    ] GC(42)   Post Evacuate Collection Set: 1.1ms
[2026-07-05T10:30:15.160+0800][gc,heap      ] GC(42) Eden: 2048M(2048M)->0B(1536M) Survivor: 256M->256M Heap: 3686M(8192M)->1669M(8192M)
[2026-07-05T10:30:15.161+0800][gc,cpu       ] GC(42) User=0.34s Sys=0.01s Real=0.04s
```

关注点：
- **Pause 类型**：Young / Mixed / Full —— Full GC 连续出现是红色警报
- **Pause 耗时**：Real 值。G1 的目标是 < 200ms，但实际通常在 10-50ms
- **Eden 回收后大小**：回收后 Eden 清空为 0B 是正常的（Minor GC 后 Eden 全清空）
- **Heap 回收量**：Heap before → Heap after，评估回收效率
- **GC 频率**：如果 GC 日志中每次间隔 < 1 秒，说明分配压力太大

**GCEasy 在线分析**：上传 gc.log 到 [gceasy.io](https://gceasy.io) 可直接获得：
- 吞吐量占比（GC 时间 / 总运行时间）
- 各 GC 暂停的 P50/P99/P99.9
- 内存分配速率
- 优化建议

### 1.2 GC 调优目标

| 场景 | 优先目标 | 推荐收集器 | 典型参数 |
|------|----------|------------|----------|
| **高吞吐**（批处理/离线计算） | 吞吐量 > 99%，可容忍秒级暂停 | Parallel GC | `-XX:+UseParallelGC` |
| **低延迟**（在线服务/API） | P99 暂停 < 100ms | **G1 GC** (默认) | `-XX:+UseG1GC -XX:MaxGCPauseMillis=100` |
| **超低延迟**（交易/实时风控） | P99 暂停 < 10ms | **ZGC** | `-XX:+UseZGC` |
| **超大堆**（> 32GB，离线分析） | 吞吐量优先 | G1 或 Parallel | `-XX:+UseParallelGC` |

**JDK 各版本默认收集器：**
- JDK 8：Parallel GC
- JDK 9-16：**G1 GC**（推荐用于绝大多数在线服务）
- JDK 17+：G1 GC（ZGC 在 JDK 21 成为备选推荐）

### 1.3 G1 GC 调优参数

G1 的设计哲学是**自适应**——大部分参数无需手动设置，但以下几个值得关注：

```bash
`
```
