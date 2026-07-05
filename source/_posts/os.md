---
title: 操作系统
date: 2020-01-01 00:00:00
updated: 2020-01-01 00:00:00
tags:
  - 操作系统
  - 进程
  - 内存管理
  - IO
  - Linux
categories:
  - CS基础
---

[TOC]

## 一、进程与线程

### 1.1 进程

进程是操作系统资源分配的基本单位。内核通过 PCB（Process Control Block，即 `task_struct`）管理每个进程，其中包含 PID、进程状态、打开的文件描述符表、内存映射、信号处理表、调度信息等。

**进程状态转换**（Linux `task_struct` 中的 `__state`）：

```
创建(fork) → 就绪(TASK_RUNNING) → 运行(正在CPU上) → 终止(EXIT_ZOMBIE/EXIT_DEAD)
                ↑                    ↓
          等待IO/信号/锁 ← 睡眠(TASK_INTERRUPTIBLE/TASK_UNINTERRUPTIBLE)
```

- `TASK_INTERRUPTIBLE`：可被信号唤醒（如等待 socket 数据的进程）。
- `TASK_UNINTERRUPTIBLE`：不可被信号中断（如等待磁盘 I/O 完成的 D 状态）。`ps aux` 中看到 D 状态的进程就是此状态，通常意味着 I/O 瓶颈。
- `TASK_STOPPED`：收到 SIGSTOP 或调试时被暂停。
- `EXIT_ZOMBIE`：进程已退出，但父进程尚未调用 `waitpid()` 回收——即僵尸进程。

**fork 与 exec**：

`fork()` 通过写时复制（Copy-On-Write, COW）创建子进程——子进程共享父进程的物理页，仅当一方写入时才真正复制。`execve()` 替换当前进程的地址空间，加载新程序。这意味着 `fork` 的代价很低（只复制页表和 task_struct），而 `exec` 的开销主要在于加载 ELF 文件、建立新地址空间。

> **生产经验**：Redis 在生成 RDB 快照时调用 `fork()`，利用 COW 避免阻塞主线程。但如果进程内存 50GB，`fork` 仍需复制页表，耗时可能达数百毫秒，触发 latency spike。这也是 Redis 建议单实例不超过 10-20GB 内存的原因之一。

**孤儿进程与僵尸进程**：

- 孤儿进程：父进程先于子进程退出，子进程被 init（PID 1）收养，init 会定期 `waitpid()` 清理。
- 僵尸进程：子进程已退出但父进程未回收。大量僵尸进程会耗尽 PID，导致无法创建新进程。

排查命令：`ps aux | grep Z` 或 `top` 看 zombie 计数。根治方式：父进程注册 SIGCHLD 信号处理函数调用 `waitpid()`，或显式忽略 SIGCHLD（`signal(SIGCHLD, SIG_IGN)`）。

**进程间通信（IPC）**：

| 方式 | 特点 | 典型场景 |
|---|---|---|
| 管道 (pipe/FIFO) | 单向、字节流、内核缓冲区 | shell `|` 管道 |
| 信号 (signal) | 异步通知，信息量小 | SIGTERM 优雅关闭 |
| 消息队列 | 有边界的消息，支持按类型读取 | POSIX mq |
| 共享内存 | 最快，需配合信号量同步 | 高性能 IPC |
| 信号量 | 进程间同步原语 | 生产者-消费者 |
| Socket | 通用，支持跨主机 | 网络通信 |

### 1.2 线程

线程是 CPU 调度的基本单位。同一进程的线程共享地址空间、文件描述符表、信号处理等，但各自拥有独立的栈、寄存器和线程局部存储（TLS）。

**内核线程 vs 用户线程**：

- 内核线程（1:1 模型）：每个用户线程对应一个内核调度实体。Linux 通过 `clone()` 系统调用创建线程，与进程共用 `task_struct`，只是资源共享标识不同。POSIX Threads (pthread) 即 1:1 模型。
- 用户线程（N:1 模型）：多用户线程映射到一个内核线程，由用户态调度器管理。优势是上下文切换零开销，劣势是一旦某个线程阻塞整个进程阻塞。曾经 Java "green thread" 即此模型。
- 混合模型（M:N 模型）：Go runtime 的 GMP 模型是经典案例——M 个 goroutine 映射到 N 个 OS 线程，runtime 调度器负责在 OS 线程上调度 goroutine。

**多线程优势**：利用多核并行、重叠计算与 I/O、共享地址空间使数据传递更简单。

**多线程开销**：`pthread_create()` 涉及 `clone()` 系统调用、内核栈分配等，耗时约 10-50μs。线程栈默认 8MB（Linux glibc），1000 个线程占用 8GB 虚拟内存。频繁的上下文切换带来 cache miss 和 TLB flush 开销。

### 1.3 协程

协程是用户态的轻量级执行单元，由程序自身调度而非 OS。优势是切换成本远低于线程上下文切换（仅需保存少量寄存器，不陷入内核）。

**有栈协程 vs 无栈协程**：

- 有栈协程：每个协程拥有独立栈，可以在任意调用深度中暂停和恢复。Go goroutine、Lua coroutine 属于有栈协程。Goroutine 初始栈仅 2KB，可动态增长，因此可以轻松创建百万级 goroutine。
- 无栈协程：不维护独立栈，通过编译器将协程体转换为状态机。C++20 coroutine、JavaScript async/await、Kotlin suspend 均属此类。优势是内存开销极小（通常仅为状态对象大小），劣势是只能在已知暂停点挂起（即"染色"问题）。

**Go goroutine 原理简述**：

