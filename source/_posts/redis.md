---
title: Redis 深度解析
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - Redis
  - 缓存
  - 数据结构
  - 高可用
  - 分布式
categories:
  - 存储
---
## 一、核心数据结构底层实现

Redis 的每种数据类型内部都可能对应多种底层编码实现，Redis 会根据数据规模和元素特征自动选择最优的内部编码。理解这些底层结构，是调优 Redis 性能、排查内存问题的基础。

### 1.1 SDS：简单动态字符串

C 语言原生字符串以 `\0` 结尾，获取长度需要 O(n) 遍历，且极易引发缓冲区溢出。Redis 封装了 SDS（Simple Dynamic String）替代 C 字符串，几乎所有字符串操作都基于 SDS。

```c
// SDS 结构 (sds.h, Redis 3.2+ 使用多种 header)
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;         // 已用长度（不含 \0）
    uint8_t alloc;       // 总分配长度（不含 \0 和 header）
    unsigned char flags; // 低 3 位标识 header 类型
    char buf[];          // 实际字符数组，末尾仍保留 \0 兼容 C 函数
};
```

SDS 相比 C 字符串的优势：

| 特性 | C 字符串 | SDS |
|------|---------|-----|
| 获取长度 | O(n) 遍历 | O(1) 读 len 字段 |
| 缓冲区安全 | 需手动管理，易溢出 | API 自动检查扩容 |
| 内存分配 | 每次修改都重新分配 | 预分配 + 惰性释放 |
| 二进制安全 | 遇 `\0` 截断 | 以 len 为准，可存任意二进制 |

**空间预分配策略**：当 SDS 长度小于 1MB 时，扩容为所需空间的 2 倍；超过 1MB 时，每次只多分配 1MB。**惰性释放**：字符串缩短时不立即回收内存，而是用 `free` 字段记录，供后续扩展复用。

**SDS Header 类型选择**：sdshdr5 用 5 位存长度（最大 31），sdshdr8/16/32/64 分别用 uint8/16/32/64 存 len 和 alloc。Redis 根据实际字符串长度选择最小的 header，极致节省内存。

### 1.2 Ziplist：压缩列表

Ziplist 是 Redis 为了节约内存而设计的顺序存储结构，本质是一块连续内存，每个节点紧挨排列，适合元素较少、值较小的场景。在 Redis 7.0 之前，Hash 和 ZSet 在小数据量时默认使用 ziplist。

```
<zlbytes><zltail><zllen><entry1><entry2>...<entryN><zlend>
```

- **zlbytes**：4 字节，记录整个 ziplist 占用的字节数
- **zltail**：4 字节，记录最后一个 entry 的偏移量，支持双向遍历
- **zllen**：2 字节，entry 数量（超过 65535 时需遍历获取）
- **entry**：每个元素，包含 prevlen（前一个 entry 长度）、encoding（编码类型+数据长度）、data
- **zlend**：1 字节，固定值 0xFF 标记结尾

**连锁更新问题**：每个 entry 的 `prevlen` 字段，当前一个 entry 长度小于 254 字节时占 1 字节，否则占 5 字节。极端情况下，若所有 entry 长度都是 253 字节，当头部的 entry 被修改为 254 字节时，会触发后续所有 entry 的 prevlen 逐级扩容，导致 N 次内存重分配——这被称为"连锁更新"（cascade update）。实际生产中极少遇到，但设计上需要了解。

### 1.3 Quicklist：快速列表

Redis 3.2 后，List 的底层实现从 `linkedlist + ziplist` 的混合体统一为 **quicklist**。quicklist 是一个双向链表，但每个链表节点存储的是一个 ziplist。

```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        // 所有 ziplist 中 entry 总数
    unsigned long len;          // quicklistNode 数量
    int fill : QL_FILL_BITS;   // ziplist 填充因子
    unsigned int compress : QL_COMP_BITS; // 节点压缩深度
} quicklist;
```

- **fill 填充因子**：默认 -2，表示每个 ziplist 最大 8KB。值越小限制越严格。
- **compress 压缩深度**：链表两端各有 N 个节点不压缩，中间的节点使用 LZF 算法压缩。默认 0（不压缩）。适合消息队列场景——最新和最老的数据常被访问，中间的历史数据可以压缩。

这样设计兼顾了 ziplist 的内存紧凑和 linkedlist 的快速头尾操作。执行 `LPUSH`/`RPOP` 时，如果头/尾 ziplist 已满则创建新节点，单个 ziplist 内的插入是 O(n) 的，但由于 ziplist 很小（8KB 左右），实际影响可控。

### 1.4 Listpack：紧凑列表

Redis 7.0 起，listpack 全面替代 ziplist，成为 Hash、ZSet、Stream 等数据类型的紧凑编码方案。核心改进是**彻底消除了连锁更新问题**。

```
<totalBytes><numEntries><entry1><entry2>...<entryN><end>
```

每个 entry 结构：
```
<encoding><data><backlen>
```

关键变化：每个 entry **不再记录前一个 entry 的长度**，而是记录**自己的长度**（backlen），并且 backlen 放在 entry 尾部。遍历时从前向后解析 encoding 确定跳过多少字节，反向遍历时从后向前解析 backlen。由于不依赖前驱节点的大小，连锁更新问题不复存在。

**backlen 的编码**：长度小于 127 时占 1 字节，否则使用多字节编码（每字节最高位为 1 表示后续还有字节，类似 UTF-8），最大可表示 5 字节回退长度。

### 1.5 Skiplist：跳表

Redis 的 ZSet 在数据量较大时使用 skiplist + dict 的组合结构（skiplist 负责范围查询和排序，dict 负责 O(1) 按成员查分）。其中 skiplist 是核心。

**结构定义**：
```c
typedef struct zskiplistNode {
    sds ele;                // 成员对象
    double score;           // 分值
    struct zskiplistNode *backward; // 后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进指针
        unsigned long span;            // 跨度（该层两节点间跳过了多少个节点）
    } level[]; // 柔性数组，每个节点层数不同
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;   // 节点总数
    int level;              // 当前最大层数（不含 header）
} zskiplist;
```

**层级生成算法**：新节点插入时，Redis 使用幂次定律（power law）随机生成层数。默认 `ZSKIPLIST_P = 0.25`，即每个节点有 25% 的概率上升到第 n+1 层。数学上保证平均每 4 个节点中有 1 个在第 2 层，每 16 个有 1 个在第 3 层，以此类推。实测最大层数限制为 32 层。

```c
// t_zset.c 中的随机层数生成
int zslRandomLevel(void) {
    int level = 1;
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

**查找复杂度**：O(log N)。从最高层开始，若 forward 节点的 score 仍小于目标值则继续前进，否则降一层。每降一层，搜索范围急剧缩小，期望比较次数大约为 `log(N) / log(1/p)`。

**插入复杂度**：O(log N)。查找插入位置时，用 `update[]` 数组记录每层的前驱节点，用 `rank[]` 记录每层累计跨度。逐层插入并更新 span 值。

**为什么不用红黑树**：跳表实现简单（无需旋转和染色），支持 O(log N) 的范围查询（找到起点后向后遍历即可），且天然适合并发（红黑树 rebalance 需要锁大片区域）。

**ZSet 为什么同时使用 dict 和 skiplist**：dict 保证 O(1) 按成员查分的操作（`ZSCORE`），skiplist 保证 O(log N) 范围查询（`ZRANGE`/`ZRANGEBYSCORE`）。两者共存会带来内存开销——每个元素在 dict 和 skiplist 中各存一份，但 Redis 通过指针共享元素对象（ele 是 sds 指针，score 是值拷贝），实际附加开销主要是 skiplist 的层级指针。

### 1.6 Dict：字典与渐进式 Rehash

Redis 的字典（dict）是所有 KV 数据、Hash 类型大数据的底层实现。使用**链式哈希**解决哈希冲突。

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];       // 两个哈希表，用于渐进式 rehash
    long rehashidx;     // -1 表示未在 rehash，>=0 表示正在 rehash 的桶索引
    unsigned long iterators; // 安全迭代器数量
} dict;
```

