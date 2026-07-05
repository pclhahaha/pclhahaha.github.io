---
title: 后端性能优化方法论
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 性能优化
  - JVM
  - GC
  - MySQL
  - Redis
categories:
  - 基础设施
---

[TOC]

## 一、性能理论基础

性能优化不是凭感觉调参数，而是基于指标体系、数学模型和科学方法论的工程实践。脱离理论的优化只是在碰运气。

### 1.1 核心性能指标

| 指标 | 定义 | 说明 |
|------|------|------|
| **QPS** (Queries Per Second) | 每秒查询数 | 衡量系统吞吐能力，服务端视角："系统每秒能处理多少请求" |
| **TPS** (Transactions Per Second) | 每秒事务数 | 衡量业务处理能力，一个事务可能包含多个查询；压测时 TPS 比 QPS 更能反映真实业务能力 |
| **RT** (Response Time) | 响应时间 | 从客户端发出请求到收到完整响应的时间，含网络传输、排队、处理 |
| **P50/P90/P99/P999** | 分位数延迟 | P99 = 99% 的请求延迟在此值以下；**均值掩盖真相，分位数暴露长尾** |
| **并发数** | 同时处理的请求数 | 狭义：系统内正在处理中的请求数；广义：同时连到系统的用户/连接数 |
| **错误率** | Error Rate | 失败的请求占比；目标通常 < 0.01%（4 个 9），核心支付链路要求更高 |

**为什么平均值具有欺骗性？** 假设 100 个请求，99 个在 10ms 内完成，1 个耗时 10s。平均 RT ≈ 110ms——看起来还行，但实际上有 1% 的用户等了 10 秒。P99 = 10s 才是真实用户体验。

```
请求耗时分布示例（单位 ms）:
  10, 10, 10, ..., 10 (99个)  |  10000 (1个)
  → avg = 110ms  ← 看起来正常
  → P50 = 10ms, P99 = 10000ms  ← 暴露长尾问题
```

### 1.2 吞吐量与延迟的关系

很多人以为：吞吐量上升 → 延迟也上升。**这不总是对的。** 实际关系有三个阶段：

```
延迟
 │
 │          ┌──────────────────────  (3) 过饱和区
 │         ╱     延迟急剧恶化，
 │        ╱      吞吐量可能下降
 │       ╱
 │      ╱
 │  ───╱──────────────────────────  (2) 线性增长区
 │   ╱      延迟随吞吐线性增长，
 │  ╱       系统排队效应显现
 │ ╱
 │╱───────────────────────────────  (1) 空闲区
 └──────────────────────────► 吞吐量
      延迟接近恒定（零排队）
```

三条线的本质差异：
- **空闲区**：请求几乎无需排队，RT ≈ 服务处理时间。增加并发，吞吐线性增长，延迟几乎不变。
- **线性增长区**：CPU/线程开始排队，RT 包含排队时间。吞吐继续增长但斜率下降。
- **过饱和区**：某个资源（CPU/线程池/连接池/DB）达到瓶颈，排队无限堆积，RT 急剧恶化，吞吐不升反降。

**核心结论**：你在压测时要找到"线性增长区 → 过饱和区"的拐点，那就是系统的**最大安全吞吐量**，线上流量应控制在此拐点的 60%-70% 以内。

### 1.3 Little's Law

Little's Law 是排队论中最简单也最强大的公式：

```
L = λ × W

其中：
  L = 系统中平均并发数（正在处理的请求数）
  λ = 平均到达率（QPS）
  W = 平均响应时间（RT）
```

**实际应用：**

- **容量推导**：已知目标 QPS = 1000，RT ≤ 100ms，需要多少线程？L = 1000 × 0.1s = 100。因此线程池至少需要 100 个线程。
- **瓶颈反向推断**：线上 P99 RT = 200ms，QPS 峰值 = 5000，则峰值并发 L = 5000 × 0.2s = 1000。如果线程池只有 200 个线程，请求必然排队积压——不是代码慢，是池太小。
- **Tomcat 线程池规划**：`maxThreads` 应 ≥ L_peak（峰值并发数），否则请求在 OS accept queue 排队。

```
实际计算示例：
  一个支付服务，目标 QPS = 2000，P99 RT ≤ 100ms
  → L = 2000 × 0.1 = 200
  → 线程池需 ≥ 200，留 30% 余量 → 260 线程
  → DB 连接池：每个请求约 2 次 DB 操作，每操作 5ms
  → DB 并发 = L × (DB时间/总RT) ≈ 200 × (10/100) = 20
  → 连接池配置 20-30 即可（注意：单 DB 总连接数通常 100-300 是安全上限）
```

### 1.4 Amdahl's Law 与 Gustafson's Law

**Amdahl's Law（阿姆达尔定律）——并行加速的极限由串行部分决定：**

```
Speedup = 1 / (S + (1 - S) / N)

  S = 程序中必须串行执行的比例
  N = 处理器数量（并行度）

示例：S = 20%（80% 可并行），N = ∞
  → Speedup = 1 / 0.2 = 5 倍（上限！）
```

这是悲观主义者的定律——它告诉你**加再多机器也突破不了串行瓶颈**。在微服务架构中，一个需要串行调用 5 个下游依赖的接口，即使每个下游都完美水平扩展，串行的调用链本身也会成为瓶颈。

**Gustafson's Law（古斯塔夫森定律）——大规模问题可获近乎线性加速：**

```
Scaled Speedup = S + N × (1 - S)

与 Amdahl 不同，Gustafson 认为问题规模会随处理器数增长。串行部分 S 是常数，并行部分比例增大。
```

两个定律不是矛盾的，而是回答了不同问题：
- Amdahl 回答：**固定问题规模，加机器能快多少？**（悲观）
- Gustafson 回答：**机器越多，能做多大的问题？**（乐观）

**性能优化的启示**：
1. 识别串行瓶颈比增加并行度更重要——先做 benchmark 找到 Amdahl 串行部分
2. 消除全局锁、单点串行化、同步等待是提升水平扩展能力的前提
3. 如果一个接口串行调用了 8 个下游，即使每个都很快（10ms），总 RT 也至少 80ms——考虑用 `CompletableFuture` 并行化或 `Reactive` 异步合并

### 1.5 性能测试金字塔

```
                /\
               /容量\
              /------\
             /稳定性测试\
            /----------\
           /  压力测试   \
          /--------------\
         /   负载测试      \
        /------------------\
       /    基准测试        \
      /---------------------\
```

| 层级 | 测试类型 | 目标 | 典型问题 |
|------|----------|------|----------|
| L1 | **基准测试** | 建立性能基线 | microbenchmark (JMH)；单接口/单 SQL 的最佳性能 |
| L2 | **负载测试** | 验证预期负载下的行为 | 给定 QPS 下 RT/错误率是否达标 |
| L3 | **压力测试** | 找到系统极限拐点 | 持续加压直到 RT 飙升/错误率突破——找到最大吞吐 |
| L4 | **稳定性测试** | 验证长时间运行不退化 | 内存泄漏、连接泄漏、GC 恶化、日志膨胀 |
| L5 | **容量测试** | 推算线上极限容量 | 单机上限 → N 台集群上限；规划扩容时机 |

**压测前置条件**：
- 独立压测环境（非生产），但配置与生产一致
- 压测数据量要与生产同量级（索引行为/缓存命中率对数据量敏感）
- 消除单测干扰——压测期间排除定时任务、其他批处理
- 预热完成后再采样（JIT 编译、缓存、连接池初始化均需要时间）

---

## 二、性能分析方法论

调优之前先定位。盲目修改配置是性能优化最忌讳的事——**没有数据支撑的优化都是玄学**。

### 2.1 CPU 密集型 vs IO 密集型识别

大部分性能问题的根因都可以归为两类：