Goroutine 运行在 GMP 模型上——G（goroutine）、M（OS 线程）、P（逻辑处理器，默认等于 GOMAXPROCS）。当 goroutine 执行系统调用阻塞时，M 与 P 分离，P 绑定新 M 继续执行其他 G，原 M 等待系统调用返回后 G 放回队列。

Goroutine 在以下时机触发调度：channel 操作、`time.Sleep`、`sync.Mutex` 操作、函数调用（在栈增长检查点插入）等。Go 1.14+ 加入了基于信号的抢占式调度，通过 SIGURG 信号打断长时间运行的 goroutine，解决死循环 goroutine 长时间占据线程的问题。

**核心 vs 线程创建开销对比**：

| 资源类型 | 创建耗时 | 初始栈大小 | 内存占用 | 数量上限 |
|---|---|---|---|---|
| OS 线程 (pthread) | ~10-50 μs | 8MB (glibc) | ~16KB (task_struct + kernel stack) | ~数千 |
| Go goroutine | ~2-4 μs | 2KB (动态增长) | ~4KB | ~百万级 |
| Java Virtual Thread | ~1-3 μs | ~几百字节 (stack chunk) | ~1KB | ~百万级 |

> **实战建议**：传统 thread-per-request 模型（如 Tomcat 200 线程池）在 10K QPS 下上下文切换成为瓶颈。协程模型（Go/虚拟线程）允许每请求一个协程而不必担心 OS 线程开销。但需警惕：协程数量爆发时内存膨胀和 GC 压力同样会成为瓶颈。

**Java Loom Virtual Thread**：

Java 21 正式发布的 Virtual Thread（基于 Loom 项目）是 JVM 级别的有栈协程。与传统的 Platform Thread（1:1 映射到 OS 线程）不同，Virtual Thread 由 JVM 调度器管理，在遇到阻塞 I/O 时自动将底层 OS 线程释放给其他 Virtual Thread。优点是与现有 Java 生态无缝兼容，无需引入 async/await 风格的 API 变更。

### 1.4 上下文切换开销分析

上下文切换包括：保存当前进程/线程的寄存器、程序计数器、栈指针 → 加载新进程/线程的对应状态。这三类切换开销差异显著：

| 切换类型 | 典型耗时 | 关键开销 |
|---|---|---|
| 进程切换 | 3-5 μs | 页表切换、TLB 全部失效、L1/L2 cache 变冷 |
| 线程切换（同进程） | 1-2 μs | 页表不变、TLB 保留、但寄存器/栈/L1 cache 仍受影响 |
| 协程切换 | ~10 ns | 仅保存/恢复少量寄存器，无内核态切换、无 TLB 失效 |

> **生产经验**：高并发服务器中，每秒数十万次线程上下文切换意味着 CPU 有 30-50% 时间消耗在切换本身。`pidstat -w` 可观察进程级别的上下文切换率。优化方向：减少线程数、使用协程模型、增大线程池与任务粒度的比值。

---

## 二、内存管理

### 2.1 虚拟内存

虚拟内存为每个进程提供独立的地址空间（Linux 32 位下 3G 用户空间 + 1G 内核空间，64 位下 128TB 用户空间 + 128TB 内核空间）。CPU 通过 MMU（Memory Management Unit）将虚拟地址转换为物理地址。

**页表与多级页表**：

页表是虚拟页号到物理页框号的映射表。x86-64 使用四级页表（PGD→PUD→PMD→PTE），每级 9 位索引，加上页内 12 位偏移，构成 48 位虚拟地址（256TB）。多级页表的好处是稀疏——未使用的虚拟地址区域无需分配中间页表，大幅节约内存。（4 级页表完整填充需 4KB×512⁴ = 256TB 地址空间的代价，但实际仅按需分配）。

**TLB（Translation Lookaside Buffer）**：

TLB 是 CPU 内的高速缓存，缓存最近使用的虚拟页到物理页的映射。TLB 命中延迟 < 1 cycle，未命中需走页表遍历（4 次内存访问，约 50-100 cycles）。上下文切换时，进程切换会导致 TLB 全部失效（除非使用 PCID 技术）。

**HugePage（大页）**：

默认页大小为 4KB。对于大内存应用（如 JVM 堆、数据库 Buffer Pool），4KB 页表条目数激增，TLB 覆盖范围有限（TLB 条目数约 64-1024 条，4KB 页覆盖仅 256KB-4MB）。

- 2MB HugePage（PMD 级别映射）：TLB 覆盖范围扩大 512 倍。
- 1GB HugePage（PUD 级别映射）：TLB 覆盖范围扩大 262144 倍。

> **生产经验**：MySQL/PostgreSQL/Redis/JVM 均可通过配置启用 HugePage，能提升 5-15% 的吞吐。配置方式：
> ```bash
> echo 512 > /proc/sys/vm/nr_hugepages    # 预留 512 个 2MB 大页
> # JVM 使用: -XX:+UseLargePages
> ```

**透明大页（THP，Transparent Huge Pages）**：

内核自动尝试将连续 4KB 页合并为 2MB 大页，对应用透明。但 THP 的压缩/分裂过程可能造成延迟抖动——数据库类应用通常建议禁用：
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### 2.2 缺页中断（Page Fault）

当 CPU 访问的虚拟地址在页表中不存在（或权限不对）时，触发缺页中断：

- **Minor Fault**：物理页已分配但未建立页表映射（如 `mmap` 后首次访问、COW 触发）。处理时间约 1-10 μs。
- **Major Fault**：物理页不在内存中，需从磁盘/swap 读取。处理时间约 1-10 ms——与内存访问相差 5-6 个数量级。
- **Invalid Fault**：访问非法地址，触发 SIGSEGV。