**哈希算法**：默认使用 SipHash（Redis 5.0+，抗哈希洪水攻击），替代了早期的 MurmurHash2。

**渐进式 Rehash 核心流程**：

1. 为 `ht[1]` 分配空间，大小为 `ht[0].used * 2` 后的第一个 2^n。
2. 将 `rehashidx` 置为 0，开始 rehash。
3. 每次对 dict 执行增删改查操作时，顺带将 `ht[0]` 中 `rehashidx` 对应的桶迁移到 `ht[1]`，`rehashidx++`。
4. 当 `rehashidx == ht[0].size` 时，rehash 完成，释放 `ht[0]`，将 `ht[1]` 赋值给 `ht[0]`，重置 `rehashidx = -1`。

**在 Rehash 期间的查询/插入/删除**：先在 `ht[0]` 中找，没找到再到 `ht[1]` 中找。**新增数据直接写入 `ht[1]`**，确保 `ht[0]` 只减不增，最终变空。

**定时 Rehash**：Redis 在 `serverCron` 中每次执行 1ms 的渐进式 rehash，使用 `dictRehashMilliseconds()`。同时，Redis 6.0+ 支持在 IO 线程中并行 rehash（`activerehashing yes`）。

**扩容与缩容触发条件**：
- 扩容：负载因子 `used/size >= 1` 且无 BGSAVE/BGREWRITEAOF 时；若有 BGSAVE 等子进程则阈值提升到 5（为了减少写时复制内存开销）。
- 缩容：`used/size < 0.1` 时（数据量比值小于 10%）。

### 1.7 Intset：整数集合

当 Set 中所有元素都是整数且数量不超过 `set-max-intset-entries`（默认 512）时，Redis 使用 intset 存储。

```c
typedef struct intset {
    uint32_t encoding; // INTSET_ENC_INT16 / INT32 / INT64
    uint32_t length;
    int8_t contents[]; // 实际按 encoding 位宽读取
} intset;
```

intset 内部是**有序数组**，二分查找 O(log N)。当插入的元素超出当前编码范围时，触发**升级**——例如当前是 INT16，插入 INT32 值时，整个 intset 升级到 INT32 编码，所有已有元素统一按新位宽重新排列。**不支持降级**，升级后即使删除了大值元素，编码也不会回退。

### 1.8 各数据类型编码切换阈值

下面是各数据类型在 Redis 7.0 中的默认编码切换规则：

| 类型 | 小数据编码 | 阈值配置 | 大数据编码 |
|------|-----------|----------|-----------|
| String | embstr (<44B) / raw | — | — |
| List | quicklist | `list-max-ziplist-size`(-2=8KB) | quicklist |
| Hash | listpack | `hash-max-listpack-entries`(512), `hash-max-listpack-value`(64B) | hashtable |
| Set | intset | `set-max-intset-entries`(512) | hashtable |
| ZSet | listpack | `zset-max-listpack-entries`(128), `zset-max-listpack-value`(64B) | skiplist+dict |

当 Hash 或 ZSet 的 entry 数量超过阈值，或单个 value 长度超过阈值时，触发编码升级转换，此过程不可逆（listpack 不会自动降级回 listpack）。

## 二、对象系统

### 2.1 RedisObject 结构

Redis 中所有数据都以 `redisObject` 包裹：

```c
typedef struct redisObject {
    unsigned type:4;      // 对外类型：string/list/set/zset/hash/stream/module
    unsigned encoding:4;  // 内部编码：raw/embstr/int/ziplist/quicklist/listpack/skiplist...
    unsigned lru:24;      // LRU 时钟或 LFU 计数器
    int refcount;         // 引用计数
    void *ptr;            // 指向实际数据的指针
} redisObject;  // 共 16 字节
```

- **type**：对客户端暴露的逻辑类型（`TYPE` 命令返回）。
- **encoding**：内部实际存储结构（`OBJECT ENCODING` 命令返回）。
- **lru**：24 位，存的是 LRU 秒级时间戳或 LFU 计数（高 16 位是最后访问时间分钟级、低 8 位是对数访问频率）。
- **refcount**：引用计数，用于共享对象和内存回收。
- **ptr**：指向 sdshdr / ziplist / skiplist 等实际数据结构。

**embstr vs raw**：String 长度 <= 44 字节（OBJ_ENCODING_EMBSTR_SIZE_LIMIT）时使用 embstr 编码——redisObject 和 sdshdr8 在同一块连续内存中，只需一次 malloc，且缓存局部性更好。超过阈值后使用 raw 编码，redisObject 和 sds 分两块内存。embstr 是只读的，任何修改操作都会将其转为 raw。

### 2.2 共享对象池

Redis 启动时会创建 0~9999 的整数 String 对象，存放在共享对象池中。命令中引用的整数常量不会新建 redisObject，而是直接使用池中对象，节省内存和 CPU。

```c
// server.c 初始化
for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
    shared.integers[j] = createObject(OBJ_STRING, ...);
}
```

**限制**：共享对象池仅在单机模式下有效。Redis Cluster 或 Sentinel 模式下，跨节点共享没有意义；而且对象共享要求被共享的对象完全相同（值和类型），由于共享对象本身需要被 refcount 追踪，嵌套数据结构的共享实现复杂且收益不高，因此 Redis 仅对 0~9999 整数做共享。

### 2.3 内存回收与对象淘汰

- **引用计数**：`INCR` 操作不修改原对象而是创建新对象，因此共享对象可能被多个 key 引用。`refcount` 减到 0 时释放内存。
- **LRU/LFU**：`lru` 字段用于内存淘汰。24 位中，LRU 模式存的是 `server.lruclock` 的秒级时间戳（通过 `serverCron` 每 100ms 更新一次）；LFU 模式高 16 位存最后访问时间（分钟级），低 8 位存对数衰减的访问频率计数器。具体算法在后文第 10 节展开。

## 三、持久化

### 3.1 RDB：快照持久化

RDB 将某一时刻的内存数据以**二进制快照**形式写入磁盘，文件紧凑、恢复速度快，适合灾难恢复和数据迁移。

**触发方式**：

- **手动触发**：`SAVE`（阻塞当前进程，期间拒绝所有请求）、`BGSAVE`（fork 子进程执行，不阻塞主进程）。
- **自动触发**：配置 `save <seconds> <changes>`，如 `save 900 1` 表示 900 秒内至少 1 次修改则触发 BGSAVE。可配置多条规则，满足任一即触发。

**写时复制（Copy-On-Write）**：`BGSAVE` 执行时，主进程 fork 出子进程。fork 时父子进程共享物理内存页，内核将内存页标记为只读。当主进程需要修改某页时，触发**缺页异常**，内核为该页创建副本供主进程修改，子进程仍读取原页。因此 RDB 快照是 fork 时刻的瞬时一致性视图。

```bash
# 查看最近一次 BGSAVE 是否成功
INFO Persistence
# rdb_last_bgsave_status: ok
# rdb_last_cow_size: 写时复制内存大小
```

**RDB 风险**：两次快照之间的数据可能丢失。`BGSAVE` 期间 COW 内存翻倍风险——如果写操作频繁，COW 复制的内存页数量可能接近进程内存总量。

**触发条件补充**：
- 主从全量复制时，主节点自动 BGSAVE 生成 RDB 发给从节点。
- `SHUTDOWN` 或 `DEBUG RELOAD` 时也会触发生成 RDB。
- `FLUSHALL` / `FLUSHDB` 会清空当前数据，但不影响已落盘的 RDB 文件。