| 类型 | CPU 密集型 | IO 密集型 |
|------|-----------|-----------|
| 特征 | CPU 使用率持续 > 80% | CPU 使用率低，大量线程处于 Waiting/Blocked 状态 |
| 典型表现 | GC 频繁、序列化开销大、计算逻辑重 | 数据库/Redis/RPC 调用慢，连接池耗尽等待 |
| 定位命令 | `top -H`, `perf top`, 火焰图 | `thread dump`, 连接池监控, Arthas trace |
| 优化方向 | 减少对象创建、优化算法、并行计算、GC 调优 | 批量操作、异步化、连接池优化、缓存 |

**快速判断命令（Linux）**：
```bash
# CPU 使用率
top -bn1 | head -5
vmstat 1 10          # r列(running队列): 持续 > CPU核数 → CPU瓶颈

# IO 等待
iostat -xz 1         # %util 接近 100% → 磁盘IO瓶颈
                     # await 高 → 磁盘响应慢
```

### 2.2 自顶向下法（Top-Down）

不要上来就翻代码。正确的排查顺序是 **宏观 → 微观**，避免"拿着放大镜找问题"。

```
第1层：业务指标         ← 哪一个接口慢？多慢？QPS 多少？
    ↓
第2层：系统资源         ← CPU/内存/磁盘/网络，哪个饱和？
    ↓
第3层：JVM 层面        ← GC 频率？线程池满？锁争用？
    ↓
第4层：下游依赖         ← DB/Redis/RPC 哪个是瓶颈？
    ↓
第5层：代码层面         ← 哪一段代码、哪条 SQL、哪个方法慢？
```

**实例**：线上某个接口 P99 从 50ms 恶化到 500ms。

```
L1: 监控发现 /api/order/create P99=500ms, QPS 无明显变化
L2: CPU 40%, 内存正常，排除资源瓶颈
L3: GC 日志正常（YGC 30ms/次, 5次/min），排除 GC
L4: Arthas trace 发现 → 调用库存服务一次耗时 300ms（正常 5ms）
L5: 排查库存服务 → 发现 DB 连接池耗尽 → 连接池配置仅 10，峰值并发 50+
    根因：连接池配置太小 → 排队等待连接
```

### 2.3 USE 方法（Utilization/Saturation/Errors）

Brendan Gregg 提出的 USE 方法，适用于分析任何系统资源——CPU、内存、磁盘、网络、连接池等。

对每种资源，回答三个问题：

| 维度 | 含义 | CPU 示例 |
|------|------|----------|
| **利用率 (Utilization)** | 资源忙碌的时间比例 | CPU 使用率 85%（可能有问题） |
| **饱和度 (Saturation)** | 排队等待的工作量 | 运行队列长度 (load average) 持续 > CPU 核数 |
| **错误 (Errors)** | 错误事件的数量 | `top` 中 `%iowait` 持续 > 10% 可能会引发 IO 错误；应用层 5xx 错误 |

```
USE 检查清单（每次排查问题必过一遍）：

CPU:    Utilization: vmstat/sar  →  avg % busy
        Saturation:  vmstat 'r'列 →  run queue length
        Errors:      无硬错误（硬件级别极少）

内存:   Utilization: free -h →  已用/总量
        Saturation:  vmstat si/so列 →  换入/换出
        Errors:      OOM Killer 日志

磁盘:   Utilization: iostat -xz  →  %util
        Saturation:  iostat avgqu-sz →  平均队列深度
        Errors:      dmesg  |  grep -i error

网络:   Utilization: sar -n DEV → 带宽占用
        Saturation:  netstat -s → 重传/丢包统计
        Errors:      ifconfig →  errors/dropped
```

### 2.4 火焰图分析

火焰图是定位 CPU 热点和阻塞点的终极工具。它不是看"某个方法耗时多少"，而是**可视化所有正在执行的函数调用栈**。

**On-CPU 火焰图**：回答"CPU 时间花在哪了"。适用于 CPU 使用率高的情况。

```
  顶部宽的平台 → 该函数占用 CPU 多
  平台越宽 → 此函数被采样的次数越多

  perf record -F 99 -p <pid> -g -- sleep 30
  perf script | stackcollapse-perf.pl | flamegraph.pl > on-cpu.svg
```

**Off-CPU 火焰图**：回答"线程在等什么"。适用于 CPU 不高但 RT 高的情况。

```
  线程不在 CPU 上运行的原因 → 等锁、等 IO、等网络
  Off-CPU 火焰图顶部宽 = 等待集中在此处

  使用 bcc/eBPF 工具的 offcputime:
  /usr/share/bcc/tools/offcputime -df -p <pid> 30 > out.stacks
  stackcollapse.pl out.stacks | flamegraph.pl --color=io > off-cpu.svg
```

**Java 火焰图获取**：直接 perf 拿到的栈帧是 JIT 编译后的，需要配合 perf-map-agent 映射回 Java 方法名。更简单的方式是用 **async-profiler**：

```bash
# async-profiler 开箱即用，自动解析 Java 方法名
./profiler.sh -d 30 -f /tmp/flame.svg <pid>         # On-CPU
./profiler.sh -d 30 -e wall -f /tmp/flame.svg <pid>  # Wall-clock（含 off-CPU）
./profiler.sh -d 30 -e alloc -f /tmp/flame.svg <pid> # 内存分配
```

**火焰图阅读技巧**：
- 宽度代表耗时/采样次数，关注最宽的平台
- 先看顶层的宽平台（叶子函数），再看调用链
- 出现大量 `pthread_cond_wait` / `epoll_wait` / `futex` → IO 等待
- 出现大量 `malloc` / `memcpy` → 内存操作密集

### 2.5 JFR + JMC 分析

Java Flight Recorder (JFR) 是 JVM 内置的低开销（< 2%）性能分析工具，数据以二进制 `.jfr` 文件保存，用 JDK 自带的 JMC (Java Mission Control) 打开分析。

**JFR 能回答的关键问题：**
- 哪个方法分配了最多内存？→ Memory → Allocations
- 锁竞争发生在哪里？→ Java Application → Lock Instances
- GC 各阶段耗时？→ 内存 → GC
- IO 操作耗时分布？→ I/O → File Read/Write, Socket Read/Write
- 线程阻塞在什么操作上？→ Threads → Thread Dump

```bash
# 启动时开启 JFR
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp

# 运行时动态开启（jcmd）
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr
jcmd <pid> JFR.check
jcmd <pid> JFR.dump name=1 filename=/tmp/final.jfr
jcmd <pid> JFR.stop name=1
```

**JFR vs async-profiler 选型：**
- JFR：适合生产环境长期监控，开销极低，可连续数小时录制
- async-profiler：适合开发/测试环境深度分析，火焰图更直观，可追踪内核级调用栈

### 2.6 Arthas 在线诊断

Arthas（阿尔萨斯）是阿里开源的 Java 在线诊断工具，无需重启应用即可追查线上问题。它是排查**非可复现**线上性能问题的利器。

```bash
# 启动
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar          # 选择目标 Java 进程
```

**六大核心命令：**

| 命令 | 功能 | 典型场景 |
|------|------|----------|
| **dashboard** | 实时面板：线程/内存/GC/Runtime | 第一眼：现在系统状态如何？ |
| **thread** | 线程分析 | `thread -n 3` 看 top 3 CPU 线程；`thread -b` 找死锁 |
| **trace** | 方法调用链路+耗时 | `trace com.x.Service method -n 5` 看调了什么、慢在哪 |
| **watch** | 方法入参/返回值/异常 | `watch com.x.Service method '{params,returnObj,throwExp}' -x 3` |
| **vmtool** | 强制 GC、查看内存 | `vmtool --action forceGc` |
| **jad** | 在线反编译 | `jad com.x.Service` 确认线上运行的代码版本 |