> **排查命令**：`ps -o min_flt,maj_flt -p <PID>` 查看进程的缺页次数。JVM 启动时大量 minor fault 是正常的（堆内存按需分配）。但运行时持续产生 major fault 说明内存不足、频繁 swap，需要扩容或降配 heap。

### 2.3 内存分配

**brk 与 mmap**：

用户态内存分配（`malloc`）通过两种系统调用向内核申请内存：
- `brk/sbrk`：调整数据段边界（program break），申请的是堆顶的连续区域。释放时高地址内存未全部释放则低处无法归还内核，易产生内存碎片。
- `mmap`：在进程地址空间中映射一块独立区域，可独立释放，无碎片问题。每次调用至少分配一页（4KB），有系统调用开销。

glibc 的 `malloc` 策略：小内存（默认 < 128KB）用 `brk`，大内存用 `mmap`。可通过 `mallopt()` 调整阈值。

**jemalloc 与 tcmalloc**：

glibc 的 `ptmalloc` 在多线程环境下存在严重的锁竞争和内存碎片问题。两种替代方案成为工业标准：

- **jemalloc**（Facebook）：以 Arenas 为核心，将内存划分为多个独立区域，线程分散到不同 Arena 减少竞争。对内存碎片控制极优，是 Redis、RocksDB 的默认分配器。通过 `LD_PRELOAD=libjemalloc.so` 即可替换。核心调优参数：`narenas`（arena 数量）、`lg_dirty_mult`（脏页回收阈值）。
- **tcmalloc**（Google）：以 CentralCache + ThreadCache 两层缓存实现低锁竞争，对小对象分配极快。Golang 早期即使用 tcmalloc 思路设计内存分配器。优势是多线程下分配/释放的性能。

**Broker/Router 类应用**（如 Kafka、RocketMQ）：属于消息量极大、大量小对象生灭频繁的场景，常替换为 jemalloc 以减少碎片和 RSS 占用。

> **生产经验**：替换分配器后应用 RSS 降低 20-40% 是常态。验证命令：
> ```bash
> LD_PRELOAD=/usr/lib/libjemalloc.so.2 ./your_app
> # 或通过环境变量
> export LD_PRELOAD=/usr/lib/libtcmalloc.so.4
> ```

### 2.4 内存回收与交换

Linux 内存回收分两条路径：

1. **后台回收**：`kswapd` 内核线程在空闲内存低于阈值（`vm.watermark_scale_factor` 相关）时异步回收。
2. **直接回收**：进程分配内存时发现内存不足，在分配路径上同步回收（`direct reclaim`），导致分配延迟显著增加。

**swappiness**（`/proc/sys/vm/swappiness`，默认 60）：

控制内核回收时优先回收 page cache 还是换出匿名页。值越大越倾向于 swap。对于纯缓存类服务（如 Redis），建议设为 0-10（优先淘汰 cache）；对于有大量冷匿名页的场景，可设高些。

**OOM Killer**：

当内存彻底耗尽且回收无果时，OOM Killer 根据 `oom_score` 选择进程杀死。`oom_score` 综合考虑进程内存占用、运行时间、`oom_score_adj` 等因素。关键进程可通过降低 `oom_score_adj` 保护：
```bash
echo -1000 > /proc/<PID>/oom_score_adj  # 完全排除
```
`dmesg | grep -i oom` 可回溯 OOM 事件日志。

### 2.5 内存映射与零拷贝

`mmap` 将文件映射到内存，访问时按需加载（Demand Paging）。`mmap` 写入可通过 `msync` 或自动回写持久化。优势是省去用户态与内核态的数据拷贝（文件 → page cache → 用户缓冲区 变为 文件 → page cache，用户直接访问 page cache）。

**零拷贝技术对比**：

| 方式 | 数据路径 | 拷贝次数 |
|---|---|---|
| 传统 read+write | 磁盘→内核→用户→内核→socket | 4 次上下文切换 + 2 次 CPU 拷贝 + 2 次 DMA 拷贝 |
| sendfile | 磁盘→内核(page cache)→socket | 2 次上下文切换 + 1 次 CPU 拷贝 + 2 次 DMA 拷贝 |
| sendfile + DMA Gather | 磁盘→内核→socket（DMA Gather） | 2 次上下文切换 + 0 次 CPU 拷贝 + 2 次 DMA 拷贝 |
| splice | 两个 fd 间通过 pipe 在内核空间传输 | 2 次上下文切换 + 0 次 CPU 拷贝 |

Kafka 和 Nginx 均使用 `sendfile` 实现静态文件传输的零拷贝。但注意零拷贝绕过了用户态，无法对数据做加工处理（如 SSL 加密、压缩）。

**mmap 在消息队列中的应用**：

Kafka 利用 `mmap` 将索引文件和分段日志映射到内存，Broker 读取消息时直接通过 page cache 实现文件读取的零拷贝——消息从磁盘到网络不需要经过用户态拷贝。但 mmap 并非完美：当 page cache 容量不足以容纳活跃的写数据时，mmap 的性能会断崖式下降。

RocketMQ 则采用了不同的策略——使用 `mmap` 映射 CommitLog 写入，通过 `MappedByteBuffer` 批量刷盘，配合堆外缓存在文件 I/O 和网络 I/O 之间形成了高效的管道。

### 2.6 内存屏障与缓存一致性