### 3.2 AOF：追加文件持久化

AOF（Append Only File）记录所有写命令，通过重放命令恢复数据，数据安全性更高。

**三种刷盘策略（`appendfsync`）**：

| 策略 | 行为 | 安全性 | 性能 |
|------|------|--------|------|
| always | 每条写命令都 fsync | 最高，最多丢失一条命令 | 最低，磁盘 IO 瓶颈 |
| everysec | 每秒 fsync 一次（异步线程） | 默认值，最多丢 1 秒 | 折中，推荐 |
| no | 由操作系统决定刷盘时机 | 不可控 | 最高 |

**AOF 重写机制**：AOF 文件随运行时间不断膨胀。Redis 通过 **BGREWRITEAOF** 生成新的 AOF 文件，新文件只包含重建当前数据集的最小命令集（不包含中间历史）。重写过程 fork 子进程，利用 COW 机制读取快照数据；主进程在重写期间的写操作同时写入 **AOF 重写缓冲区**，子进程完成快照写入后，再将缓冲区数据追加到新文件尾部，最后原子 `rename` 覆盖旧文件。

```bash
# 手动触发 AOF 重写
BGREWRITEAOF

# 自动触发条件配置
auto-aof-rewrite-percentage 100  # AOF 文件比上次重写后增长 100%
auto-aof-rewrite-min-size 64mb   # AOF 文件最小 64MB 才考虑重写
```

### 3.3 混合持久化（RDB-AOF）

Redis 4.0 引入混合持久化，配置 `aof-use-rdb-preamble yes`。AOF 文件的前半部分是 RDB 格式的快照数据（二进制紧凑），后半部分是 AOF 格式的增量命令。重启时先加载 RDB 部分（速度快），再重放后续 AOF 增量（数据完整），兼顾恢复速度和数据安全性。

## 四、主从复制

Redis 主从复制解决数据冗余、读写分离、故障恢复问题。默认为异步复制。

### 4.1 全量复制（Full Resync）

适用于从节点初次连接、或复制偏移量已超出主节点复制积压缓冲区范围的情况：

1. 从节点发送 `PSYNC ? -1`（首次连接）。
2. 主节点执行 `BGSAVE` 生成 RDB 文件，同时将 RDB 生成期间的写命令记录到 **client-output-buffer**（复制缓冲区）。
3. RDB 生成完毕，主节点先将 RDB 文件发给从节点，从节点清空旧数据，加载 RDB。
4. 主节点再将缓冲区中的增量命令发给从节点执行，完成同步。

**全量复制开销**：主节点 BGSAVE 消耗 CPU 和内存（COW），RDB 文件网络传输占用带宽，从节点清空旧数据并加载 RDB 耗时（6GB RDB 可能需数分钟）。

### 4.2 部分复制（Partial Resync）

适用于从节点短暂断线重连的场景，避免全量复制的巨大开销：

1. 从节点重连后发送 `PSYNC <replid> <offset>`。
2. 主节点检查 `replid` 是否匹配，且 `offset` 是否在 `repl-backlog` 范围内。
3. 若满足条件，主节点仅将 backlog 中从 offset 开始的增量命令发给从节点。
4. 从节点执行增量命令，恢复数据一致。

**核心依赖**：
- **Replication ID（replid）**：数据集标识，主节点生成，变动时更新。
- **Replication Offset**：主从各自维护的复制偏移量，用于表示同步进度。

### 4.3 Replication Buffer vs Replication Backlog

| 概念 | Replication Buffer | Replication Backlog |
|------|-------------------|---------------------|
| 级别 | 每个从节点独享 | 主节点全局，所有从节点共享 |
| 内容 | 完整 RDB + 增量命令 | 仅增量命令（环形缓冲区） |
| 作用 | 全量复制期间暂存增量 | 支持部分复制 |
| 大小 | 默认 slave 的 client-output-buffer-limit 256mb | `repl-backlog-size` 默认 1MB |
| 生命周期 | 全量复制完成后释放 | 持续存在，环形覆盖 |

**Backlog 环形缓冲区**：固定大小，写入超过容量时覆盖最老的数据。若从节点断线期间 backlog 覆盖了该节点断点位置，则只能退化为全量复制。生产环境建议 backlog 设为 `(断线时长秒数 * 写QPS * 每条命令平均字节数)` 的 2~3 倍。

### 4.4 无盘复制（Diskless Replication）

Redis 2.8.18 引入，通过 `repl-diskless-sync yes` 启用。主节点不将 RDB 写入磁盘，而是直接通过 socket 将 RDB 流式传输给从节点，适合磁盘 IO 压力大但网络带宽充足的场景。

```
主节点 fork 子进程 → 子进程直接通过 socket 写 RDB 到从节点 → 增量子进程和主进程配合传输
```

### 4.5 主从切换与复制拓扑

常见拓扑：

- **一主多从**：最简单，读负载均衡，但主故障需人工介入。
- **链式复制（级联）**：主 → 从1 → 从2，减轻主节点的复制压力（fork 次数减少），但链路变长，延迟叠加。中间节点需开启 `replica-serve-stale-data yes` 保证对下游可用。
- **哨兵/集群托管**：自动故障转移。

## 五、哨兵（Sentinel）

Sentinel 是 Redis 官方高可用方案，本质是一个分布式监控系统，负责**监控、通知、自动故障转移**。生产环境至少部署 3 个 Sentinel 实例（通常奇数个），独立于 Redis 节点运行。

### 5.1 主观下线（SDOWN）vs 客观下线（ODOWN）

- **SDOWN（Subjective Down）**：单个 Sentinel 在 `down-after-milliseconds` 时间内未收到目标节点响应，独立判定该节点"可能挂了"。这是一个不可信的局部判断。
- **ODOWN（Objective Down）**：当判定主节点 SDOWN 的 Sentinel 达到 `quorum`（法定人数）时，该主节点被标记为客观下线，触发故障转移。需要其他 Sentinel 投票确认。

关键区别：SDOWN 的触发对象可以是主/从/其他 Sentinel；ODOWN **仅针对主节点**。

### 5.2 Leader 选举

当主节点被标记为 ODOWN 后，需要从所有 Sentinel 中选出一个 Leader 来执行故障转移。使用 **Raft 算法**的简化版：

1. 每个 Sentinel 都可以发起选举，要求其他 Sentinel 投票给自己。
2. 每个 Sentinel 在一个 epoch（配置纪元）内只能投一票，先到先得。
3. 获得 `max(quorum, N/2+1)` 票的 Sentinel 成为 Leader。
4. 若规定时间内没有选出 Leader，epoch++，重新选举（选举超时取 `2 * failover-timeout` 或 `10s` 中的较大值）。

### 5.3 故障转移流程

1. Leader Sentinel 从该主节点的所有在线的从节点中，选出新的主节点（过滤器 + 排序器）。
   - **过滤**：排除下线、断线、5 秒内未响应 INFO 的从节点。
   - **排序**：优先级最高（`replica-priority` 小的） → 复制偏移量最大（数据最新） → Run ID 字典序最小。
2. Leader 向选中的从节点发送 `SLAVEOF NO ONE`，令其提升为主节点。
3. Leader 向其他从节点发送 `SLAVEOF <new_master>`，改为从属新主。
4. Leader 将旧主节点标记为"待上线"，旧主恢复后自动变为新主的从节点。

### 5.4 配置纪元（Configuration Epoch）

每个 Sentinel 维护一个 `current_epoch`（类似 Raft 的 term），全局单调递增。每次选举时 `current_epoch++`，投票时携带 epoch。epoch 的作用：
- 防止过期消息干扰（低 epoch 消息被忽略）。
- 保证一次选举中只有一个 Leader（同一 epoch 一人一票）。
- 写入配置文件 `sentinel.conf` 持久化，重启不丢失。