**核心排查套路（线上接口慢）**：
```bash
# 1. 看全局
dashboard

# 2. 看是否有线程堆积
thread | grep BLOCKED

# 3. 跟踪慢调用链
trace com.example.controller.OrderController createOrder -n 5 --skipJDKMethod false

# 输出示例：
# `---[0.52% 10.43ms] com.example.controller.OrderController:createOrder()
#     +---[63.87% 312.32ms] com.example.service.InventoryService:deduct()
#     |   `---[99.02% 309.11ms] com.example.dao.InventoryDao:update()
#     |                                ↑ 这里瓶颈：一次 update 309ms
#     +---[20.15% 98.51ms] com.example.service.CouponService:issue()
#     `---[15.46% 75.61ms] com.example.service.NotificationService:send()
```

---

## 三、JVM 调优

JVM 的性能瓶颈 80% 以上与 GC 有关。理解 GC 行为是后端工程师的基本功。

### 3.1 GC 日志分析

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

### 3.2 GC 调优目标

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

### 3.3 G1 GC 调优参数

G1 的设计哲学是**自适应**——大部分参数无需手动设置，但以下几个值得关注：

```bash
# ===== 核心参数 =====
-XX:MaxGCPauseMillis=100          # GC 暂停目标（软目标，G1 尽力接近）
                                   # 不要设太小（如 10ms），GC 会过度频繁
                                   # 推荐 100-200ms（在线服务）

-XX:InitiatingHeapOccupancyPercent=45  # 触发 Mixed GC 的堆占用阈值
                                        # 默认 45%，增大可减少 Mixed GC 频率
                                        # 但可能导致并发标记时堆空间不足

-XX:G1HeapRegionSize=4M           # Region 大小：1M/2M/4M/8M/16M/32M
                                   # 堆 = 2048 个 Region；堆大 → Region 大

# ===== 高级参数（通常不改）=====
-XX:G1NewSizePercent=5            # 年轻代最小占比
-XX:G1MaxNewSizePercent=60        # 年轻代最大占比
-XX:ConcGCThreads=2               # 并发标记线程数（通常 = 1/4 * ParallelGCThreads）
-XX:ParallelGCThreads=8           # STW 阶段并行线程数
-XX:+PrintAdaptiveSizePolicy      # 打印年轻代动态调整信息（调试用）
```

**G1 调优实战流程**：
```
1. 设 MaxGCPauseMillis = 100（初始值）
2. 运行压测，收集 GC 日志
3. GC 暂停超目标？
   ├─ 是 → 检查是否 Full GC → 是 Full GC → 增大堆 或 降低 IHOP
   │                            → Young GC 超时 → 减小年轻代(降低 G1MaxNewSizePercent)
   └─ 否 → 检查 GC 频率是否过高 → 频繁 YGC → 增大年轻代(提高 G1NewSizePercent)
4. 迭代直到 GC P99 在目标范围内 & GC 吞吐 > 99%
```

### 3.4 ZGC

ZGC 是 JDK 11 引入的低延迟垃圾收集器，最大特点是**暂停时间与堆大小无关**（JDK 17+ 版本保证 < 1ms）。

```bash
-XX:+UseZGC                      # 启用 ZGC
-Xms4g -Xmx4g                    # 堆大小
# ZGC 基本无需额外调参！
```

ZGC 的核心优势：
- 暂停时间 < 1ms（不随堆大小增长）
- 支持 TB 级堆（生产环境有 16TB 堆的案例）
- 染色指针技术，通过指针中的 metadata bit 实现并发整理

**ZGC 调优要点**（虽然它"基本无需调参"，但仍需留意）：
- 给堆留足余量：并发回收期间**不停止应用**，回收速度跟不上分配速度会导致 Allocation Stall
- 设置 `-XX:ZAllocationSpikeTolerance`（分配尖峰容忍度），默认 2.0
- 观察 `-Xlog:gc+zgc*=info` 日志中的 Allocation Stall 事件

### 3.5 堆大小规划

**基本原则**：
- `-Xms` 和 `-Xmx` 设为相同值，避免 heap resize 带来的 GC 开销
- 堆大小不超过物理内存的 50%-75%（给 OS page cache 和 Metaspace 留余地）
- 容器化环境：Java 10+ 支持 `-XX:MaxRAMPercentage` 替代固定 `-Xmx`

```bash
# 物理机/VM 部署（16G 内存）
-Xms8g -Xmx8g                    # 堆 8G → 50% 内存

# 容器化部署（K8s, 8G limit）
-XX:MaxRAMPercentage=70.0 -XX:InitialRAMPercentage=70.0   # 堆约 5.6G
-XX:MinRAMPercentage=50.0        # 仅当内存 < 200MB 时生效（忽略）

# 或者直接指定
-Xms6g -Xmx6g
```

**堆大小规划公式**：
```
堆大小 ≥ 存活数据大小 × 1.5 ~ 2.0 + 每次请求临时对象 × 并发数

存活数据 = Full GC 后老年代占用量（可在 GC 日志中查看）
临时对象速率 = Eden 区的填充速度（单位：MB/s）
```

### 3.6 元空间与直接内存

```bash
# 元空间（Metaspace）——存储类元数据
-XX:MetaspaceSize=256M            # 初始大小，达到后触发 GC
-XX:MaxMetaspaceSize=512M         # 上限，超过后 OOM
-XX:+PrintMetaspaceStatistics     # 查看元空间使用详情（调试）

# 直接内存（Direct Memory）——NIO ByteBuffer
-XX:MaxDirectMemorySize=512M      # 上限，默认 = -Xmx
# 监控：jcmd <pid> VM.native_memory summary
```

**元空间 OOM 常见原因**：
- 动态代理/反射大量生成类（CGLIB、动态代理、Lambda）
- Groovy 脚本热加载（类泄漏）
- 排查：`jcmd <pid> VM.classloader_stats` 查看各 ClassLoader 加载的类数量

### 3.7 常见 GC 问题诊断

| 症状 | 根因 | 解决方案 |
|------|------|----------|
| **频繁 YGC** (< 1 秒/次) | 年轻代太小或分配速率太高 | 增大年轻代 (`-XX:G1NewSizePercent`) 或减少代码中临时对象 |
| **YGC 暂停时间长** (> 200ms) | Survivor 区溢出或对象复制开销大 | 增大 Survivor 空间或调整 `-XX:SurvivorRatio` |
| **连续 Full GC** | (1) System.gc() 被调用、(2) 大对象分配失败、(3) 元空间不足、(4) 老年代碎片 | (1) `-XX:+DisableExplicitGC`、(2) 增大 `-XX:G1HeapRegionSize`、(3) 增大 Metaspace、(4) 增大堆或排查大对象 |
| **Promotion Failed** | Old 区碎片化，无法找到连续空间容纳晋升对象 | 调整 IHOP 或增大堆 |
| **GC 吞吐低于 95%** | GC 太频繁 → 堆太小 | 增大堆 或 排查代码中过多的临时对象创建 |

**G1 Humongous Object 问题**：超过 Region 大小 50% 的对象直接分配到老年代的 Humongous Region。如果频繁分配大对象，Region 碎片化严重会触发 Full GC。排查：`-XX:+PrintAdaptiveSizePolicy` 配合 GC 日志查看 Humongous allocation 次数。

### 3.8 内存泄漏排查流程

```
步骤1：获取 Heap Dump
  # 自动（OOM 时）
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/app/

  # 手动（运行中）
  jmap -dump:live,format=b,file=heap.hprof <pid>
  # 或使用 Arthas: heapdump /tmp/heap.hprof

步骤2：MAT / JProfiler 分析
  → 打开 hprof 文件
  → Leak Suspects Report（MAT 自动分析）
  → Histogram：按类统计对象数量和总大小
  → Dominator Tree：谁持有最多的内存

步骤3：引用链分析
  → 找到可疑对象 → Path to GC Roots
  → 为什么这个对象没有被回收？
      常见原因：
      ├─ ThreadLocal 未 remove()（线程池线程复用导致）
      ├─ HashMap/HashSet 中存入未重写 equals/hashCode 的对象
      ├─ 静态集合不断 add 从未 remove
      ├─ 监听器/回调未注销
      └─ 连接池/流未关闭

步骤4：确认修复后验证
  → 修改代码 → 重新压测 → 观察 GC 后老年代占用是否稳定