**MESI 协议**是现代多核 CPU 缓存一致性的基础。每个 cache line 处于四种状态之一：Modified（已修改）、Exclusive（独占）、Shared（共享）、Invalid（失效）。多核并发写同一 cache line 时会触发状态迁移和 cache line 在核间传递，即 cache line bouncing——一个典型的性能杀手。

**内存屏障**：编译器优化和 CPU 乱序执行可能导致内存操作的实际顺序与代码顺序不一致。内存屏障（memory barrier/fence）用于强制顺序约束：

- StoreStore Barrier：保证在此之前的 store 完成后才执行后续 store。
- StoreLoad Barrier：最强的屏障（也是 `mfence`），保证之前所有 store 完成后才执行后续 load。x86 上 `lock` 前缀指令隐含着 StoreLoad 语义。
- LoadLoad / LoadStore Barrier：分别是 load-load 和 load-store 的顺序保证。

**JVM 中的应用**：
- `volatile` 变量的写在 x86 上生成 `lock addl $0x0, (%rsp)`（StoreLoad barrier），保证写后可见。
- `synchronized` 的 monitor enter 和 exit 分别插入对应屏障。
- `Unsafe.putOrderedObject` / `AtomicInteger.lazySet` 只保证有序不保证立即可见（无 StoreLoad barrier），在特定场景下更高效。

---

## 三、文件系统

### 3.1 VFS 与 inode

VFS（Virtual File System）是 Linux 的文件系统抽象层，定义了统一的接口（`file_operations`、`inode_operations`、`address_space_operations`），使上层无需关心底层文件系统类型。VFS 管理的四大核心对象：
- **super_block**：挂载的文件系统元信息。
- **inode**：文件/目录的元数据（大小、权限、时间戳、数据块位置）。
- **dentry**：目录项缓存，加速路径解析。
- **file**：进程打开的文件，包含偏移量、访问模式等运行时信息。

**硬链接 vs 软链接**：
- 硬链接：目录项直接指向同一 inode，共享数据块。删除原文件不影响硬链接（引用计数 > 0）。不可跨文件系统，不可链接目录。
- 软链接（符号链接）：独立 inode，存储目标路径字符串。可跨文件系统，可链接目录。目标删除后变成悬空链接。

### 3.2 ext4 与 xfs 简介

**ext4**：Linux 使用最广泛的文件系统。最大支持 1EB 卷、16TB 单文件。使用 extent 代替 ext3 的间接块映射，提升大文件性能。支持延迟分配（delayed allocation）和日志校验。适合通用场景和中小文件。

**xfs**：SGI 开发的高性能 64 位日志文件系统。使用 B+ 树（B-tree）管理空闲空间和 inode，分配和回收极快。支持动态 inode 分配、在线扩容（不可缩容）、极高的并发性能（分配组 AG 设计）。适合大文件和高并发场景，是 RHEL 7+ 的默认文件系统。

> **生产建议**：MySQL/Kafka 的数据目录推荐 xfs，ext4 的 `data=ordered` 模式对写密集场景性能略差。但需要缩容的场景选 ext4。此外，文件系统挂载参数对性能影响巨大——**noatime** 选项禁止更新文件访问时间戳，可显著减少元数据写入（所有生产数据盘均应添加此选项）：
> ```bash
> mount -o noatime,nodiratime /dev/sdb1 /data
> ```

### 3.3 Page Cache

Page Cache 是 Linux 最核心的 I/O 缓存机制。所有通过 `read/write` 进行的文件 I/O 都经过 page cache（除非使用 Direct I/O）。Page Cache 本质是内核中以页（4KB）为单位缓存文件数据的 Radix Tree（Linux 4.x 后改为 XArray）。

**writeback（回写）机制**：

应用调用 `write()` 后数据写入 page cache 即返回（writeback 模式），并不立即写入磁盘。内核在以下时机触发脏页回写：
1. 定时回写：`dirty_writeback_centisecs`（默认 5 秒）触发一次。
2. 脏页比例超过 `dirty_background_ratio`（默认 10%）时后台回写。
3. 脏页比例超过 `dirty_ratio`（默认 20%）时，写入进程自身执行回写（阻塞式）。
4. 用户调用 `fsync/fdatasync/sync`。

> **生产经验**：高写入应用可通过调大脏页比例减少回写频率，但会增加宕机丢失数据和磁盘突发写负载。调整：
> ```bash
> sysctl -w vm.dirty_background_ratio=5
> sysctl -w vm.dirty_ratio=10
> ```

**fsync vs fdatasync**：
- `fsync`：文件数据 + 元数据（inode 大小、时间戳等）全部刷盘。通常导致两次磁盘写入（数据+日志）。
- `fdatasync`：仅刷文件数据，若元数据中仅大小变化也刷。比 fsync 少一次磁盘 I/O。
- Kafka 使用 `flush.fdatasync` 配置项控制刷盘策略。

### 3.4 Direct I/O vs Buffered I/O

| 特性 | Buffered I/O | Direct I/O |
|---|---|---|
| Page Cache | 使用 | 绕过 |
| 写延迟 | 低（写内存） | 高（直接落盘） |
| 读命中 | 可命中缓存 | 每次读磁盘 |
| 内存拷贝 | 内核→用户 | 无需额外拷贝（DMA→用户） |
| 对齐要求 | 无 | 需要扇区对齐（512B） |

> **生产建议**：数据库（如 MySQL InnoDB）和消息队列（Kafka）倾向使用 Direct I/O，因为自身实现了 Buffer Pool 和缓存管理，page cache 反而导致双重缓存浪费内存。应用自身有缓存策略的场景应使用 Direct I/O。