### 5.5 哨兵模式的局限性

- Sentinel 本身不是强一致的，选举过程中可能出现短暂的脑裂。
- 故障转移期间（通常数十秒），主节点不可写。
- 客户端需要支持 Sentinel 协议（如 JedisSentinelPool）来动态发现主节点。
- 不负责数据分片，主节点数据量受单机内存限制。

## 六、集群（Cluster）

Redis Cluster 是官方提供的**去中心化**分布式方案，解决水平扩展问题。内置自动分片、故障转移、配置管理。

### 6.1 Hash Slot 与数据分布

Redis Cluster 将全体键空间划分为 **16384 个哈希槽（hash slot）**，每个主节点负责一部分 slot。

```
slot = CRC16(key) % 16384
```

**为什么是 16384？**来自 antirez 的解释：
- 心跳包头用 2KB bitmap 携带节点负责的 slot 信息，16384 个位恰好 2KB（2048 字节 = 16384 bits），不需要额外的带宽。
- 集群规模不会大到超过 1000 个主节点，每个节点 16 个 slot 的平均粒度足够细。65536 的 bitmap 需要 8KB，大部分是浪费。
- CRC16 算法用 16384 取模，计算效率高。

**Hash Tag**：`{` 和 `}` 之间的内容参与 CRC16 计算，保证关联的 key 落入同一 slot。例如 `user:{1001}:id` 和 `user:{1001}:name` 使用 `1001` 做 hash，确保在同一节点，支持多 key 操作。

```bash
# 查看 key 属于哪个 slot
CLUSTER KEYSLOT mykey
# 返回 0~16383
```

### 6.2 MOVED 重定向 vs ASK 重定向

客户端向任意节点发送请求时：
- 若 key 所在 slot 属于当前节点：正常处理。
- 若 key 所在 slot 不属于当前节点：返回 `MOVED <slot> <ip:port>`，客户端收到后**永久**更新自己的 slot→node 映射表。
- 若 key 所在 slot 正在从当前节点**迁出**：返回 `ASK <slot> <ip:port>`，客户端先发送 `ASKING` 命令到目标节点，再执行原命令。ASK 是**临时**重定向，不更新客户端缓存。

**两者本质区别**：MOVED 表示 slot 的所有权已转移；ASK 表示迁移进行中，slot 在两个节点各有一部分，只是临时转发。

```java
// Jedis Cluster 自动处理 MOVED 和 ASK
// 底层依赖 JedisClusterConnectionHandler 维护 slot → JedisPool 映射
JedisCluster jedisCluster = new JedisCluster(
    new HostAndPort("127.0.0.1", 7001)
);
jedisCluster.set("key", "value"); // 自动重定向
```

### 6.3 集群总线（Gossip 协议）

Redis Cluster 节点之间通过 **Cluster Bus**（端口 17000，= client_port + 10000）使用 Gossip 协议通信。每个节点定期（每秒）随机选取若干节点发送 PING，接收 PONG 回复。

**Gossip 消息体包含**：
- 自身状态（节点 ID、epoch、flags）
- 自身负责的 slot bitmap（2KB）
- 随机携带 1/10 已知节点的信息（保证最终一致性）

**为什么 Gossip 而非集中式**：
- 去中心化：无需单独的元数据节点，避免了单点故障。
- 最终一致性：cluster规模不大（<1000节点）时，元数据传播收敛迅速（数秒）。
- 自动发现：新节点通过 `CLUSTER MEET` 引入集群后，Gossip 将其信息扩散给全网。

**Gossip 消息类型**：PING、PONG（回复和广播）、MEET（邀请加入）、FAIL（标记下线）、PUBLISH（Pub/Sub 广播）。

### 6.4 集群的主从切换

集群中每个主节点可配若干从节点，从节点不断复制主节点数据（同普通主从复制）。

**故障检测**：集群使用类似 Sentinel 的机制——节点间通过 Gossip 交换 `PFAIL`（疑似下线）信息，当超过半数主节点认为某主节点 `PFAIL` 时，该节点被标记为 `FAIL`，其从节点发起选举。

**从节点选举**：
1. 从节点发现主节点 FAIL 后，等待 `500ms + 随机(0~500ms)` 后发起选举。
2. 向所有主节点请求投票，投票依据是**复制偏移量**——仅在从节点的 offset 足够新时主节点才会投票。
3. 获得 `N/2+1` 票的从节点胜出，执行 `clusterSetNodeAsMaster()`，接管 slot。
4. 向整个集群广播新的 slot→节点映射（`PONG` 消息包含新配置纪元）。

### 6.5 数据迁移与 ASKING

在线扩缩容时需要使用 `redis-cli --cluster reshard` 或在客户端使用 `CLUSTER SETSLOT ... MIGRATE` 命令进行槽迁移。迁移过程中，单个 slot 的 key 在两个节点各有一部分（迁移中的 key）：

1. 源节点 `CLUSTER SETSLOT <slot> MIGRATING <target-node-id>` → 此 slot 上的请求若无本地对应 key，返回 ASK 重定向。
2. 目标节点 `CLUSTER SETSLOT <slot> IMPORTING <source-node-id>` → 收到 ASKING + 原命令时允许执行。
3. 源节点逐 key 执行 `MIGRATE`（内部：`DUMP` 序列化 key → `RESTORE` 到目标 → `DEL` 源端），MIGRATE 是**原子**操作，单个 key 迁移期间阻塞。
4. 迁移完成后，双方执行 `CLUSTER SETSLOT <slot> NODE <target-node-id>`，slot 归属变更。

**大规模迁移实践**：在线迁移时使用 `pipeline` 批量 MIGRATE 减少 RTT，控制 `migrate-timeout` 防止长尾阻塞。迁移速率通过 `--cluster-timeout` 和 `--pipeline <N>` 参数控制。

## 七、缓存策略

### 7.1 缓存穿透

**现象**：查询一个数据库中根本不存在的 key，缓存层和数据库都查不到，大量请求直接穿透到数据库。

**解决方案**：

1. **布隆过滤器（Bloom Filter）**：在缓存层前加一层布隆过滤器，将已有的 key 存入位数组。查询前先判断 key 是否可能存在，若不存在则直接拒绝。Guava 的 BloomFilter 或 Redis 的 `BF.RESERVE` / `BF.ADD`（Redis Stack）都可以实现。缺点是有误判率（可调整），且无法删除 key（需用 Counting Bloom Filter 或布谷鸟过滤器）。

```java
// 使用 Redisson 的布隆过滤器
RBloomFilter<String> bloomFilter = redisson.getBloomFilter("product");
bloomFilter.tryInit(1000000L, 0.03); // 预期 100w 元素，3% 误判率
bloomFilter.add("product:1001");
if (!bloomFilter.contains("product:9999")) {
    return null; // 拒绝穿透
}
```

2. **空值缓存**：对查询不到的 key 也缓存一个空值（TTL 较短，如 1~5 分钟），防止短时间内反复穿透。但若恶意攻击针对不同 key，此方案无效。

3. **接口层校验**：对查询参数做合法性校验（如 ID 范围、格式），过滤明显的非法请求。

4. **数据预热**：启动时加载全量 ID 到布隆过滤器，保证覆盖率。

### 7.2 缓存击穿

**现象**：热点 key 过期瞬间，大量并发请求同时穿透到数据库，造成数据库瞬时压力过大。

**解决方案**：

1. **互斥锁（Mutex）**：发现缓存失效后，只有一个线程能重建缓存，其他线程等待（或快速失败）。使用 Redis 的 `SETNX` 实现分布式锁。