```

---

## 四、Java 代码级优化

GC 调优解决的是"如何高效回收"，代码级优化解决的是"为什么产生这么多垃圾"。

### 4.1 对象生命周期管理

**不要为了"优雅"而过度创建对象**。每个 `new` 都是一次堆分配，分配压力大 → YGC 频繁 → RT 抖动。

```java
// ❌ 每个请求都创建 Calendar/DateFormat
public String formatDate(Date date) {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    return sdf.format(date);  // sdf 内部还创建 Calendar！
}

// ✅ SimpleDateFormat 线程不安全 → 用 ThreadLocal 或 DateTimeFormatter
private static final DateTimeFormatter FMT =
    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
public String formatDate(LocalDateTime dt) {
    return dt.format(FMT);  // DateTimeFormatter 不可变，线程安全
}
```

**对象池 vs 垃圾回收**：
- 轻量级对象（生命周期短）：让 GC 处理更高效
- 重量级对象（线程、连接）：池化是必须的
- 中间地带（ByteBuffer、StringBuilder）：连接复用或 ThreadLocal 可能优于反复创建

### 4.2 String 优化

String 是 Java 中最常用的不可变对象，也是 GC 压力的重要来源。

```java
// ❌ 循环内字符串拼接：每次迭代创建新的 StringBuilder + String
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;  // 编译后 ≈ result = new StringBuilder().append(result).append(i).toString();
}

// ✅ 显式使用同一个 StringBuilder
StringBuilder sb = new StringBuilder(1024); // 预设容量避免扩容
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

**字符串常量池**：`String.intern()` 可以将动态生成的字符串放入常量池，减少重复字符串的内存占用。但需要权衡——intern 操作需要锁 StringTable（HotSpot 中为哈希表），并发调用开销不小。

**G1 字符串去重（JDK 8u20+）**：
```bash
-XX:+UseStringDeduplication    # 自动去重相同内容的 char[] 底层数组
                                # 只作用于多次 GC 后仍存活的老年代字符串
                                # 实测可减少 10%-25% 的堆内存占用
```

### 4.3 集合优化

**指定初始容量**，避免扩容带来的数组复制和中间垃圾：
```java
// ❌ 默认容量 16 → 扩容 16→32→64→128→256... 每次扩容都复制 + 丢弃旧数组
Map<String, Object> map = new HashMap<>();

// ✅ 预期 100 个元素，考虑 0.75 负载因子 → 100/0.75 + 1 = 135
Map<String, Object> map = new HashMap<>(135);
new ArrayList<>(expectedSize);
new HashSet<>(expectedSize);   // 负载因子也是 0.75
```

**遍历效率对比**（100 万元素循环，平均耗时）：
```
传统 for (ArrayList, 按索引)  →  最快（无额外开销）
for-each (语法糖)              →  接近传统 for
Stream.forEach                  →  慢 20%-50%（Lambda + Spliterator 开销）
Iterator                        →  最慢（每次迭代 hasNext() + next()）
```

结论：**性能敏感的热点路径用传统 for 或 for-each，Stream 用于复杂数据转换（filter/map/reduce）场景而非纯遍历。**

### 4.4 序列化优化

| 序列化方案 | 速度 | 体积 | 跨语言 | 适用场景 |
|-----------|------|------|--------|----------|
| JSON (Jackson) | ★★★ | ★★★ | ✓ | 通用 API，对前端友好 |
| Protobuf | ★★★★★ | ★★★★★ | ✓ | 内部服务通信，高吞吐 |
| Kryo | ★★★★★ | ★★★★★ | ✗ | 纯 Java 环境（Dubbo/Spark） |
| Java Serializable | ★ | ★ | ✗ | 仅限 Legacy 兼容 |

**实际数据对比**（10 万条订单对象）：
```
JSON:        18.7 MB,  序列化 450ms,  反序列化 780ms
Protobuf:    4.2 MB,   序列化 120ms,  反序列化 210ms
Java Ser:     23.1 MB,  序列化 890ms,  反序列化 1320ms
```

选择序列化方案时还要考虑**长连接策略**——Protobuf 的二进制格式天然支持基于长连接的流式传输（如 gRPC），减少 TCP 握手开销。

### 4.5 并发优化

**减少锁粒度**：
```java
// ❌ 粗粒度锁——整个方法串行化
public synchronized void addStat(String key, int val) {
    stats.put(key, stats.getOrDefault(key, 0) + val);
}

// ✅ 细粒度锁——只锁必要的操作
private final ConcurrentHashMap<String, LongAdder> stats = new ConcurrentHashMap<>();
public void addStat(String key, int val) {
    stats.computeIfAbsent(key, k -> new LongAdder()).add(val);
}
```

**锁选型决策树**：
```
需要加锁保护的数据操作：
├─ 读多写少 → ReentrantReadWriteLock 或 StampedLock
├─ 读写均衡 → synchronized（JDK 6+ 已高度优化，无复杂需求默认用这个）
├─ 高竞争场景（> 4 线程争用）→ LongAdder（累加器）/ ConcurrentHashMap
├─ 需要尝试获取锁/超时 → ReentrantLock
└─ 极高的并发写操作 → 考虑无锁方案（CAS + 自旋）或分片锁
```

**避免伪共享（False Sharing）**：
```java
// CPU 缓存行通常 64 字节。两个变量在同一缓存行时，一个线程写 → 另一个线程的缓存行失效
// JDK 8+ 使用 @Contended 注解自动填充对齐
@Contended
public class Counter {
    volatile long value;
}
// JVM 参数：-XX:-RestrictContended（否则仅 JDK 内部类可用）
```

### 4.6 缓存预热与缓存穿透

**缓存预热**：服务启动后、上线前，预先加载热点数据到缓存（本地 Caffeine/Guava Cache 或 Redis）。避免上线瞬间流量直接打到数据库。

```java
@Component
public class CacheWarmer implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        List<HotItem> hotItems = itemService.loadHotItems();
        hotItems.forEach(item -> localCache.put(item.getId(), item));
        log.info("Cache warm-up complete: {} items loaded", hotItems.size());
    }
}
```

**缓存穿透防护**：大量请求查询一个不存在的数据（DB 也没有），缓存不命中，请求全打到 DB上。

解法：
- **布隆过滤器**（Bloom Filter）：用极小的内存判定 key 是否"可能存在"
- **缓存空值**：DB 返回 null 时，缓存一个短期空对象（TTL 短，如 1min）
- **请求合并**：同一 key 的并发查询只放一个到 DB，其余等待首个完成

---

## 五、数据库优化

> 详细的 MySQL 底层原理请参阅 [MySQL 深度解析](/data/mysql/)，本节聚焦性能优化方法论。

### 5.1 SQL 优化策略

SQL 优化是性价比最高的优化手段。一条慢 SQL 可能吃掉 80% 的 DB CPU。

**索引优化是 SQL 调优的起点（详见 MySQL 文章 第六章）**：
- 覆盖索引：查询列完全在索引中，避免回表
- 最左前缀原则：联合索引 `(a, b, c)` 的过滤条件必须从 a 开始
- 避免索引失效：函数运算 `WHERE DATE(create_time) = '2026-01-01'`、隐式类型转换 `WHERE phone = 13800138000`（phone 是 varchar）、前置模糊 `LIKE '%keyword'`
- 索引下推（ICP, Index Condition Pushdown）：MySQL 5.6+ 在索引层过滤，减少回表

**Join 优化**：
- 小表驱动大表（小表作为驱动表，减少被驱动表循环次数）
- Join 字段必须有索引（Nested-Loop Join 中被驱动表走索引）
- 超过 3 表的 Join 考虑拆分——在应用层组装或用临时表

**子查询改写**：
```sql
-- ❌ 相关子查询：外表每行都执行一次子查询
SELECT * FROM orders o WHERE o.amount > (
    SELECT AVG(amount) FROM orders WHERE user_id = o.user_id
);

-- ✅ 改写为 JOIN + 派生表
SELECT o.* FROM orders o
JOIN (SELECT user_id, AVG(amount) AS avg_amt FROM orders GROUP BY user_id) t
ON o.user_id = t.user_id AND o.amount > t.avg_amt;
```