### 3.5 日志文件系统

日志（Journal/Journaling）保证文件系统在崩溃后的快速恢复和一致性：

- **物理日志**：记录数据块的完整修改前后值。体积大但最安全。
- **逻辑日志**：记录元数据变化操作（如"修改 inode X 的 size 字段"）。恢复时需要回放操作，快于物理日志。

ext4 提供三种日志模式：
- `data=journal`：数据和元数据都写日志。最安全但性能最差。
- `data=ordered`（默认）：先写数据块，再写元数据日志。保证文件内容不出现垃圾数据（跨 inode 元数据可能不完整）。
- `data=writeback`：只写元数据日志，数据块任意时序写回。最快但崩溃后新文件可能包含旧数据。

xfs 只对元数据做日志（逻辑日志），不对文件数据做日志。

> **崩溃恢复实测**：ext4 崩溃后启动时执行 `journal replay` 恢复日志，脏页数据可能丢失；使用 `data=ordered` 模式能保证文件内容一致性但不保证新文件一定被看到。数据库在崩溃后必须执行 WAL 重放来恢复未持久化的事务数据——这是为什么文件系统的 Journal 不能替代数据库 WAL。

---

## 附：生产环境内核参数调优速查

| 参数 | 推荐值 | 原因 |
|---|---|---|
| `vm.swappiness` | 1（纯缓存类服务）/ 10-20（通用） | 减少不必要的 swap |
| `vm.dirty_background_ratio` | 5 | 提前触发后台回写，避免突发 |
| `vm.dirty_ratio` | 10 | 降低回写阻塞概率 |
| `vm.dirty_expire_centisecs` | 3000 (30s) | 脏页过期时间 |
| `vm.min_free_kbytes` | 131072 (128MB) | 给内核留足回收缓冲 |
| `vm.overcommit_memory` | 0（启发式）/ 1（允许 overcommit） | JVM/Redis 场景避免 OOM 假阳性 |
| `vm.zone_reclaim_mode` | 0 | 禁止 NUMA 节点内单独回收（跨节点访问优于 swap） |
| `net.core.somaxconn` | 65535 | 增大 TCP accept 队列 |
| `fs.file-max` | 6553560 | 全局文件描述符上限 |
| `kernel.threads-max` | 与内存相关 | 限制总线程数防止线程泄漏拖垮系统 |

应用这些参数后写入 `/etc/sysctl.d/99-app.conf` 并通过 `sysctl -p` 使其立即生效。

---

## 四、I/O 模型

### 4.1 五种 I/O 模型

| 模型 | 发起方式 | 数据拷贝方式 | 是否阻塞 | 典型系统调用 |
|---|---|---|---|---|
| 阻塞 I/O (BIO) | 同步 | 同步 | 全程阻塞 | `read/write`（默认） |
| 非阻塞 I/O (NIO) | 同步（轮询） | 同步 | 返回 EAGAIN 时不阻塞 | `read` + `O_NONBLOCK` |
| I/O 多路复用 | 同步（事件通知） | 同步 | 事件等待时阻塞 | `select/poll/epoll` |
| 信号驱动 I/O | 信号通知 | 同步 | 不阻塞 | `sigaction` + `F_SETSIG` |
| 异步 I/O (AIO) | 异步 | 异步 | 全程不阻塞 | `aio_read/aio_write`（POSIX） |

关键理解：前四种模型的第二阶段（数据从内核到用户）都是阻塞的；只有真正异步 I/O（第 5 种）两阶段均不阻塞。

实际上 POSIX AIO 在 Linux 上实现不佳（基于线程池模拟），生产使用极少。真正成熟的异步 I/O 是 Linux 5.1 引入的 **io_uring**。

### 4.2 select / poll / epoll 原理对比

| 特性 | select | poll | epoll |
|---|---|---|---|
| 数据结构 | 位图 (fd_set) | 链表 (pollfd) | 红黑树 + 就绪链表 |
| 最大 fd 数 | 1024 (FD_SETSIZE) | 无限制 | 无限制 |
| 时间复杂度 | O(n) 遍历所有 fd | O(n) 遍历所有 fd | O(1) 仅遍历就绪 fd |
| 内核/用户态拷贝 | 每次调用全量拷贝 | 每次调用全量拷贝 | fd 注册一次，事件通知时仅拷贝就绪事件 |
| 触发模式 | 水平触发 (LT) | 水平触发 (LT) | 水平触发 + 边缘触发 (LT/ET) |
| 适用场景 | fd 数 < 100 | fd 数 < 1000 | 高并发 (10K+) |

**epoll 实现机制**：

`epoll_create()` 在内核创建 eventpoll 对象，包含：
- **红黑树**（rbtree）：存储所有注册的 fd，用于 `epoll_ctl` 的 O(log n) 增删。
- **就绪链表**（rdllist）：存储已就绪的 fd。当 fd 就绪时，内核调用 `ep_poll_callback` 将事件节点插入就绪链表。
- **等待队列**：`epoll_wait` 时进程在此等待。

这种设计使得 `epoll_wait` 仅需遍历就绪链表（而非所有 fd），实现 O(1) 级事件获取。

**LT（Level Triggered）与 ET（Edge Triggered）**：

- **LT（水平触发）**：只要 fd 仍有数据可读/可写，每次 `epoll_wait` 都会返回该事件。是默认模式，编程简单但可能多触发。
- **ET（边缘触发）**：仅在 fd 状态发生变化时通知一次（如不可读→可读）。必须配合非阻塞 I/O，一次性读空/写空，否则可能丢失事件。优势是减少事件通知次数，适合高吞吐场景。