```java
public String getWithMutex(String key) {
    String value = redis.get(key);
    if (value != null) return value;

    String lockKey = "lock:" + key;
    try {
        if (redis.setnx(lockKey, "1")) {
            redis.expire(lockKey, 10); // 10s 过期防止死锁
            value = db.query(key);     // 真正重建
            redis.set(key, value, 60); // 缓存 60s
        } else {
            Thread.sleep(50);          // 等待重试
            return getWithMutex(key);  // 递归重试
        }
    } finally {
        redis.del(lockKey);
    }
    return value;
}
```

2. **永不过期（逻辑过期）**：不给热点 key 设物理 TTL，而是在 value 中嵌入一个逻辑过期时间。读取时若发现逻辑过期，开异步线程重建缓存，当前请求仍返回旧值。适合对一致性要求不严格的场景。

3. **提前异步刷新**：监控热点 key 的 TTL，在剩余时间低于阈值时（如剩余 30s），提前触发异步刷新，避免集中过期。

### 7.3 缓存雪崩

**现象**：大量缓存 key 同时过期（或缓存服务器宕机），所有请求直接打到数据库，导致数据库崩溃，进而扩散为全链路雪崩。

**解决方案**：

1. **过期时间加随机值**：在基础 TTL 上叠加一个随机偏移量（如 ±10%），打散过期时间，避免集中失效。

```java
int baseTTL = 3600; // 1 小时
int randomTTL = ThreadLocalRandom.current().nextInt(300); // 0~300s 随机
redis.setex(key, baseTTL + randomTTL, value);
```

2. **多级缓存**：本地缓存（Caffeine/Guava Cache） + Redis + 数据库，每层 TTL 错开。Redis 失效时本地缓存仍可扛住部分流量。

3. **限流降级**：在网关或业务层做限流（Sentinel/令牌桶），缓存失效时只放行少量请求到数据库，其余快速失败或返回降级值。Hystrix/Sentinel 的熔断策略可在此发挥作用。

4. **高可用部署**：Redis 使用 Sentinel/Cluster 保证缓存层高可用，避免单点故障。

5. **预热机制**：系统上线或大促前，将热点数据提前加载到缓存，避免冷启动击穿。

### 7.4 缓存一致性

缓存和数据库双写时，如何保证二者数据一致，是分布式系统的经典难题。

**核心矛盾**：CAP 理论下，更新操作无法同时保证缓存和数据库的原子性。常用策略是追求**最终一致性**。

**先删缓存再更新 DB**：

```
(1) 删除缓存
(2) 更新数据库 ──→ 在此期间另一个请求读到旧数据，写回缓存
结果：缓存是旧数据，直到再次更新才修正
```

**先更新 DB 再删缓存**（推荐）：

```
(1) 更新数据库
(2) 删除缓存 ──→ 若删除失败（网络超时），出现长期不一致
结果：绝大多数场景下不一致窗口很短（几十 ms）
```

以上两种都有失败可能。工业界常用补偿方案：

**延迟双删**：先删缓存 → 更新数据库 → 等待一段时间（如 300ms）→ 再删一次缓存。第二次删除用**异步**方式，即使失败也有重试兜底。

```java
public void updateData(Long id, Object data) {
    redis.del("data:" + id);          // 第一次删除
    db.update(data);                   // 更新数据库
    // 异步延迟双删
    CompletableFuture.runAsync(() -> {
        try { Thread.sleep(300); } catch (InterruptedException e) {}
        redis.del("data:" + id);      // 第二次删除
    });
}
```

**Canal + MQ 最终一致性**（阿里巴巴开源方案）：

```
应用更新 DB → MySQL binlog → Canal 模拟 slave 订阅 binlog → 发送 MQ → 消费者更新 Redis
```

Canal 监听 binlog 变更，异步更新缓存，最大程度保证最终一致性。适合对一致性要求较高的场景，但架构复杂度显著增加。

**订阅数据库变更的回调**：部分 ORM 框架支持数据库变更后触发回调，在回调中删除/更新缓存。

**缓存策略总结表**：

| 策略 | 不一致窗口 | 复杂度 | 适用场景 |
|------|-----------|--------|---------|
| 先删缓存再更新 DB | 大 | 低 | 不推荐 |
| 先更新 DB 再删缓存 | 小（几十ms） | 低 | 一般场景，配合重试机制 |
| 延迟双删 | 可接受 | 中 | 并发高的读写场景 |
| Canal + MQ | 极小（<100ms） | 高 | 强一致性要求 |

## 八、分布式锁

### 8.1 SET NX PX：最简单实现

Redis 2.6.12 起，`SET` 命令支持 `NX`（Not Exists）和 `PX`（毫秒级过期）参数，实现了原子性的加锁操作。

```bash
SET lock:order:1001 unique_value NX PX 30000
```

- `NX`：key 不存在时才 SET 成功（加锁）。
- `PX 30000`：30s 后自动过期，防止死锁。
- `unique_value`：使用 UUID 或线程 ID，释放锁时校验身份，防止误删他人持有的锁。

**释放锁的 Lua 脚本**（必须原子执行）：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**Java 实现**：

```java
public boolean lock(String key, String value, long expireMs) {
    String result = jedis.set(key, value, "NX", "PX", expireMs);
    return "OK".equals(result);
}

public void unlock(String key, String value) {
    String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
                    "return redis.call('DEL', KEYS[1]) else return 0 end";
    jedis.eval(script, Collections.singletonList(key),
               Collections.singletonList(value));
}
```

**单机版 SET NX 的问题**：
- **锁自动释放但任务未完成**：锁的 TTL 是固定的，如果业务执行时间超过 TTL，锁会被自动释放，其他线程获取锁，导致并发安全问题。
- **单点故障**：Redis 宕机则锁服务不可用。
- **主从切换时锁丢失**：主节点加锁后，还未同步到从节点就宕机——从节点提升为主，锁信息丢失。

### 8.2 Redlock 算法（Redis 官方分布式锁）

antirez 提出的 Redlock 算法旨在解决单点问题，适用于多个独立的 Redis Master 节点（非主从/集群）。

**加锁流程**：
1. 客户端获取当前时间（毫秒）。
2. 依次向 N 个 Redis 节点请求 SET NX PX，设置相同的 key 和 value，超时时间远小于锁的 TTL。
3. 统计成功获取锁的节点数，若 >= N/2+1（多数派），且总耗时 < 锁 TTL，则加锁成功。
4. 锁的有效时间 = TTL - 总耗时。
5. 若加锁失败，向所有节点发送解锁。

**释放流程**：向所有节点广播解锁 Lua 脚本（无论是否加锁成功），保证清理干净。

**争议**：分布式系统大神 Martin Kleppmann 发文批评 Redlock，核心论点：
- 锁依赖各节点的时钟单调性，若某节点时钟跳跃（如 NTP 校正），会导致锁提前过期。
- 在 GC 暂停、网络延迟等场景下，Redlock 无法提供强一致的互斥保证。
- 如果需要一个真正安全的锁，应该使用 Zookeeper（ZAB 协议）或 etcd（Raft 协议），而非 Redis。

antirez 的回应：Redlock 面向的是**非强一致性**场景，大多数业务不需要严格的互斥保证。争议至今仍在持续。生产环境选择时需评估对一致性的需求等级。

### 8.3 Redisson 看门狗机制

Redisson 是 Redis Java 客户端中最完善的分布式锁实现，通过**看门狗（Watchdog）**自动续期解决"锁到期但任务未完成"的问题。

```java
// Redisson 基本用法
RLock lock = redisson.getLock("lock:order:1001");
try {
    // 默认 30s 过期，看门狗每 10s 自动续期到 30s
    lock.lock();
    // 或者指定更短时间（不给 -1 就会禁用看门狗）
    // lock.lock(10, TimeUnit.SECONDS);
    // 执行业务逻辑...
} finally {
    lock.unlock();
}
```