**分页优化**（深分页问题）：
```sql
-- ❌ 偏移量 100000 → 扫描并丢弃前 100000 行
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- ✅ 延迟关联：先通过覆盖索引获取 ID，再关联获取完整行
SELECT o.* FROM orders o
JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 20) t
ON o.id = t.id;

-- ✅ 或者游标分页（基于上一次的最大 ID，适用于"下一页"而非跳页场景）
SELECT * FROM orders WHERE id > #{lastId} ORDER BY id LIMIT 20;
```

### 5.2 连接池优化（HikariCP）

HikariCP 是 Spring Boot 2.x 默认连接池，以性能著称。优化不在调参数，**在合理的池大小**。

**连接池大小计算公式**：
```
maximumPoolSize = ((core_count * 2) + effective_spindle_count)

解释：
- core_count: CPU 核心数
- effective_spindle_count: SSD 通常为 1，HDD 通常为磁盘数
- 对于大多数 Web 应用，DB 瓶颈不在连接数而在 CPU/IO → 连接数不是越大越好

生产环境经验值：
  单服务 → 10~30 个连接足够
  集群 N 个节点 → 单服务 10~20，总连接 = N × 20，确保 < DB max_connections
```

> 详细公式推导和连接池原理参见 [MySQL 深度解析 第五章 连接管理与线程池](/data/mysql/#五连接管理与线程池)。

**关键参数**：
```yaml
spring.datasource.hikari:
  maximum-pool-size: 20       # 最大连接数（生产建议 10-30）
  minimum-idle: 10            # 最小空闲连接（通常 = maximum-pool-size）
  connection-timeout: 3000    # 获取连接超时（ms），超时抛出 SQLException
  idle-timeout: 600000        # 空闲连接最大存活时间（10 分钟）
  max-lifetime: 1800000       # 连接最大存活时间（30 分钟，应小于 DB 的 wait_timeout）
  leak-detection-threshold: 10000  # 连接泄漏检测阈值（10s）, 日志告警
```

### 5.3 读写分离与分库分表

**读写分离**：一主多从，写走主库，读走从库。适合读多写少的场景。

```
                   ┌── 主库 (读写) ──┐
                   │                │
              Binlog 同步     Binlog 同步
                   │                │
              ┌────▼──┐        ┌────▼──┐
              │ 从库1  │        │ 从库2  │
              │ (只读) │        │ (只读) │
              └───────┘        └───────┘
```

> 详解参见 [MySQL 深度解析 第九章 高可用架构](/data/mysql/#九高可用架构)。

**分库分表**：水平拆分解决单库单表的写入瓶颈。本质上用"空间（更多机器）"换"写入吞吐"。

| 拆分维度 | 策略 | 适用场景 |
|----------|------|----------|
| 按 user_id 取模 | hash(user_id) % N | 用户体系均匀分布 |
| 按时间 | 按月/年分表 | 日志/流水类数据 |
| 按地市/业务线 | 按业务维度分库 | 多租户 SaaS |

> 详细分片策略、分布式 ID 生成、跨片查询问题参见 [MySQL 深度解析 第十章 分库分表实战](/data/mysql/#十分库分表实战)。

### 5.4 批量操作

**批量插入**是优化写入吞吐最直接的手段——N 次网络往返合并为 1 次。

```sql
-- ❌ 逐条插入（10000 条 = 10000 次网络往返 = 10000 次事务）
INSERT INTO logs (...) VALUES (...);  -- 循环执行

-- ✅ 批量插入（10000 条 = 1 次网络往返 + 1 次事务）
INSERT INTO logs (col1, col2, col3) VALUES
  (v1, v2, v3),
  (v1, v2, v3),
  ...;  -- 建议每批 500-2000 条
```

JDBC 层面使用 `rewriteBatchedStatements=true`（MySQL 驱动），否则每条 SQL 仍是一次独立发送。

```yaml
spring.datasource.hikari.data-source-properties:
  rewriteBatchedStatements: true       # 必须开启！
  cachePrepStmts: true
  prepStmtCacheSize: 250
  prepStmtCacheSqlLimit: 2048
```

**批量更新**同理，使用 JDBC batch API 或框架提供的批量更新能力（MyBatis BatchExecutor、JdbcTemplate.batchUpdate）。

### 5.5 慢查询分析

```sql
-- 开启慢查询日志（生产建议开启）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.1;         -- 100ms 算慢查询
SET GLOBAL log_queries_not_using_indexes = ON;

-- 分析工具
pt-query-digest /var/log/mysql/slow.log   # Percona Toolkit，慢查询聚合分析
```

**线上紧急排查**（不靠慢查询日志）：
```sql
-- 查看当前正在执行的所有 SQL（执行中 > 5 秒的）
SELECT * FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 5
ORDER BY time DESC;

-- EXPLAIN 当前慢 SQL
EXPLAIN SELECT ...;
-- 重点关注：type (ALL/full scan → 缺索引)、rows (扫描行数是否过大)、Extra (Using filesort/Using temporary → 排序/临时表优化)
```

---

## 六、中间件优化

### 6.1 Redis 优化

> 详细的数据结构底层原理和高可用架构请参阅 [Redis 深度解析](/data/redis/)，本节聚焦性能调优。

**Pipeline 批处理**：将多次命令的 RTT（Round Trip Time）合并为一次网络往返。

```java
// ❌ 逐条执行：每次 RTT = 1ms，1000 条 = 1000ms
for (int i = 0; i < 1000; i++) {
    redisTemplate.opsForValue().get("key:" + i);
}

// ✅ Pipeline：1000 条 ≈ 1 次 RTT + 执行时间
redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    for (int i = 0; i < 1000; i++) {
        connection.get(("key:" + i).getBytes());
    }
    return null;
});
```

> 详细命令的复杂度分析和高效使用参见 [Redis 深度解析 第三章 命令高效应用](/data/redis/#三命令高效应用)。

**大 Key / 热 Key 处理**：
| 问题 | 影响 | 解决方案 |
|------|------|----------|
| **大 Key** (String > 10KB, 集合 > 10000 元素) | 读取阻塞单线程，删除耗时长，迁移困难 | 拆分大 Key（大 Hash 按域拆分为多个小 Hash）；集合元素改为分页读取；删除用 `UNLINK`（异步删除） |
| **热 Key** (单 Key 的 QPS > 5000) | 单分片 CPU 打满，导致局部热点 | 多副本（客户端缓存本地备份）；前綴拆分（`key_1`, `key_2`, ... `key_n`）；Proxy 层 Hot Key 探测+自动副本 |

**连接池优化（Lettuce）**：
```yaml
spring.redis.lettuce.pool:
  max-active: 8         # 最大活跃连接（Lettuce 是单连接多路复用，无需太多）
  max-idle: 8
  min-idle: 0
  max-wait: -1ms        # 获取连接最大等待时间
```

> 详细持久化（RDB/AOF）及主从架构的可靠性影响参见 [Redis 深度解析 第六章 持久化与高可用](/data/redis/#六持久化与高可用)。

### 6.2 MQ 优化

> 详细的技术选型和消息可靠性保证请参阅 [消息队列](/systems/mq/)，本节聚焦性能优化。

**批量发送**：消息不逐条投递，而是积攒到一定数量或时间后批量发送，显著提升吞吐。

| MQ | 批量发送方式 | 配置要点 |
|----|-------------|----------|
| RocketMQ | `DefaultMQProducer.send(messages)` 批量 List | 同批次消息 Topic 必须相同；批次总大小 < 4MB |
| Kafka | `linger.ms` + `batch.size` | `linger.ms=5`（等待 5ms 攒批），`batch.size=16384`（16KB 批次） |
| RabbitMQ | 暂无原生批量 API，通过 `Publisher Confirms` + 批量 confirm 模拟 | 使用 `BatchingRabbitTemplate`（Spring AMQP 封装） |

> MQ 的消息可靠性三个维度（不丢/不重/有序）详见 [消息队列 第五章 消息可靠性](/systems/mq/#五消息可靠性)。

**消费优化**：
```java
// ❌ 逐条消费 + 逐条提交（offset/ACK）
@KafkaListener(topics = "orders")
public void consume(Order order) {
    process(order);  // 每条消息独立处理
}

// ✅ 批量消费
@KafkaListener(topics = "orders")
public void consume(List<Order> orders) {
    // 批量处理：批量写库、批量更新缓存
    orderService.batchProcess(orders);
}
// 配置：spring.kafka.consumer.max-poll-records=500, fetch.min.bytes=1024
```

**消息压缩**：对传输中的消息体进行压缩，减少网络带宽和存储开销。

```yaml
# RocketMQ
compression.type: LZ4

# Kafka
compression.type: lz4   # gzip/snappy/lz4/zstd; lz4 兼顾速度和压缩比
```

### 6.3 Elasticsearch 优化

| 优化维度 | 策略 | 具体做法 |
|----------|------|----------|
| **索引分片** | 分片数 = 节点数的 1.5~3 倍 | 单分片 10-50GB 为最佳实践；避免分片过多（集群管理开销大） |
| **Refresh 频率** | 降低刷新频率 | `index.refresh_interval: 30s`（默认 1s，写入密集场景可调大或关闭） |
| **Bulk 写入** | 批量写入 + 关闭 refresh | 先关 refresh → 批量写入 → 开 refresh；批量大小 5-15MB |
| **查询优化** | 避免深分页 | `from + size` 深度 > 10000 时改用 `search_after`；避免全量 `size: 0` 后用聚合 |
| **Mapping** | 关闭不需要的索引功能 | 不需要全文检索 → `"index": false`；不需要评分 → `"norms": false` |
| **Translog** | 异步持久化 | `index.translog.durability: async`（少量写入可靠性换取巨大吞吐提升） |

### 6.4 Nginx 优化

```nginx
# worker 配置
worker_processes auto;                # 自动匹配 CPU 核数
worker_connections 10240;             # 每个 worker 最大连接数
worker_rlimit_nofile 65535;           # 文件描述符上限

# 长连接
keepalive_timeout 65;
keepalive_requests 100;               # 单个长连接最大请求数
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    keepalive 32;                     # 与 upstream 的长连接池
}

# Gzip 压缩
gzip on;
gzip_min_length 1024;                 # > 1KB 才压缩（小文件不划算）
gzip_comp_level 4;                    # 压缩级别 4（1-9，4 是性价比最高的点）
gzip_types text/plain application/json application/javascript;
gzip_vary on;

# 静态文件缓存
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    expires 7d;
    add_header Cache-Control "public, immutable";
}

# 限流 (保护后端)
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://backend;
}

# 代理缓冲（避免大响应拖垮 Nginx）
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
```

---

## 七、系统架构优化

如果前面的微观调优是"修修补补"，架构优化就是"改设计"。有些性能问题从代码层面无解，必须在架构层面重构。

### 7.1 多级缓存

```
┌─────────────────────────────────────────────────────────┐
│  L1: 本地缓存 (Caffeine/Guava)  ← JVM 堆内，纳秒级      │
│  容量: 几百 MB ~ 几 GB，TTL: 秒到分钟                   │
│  命中 → 直接返回                                        │
├─────────────────────────────────────────────────────────┤
│  L2: 分布式缓存 (Redis/Memcached)  ← 网络级，毫秒级      │
│  容量: 几十 GB ~ 几百 GB，TTL: 分钟到小时               │
│  命中 → 回写到 L1                                       │
├─────────────────────────────────────────────────────────┤
│  L3: 数据库 (MySQL/PG)  ← 磁盘级，毫秒到秒              │
│  容量: TB 级                                            │
│  命中 → 回写到 L2 + L1                                  │
└─────────────────────────────────────────────────────────┘
```

**一致性策略**：
- **Cache Aside**（旁路缓存）：读 → 缓存未命中 → 读 DB → 写缓存；写 → 更新 DB → 删缓存。最常用。
- **Write Through**：写 → 同步写缓存 + DB。一致性最强但写入延迟高。
- **Write Behind**：写 → 写缓存 → 异步批量写 DB。性能最好但存在数据丢失窗口。

**热点探测与缓存预热**：在本地缓存中维护热点 Key 统计，高 QPS 的 Key 标记为热点，提前预热到 L1 甚至预加载到入站请求队列中。

### 7.2 异步化

**MQ 解耦**是最经典的异步手段。非核心链路从同步调用剥离为异步消息，参见 6.2 节及 [消息队列](/systems/mq/)。

**CompletableFuture 并行化**：一个接口需要同时调用多个下游服务时，串行调用总 RT = 各服务 RT 之和；并行调用总 RT = max(各服务 RT)。

```java
// ❌ 串行调用：200ms + 300ms + 150ms = 650ms
OrderInfo order = orderService.getOrder(orderId);          // 200ms
UserInfo user = userService.getUser(order.getUserId());    // 300ms
CouponInfo coupon = couponService.getCoupon(orderId);     // 150ms

// ✅ 并行调用：max(200, 300, 150) = 300ms
CompletableFuture<OrderInfo> orderFuture =
    CompletableFuture.supplyAsync(() -> orderService.getOrder(orderId));
CompletableFuture<UserInfo> userFuture =
    CompletableFuture.supplyAsync(() -> userService.getUser(userId));
CompletableFuture<CouponInfo> couponFuture =
    CompletableFuture.supplyAsync(() -> couponService.getCoupon(orderId));

CompletableFuture.allOf(orderFuture, userFuture, couponFuture).join();
// RT ≈ 300ms（取决于最慢的那个），减少了 54%
```

**响应式编程**（Reactive Programming / WebFlux / Project Loom Virtual Thread）：
- 适用场景：高并发 + IO 密集型（网关、消息推送、实时流处理）
- 核心逻辑：用少量线程处理大量并发请求，线程不被阻塞在 IO 等待上
- 代价：开发复杂性高，调试困难，代码风格不直观
- Loom 虚拟线程（JDK 21+）让传统阻塞代码获得近似响应式的可伸缩性

### 7.3 批处理与合并

**请求合并**：对高并发下请求相同资源的场景，将多个请求合并为一次后端查询。

```
场景：1000 个请求同时查询同一个 productId 的商品信息

方案：
  ├─ 不加合并：Redis → 1000 次 Redis GET
  ├─ Flux 合并：请求聚合窗口 (如 10ms) 内，相同 productId 的只查一次
  │             其他 999 个请求等待并共享结果
  └─ 本地缓存：Caffeine GET productId → 只有第一个请求穿透到 Redis
```

**批量操作**：见 5.4 和 6.1，将逐条操作合并为批量是通用优化模式。

### 7.4 池化技术

| 池类型 | 调优要点 | 常见问题 |
|--------|----------|----------|
| **线程池** | `coreSize` = 峰值并发 × 1.3；`maxSize` 可稍大；队列选 `LinkedBlockingQueue`（无界慎用）或 `ArrayBlockingQueue`（有界+拒绝策略） | 拒绝策略选 `CallerRunsPolicy`（让调用线程执行，天然限流）或 `DiscardOldestPolicy` |
| **DB 连接池** | 见 5.2 | 连接泄漏（未 close）、连接过多（DB 拒绝） |
| **Redis 连接池** | 见 6.1 | Lettuce 单连接多路复用，通常无需大池 |
| **HTTP 连接池** | 连接池大小 = 目标服务实例数 × 每个实例的最大连接数；`maxPerRoute` < 目标服务 accept 上限 | 连接池耗尽 → 请求排队 → 雪崩 |

### 7.5 预计算

**空间换时间的典型案例**：
- 电商首页分类聚合：定时任务每天凌晨预计算 Top 类目 → 缓存中直接取，而非每次请求实时聚合
- 用户未读消息数：写入消息时同步更新 counters 表，而非每次查询时 `COUNT(*)`
- 报表/大屏：T+1 预聚合到指标表，实时要求不高的数据不走实时 SQL
- 搜索推荐索引：提前构建倒排索引（Elasticsearch 本质上就是空间换时间）

### 7.6 读写分离与 COW（Copy on Write）

**读写分离**（见 5.3）本质是用冗余（从库）换读取并发能力。同样思路适用于应用层：
- 一份数据的"读写版本"分离——读走快照副本，写操作在后台生成新版本
- COW：对于读多写少的配置数据，更新时复制一份新数据，修改完成后原子替换引用（`volatile` 或 `AtomicReference`）

```java
// COW 实现配置热更新
private volatile Map<String, Config> config = new HashMap<>();

public void updateConfig(Map<String, Config> newConfig) {
    // 复制一份，在新副本上修改，完成后原子替换
    Map<String, Config> copy = new HashMap<>(newConfig);
    this.config = copy;  // volatile 保证可见性
}

public Config getConfig(String key) {
    return config.get(key);  // 读无锁，性能极高
}
```

### 7.7 数据结构选择

数据结构的选择直接影响算法复杂度，不同场景的容器选型能带来数量级的性能差异：

| 场景 | 推荐结构 | 时间复杂度 | 避免使用的结构 |
|------|----------|-----------|---------------|
| 频繁按 key 查找 | `HashMap` | O(1) | `ArrayList` 线性查找 O(n) |
| 需要有序查找 | `TreeMap` | O(log n) | `ArrayList` 按 key 查找后再排序 O(n + n log n) |
| 去重 + 快速 contains | `HashSet` | O(1) | `ArrayList.contains()` → O(n) |
| 最值/排行榜 | `PriorityQueue` 或 `TreeSet` | O(log n) | 每次 Collections.sort() → O(n log n) |
| 前缀/模糊匹配 | Trie（前缀树） | O(k), k=长度 | 遍历 + startsWith 每个 O(n) |

---

## 八、压测实战

### 8.1 压测工具选型

| 工具 | 类型 | 特点 | 适用场景 |
|------|------|------|----------|
| **JMH** | 微基准测试 | JVM 级微基准，控制 JIT 预热/死代码消除 | 单方法/单算法性能对比 |
| **wrk / wrk2** | HTTP 压测 | Lua 脚本可编程，吞吐极高 | 单接口压测，快速获取系统上限 |
| **ab** (Apache Bench) | HTTP 压测 | 简单，无脚本能力 | 快速验证，功能有限 |
| **JMeter** | 全链路压测 | GUI + 丰富协议支持 + 分布式 | 多接口、有依赖关系的复杂业务 |
| **K6** | 现代 JS 压测 | 脚本化（JS），CI/CD 友好 | 性能回归测试，持续集成 |
| **Locust** | Python 压测 | Python 脚本，分布式原生 | 复杂的用户行为模拟 |

**JMH 示例**（基准测试一定要用 JMH，不要自己写 `System.nanoTime()` 循环）：
```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Thread)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 2)
public class StringConcatBenchmark {
    @Benchmark
    public String plus() {
        return "hello" + " " + "world" + " " + "!";  // 编译器常量折叠
    }

    @Benchmark
    public String builder() {
        return new StringBuilder().append("hello").append(" ").append("world").toString();
    }
}
```

### 8.2 压测流程

```
阶段1: 目标定义
  ├─ 量化目标: QPS ≥ 10000, P99 RT ≤ 50ms, 错误率 < 0.01%
  └─ 业务场景: /api/order/create, /api/product/detail 等核心接口

阶段2: 场景设计
  ├─ 单接口压测 (探测单功能极限)
  ├─ 混合场景压测 (模拟真实流量配比：搜索 40%, 详情 30%, 下单 20%, 支付 10%)
  └─ 梯级施压: 从 10% 目标流量开始，每次增加 10%

阶段3: 数据准备
  ├─ 数据量 = 生产量级 (1 亿条订单 vs 100 条订单，索引行为完全不同！)
  ├─ 数据分布 = 生产分布 (用户 id 的散列、热点商品占比)
  └─ 脱敏: 生产数据打码后复用

阶段4: 逐步施压
  ├─ 预热 10 分钟 (JIT 编译、缓存加载、连接池初始化)
  ├─ 逐级增加并发，每个级别维持 5-10 分钟
  └─ 记录每级的 QPS/RT/错误率/CPU/内存/GC/JVM 线程

阶段5: 瓶颈定位
  └─ 当 QPS 不再增长或 RT/错误率突破阈值 → 停止加压
     → 用 USE 方法 + Arthas/火焰图定位当前瓶颈
     → 是 CPU? 是 DB? 是锁? 是 GC?

阶段6: 优化 → 验证
  └─ 根据定位结果优化 → 重新压测验证
     └─ 直到达到目标 or 识别出了系统架构瓶颈
```

### 8.3 全链路压测简介

全链路压测与单服务压测的本质区别：**在真实的线上集群中，用构造的测试流量叠加在生产流量上**，同时保证测试数据不污染真实业务数据。

**核心技术挑战**：
1. **流量染色**：测试流量打上特殊标记（RPC 透传 Header / HTTP Header），全链路传递
2. **数据隔离**：测试写操作路由到影子表/影子库，或使用特殊数据范围（如 uid > 某个阈值）
3. **流量控制**：测试流量 QPS 可实时控制，出现异常可一键终止
4. **风险控制**：选择低峰期，从极小流量起步，监控告警阈值调严，随时可回滚

> 全链路压测是大型互联网公司（阿里双十一、美团外卖日高峰）的必备能力，中小团队不必一步到位——先做好单服务压测。

### 8.4 容量规划

基于压测结果，推算线上容量并制定扩容节奏：

```
单机最大安全 QPS = 测得的 QPS_max × 0.6  （60% 余量）

线上预计峰值 QPS → 需要的实例数 = 峰值 QPS / 单机安全 QPS

扩容触发条件（余量不足时触发）：
  ├─ 当前 QPS > 安全 QPS × 0.7  → 预警，准备扩容
  ├─ 当前 QPS > 安全 QPS × 0.85 → 告警，立即扩容
  └─ 当前 QPS > 安全 QPS → 限流降级启动

示例：
  压测单机 QPS_max = 1200 → 安全 QPS = 720
  5 台实例 → 总安全 QPS = 3600
  大促预期峰值 QPS = 8000 → 需扩容到 8000 / 720 ≈ 12 台
```

---

## 九、性能优化案例

### 9.1 案例一：接口从 2s 优化到 80ms

**症状**：订单详情接口 P99 RT = 2000ms，用户投诉页面加载慢。

**排查过程**：
```
L1: 监控显示 /api/order/detail P99 = 2000ms, P50 = 500ms（长尾严重）
L2: CPU 使用率 25%, 内存正常，排除 GC 和资源瓶颈
L3: Arthas trace → 发现接口串行调用了 8 个下游服务
    各服务 RT: 200ms + 150ms + 180ms + 100ms + 300ms + 120ms + 250ms + 160ms = 1460ms
    + 内部 DB 查询 400ms + 序列化/逻辑 140ms ≈ 2000ms
```

**优化方案**：
1. **并行化**：8 个下游调用改为 CompletableFuture 并行 → 最长 300ms
2. **缓存**：商品信息、店铺信息 TTL 缓存（几乎不变的数据）→ 减少 4 个 RPC
3. **SQL 优化**：订单明细查询改为覆盖索引，移除 `ORDER BY rand()` + 无用 join

**结果**：P99 RT = 2000ms → **80ms**（并行调用 3 个 + 缓存命中），P50 = **35ms**。CPU 上涨 5%（CompletableFuture 线程开销），完全可接受。

### 9.2 案例二：Full GC 频繁导致定时抖动

**症状**：压测时 P99 RT 间歇性飙升到 5s，但 P50 只有 30ms。

**排查过程**：
```
L1: GC 日志分析 → Full GC 每 2-3 分钟发生一次，每次耗时 4s+（→ P99 5s 的根因）
L2: jmap -histo 发现 HashMap 中存在大量未回收的 POI 对象（Point of Interest）
    类实例达 8,000,000 个 → 占用约 1GB 堆内存
L3: MAT Dominator Tree → 所有 POI 对象被一个 @Cacheable 的静态 Map 持有
    → 缓存 TTL = 永不过期，数据不断增长
```

**优化方案**：
1. 为缓存加 TTL（60 分钟），过期的 POI 自动淘汰
2. 改用 Caffeine 替代手动 Map 缓存（内置 LRU 淘汰策略）
3. 最大缓存条数限制：`maximumSize(100000)`
4. 补充：异步定期清理过期的 POI 引用

**结果**：Full GC 消失，堆内存从 7G 降至 3G 稳定，P99 ≤ 150ms。

### 9.3 案例三：数据库连接池耗尽

**症状**：每天业务高峰期接口频繁出现 `Cannot get JDBC Connection`，持续 5-10 分钟后自动恢复。

**排查过程**：
```
L1: HikariCP 连接池监控 → active=20, idle=0, pending=50, wait=8s
    连接池只有 20 个连接，高峰期有 50+ 并发请求在等待
L2: 为什么一个请求占用连接这么久？
    Arthas trace → 发现某个导出报表的接口占用一条 DB 连接长达 30s+（大分页查询）
L3: 这个导出接口的调用频率比预期高 5 倍（运营人员频繁导出）
```

**优化方案**：
1. 连接池扩容：`maximum-pool-size` 从 20 → 40（临时止血）
2. 报表导出改为异步：提交导出 → MQ → 异步生成文件 → 通知下载（不再同步占用连接）
3. 数据库连接超时限制：`connection-timeout: 3000` → 3s 拿不到连接直接失败，不再堆积
4. 报表 SQL 优化：深分页改为游标分页

**结果**：高峰期连接池 active 稳定从 20 → 8，等待队列消除。真正解决的是异步化改造——**阻塞操作必须脱离请求主链路**。

### 9.4 案例四：缓存热 Key 击穿

**症状**：秒杀开始瞬间，Redis 中商品库存 Key 过期（或有大量并发读取同一个 Key），Redis 单分片 CPU 瞬间 100%，大量请求超时，然后穿透到 MySQL 瞬间打挂。

**排查过程**：
```
L1: Redis 监控：单分片 CPU 100%，QPS 集中在一个 Key（秒杀商品）
L2: 该 Key 过期时间 60s → 秒杀时大量并发读取 + 更新库存
L3: MySQL 慢查询日志 → SELECT stock FROM product WHERE id=xxx 频繁执行
```

**优化方案**：
1. **热点 Key 前置探测**：实时统计 Redis Key 的 QPS，识别热点 Key → 通知客户端本地缓存该 Key
2. **互斥锁**：缓存过期时，只有一个请求能去 MySQL 加载数据，其余等待
3. **逻辑过期**：缓存设置永不过期，value 中包含逻辑过期时间，后台线程异步刷新
4. 秒杀场景特殊处理：库存扣减收口到单线程队列（如 Redis + Lua 原子操作）

**结果**：热点 Key 从 Redis 分片 CPU 100% → 30%，无请求穿透到 MySQL。核心经验：**区分物理过期和逻辑过期，用异步刷新替代被动过期**。

---

## 十、性能优化检查清单

按优先级从高到低排列。这不是一次性全部执行的列表，而是**每次上线前、每次压测后、每次排查问题时**应该过一遍的检查清单。

### 第一优先级：低风险、高收益（必做）

- [ ] SQL 慢查询日志已开启，定期 Review 并优化（目标：90% 查询 < 10ms）
- [ ] DB 连接池 `maximum-pool-size` 经过合理计算（≤ DB `max_connections` / 服务数）
- [ ] GC 日志已开启，可通过 GCEasy 分析
- [ ] JVM 堆大小已按规划设置（Xms = Xmx）
- [ ] 关键接口有缓存策略（本地缓存 + 远程缓存）
- [ ] 缓存 Key 设置了合理的 TTL（不过期也不永不过期）

### 第二优先级：中等风险、高收益（推荐做）

- [ ] 核心接口串行调用改为 CompletableFuture 并行化
- [ ] Redis bulk 操作使用 Pipeline，而非逐条执行
- [ ] 批量 SQL INSERT/UPDATE 使用 `rewriteBatchedStatements=true`
- [ ] 热点路径代码消除无意义的 `new` 对象
- [ ] 接口加入了限流/熔断/降级机制（Sentinel/Resilience4j）
- [ ] HTTP Client 配置了连接池和超时（connect/read/write timeout）
- [ ] 大查询（导出、报表）改为异步处理（MQ + 回调/轮询）
- [ ] 深分页 SQL 改为游标分页或延迟关联

### 第三优先级：低风险、中收益（选做）

- [ ] Nginx 开启 Gzip + 静态资源缓存 + upstream keepalive
- [ ] ES Refresh 间隔根据写入频率调整（> 30s）
- [ ] 数据库读写分离（读多写少场景）
- [ ] 建立了中央配置管理的线程池/连接池参数
- [ ] 对不可变数据使用 COW（Copy on Write）减少读锁开销
- [ ] JVM 开启 `-XX:+UseStringDeduplication`（G1 GC）

### 第四优先级：深层优化（按需）

- [ ] JFR 录制分析热点方法的对象分配
- [ ] 火焰图分析 CPU 热点函数（On-CPU / Off-CPU）
- [ ] MQ 消息压缩（LZ4/Zstd）
- [ ] Redis 大 Key / 热 Key 拆分
- [ ] 引入 Protobuf 替代 JSON 作为内部 RPC 序列化
- [ ] 使用虚拟线程（JDK 21+）改造高并发阻塞 IO 场景
- [ ] 建立全链路压测体系（中大型团队）

### 持续监控

- [ ] 每个核心接口的 QPS / P99 RT / 错误率 有 Dashboard
- [ ] JVM 堆内存 / GC 频率 / 线程数 / 连接池 有 Dashboard
- [ ] DB 慢查询告警 / 连接数告警 / 主从延迟告警
- [ ] Redis 内存使用率 / 慢查询 / 连接数告警
- [ ] 定期（每季度）执行一次全量压测，建立性能基线

---

## 附录：快速诊断指南

一个接口"慢"时，按以下顺序排查——这是十年来最有效的一线排查路径：

```
用户投诉"页面很慢"
  → 1. 定位到具体接口（APM/日志 TraceId）
  → 2. 确认是间歇性还是持续性
  → 3. 查看监控：QPS/RT 近期有变化？ 流量异常？
  → 4. USE 方法过一遍系统资源（CPU/内存/磁盘/网络）
  → 5. GC 日志：Full GC 频繁吗？GC Pause 长吗？
  → 6. Arthas trace：具体慢在哪个方法/哪个下游调用？
  → 7. 若是 DB 调用慢 → EXPLAIN + Optimize SQL
  → 8. 若是 RPC 下游慢 → 联动下游排查
  → 9. 若是锁等待 → thread -b 找死锁，thread dump 分析 BLOCKED
  → 10. 修复 → 验证 → 复盘 → 固化到监控告警
```

> **后记**：性能优化没有银弹。最重要的不是记住哪些参数、哪些命令，而是建立一套系统化的"分析 → 定位 → 优化 → 验证"方法论。遇到任何性能问题，先问三个问题：**在哪**（哪个环节慢）、**为什么**（根因是什么）、**值不值**（优化的 ROI 是否合理）。
>
> 更多底层原理请参阅：
> - [MySQL 深度解析](/data/mysql/) — InnoDB 引擎、索引原理、事务与锁、高可用架构、分库分表实战
> - [Redis 深度解析](/data/redis/) — 核心数据结构、持久化与高可用、分布式集群、缓存设计方法论
> - [消息队列](/systems/mq/) — 三大 MQ 对比、消息可靠性、事务消息、最佳实践
> - [可观测性](/systems/observability/) — Metrics / Logging / Tracing 三支柱落地实践