> **生产经验**：Nginx 使用 ET 模式 + 非阻塞 socket，在一次事件中循环读写直到 EAGAIN，最大化吞吐。除非确有必要，否则使用 LT 就够了——ET 的编程复杂度高，收益有限。

### 4.3 io_uring

io_uring 是 Linux 5.1 引入的革命性异步 I/O 接口，解决了两大问题：1) 真正的异步（非线程池模拟）；2) 极低的系统调用开销。

**核心设计**：通过两个共享内存环形缓冲区（ring buffer）与内核通信：
- **提交队列（SQ, Submission Queue）**：用户态写入 I/O 请求 → 内核消费。
- **完成队列（CQ, Completion Queue）**：内核写入 I/O 完成事件 → 用户态消费。

用户态提交请求时无需系统调用（使用 `IORING_SETUP_SQPOLL` 模式时内核线程持续轮询 SQ）只需修改共享内存；同样获取完成事件时也只需读取共享内存。在最佳场景下单次 I/O 的 CPU 开销比传统方式降低 50-90%。

io_uring 支持几乎所有 I/O 操作类型（read/write/fsync/send/recv/accept/connect 等），且支持批量提交（SQE array）和批量收割（CQE array）以摊薄开销。

ScyllaDB（Seastar 框架）、RocksDB 的异步 I/O 实现已迁移到 io_uring。Java 17+ 通过 Foreign Function & Memory API（Panama）可调用 io_uring。

---

## 五、进程调度

### 5.1 CFS 完全公平调度器

Linux 默认调度器 CFS（Completely Fair Scheduler）自 2.6.23 引入。设计理念：每个可运行进程应获得公平的 CPU 份额。

**核心数据结构**：红黑树，key 为进程的 `vruntime`（虚拟运行时间）——实际运行时间按 nice 值加权后的归一化值。CFS 每次选择 `vruntime` 最小的进程（即红黑树最左节点）运行。

**调度周期与时间片**：
- `sched_min_granularity_ns`（默认 0.75ms）：一个进程至少运行的时间。
- `sched_latency_ns`（默认 6ms）：期望所有可运行进程都获得 CPU 的周期。如果进程数太多，每个进程的时间片按 `调度周期 / 进程数` 动态计算。

**nice 值与权重**：nice 值 [-20, 19]，对应权重（约 1.25^n 缩放）。nice 每降低 1 级，CPU 份额增加约 10%。内核维护 `sched_prio_to_weight` 数组精密映射 nice 到权重。

### 5.2 调度策略

| 策略 | 优先级范围 | 特点 |
|---|---|---|
| `SCHED_NORMAL` (SCHED_OTHER) | nice -20~19 | CFS 调度，默认策略 |
| `SCHED_BATCH` | nice -20~19 | 类似 NORMAL，但唤醒频率低，适合批处理 |
| `SCHED_FIFO` | 实时优先级 1~99 | 静态优先级，不抢占不轮转，运行到主动放弃或更高优先级抢占 |
| `SCHED_RR` | 实时优先级 1~99 | 时间片轮转，同优先级下轮流运行 |
| `SCHED_DEADLINE` | - | 基于 EDF (Earliest Deadline First) 的实时调度 |

> **注意**：`SCHED_FIFO/RR` 是实时调度策略，优先级高于所有 `SCHED_NORMAL` 进程。应用错误使用实时策略可导致系统被 Starve（饥饿），SSH 都无法连接。

### 5.3 CPU 亲和性

`taskset` 可以将进程绑定到指定的 CPU 核心：

```bash
taskset -c 0-3,8-11 <pid>        # 绑定到 CPU 0-3 和 8-11（同一 NUMA node）
taskset -c 0-3 <command>         # 启动并绑定
```

**绑定原因**：避免跨 NUMA node 的内存访问延迟（远端内存访问 2x 延迟）、提高 L1/L2 cache 命中率、降低 TLB miss。

也可以在代码中通过 `pthread_setaffinity_np()` 或 `sched_setaffinity()` 设置线程级亲和性。Netty 的 NioEventLoop、DPDK 的 lcore 均使用此技术。

### 5.4 负载均衡

CFS 在每个调度 tick 检查 CPU 的负载平衡状态。主要涉及：
- **CPU 级别的负载均衡**：每个调度域的负载均衡周期，将任务从忙 CPU 迁移到闲 CPU。
- **NUMA 级别的负载均衡**：尽量避免跨 NUMA node 迁移，优先在同 NUMA node 内平衡。

负载均衡的决策由 `sd->balance_interval`（调度域平衡间隔）、`imbalance_pct`（不平衡阈值）等参数控制。可通过 `/proc/sys/kernel/sched_*` 调整。

### 5.5 NUMA 架构对后端应用的影响

多路服务器通常采用 NUMA（Non-Uniform Memory Access）架构——每个 CPU socket 拥有本地内存（本地 node 访问延迟 ~100ns，远端 node 访问延迟 ~200-300ns）。

**NUMA 陷阱**：
- **跨 NUMA 内存分配**：进程分配到 Node 0 的 CPU 但内存分配到了 Node 1，导致每次内存访问都有 2x 延迟变高。`numastat -p <PID>` 可查看进程在各 NUMA node 上的内存分布。
- **CPU 迁移**: 进程/线程在不同 NUMA node 间漂移会导致本地 cache 和 TLB 全部失效。
- **内存交错模式**：如果系统 BIOS 中启用了 Node Interleaving，内存跨 node 均匀分配，保证所有 CPU 访问延迟一致但整体访问延迟更高（适合不确定 NUMA 优化的场景）。