**看门狗工作原理**：
1. `lock()` 不传 leaseTime 时，默认 30s TTL。
2. 内置的 `Watch Dog` 定时任务每 `internalLockLeaseTime / 3 = 10s` 执行一次，若锁仍被当前线程持有，则用 Lua 脚本将 TTL 续期回 30s。
3. 通过 Netty 的定时任务（`TimerTask`）驱动，不依赖独立的线程池。
4. 若客户端宕机，看门狗停止续期，锁在 30s 后自动释放。

**lock() 的 Lua 脚本**（Redisson 内部分析）：

```lua
-- KEYS[1]: 锁 key, ARGV[1]: 过期时间(ms), ARGV[2]: hash key(UUID:threadId)
if (redis.call('EXISTS', KEYS[1]) == 0) then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1);       -- 可重入计数 +1
    redis.call('PEXPIRE', KEYS[1], ARGV[1]);            -- 设置过期时间
    return nil;                                          -- 加锁成功
end;
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 1) then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1);         -- 重入计数 +1
    redis.call('PEXPIRE', KEYS[1], ARGV[1]);            -- 刷新过期时间
    return nil;                                          -- 重入成功
end;
return redis.call('PTTL', KEYS[1]);                      -- 加锁失败，返回剩余 TTL
```

说明：Redisson 使用 **Hash** 而非 String 存储锁信息——Hash Key 为锁名称，Hash 中的 field 为 `UUID:threadId`，field 的 value 为**重入计数**。因此天然支持可重入锁。

### 8.4 可重入锁、读写锁与其他

**可重入锁（ReentrantLock）**：默认的 `RLock` 就是可重入的，同一线程多次 lock 只需要对应次数 unlock，底层就是 Hash field 的重入计数。

**公平锁**：`RLock fairLock = redisson.getFairLock("lock:order");` 按请求顺序排队。

**读写锁**：

```java
RReadWriteLock rwLock = redisson.getReadWriteLock("lock:rw");
// 读锁：共享，并发读
RLock readLock = rwLock.readLock();
// 写锁：独占，排斥读写
RLock writeLock = rwLock.writeLock();
```

读写锁原理：使用额外的 key 记录读锁持有者列表，读锁之间允许并发，写锁排斥一切。

**联锁（MultiLock）**：

```java
RLock lock1 = redisson.getLock("lock1");
RLock lock2 = redisson.getLock("lock2");
RLock multiLock = redisson.getMultiLock(lock1, lock2);
multiLock.lock();   // 同时对多个锁加锁（所有都成功才算成功）
```

联锁本质是 Redlock 算法的 Redisson 实现，适用于跨 Redis 实例的分布式锁场景。

**红锁（RedLock）**：`redisson.getRedLock(lock1, lock2, lock3)` 是 MultiLock 的严格版本，遵循 Redlock 协议。

**生产选型建议**：
- 一般场景直接用 Redisson 的 `RLock` + 看门狗。
- 如果 Redis 是 Sentinel/Cluster 模式，单点锁即可（反正有自动故障转移），Redlock 的复杂度通常不必要。
- 对可靠性有极致要求，考虑 etcd 或 ZooKeeper 的线性一致性锁。

## 九、高级特性

### 9.1 Pipeline：批量执行

Redis 是 Request/Response 模型，每条命令独立往返一次网络。Pipeline 允许客户端批量发送多个命令，然后一次性读取所有响应，大幅减少网络 RTT。

```java
// Jedis Pipeline
Pipeline pipeline = jedis.pipelined();
for (int i = 0; i < 10000; i++) {
    pipeline.set("key:" + i, "value:" + i);
}
pipeline.sync(); // 一次性发送并接收所有结果
```

**注意事项**：
- Pipeline 是非事务的，命令之间互相独立，中间失败不影响后续。
- 单次 Pipeline 不宜缓存过多命令（建议 < 10k），否则一次性内存占用大且主线程处理耗时。
- 集群模式下，需要按 slot 分组 → 每个 slot 对应一个 pipeline，否则 MOVED 重定向会中断流程。

### 9.2 Lua 脚本

Lua 脚本在 Redis 中**原子性执行**（执行期间不处理其他命令，类似数据库存储过程），适合需要依赖前一步结果的复杂操作。

```lua
-- 限流示例：滑动窗口计数器
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)  -- 清理过期请求
local current = redis.call('ZCARD', key)
if current < limit then
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)
    return 1  -- 通过
else
    return 0  -- 限流
end
```

**Lua 脚本管理**：
- `SCRIPT LOAD` 将脚本缓存到 Redis，返回 SHA1 摘要。
- `EVALSHA <sha1>` 通过 SHA1 调用缓存脚本，省去网络传输（推荐）。
- `SCRIPT FLUSH` 清除脚本缓存。
- `SCRIPT KILL` 终止正在运行的只读脚本。

**注意事项**：
- Lua 脚本执行期间**阻塞主线程**，应避免耗时过长。脚本内部不要做 O(N) 全量扫描。
- 脚本必须是**纯函数风格**：给定相同输入，在任意 Redis 节点上执行结果一致（尤其是 Cluster 模式下）。
- 脚本写死 key 传参，不要动态拼接 key，以支持 Cluster 的 slot 校验。
- Redis 7.0 引入 `FUNCTION` 替代 `EVAL`（通过 `FUNCTION LOAD` 加载），代码管理更规范，但核心原子执行逻辑不变。

### 9.3 Redis Stream：消息队列

Redis 5.0 引入 Stream，彻底补上了 Redis 做轻量级消息队列的短板。数据结构类似 Kafka 的 topic-partition。

**基础命令**：

```bash
# 添加消息（可自动生成 ID *）
XADD mystream * sensor 1001 temp 36.5
# → "1688123456789-0"（格式：毫秒时间-序号）

# 读取消息
XREAD COUNT 2 STREAMS mystream 0      # 从头读
XREAD BLOCK 5000 STREAMS mystream $    # 阻塞读取最新

# 范围查询
XRANGE mystream - + COUNT 10
```

**消费者组**（对标 Kafka Consumer Group）：

```bash
# 创建消费者组（从头部消费 $ 不消费历史）
XGROUP CREATE mystream mygroup 0 MKSTREAM
# 消费者组读取（消费后需手动 ACK）
XREADGROUP GROUP mygroup consumer1 COUNT 1 BLOCK 5000 STREAMS mystream >
# 确认消息
XACK mystream mygroup 1688123456789-0
# 查询未 ACK 的待处理消息（用于重试）
XPENDING mystream mygroup - + 10
```

**核心特点**：
- Stream 底层使用 **Rax Tree（基数树）** 索引消息 ID，支持 O(log N) 的范围查询和 ID 定位。
- 消息 ID 由 `毫秒时间戳-序列号` 构成，严格递增，保证有序性。
- 消费组支持**多消费者并行消费**（同一组的消费者分摊消息），且每个消费者的 ACK 独立跟踪。
- 消息**不会自动删除**，ACK 只是标记已处理，需要 `XTRIM / XDEL / MAXLEN` 手动裁剪。

**适用场景**：轻量异步解耦、事件溯源、日志收集。对于高吞吐（百万QPS）或严格顺序的场景，仍建议 Kafka/RocketMQ。

### 9.4 Pub/Sub：发布订阅

简单的发布订阅，**无持久化**，消息即发即忘。

```bash
# 订阅
SUBSCRIBE channel1 channel2
# 发布
PUBLISH channel1 "hello world"

# 模式订阅（通配符）
PSUBSCRIBE news.*
```

**注意事项**：
- 消息**不持久化**，订阅者离线期间的消息**不会**被补推。
- 无法保证消息顺序和送达（at-most-once 语义）。
- 生产环境若需可靠消息传递，请用 Stream 替代。
- 当 client 执行 `SUBSCRIBE` 后，连接进入 `pub/sub` 模式，不能再执行其他命令，需另开连接。

**Cluster 中的 Pub/Sub**：消息发布到任意节点后，通过集群总线广播到所有节点。每个节点再分发给本地订阅的客户端。这意味着集群中 Pub/Sub 是全量广播，有网络放大效应。

### 9.5 事务：WATCH / MULTI / EXEC

Redis 事务提供**非原子**的命令批量执行，与传统数据库事务概念不同。

```bash
WATCH inventory:1001      # 乐观锁：监控 key 变化
MULTI                      # 事务开始（命令入队）
DECRBY inventory:1001 1
INCRBY order:2001 1
EXEC                       # 执行（若 WATCH 的 key 被改动则返回 nil，全部命令不执行）
```

**关键理解**：
- **入队时报语法错**：`EXEC` 时不执行任何命令。
- **执行时报运行错**：正确命令依然执行，**不会回滚**（非原子性质！）。
- `WATCH` 实现 CAS 乐观锁：若被监控的 key 在执行 EXEC 前被其他客户端修改，事务自动取消。
- 一般在 `EXEC` 失败时进行重试（通常在应用层循环重试）。

```java
// Jedis 事务重试示例
public boolean transfer(String from, String to, int amount) {
    jedis.watch(from);
    int balance = Integer.parseInt(jedis.get(from));
    if (balance < amount) { jedis.unwatch(); return false; }
    
    Transaction tx = jedis.multi();
    tx.decrBy(from, amount);
    tx.incrBy(to, amount);
    List<Object> results = tx.exec();
    return results != null; // null 表示事务被取消，需重试
}
```

### 9.6 HyperLogLog、GEO、Bitmap

**HyperLogLog**：基数估计，极省内存。每个 key 固定使用 12KB 内存，统计 UV 误差仅 0.81%。

```bash
PFADD uv:page1 user1 user2 user3
PFCOUNT uv:page1    # 返回 3
PFMERGE uv:total uv:page1 uv:page2  # 合并多个 HLL
```

原理：对每个元素 hash 后的比特串，记录第一个 1 出现的位置的最大值 k，基数 ≈ 2^k。使用 **调和平均数** + **分桶（16384 个寄存器）** 降低方差，最终 12KB=16384×6bit。

**GEO**：地理位置索引，底层使用 ZSet 的 `geohash` 编码（52bit 整数做 score）。

```bash
GEOADD cities 116.397 39.908 "Beijing" 121.473 31.230 "Shanghai"
GEODIST cities Beijing Shanghai km    # 两城市距离
GEORADIUS cities 116 39 200 km       # 200km 半径内的城市
```

**Bitmap**：位图操作，适合签到、布隆、在线状态、权限等二值场景。

```bash
SETBIT sign:user:1001 100 1     # 第 100 天签到
BITCOUNT sign:user:1001         # 总签到天数
BITPOS sign:user:1001 0         # 第一个未签到日期
BITOP AND result sign:u1 sign:u2 # 位操作
```

### 9.7 Redis Search 与 Redis Stack

**Redis Search**（RediSearch）是一个基于 Redis 的全文搜索引擎，支持：全文索引、模糊匹配、聚合查询、自动补全。相当于内置了 Elasticsearch 的核心能力。

```bash
FT.CREATE idx:products ON hash PREFIX 1 product: SCHEMA name TEXT SORTABLE price NUMERIC SORTABLE
FT.SEARCH idx:products "@name:phone @price:[0 5000]"
FT.AGGREGATE idx:products "*" GROUPBY 1 @category REDUCE AVG 1 @price AS avg_price
```

**Redis Stack**：Redis 的增强套件，打包了 Redis Search、RedisJSON、RedisTimeSeries、RedisGraph、RedisBloom 等模块，一站式覆盖搜索、时序、图、JSON 等场景。

## 十、性能与内存

### 10.1 单线程模型 vs 多线程（Redis 6.0+）

**传统单线程模型**：Redis 核心命令处理（网络 IO + 命令解析 + 数据结构操作 + 响应）在单个线程中串行执行。线程模型是 Reactor 模式——主线程用 `epoll`（Linux）或 `IOCP`（Windows）等 IO 多路复用监听多个 socket 事件，事件就绪后同步处理。

**为什么单线程还能高性能**：
- 纯内存操作，CPU 通常不是瓶颈。
- 避免多线程的锁竞争和上下文切换开销。
- IO 多路复用 + 非阻塞 IO 充分利用网络带宽。

**Redis 6.0+ 多线程 IO**：命令执行仍保持单线程（保证原子性），但**网络数据的读写**引入 IO 线程池，利用多核提升大流量下的网络吞吐。

```
主线程:
  1. epoll_wait 获取就绪事件
  2. 将读事件分发到 IO 线程 → 多线程并行 read + 解析 → 加入待处理队列
  3. 主线程单线程执行所有命令
  4. 将写事件分发到 IO 线程 → 多线程并行 write 响应
```

配置项：
```conf
io-threads 4                # IO 线程数（建议 ≤ CPU 核数）
io-threads-do-reads yes     # 启用 IO 线程处理读（默认仅处理写）
```

**Redis 7.0 进一步优化**：AOF 写入使用后台线程 `bio_aof_fsync`，LZF 压缩使用专用线程，`DEL` 大对象（unlink）继续用异步线程。

### 10.2 大 Key 与热 Key

**大 Key 问题**指某个 Key 对应的 Value 过大（String >10MB，或集合元素超过数万个），引发一系列连锁问题：

- **阻塞风险**：`DEL` 大 key 时主线程阻塞数百毫秒，导致连接超时、连接池打满、CPU 飙升。
- **内存不均**：Cluster 模式下大 key 导致 slot 数据倾斜，个别节点 OOM。
- **网络带宽**：主从复制或迁移时，大 key 序列化传输占用大量带宽。
- **慢查询**：`HGETALL`、`LRANGE 0 -1` 等对大 key 操作极慢。

**大 Key 检测**：

```bash
# 使用 redis-cli 的 --bigkeys 选项（遍历所有 key，评估 value 大小）
redis-cli --bigkeys

# 使用 memory usage 命令精确查看单个 key 内存占用
MEMORY USAGE myBigHashKey

# 查看 Redis 内部各 key 大小
DEBUG OBJECT mykey  # 返回序列化长度
```

也可借助工具如 `redis-rdb-tools` 离线分析 RDB 文件，导出所有 key 的大小分布报告。

**大 Key 删除**：

- **Redis 4.0+**：使用 `UNLINK` 替代 `DEL`。`UNLINK` 在判断 key 为大型集合时，只做逻辑删除（从字典中移除 key），将内存回收交给后台 `bio` 线程异步执行，不阻塞主线程。
- **老版本**：分批"慢删"。对于 List 用 `LTRIM` 逐步裁剪；对于 ZSet 用 `ZREMRANGEBYRANK` 逐步删；对于 Hash 用 `HDEL` + `HSCAN` 分批；对于 Set 用 `SSCAN` + `SREM` 分批。

```bash
# Hash 大 key 分批删除示例（Lua 脚本）
local cursor = 0
repeat
    local result = redis.call('HSCAN', KEYS[1], cursor, 'COUNT', 100)
    cursor = tonumber(result[1])
    if #result[2] > 0 then
        redis.call('HDEL', KEYS[1], unpack(result[2]))
    end
until cursor == 0
```