**优化策略**：
```bash
numactl --cpunodebind=0 --membind=0 <command>      # 绑定到同一 NUMA node
numactl --cpunodebind=0 --preferred=0 <command>    # 优先使用 Node 0 内存，不足时跨 node
```
高效应用（如 DPDK、Redis、数据库）应尽量部署在单一 NUMA node 内，必要时启动多实例分 node 绑定。`lscpu` 可查看服务器 NUMA 拓扑。

---

## 六、Linux 系统调用与内核

### 6.1 用户态/内核态切换

CPU 通过特权级别（x86 的 Ring 0~3，ARM 的 EL0~EL3）隔离用户态和内核态。系统调用是用户态主动进入内核态的唯一合法入口（中断和异常是硬件触发）。

**系统调用流程**：
1. 用户程序调用 glibc 封装的函数（如 `read()`）。
2. glibc 将系统调用号存入 `rax`，参数存入 `rdi/rsi/rdx/r10/r8/r9`。
3. 执行 `syscall` 指令（x86-64），CPU 切换到内核栈、提升特权级。
4. 内核 `entry_SYSCALL_64` 处理入口，根据 `rax` 调用系统调用表中的对应函数。
5. 执行完成后通过 `sysret` 返回用户态。

**切换开销**：单次系统调用耗时约 50-200ns（不含业务逻辑），主要开销在于：保存/恢复寄存器、切换栈、TLB/Cache 影响。持续高频的系统调用会显著拖累吞吐——这也是 io_uring 和 DPDK 设计核心理念（减少/消除系统调用）。

### 6.2 常用系统调用

| 系统调用 | 功能 | 生产场景 |
|---|---|---|
| `read/write` | 文件读写 | 通用 I/O |
| `sendfile` | 文件→socket 零拷贝 | Nginx/Kafka 静态文件发送 |
| `splice` | 两个 fd 间零拷贝传输 | Nginx 代理场景 |
| `mmap` | 文件映射到内存 | 数据库 WAL 读取、JVM 堆外内存 |
| `madvise` | 给内核内存使用建议 | `MADV_DONTNEED` 回收、`MADV_HUGEPAGE` 启用大页 |
| `mlock` | 锁定内存不 swap | 加密密钥存储、JVM `-XX:+UseLargePages` |
| `prctl` | 进程控制（名称、core dump 等） | 守护进程改名 |
| `setrlimit` | 资源限制 | `nofile`（文件描述符限制）、`nproc`（进程数限制） |
| `epoll_create/wait` | I/O 多路复用 | 所有 Reactor 模型网络框架 |
| `futex` | 快速用户态锁 | `pthread_mutex`、`Semaphore` 的底层实现 |

### 6.3 /proc 文件系统常用文件

| 文件路径 | 内容 | 排查场景 |
|---|---|---|
| `/proc/cpuinfo` | CPU 型号、核心数、cache 大小 | 确认 CPU 规格 |
| `/proc/meminfo` | 内存总量/空闲/buffer/cache/swap | 内存水位分析 |
| `/proc/<pid>/status` | 进程内存、线程数、fd 数 | 单进程资源画像 |
| `/proc/<pid>/maps` | 进程虚拟内存映射详情 | 内存横向分析 |
| `/proc/<pid>/fd/` | 进程打开的文件描述符 | fd leak 排查 |
| `/proc/<pid>/limits` | 进程资源限制 (ulimit) | 检查 nofile 生效 |
| `/proc/<pid>/sched` | 进程调度策略和优先级 | 检查实时调度 |
| `/proc/<pid>/smaps` | 详细内存映射（含 RSS/PSS/swap） | 精准内存分析 |
| `/proc/interrupts` | 中断分布（含 CPU 亲和性） | 网卡中断均衡 |
| `/proc/net/dev` | 网卡收发包统计 | 网络流量分析 |

---

## 七、性能排查

### 7.1 CPU 排查

**top** 是最直观的实时 CPU 观察工具。关键指标：
- `us`（user）：用户态 CPU 占比。高则程序计算密集或频繁系统调用。
- `sy`（system）：内核态 CPU 占比。持续 >20% 意味着系统调用开销大或中断密集。
- `ni`（nice）：低优先级用户态 CPU 占比。
- `id`（idle）：空闲 CPU 占比。
- `wa`（iowait）：CPU 等待 I/O 完成的时间占比。高 `wa` 通常是磁盘瓶颈，而非 CPU 问题（此时 CPU 实际上是空闲的，只是被计入等待）。
- `si/`（softirq）：软中断占比。高 `si` 通常是网卡中断处理开销大。

**mpstat**（每 CPU 使用率）：
```bash
mpstat -P ALL 1           # 每秒输出各 CPU 使用率
```
可快速判断负载是否均衡、是否存在单 CPU 打满的整体瓶颈。

**pidstat**（进程级别 CPU）：
```bash
pidstat -u 1              # 每秒输出进程 CPU 使用率
pidstat -w 1              # 进程上下文切换率（cswch: 自发切换, nvcswch: 非自发切换）
pidstat -t -p <PID> 1     # 查看进程的线程级 CPU 分布
```

> **实战**：Java 应用中，通过 `pidstat -t -p <PID>` 识别占 CPU 最高的线程 TID → 转十六进制 → 在线程 dump（`jstack <PID>`）中搜索 `nid=0x<TID>`，定位到具体代码。

### 7.2 内存排查