**热 Key**：单个 key 被超高频率访问（如秒杀商品），热点集中在某节点，可能导致单节点 CPU 打满、带宽跑满。解决方案：
- **本地缓存**：在应用层用 Caffeine/Guava Cache 缓存热 key，减轻 Redis 压力。
- **读写分离**：增加从节点，读操作分摊到从节点。
- **Key 拆分（热 Key 多副本）**：将热 key 复制多份（如 `hotkey:1`、`hotkey:2`...），客户端随机选取，负载分散到不同 slot，分布到不同节点。
- **Cluster 模式下 slot 迁移**：将热 key 所在 slot 迁移到配置更强的节点。

### 10.3 内存淘汰策略

当 Redis 内存达到 `maxmemory` 时，根据配置的策略淘汰 key：

| 策略 | 作用域 | 淘汰规则 |
|------|--------|---------|
| noeviction | — | 不淘汰，写操作返回错误 |
| allkeys-lru | 全体 key | 淘汰 LRU 最不活跃者 |
| volatile-lru | 带 TTL 的 key | 淘汰 LRU 最不活跃者 |
| allkeys-lfu | 全体 key | 淘汰 LFU 访问频率最低者 |
| volatile-lfu | 带 TTL 的 key | 淘汰 LFU 访问频率最低者 |
| allkeys-random | 全体 key | 随机淘汰 |
| volatile-random | 带 TTL 的 key | 随机淘汰 |
| volatile-ttl | 带 TTL 的 key | 淘汰 TTL 剩余时间最短者 |

**LRU 实现**（近似 LRU）：Redis 不维护完整链表（开销太大），而是**随机采样 N 个 key**（`maxmemory-samples`，默认 5），淘汰其中 LRU 最旧的。采样数量越多，越接近真实 LRU，但 CPU 开销越大。

**LFU 实现**：24 位 `lru` 字段中：
- 高 16 位：最后访问时间（**分钟级**时间戳，精度低于 LRU 的秒级）。
- 低 8 位：对数访问计数（0~255），不是简单的累加而是**指数衰减 + 对数增长**。

LFU 计数器更新逻辑：
```
counter = 旧值, 旧值越大，增加概率越小（对数增长特征）
prob = 1.0 / (counter * server.lfu_log_factor + 1)
若随机数 < prob，counter++
同时，根据 idle_time，counter = counter - idle_time / server.lfu_decay_time

最终排序依据: 访问频率 = (counter * 255) / lfu_log_factor
```

**生产建议**：
- 纯缓存场景：`allkeys-lru`（最通用）。
- 数据有明确冷热之分：`allkeys-lfu`（适合大促等有明显访问偏好的场景）。
- 消息队列/排行榜等数据不可丢失：`noeviction`。

### 10.4 内存碎片整理

Redis 使用 `jemalloc`（可选 `libc`）管理内存。频繁的内存分配和释放会产生**外部碎片**——空闲内存总量充足但无连续大块可用，导致 RSS 远高于实际数据量。

**查看碎片率**：

```bash
INFO Memory
# mem_fragmentation_ratio = used_memory_rss / used_memory
# 正常范围：1.0 ~ 1.5；> 1.5 说明碎片严重；< 1.0 说明有 swap
```

**Redis 4.0+ 自动碎片整理**：

```conf
activedefrag yes                          # 启用自动碎片整理
active-defrag-ignore-bytes 100mb          # 碎片达到 100MB 开始整理
active-defrag-threshold-lower 10          # 碎片率 > 1.1 时开始
active-defrag-threshold-upper 100         # 碎片率 > 2.0 时全力整理
active-defrag-cycle-min 1                 # 整理占 CPU 的最小时间百分比
active-defrag-cycle-max 25                # 整理占 CPU 的最大时间百分比
active-defrag-max-scan-fields 1000        # 每次扫描的 field 上限
```

**原理**：利用 `jemalloc` 的 `madvise(MADV_DONTNEED)` 或内存迁移，将分散的空闲页归并成大块。整理过程在主线程中执行，通过 `active-defrag-cycle-min/max` 限制 CPU 占用，避免影响正常请求。对于 big key（如百万元素的 Hash），整理可能在单次迭代中耗时较长，此时需调大 `active-defrag-max-scan-fields` 并容忍短暂延迟。

### 10.5 慢查询日志

Redis 将执行时间超过 `slowlog-log-slower-than`（默认 10000 微秒 = 10ms）的命令记录到慢查询日志中。

```bash
# 配置
CONFIG SET slowlog-log-slower-than 10000   # 单位：微秒
CONFIG SET slowlog-max-len 128             # 最多记录条数

# 查看
SLOWLOG GET 10      # 最近 10 条慢查询
SLOWLOG LEN          # 当前日志条数
SLOWLOG RESET        # 清空日志
```

慢查询日志记录**不包括 IO 和网络时间**，仅计算命令在 Redis 内部的实际执行耗时。慢查询持久存储于内存，重启丢失，且每个慢查询会记录客户端 IP、命令、参数、执行时间。

**常见慢查询来源**：
- 大 Key 操作：`HGETALL`、`LRANGE 0 -1`、`SMEMBERS`、`ZRANGE 0 -1`。
- `KEYS *`：遍历所有 key（生产禁用，用 `SCAN` 替代）。
- `FLUSHALL` / `FLUSHDB`：清空所有数据。
- 复杂聚合命令：`SORT`、`SUNION` 对多个大集合。
- AOF 刷盘或 BGSAVE 期间的锁等待（不算在慢查询中，但影响性能）。

### 10.6 Pipeline 批处理

如前文 9.1 所述，Pipeline 通过合并网络 IO 提升吞吐，但另有性能细节：

- **批量大小**：建议每次 pipeline 500~5000 条命令，过大会增加内存压力，过小收益不明显。
- **集群 Pipeline**：需自行实现 slot 分组。Spring Data Redis 的 `JedisClusterConnection` 封装了自动分组逻辑。
- **Pipeline + 事务**：`MULTI/EXEC` 可以与 Pipeline 结合——MULTI 之后的所有命令入队，EXEC 时一次性执行并返回所有结果。但注意事务内不支持中间结果依赖。

```java
// Jedis 集群 Pipeline 按 slot 分组示例
Map<Integer, List<Jedis>> slotMap = ...; // 预先缓存 slot→JedisPool 映射
Map<Jedis, Pipeline> pipelineMap = new HashMap<>();
for (String key : keys) {
    int slot = JedisClusterCRC16.getSlot(key);
    Jedis jedis = slotMap.get(slot).get(0); // 取该 slot 主节点
    Pipeline pipeline = pipelineMap.computeIfAbsent(jedis, Jedis::pipelined);
    pipeline.get(key);
}
pipelineMap.values().forEach(Pipeline::sync);
```

### 10.7 性能调优检查清单

| 维度 | 检查项 | 优化措施 |
|------|--------|---------|
| 连接 | `info clients` 看连接数是否过高 | 连接池合理配置，启用 TCP keepalive |
| CPU | `info cpu` 看 sys/user CPU 占比 | 关闭 `keys *`/`flushall`，检查慢查询 |
| 内存 | `info memory` 看碎片率、是否达到 maxmemory | 设置合理 maxmemory，启用碎片整理 |
| 网络 | 监控带宽占用 | 调大 repl-backlog-size，减少大 key |
| 持久化 | `info persistence` 看 RDB 耗时 | 调整 save 参数，非高峰期触发 BGSAVE |
| 复制 | `info replication` 看 offset 差异 | 增大 backlog 或 client-output-buffer-limit |
| 磁盘 | 监控 `rdb_last_save_time` | 使用 SSD，关闭 `THP`（透明大页） |

---

*本文涵盖了 Redis 的核心数据结构、持久化、高可用架构、缓存策略、分布式锁、高级特性与性能调优，适合后端工程师系统化深入 Redis。具体配置参数以 Redis 7.0 官方文档为准。*