**free**：
```bash
free -h
#              total   used    free    shared  buff/cache   available
# Mem:          62Gi   18Gi    8.0Gi   2.0Gi   36Gi         41Gi
```
关键：`available` 才是真正可用的内存（≈ free + 可回收的 buffer/cache）。`free` 低但 `available` 高是正常的（Linux 倾向将空闲内存用作 page cache）。

**vmstat**（虚拟内存统计）：
```bash
vmstat 1
# r  b   swpd   free   buff  cache   si   so   bi   bo   in   cs
# 2  0      0  8000M  2000M  36000M   0    0    0  100  500  3000
```
- `si/so`：swap in/out。持续非零说明内存不足在 swap。
- `bi/bo`：block in/out（磁盘读写 KB/s）。高 `bi` 可能是缺页中断读盘。
- `r`：可运行进程数。持续 > CPU 核数说明 CPU 瓶颈。
- `b`：在不可中断睡眠的进程数。持续 >0 说明 I/O 瓶颈。

**pmap**（进程内存映射）：
```bash
pmap -x <PID> | sort -k3 -rn | head -20    # 按 RSS 排序，找内存占用大户
```
分析 JVM 堆外内存泄漏的利器：如果 `pmap` 中大量 `anon` 段接近堆大小，可能是 DirectByteBuffer 或 Native memory 泄漏。

### 7.3 I/O 排查

**iostat**（磁盘 I/O 统计）：
```bash
iostat -x 1
# r/s w/s  rkB/s wkB/s  await  r_await  w_await  %util
#  0  200     0  40000    3.0      0.0      3.0   60.0
```
- `await`：I/O 平均响应时间（ms）。持续 >10ms 说明磁盘延迟高（HDD 正常约 5-15ms，SSD 应 <1ms）。
- `%util`：设备繁忙时间占比。注意 SSD 并发 I/O 时 100% util 不代表瓶颈（NVMe 有队列并行），但 HDD 的 100% util 通常是瓶颈。
- `svctm`：平均服务时间。低于 `await` 说明排队严重。

**iotop**（进程级 I/O）：
```bash
iotop -o                   # 仅显示有 I/O 活动的进程
iotop -p <PID>             # 查看特定进程
```

### 7.4 综合工具

**sar**（System Activity Reporter）可回溯历史数据（通过 `sysstat` 服务的 10 分钟采集周期）：
```bash
sar -u 1 10              # CPU 使用率
sar -r 1 10              # 内存使用
sar -n DEV 1 10          # 网络流量
sar -b 1 10              # I/O 速率
sar -f /var/log/sa/saXX  # 查看历史数据（如 sa04 代表当月 4 号）
```

**perf**（性能分析）——生产环境最关键的工具：
```bash
perf top -g              # 实时热点函数 + 调用栈
perf record -g -p <PID> -- sleep 30   # 采样 30 秒
perf report -g           # 查看报告（火焰图）
perf stat -p <PID> -- sleep 10        # 事件计数（instructions/cycles/cache-miss/branch-miss）
```
火焰图（FlameGraph）是 perf 报告的可视化形式，直观展示 CPU 时间在不同函数栈中的分布。

### 7.5 常见性能问题案例

**案例 1：CPU iowait 高企**

现象：`top` 显示 `wa` 持续 >30%，应用响应缓慢。
分析：`iostat -x 1` 确认磁盘 `await` >50ms，`%util` 接近 100%。`iotop` 发现大量顺序写。
根因：应用日志 `fsync` 同步刷盘频率过高。
解决：调大日志缓冲区、合并写、使用异步日志（如 Log4j2 AsyncAppender 或直接上 Kafka 日志收集）。

**案例 2：频繁 Full GC 导致 STW**

现象：服务周期性超时，`jstat -gc` 显示 Full GC 频率异常。
分析：`pmap -x` 发现堆外内存 RSS 占用极大（Direct Memory），结合 GC 日志，确认是堆外内存不足触发 `System.gc()`。
根因：`-XX:MaxDirectMemorySize` 未配置，默认与 `-Xmx` 相同，大量 DirectByteBuffer 未及时回收。
解决：显式设置 `-XX:MaxDirectMemorySize`，升级到使用 `Cleaner` 管理直接内存的框架版本。

**案例 3：页表争抢导致性能瓶颈**

现象：32 核服务器上 32 线程应用，`perf top` 中 `clear_page` 和 `page_fault` 占比高。
分析：线程频繁分配新内存触发 COW 和 minor fault，内核锁争抢（`pte_offset_map_lock` 在 `perf` 热点中）。
根因：高频内存分配（如每次请求创建大量临时对象），内存分配路径上的内核锁无法 1:32 扩展。
解决：使用对象池、栈上分配（逃逸分析优化）、替换为 jemalloc 减少分配路径上的竞争。

**案例 4：Redis fork 延迟**

现象：Redis 定期 fork 时响应延迟抖动 >100ms。
分析：`info stats` 显示 `latest_fork_usec > 100000`。Redis 进程 RSS 已超过 20GB。
根因：`fork` 需要复制父进程所有页表（RSS/PAGE_SIZE 条 PTE），20GB → 500 万条 PTE，逐一复制耗时 >100ms。
解决：拆分大实例为多个小实例（单实例 <10GB）、启用 THP（减少 PTE 条目数）、或使用 Redis Cluster 水平扩展。

> **总结**：操作系统知识对后端工程师的价值不在于记住所有内核参数，而在于——当生产环境出问题时，能沿着进程状态、内存路径、I/O 栈、调度行为这条线索，快速定位瓶颈是从哪里产生的。
