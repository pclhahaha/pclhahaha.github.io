---
title: 数据结构与算法 -- 后端工程师视角
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 数据结构
  - 算法
  - 面试
  - 后端
categories:
  - CS基础
---
## 一、为什么后端工程师需要学数据结构与算法

大多数后端工程师对数据结构与算法的第一反应是"刷 LeetCode"。但面试只是副产品——真正重要的，是你每天用的每一个中间件、每一行代码，底层都是数据结构的精巧组合。

**几个让你无法反驳的场景：**

- 你写了一条 SQL `SELECT * FROM t WHERE id BETWEEN 1000 AND 2000`，MySQL 为什么能在几百毫秒内从几千万行数据中找到这 1000 行？因为 InnoDB 在磁盘上组织了一棵 **B+Tree**。
- 你在 Redis 里插入几十万条 `ZADD` 命令，为什么 `ZRANGEBYSCORE` 还能在毫秒级返回？因为 ZSet 底层是**跳表 (Skip List)**。
- 你给 Kafka 发消息，Producer 怎么决定发到哪个分区？可能是 `hash(key) % partition_count`，这涉及**哈希函数**和**分区策略**。
- ES 的搜索建议（"你搜的是不是..."）为什么能在你敲击键盘的瞬间给出联想？背后是 **Trie 树**和**有限状态转换器 (FST)**。
- JVM Full GC 为什么这么慢？你在优化 O(n²) 算法时和 GC 线程在竞争同一块**堆**结构的时间片。

**数据结构不是八股文，而是每个中间件作者的工程决策记录。** 这篇文章的目的不是带你刷题，而是让你把脑子里零散的 DSA 知识点，和你实际工作中使用的系统**一一对应**起来。

---

## 二、数组与链表

### 2.1 本质差异：连续内存 vs 链式内存

数组和链表是最基础的数据结构，但它们的差异不是"一个能随机访问、一个不能"这么简单。真正的差异在于**内存布局**，而内存布局决定了 CPU 缓存的友好程度。

```
数组 (连续内存):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ← 一块连续内存
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑                               ↑
  base_addr                   base_addr + 7*size

链表 (链式内存):
┌───┬───┐    ┌───┬───┐    ┌───┬───┐    ┌───┬───┐
│ 0 │ ●─┼───→│ 1 │ ●─┼───→│ 2 │ ●─┼───→│ 3 │ / │  ← 内存中散落各处
└───┴───┘    └───┴───┘    └───┴───┘    └───┴───┘
```

| 特性 | 数组 | 链表 |
|------|------|------|
| 内存布局 | 连续 | 离散 |
| 随机访问 | O(1) | O(n) |
| 插入/删除（中间） | O(n)（需要移动） | O(1)（改指针） |
| CPU L1/L2/L3 Cache 友好性 | 极高（空间局部性） | 极差（cache miss 频繁） |
| 内存开销 | 0（仅数据本身） | 每节点额外存指针（8/16 bytes） |

**CPU 缓存友好性详解：** 现代 CPU 每次从主存取数据时，会以 cache line（通常 64 bytes）为单位把整块数据加载到 L1/L2/L3 缓存中。数组元素连续排列，一次 cache miss 能一次性预取相邻的多个元素。链表节点散布在内存各处，每遍历一个节点都是**几乎必定的 cache miss**——在 L1 缓存命中约 1ns，主存访问约 100ns，这意味着**遍历链表比遍历数组慢两个数量级**。

这是为什么很多"理论上 O(1) 插入的链表"在实际基准测试中跑不过 ArrayList 的根本原因。

### 2.2 在 MySQL Page 中的应用

InnoDB 的磁盘管理以 **Page**（默认 16KB）为单位。每个 Page 内部是一块**连续的 16KB 内存**：

```
InnoDB Page 结构 (16KB 连续空间):
┌──────────────────────────────────────────────────────┐
│ File Header (38B) │ Page Header (56B) │ Infimum+Supremum │ User Records (自由组织) │ Free Space │ Page Directory │ File Trailer (8B) │
└──────────────────────────────────────────────────────┘
                    ↑ 连续内存块，大小固定 16KB
```

- **User Records** 区域：行数据按插入顺序紧挨着存储，本质上是一个**物理紧凑的数组**。每个 Record 通过链表指针（next_record，2 bytes 偏移量）串联成逻辑链表。这就是"物理数组 + 逻辑链表"的混合结构。
- **Page Directory**：每个 Page 尾部有槽 (Slots)，每个槽指向一组记录中最大的那条记录，构成**稀疏索引**。二分查找页内数据时，先二分定位 Slot，再在 Slot 所指向的小范围内顺序扫描——这是"数组二分 + 链表顺序"的组合技。

> 详细 Page 结构参见 [MySQL 深度解析](..\data\mysql.md)。

### 2.3 在 Redis quicklist 中的应用

Redis 3.2 之前的 List 底层使用 `linkedlist` 或 `ziplist`，Redis 3.2+ 统一为 **quicklist**：

```
quicklist 结构:
┌────────┬────────┬────────┬────────┐
│ node_0 │ node_1 │ node_2 │ node_3 │  ← 双向链表 (每个 node 是一个 quicklistNode)
└───┬────┴───┬────┴───┬────┴───┬────┘
    │        │        │        │
    ▼        ▼        ▼        ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│ziplist ││ziplist ││ziplist ││ziplist │  ← 每个 node 内是一段压缩数组 (ziplist/listpack)
│ [a,b,c]││ [d,e,f]││ [g,h,i]││ [j,k,l]│
└────────┘└────────┘└────────┘└────────┘
```

**设计哲学：** 每个 ziplist（或 listpack）是一段连续内存，内部元素连续存储，具有数组的 cache 友好性。多个 ziplist 通过链表串联，解决了单个连续块无限膨胀的问题。`list-max-ziplist-size` 控制每个 ziplist 最多多少元素或多大字节。

这是典型的**混合结构**——在空间局部性和插入灵活性之间取平衡。插入中间元素时，只需改目标 ziplist 内部的元素移动，而不需要像纯数组一样移动所有后续元素。

### 2.4 在 Java 集合中的应用

**ArrayList 扩容：**

```java
// ArrayList 扩容核心逻辑 (JDK 8+)
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5 倍扩容
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

扩容涉及 `Arrays.copyOf` → `System.arraycopy` —— 这是一个 native 方法，调用 `memmove`/`memcpy` 进行内存块的整块拷贝，CPU 能利用 SIMD 指令并行拷贝，速度极快。但扩容时所有元素都要搬运，所以不要在构造 ArrayList 时不传初始容量、然后插入几百万条数据——扩容 30 多次，每次都搬运全部元素，时间复杂度实际上是 O(n²)。

**LinkedList 的最佳使用场景：** 如果你确认需要在 List 头部频繁插入/删除（如实现一个 FIFO 队列），LinkedList 的 `addFirst`/`removeFirst` 是 O(1)。但绝大多数场景下 ArrayList 更快——哪怕是中间插入，`System.arraycopy` 搬运内存的速度也能抵消遍历链表的 cache miss 开销。

---

## 三、栈与队列

### 3.1 栈：后进先出

#### JVM 操作数栈与帧栈

每个 JVM 线程都对应一个 **Java 虚拟机栈 (Java Virtual Machine Stack)**，每个方法调用对应一个 **栈帧 (Stack Frame)**：

```
JVM 线程栈:
┌──────────────────────────────┐  ← 高地址
│    栈帧: method_A()          │
│    ┌─────────────────────┐   │
│    │ 局部变量表 (int a=3) │   │
│    │ 操作数栈            │   │  ← 字节码指令的运算空间
│    │ 动态链接            │   │
│    │ 返回地址            │   │
│    └─────────────────────┘   │
├──────────────────────────────┤
│    栈帧: method_B()          │  ← method_A 内部调用了 method_B
├──────────────────────────────┤
│    栈帧: method_C()          │  ← method_B 内部调用了 method_C
├──────────────────────────────┤
│    ......                    │  ← 低地址
└──────────────────────────────┘
```

**操作数栈 (Operand Stack)** 是字节码指令执行的空间。比如 `int a = 3 + 5`，字节码是：`iconst_3`（压栈）→ `iconst_5`（压栈）→ `iadd`（弹出两个、相加、结果压栈）→ `istore_1`（弹出结果存入局部变量表）。整个计算过程就是在操作数栈上做 push/pop。

**方法调用栈** 也是栈的经典用途——当异常发生时，JVM 打印的 `stacktrace` 其实就是在遍历当前线程的方法调用栈。递归过深导致的 `StackOverflowError`，就是线程栈空间不足以容纳新的栈帧。

#### 括号匹配问题 — 后端无处不在

```java
// 一个实际的括号匹配检查 —— 这在 RPC 框架的序列化/反序列化代码中频繁出现
// 例如：校验 JSON 格式合法性、校验 SQL 中括号配对、校验表达式语法
Deque<Character> stack = new ArrayDeque<>();
for (char c : s.toCharArray()) {
    if (c == '(' || c == '[' || c == '{') {
        stack.push(c);
    } else {
        if (stack.isEmpty()) return false;
        char top = stack.pop();
        if (!matches(top, c)) return false;
    }
}
return stack.isEmpty();
```

这不是面试题——你写的任何 DSL 解析器（如 Spring EL 表达式、MyBatis 动态 SQL 的 `<if>`/`<foreach>` 标签解析）、JSON/Protobuf 的编解码器，底层都大量使用栈做嵌套结构的校验。

### 3.2 队列：先进先出

#### Kafka 分区队列

Kafka 的每个 Partition 在物理上是一个**不可变的追加日志**——本质上是一个只追加的数组队列：

```
Kafka Partition:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│M_0 │M_1 │M_2 │M_3 │M_4 │M_5 │M_6 │M_7 │M_8 │M_9 │  ← 分段存储 (LogSegment)
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  ↑                         ↑
  offset=0              offset=9

Consumer Group 消费模型:
Producer ──append──→ [Partition] ──consume──→ Consumer  (FIFO 队列语义)
```

- **Producer 端**：追加写入（append-only），相当于 `enqueue`
- **Consumer 端**：顺序消费，相当于 `dequeue`
- **Offset**：消费游标，记录消费者"读到了队列的哪个位置"

Kafka 高性能的关键之一就是这种 **顺序追加** 的数组式队列——磁盘的顺序写速度可达 600MB/s 以上（机械硬盘），远高于随机写。这比链表式的消息队列（随机写磁盘）快一到两个数量级。

#### 线程池的工作队列

Java `ThreadPoolExecutor` 使用 `BlockingQueue` 存储待执行任务：

| 实现类 | 数据结构 | 特点 | 适用场景 |
|--------|----------|------|----------|
| `ArrayBlockingQueue` | 有界循环数组 | 固定大小，锁竞争集中在同一把锁 | 需要限制队列长度的场景 |
| `LinkedBlockingQueue` | 单向链表 | 可选有界/无界，两把锁（putLock/takeLock） | 吞吐量要求高的场景 |
| `SynchronousQueue` | 无容量 | 不存储任务，直接移交 | 希望任务立即被执行，不排队 |
| `DelayedWorkQueue` | 堆（小顶堆） | 按延迟时间排序 | `ScheduledThreadPoolExecutor` |

`ArrayBlockingQueue` 的底层实现是一个**循环数组**——两个指针 `putIndex` 和 `takeIndex` 在数组上循环移动：

```
ArrayBlockingQueue 循环数组 (capacity = 8):
         takeIndex=2          putIndex=5
            ↓                    ↓
┌───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │ C │ D │ E │   │   │   │  ← 有 3 个元素在队列中
└───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7
```

### 3.3 优先队列与堆

优先队列不是 FIFO——每次取出的都是**优先级最高**的元素，而"优先级最高"由比较器定义。底层通常是**堆 (Heap)** 实现。

**定时任务调度：** Java `ScheduledThreadPoolExecutor` 内部使用 `DelayedWorkQueue`（小顶堆实现），堆顶永远是最近需要执行的任务。每次 `take()` 时检查堆顶任务的到期时间：
- 如果已到期 → 立即执行
- 如果未到期 → `awaitNanos(delay)` 等待

但是存在一个问题：如果在等待期间插入了一个更早的任务怎么办？新插入的任务触发 `siftUp`，如果它比当前等待的堆顶更早，会调用 `leader` 线程的 `interrupt()` 重新排队。

**时间轮 (HashedWheelTimer)：** Netty 使用时间轮实现高性能定时器，解决大量定时任务场景下堆的 O(log n) 插入复杂度问题。时间轮把时间分成多个格子（bucket），每个格子是一个任务链表，指针按 tick 转动。插入 O(1)，但精度受 tick 间隔限制。

**Dijkstra 最短路径：** 用优先队列（最小堆）维护"当前已知最短距离的节点"，每次弹出最小距离节点进行松弛操作——这是在链路追踪、路由选择中的核心算法，详见第 7 章。

---

## 四、哈希表

### 4.1 哈希函数设计：扰动函数

哈希函数的核心目标：**把任意输入均匀、低碰撞地映射到一个整数区间**。

**Java HashMap 的哈希函数：**

```java
// JDK 8 HashMap.hash()
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这行 `(h = key.hashCode()) ^ (h >>> 16)` 是经典的**扰动函数**——把高 16 位和低 16 位做异或，让高位的变化也能影响低位。为什么需要扰动？

因为 HashMap 通过 `(n - 1) & hash` 求桶的下标（n 是数组长度，通常是 2 的幂）。如果 n 较小（如默认 16），`n-1` 只有低 4 位有效（`15 = 0b1111`），高位信息在取模运算中被截断了。扰动函数让高位参与低位运算，减少低位相同的碰撞概率。

```
不扰动：      hash = 0b11010010_01010101_10010001_01100110
              只有低 16 位参与 ───────────────────────── ↑↑↑↑
扰动后：      hash = 0b11010010_01010101 XOR 10010001_01100110
              高位 → 低位的变化被"搅拌"进来了
```

**Redis dict 的哈希函数：** Redis 使用 MurmurHash2（Redis 4.0 改为 SipHash）——选择 SipHash 的安全考虑是防止 Hash DoS 攻击（攻击者构造大量哈希碰撞的 key 使 CPU 飙升）。

### 4.2 冲突解决：链表法 vs 开放寻址

#### 链表法 (Separate Chaining) — Java HashMap

```
HashMap 结构 (数组 + 链表):
bucket[0]  → [K1|V1|●] → [K5|V5|●] → [K9|V9|/]   ← 链表
bucket[1]  → null
bucket[2]  → [K2|V2|/]
bucket[3]  → [K7|V7|●] → [K3|V3|/]
...
bucket[15] → null
```

每个桶 (bucket) 是一个链表头节点。命中同一个桶的 key 串成链表。JDK 8 的优化：**当链表长度 ≥ 8 且数组长度 ≥ 64 时，链表树化为红黑树**，将最坏 O(n) 的查找优化到 O(log n)。（详见第 5 章红黑树）

#### 开放寻址 (Open Addressing) — ThreadLocalMap

Java `Thread` 每个线程内部有一个 `ThreadLocalMap`，它使用**开放寻址**而非链表法：

```java
// ThreadLocalMap 的 set 逻辑：
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int i = key.threadLocalHashCode & (len - 1);
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) { ... }  // 找到，更新
    }
    tab[i] = new Entry(key, value);  // 找到空位，插入
}
```

碰撞后线性探测下一个位置 `i+1` 直到找到空位。为什么 ThreadLocalMap 选择开放寻址而非链表法？
1. **ThreadLocal 对象数量通常很少**（一般几个到几十个），数据规模小，开放寻址缓存更友好
2. **弱引用 Entry 需要清理**——线性探测天然支持在 set/get 时顺便清理已回收的 key（`expungeStaleEntries`）

### 4.3 Redis dict 与渐进式 Rehash

Redis 的核心数据结构 `dict` 在以下场景广为使用：全局 key 空间、Hash 类型底层、Set 类型（小集合用 ziplist，大集合用 dict）。

```c
// Redis dict 结构 (redis/src/dict.h)
typedef struct dict {
    dictType *type;
    dictEntry **ht_table[2];    // 两个哈希表：ht[0] 正常使用，ht[1] 用于 rehash
    unsigned long ht_used[2];
    long rehashidx;             // -1 表示无 rehash 进行中，≥0 表示正在 rehash
    int iterators;
} dict;
```

**渐进式 Rehash：** Redis 是单线程的，如果一次性把 `ht[0]` 所有 bucket 拷贝到 `ht[1]`，大 key 场景下会长时间阻塞。Redis 的做法是分步搬迁：

1. 创建 `ht[1]`（容量为 `ht[0]` 的 2 倍）
2. 设置 `rehashidx = 0`
3. 每次对 dict 的**增删改查**操作时，顺手把 `rehashidx` 指向的那个 bucket 从 `ht[0]` 搬迁到 `ht[1]`，然后 `rehashidx++`
4. 所有 bucket 搬迁完毕，`ht[1]` 变成新的 `ht[0]`，销毁旧的

在渐进式 rehash 期间，新增操作只写到 `ht[1]`，查询/删除/更新需要同时查两个表。

两个触发 Rehash 的条件（负载因子 = `used / size`）：
- **扩容**：负载因子 ≥ 1 且无 `BGSAVE`/`BGREWRITEAOF` 子进程；或负载因子 > 5（此时不管有没有子进程，强制扩容）
- **缩容**：负载因子 < 0.1

> 更详细的 Redis 内部结构分析参见 [Redis 深度解析](..\data\redis.md)。

### 4.4 MySQL 哈希索引

MySQL Memory 存储引擎支持 **哈希索引**——但 InnoDB 不支持用户创建的哈希索引，只有**自适应哈希索引 (Adaptive Hash Index)**：

```
自适应哈希索引 (AHI):
B+Tree 索引                              AHI 内存镜像
┌──────────────┐                         ┌──────────┐
│ [10] [20]    │    频繁等值查询触发      │ hash(10) │ → 直接定位到 B+Tree 叶子页
│  │    │   │  │ ──────────────────────→ │ hash(20) │ → 直接定位到 B+Tree 叶子页
│  │    │   └──┤                         │ hash(35) │ → ...
│ ...  ... ... │                         └──────────┘
└──────────────┘
```

AHI 是 InnoDB 内部自动维护的，无法手动干预。它的作用是：对频繁访问的索引页建立哈希映射，绕过 B+Tree 的 O(log n) 查找，实现 O(1) 等值查询。但只对等值查询有效，范围查询仍走 B+Tree。

### 4.5 一致性哈希

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

> 一致性哈希在 Redis Cluster 和客户端分区（如 Jedis Sharded）中的实现细节，参见 [分布式系统](distributed.md)。

---

## 五、树

### 5.1 二叉搜索树与退化问题

普通的二叉搜索树 (BST) 在数据有序插入时会**退化成链表**：

```
插入顺序: 1, 2, 3, 4, 5

退化 BST:              期望的平衡 BST:
    1                       3
     \                     / \
      2                   2   4
       \                 / \   \
        3               1   *   5
         \
          4
           \
            5
```

退化后的 BST 查找复杂度从 O(log n) 退化为 O(n)。这是所有平衡树（红黑树、AVL、B-Tree）要解决的核心问题：**在插入和删除时通过额外操作维持树的平衡**。

### 5.2 B-Tree / B+Tree：MySQL InnoDB 的基石

这是后端工程师最需要深入理解的数据结构——没有之一。InnoDB 的索引组织、查询性能、磁盘 I/O 模型全都建立在一棵 B+Tree 上。

#### 为什么是 B+Tree 而不是 BST/红黑树？

答案关键词：**磁盘 I/O**。

数据库索引是存在磁盘上的，数据量远超内存容量。一次磁盘寻道约 5-10ms，一次内存访问约 100ns——差距是 5 万到 10 万倍。对于数据库来说，"算法复杂度 O(log n)"不是用 CPU 执行步数衡量的，而是用**磁盘 I/O 次数**衡量的。

B+Tree 的核心设计目标：**矮胖**——让树的高度尽可能低，从而减少磁盘 I/O 次数。

```
B+Tree 结构 (InnoDB 典型: m=3 阶, 实际可达 1000+ 阶):

                     [30 | 60]              ← 非叶子节点 (存索引键 + 子节点指针)
                   /     |     \
          [10|20]      [40|50]      [70|80]  ← 非叶子节点
          /  |  \      /  |  \      /  |  \
        [1-9][10-19][20-29][30-39][40-49][50-59][60-69][70-79][80-89]  ← 叶子节点
        ←→    ←→    ←→    ←→    ←→    ←→    ←→    ←→    ←→
                    所有叶子节点通过双向链表串联 ←→
```

**B+Tree 选择矩形而非三角的原因 —— 扇出 (Fanout)：**

每个 InnoDB Page 是 16KB。假设主键为 8 bytes 的 BIGINT，指针为 6 bytes（页号），一个索引记录 14 bytes。一个 16KB 的 Page 可以存约 1170 个索引项（实际受 Page Header 等开销影响，约 1000 个）。

这就是扇出 (fanout) ≈ 1000——每个非叶子节点可以指向约 1000 个子节点。

- **高度 h=1**（只有根和叶子）：可容纳约 1,000 条记录
- **高度 h=2**：可容纳约 1,000 × 1,000 = 1,000,000 条记录
- **高度 h=3**：可容纳约 1,000³ = 1,000,000,000 条记录

一行数据 1KB 时，叶子节点每个 Page 存 16 条记录。3 层 B+Tree 可以支撑 (1000² × 16) ≈ 1600万行——查一条数据最多只需要 3 次磁盘 I/O。如果要存几十亿行，树高最多 4 层。

作为对比：如果是一棵二叉树（BST/红黑树），即使完全平衡，1 亿行数据约 2^27 需要约 27 层——也就是最多 27 次磁盘 I/O，比 B+Tree 的 3-4 次差了近 10 倍。

**这就是 InnoDB 选 B+Tree 最根本的原因——极致的磁盘 I/O 优化。**

#### B-Tree vs B+Tree 的关键区别

| 特性 | B-Tree | B+Tree (InnoDB 实际使用) |
|------|--------|--------------------------|
| 数据存储位置 | 所有节点都存数据 | **仅叶子节点存数据** |
| 非叶子节点 | 索引 + 数据 | **仅索引（占空间极小）** |
| 叶子节点之间 | 没有链接 | **双向链表串联** |
| 范围查询 | 需要回溯（中序遍历） | 叶子链表顺序扫描即可 O(k) |
| 扇出 | 较小（因为存数据） | **极大（因为只存索引）** |

B+Tree 把数据集中在叶子节点，非叶子节点只存键值 + 指针，同样的 16KB Page 能放下更多索引项 → 更高的扇出 → 更矮的树 → 更少的 I/O。

#### 聚簇索引 (Clustered Index) 组织

InnoDB 的主键索引就是聚簇索引——数据行直接存储在 B+Tree 的叶子节点上，而非叶子的 Value 就是一个 Page 指针。这与 MyISAM 的"数据文件独立、索引存数据地址"完全不同：

```
InnoDB 聚簇索引:                          MyISAM 非聚簇索引:
B+Tree 叶子节点                            B+Tree 叶子节点
┌─────────────┐                         ┌─────────────┐
│ PK=100      │                         │ PK=100      │
│ col1='Alice'│  ← 整行数据在这里！        │ 数据地址 #7 │ ───→ [数据文件 #7]
│ col2='NY'   │                         └─────────────┘
│ ...         │
└─────────────┘
```

聚簇索引的优势：主键查询一次 B+Tree 就能拿到完整行数据。劣势：二级索引的叶子节点存的是主键值（而不是数据地址），所以通过二级索引查询需要**回表**——先查二级索引拿到主键，再查聚簇索引拿到行数据。这就是为什么建议用覆盖索引避免回表。

> InnoDB 索引的详细分析（包括页内结构、联合索引、索引优化策略）参见 [MySQL 深度解析](..\data\mysql.md)。

### 5.3 LSM-Tree：LevelDB / RocksDB / Cassandra 的存储引擎

B+Tree 擅长点查和范围查，但在**写入密集型**场景下有痛点：
- 每次写入需要多次随机 I/O（更新 B+Tree 页、页分裂、维护索引）
- 页分裂时需要移动大量数据
- 机械磁盘随机写性能极差（~100 IOPS）

LSM-Tree 的核心思想：**把随机写变成顺序写**。

```
LSM-Tree 核心架构:
                    ┌──────────────────────┐
                    │    Write              │
                    │     ↓                 │
                    │  [WAL (预写日志)]      │  ← 崩溃恢复
                    │     ↓                 │
                    │  [MemTable (内存树)]   │  ← 一般是 SkipList/红黑树
                    │     ↓ (满了 flush)    │
                    └──────────────────────┘
                              │
────────── 磁盘分层的 SSTable ──────────
      ┌─────────────────────────┐
      │  Level 0: [SSTable] [SSTable] [SSTable] │  ← 各 SSTable 之间可能有重叠
      │  Level 1: [SSTable────SSTable────SSTable] │  ← 同一层内 key 不重叠
      │  Level 2: [──────────────────────]       │
      │  ...                                    │
      │  Level N: [──────────────────────]       │
      └─────────────────────────┘
```

**三层结构：**

1. **MemTable（内存表）**：接收写入请求，数据在内存中排序（跳表/红黑树实现），同时为崩溃恢复先写 WAL（Write-Ahead Log）。写操作只是 Append WAL + 插入内存树，速度极快。
2. **SSTable（Sorted String Table）**：MemTable 满了之后刷到磁盘成为 SSTable——一个不可变的、按键排序的文件。Level 0 的 SSTable 可能相互重叠（因为是多个 MemTable 直接刷下来的），但 Level 1+ 的 SSTable 在同一层内是不重叠的。
3. **Compaction（合并压缩）**：后台线程不断将低层 SSTable 与高层 SSTable 合并，消除冗余数据（同 key 的旧版本、已删除的标记），保持查询性能。

**写放大 vs 读放大 vs 空间放大：**

| 放大类型 | B+Tree | LSM-Tree |
|----------|--------|----------|
| 写放大 | ~1.1x（就地更新） | 10x-30x（WAL + 多次 Compaction 重复写入） |
| 读放大 | 1（直接定位） | 10x+（需要查 MemTable + 多层 SSTable + BloomFilter） |
| 空间放大 | ~1.1x | ~1.1x（Compaction 及时清理可保持很低） |

LSM-Tree 适合写多读少的场景（时序数据、日志、消息队列），因为写入极快但读取需要访问多层 SSTable。RocksDB 通过 BloomFilter 在每个 SSTable 上做快速排除来降低读放大。

**实际产品映射：**
- **LevelDB**：Google 开源的 LSM-Tree 实现，MemTable 用跳表
- **RocksDB**：Facebook 基于 LevelDB 的增强版，MySQL MyRocks 存储引擎的底层
- **Cassandra**：LSM-Tree + 去中心化分布，MemTable + SSTable + Compaction
- **HBase**：基于 HDFS 的 LSM-Tree 变体

### 5.4 红黑树

红黑树是一种**自平衡二叉搜索树**，通过 5 条性质保证树"近似平衡"——最长路径不超过最短路径的 2 倍，确保查找/插入/删除都是 O(log n)。

**5 条性质：**
1. 节点是红色或黑色
2. 根节点是黑色
3. 叶子节点 (NIL) 是黑色
4. 红色节点的两个子节点必须是黑色（不能两红相邻）
5. 从任意节点到其叶子节点的所有路径上，黑色节点数相同（黑高相等）

```
红黑树示例:
                    [● 10]          ← 黑
                   /       \
              [● 5]        [● 20]   ← 黑
             /    \        /    \
           nil    nil  [R 15] [R 25] ← 红
                      /  \   /  \
                    nil nil nil nil
```

**自平衡三连：左旋 + 右旋 + 变色。** 旋转：儿子变爹，爹变儿子。插入时默认插入红色节点，然后根据叔叔节点 (uncle) 的颜色分三种情况处理：
- 叔叔红 → 父黑、叔黑、爷红 → 递归处理爷爷
- 叔叔黑 + 当前是父的右子 → 绕父左旋 → 变成第三种情况
- 叔叔黑 + 当前是父的左子 → 父黑、爷红 → 绕爷右旋

**在 Java 中的应用：**
- **HashMap**：链表长度 ≥ 8 且数组 ≥ 64 时树化为红黑树 (`final void treeifyBin(Node<K,V>[] tab, int hash)`)
- **TreeMap**：底层就是红黑树，`put`、`get`、`remove` 都是 O(log n)
- **TreeSet**：基于 TreeMap 实现

**为什么 HashMap 选红黑树而不是 AVL？** AVL 树更严格（平衡因子 ≤ 1），查找更快但对插入删除的旋转次数更多。HashMap 的树化场景下插入频率高，红黑树的"松散平衡"更合适。

### 5.5 跳表 (Skip List)：Redis ZSet 底层

跳表是多层的链表——每层的节点指向同层的下一个节点，上层节点跨越多层，实现"跳跃"查找。

```
跳表结构 (Redis ZSet 实现):
Level 3:  [head] ───────────────────────────→ [30] ──────────→ NULL
Level 2:  [head] ──────────→ [15] ──────────→ [30] ──────────→ NULL
Level 1:  [head] ──→ [5] ──→ [15] ──→ [21] ──→ [30] ──→ [42] → NULL
Level 0:  [head]→[2]→[5]→[12]→[15]→[18]→[21]→[27]→[30]→[35]→[42]→NULL
                          ↑ 底层包含全部元素
```

**查找过程 (找 key=21)：** 从最高层 Level 3 开始 → 30 > 21，退到 Level 2 → 15 < 21 → 跳到 15 → 30 > 21，退到 Level 1 → 从 15 走到 21 → 命中！整个过程跳过了 Level 0 的多个节点，复杂度 O(log n)。

**时间复杂度分析：**
- 查找：O(log n)（概率期望，与红黑树相同）
- 插入：O(log n)（随机生成层数，期望 O(log n)）
- 删除：O(log n)
- 空间复杂度：O(n)（每层节点的期望比例为 1/2，总节点数 ≈ 2n）

**Redis ZSet 为什么选跳表而不是红黑树？**

| 对比维度 | 跳表 | 红黑树 |
|----------|------|--------|
| 实现复杂度 | 简单（300-400 行 C） | 复杂（旋转/变色逻辑繁琐） |
| 范围查询 | O(log n + k)，天然支持链表遍历 | 需要中序遍历（递归/栈） |
| 并发支持 | 容易实现无锁（每层独立的链表） | 几乎不可能无锁（旋转影响多个节点） |
| 内存占用 | 略大（多指针） | 较小（两个子节点指针 + 颜色） |
| 查询性能 | 稍慢（概率均衡，常数因子更大） | 稍快（确定性平衡） |

Redis 的 ZSet 需要频繁做 `ZRANGEBYSCORE` 范围查询——跳表的底层 Level 0 就是一条完整的有序链表，直接遍历即可，而红黑树需要中序遍历做栈或递归。此外，Redis 是单线程的，不需要锁，但跳表的简单性对于 C 代码的可维护性有天然优势。

```c
// Redis 跳表节点 (redis/src/server.h)
typedef struct zskiplistNode {
    sds ele;                       // 成员 (sds 字符串)
    double score;                  // 分值（排序依据）
    struct zskiplistNode *backward; // 后退指针（仅 Level 0 使用）
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;        // 跨度（用于计算 rank）
    } level[];                     // 柔性数组
} zskiplistNode;
```

### 5.6 Trie 字典树：ES 搜索建议与敏感词过滤

Trie（字典树/前缀树）按字符串的每个字符逐层展开，共享公共前缀，适合前缀匹配场景。

```
Trie 示例 (存储 "abc", "abd", "bce", "bcf"):
                    [root]
                   /      \
                 a        b
                /          \
               b            c
              / \          / \
             c   d        e   f
            (*)  (*)     (*)  (*)    ← * 表示是一个完整单词的结束
```

**ES Completion Suggester 中的 Trie：**

Elasticsearch 的 `completion` 类型底层使用 Trie（实际上是 FST，见下一节）实现搜索建议。用户在搜索框输入 "spr"，ES 快速返回 "spring", "spring boot", "spring cloud" 等建议。

**敏感词过滤：**

Trie 的一个经典应用是 Aho-Corasick (AC) 自动机——基于 Trie + 失配指针实现多模式串同时匹配。常用于：
- 内容审核系统（敏感词过滤）
- 入侵检测系统（Snort 的规则匹配）
- 搜索引擎的 Query 理解

在实际的敏感词过滤系统中，一个包含 10 万个敏感词的 Trie，内存约占 10-50MB，可以实现在一篇 5000 字的文章中在几毫秒内找出所有敏感词。

### 5.7 FST (有限状态转换器)：ES 倒排索引的 Term Index

**FST 是 Elasticsearch/Lucene 引擎的基石数据结构。** 需要先理解为什么有它：

```
ES 数据流向:
原始文档
  ↓ 分词 (Analyzer)
Term 字典 (所有不重复的词)  ← 可能数百万甚至数亿个
  ↓ 每个 Term → 倒排列表 (Posting List, 哪些文档包含这个词)
倒排索引
```

**问题：** 数亿个 Term，怎么快速找到某个词对应的倒排列表？最直接的办法是哈希表 (O(1))，但哈希表没法前缀查询。B-Tree 可以，但字符比较的存储效率低。

**FST 的解法：** FST 本质上是一个**有向无环图 (Minimal Acyclic DFA)**，将共享前缀和后缀的字符串压缩到极致。

```
普通 Trie 存储 "mon", "tues", "thurs":
        root
       / | \
      m  t  t            (多个 t 重复)
      |  |  |
      o  u  h
      |  |  |
      n  e  u
         |  |
         s  r
            |
            s

FST 存储 (共享前缀 + 后缀):
    [root]
      ├── m → o → n [$]
      └── t ─┬─ u → e → s [$]
             └─ h → u → r → s [$]
```

FST 共享了 `t` 前缀和 `s` 后缀，而且每个节点上可以挂载数值（如 Term 在倒排文件中的偏移量），既是**字典**又是**映射**。

**Lucene 的 BlockTree Terms Index：**

```
Lucene 内部术语索引:
                    [Term Index (FST)]
                           │
                    ┌──────┼──────┐
                    │      │      │
            term_a ─┘ term_m ─┘ term_z ─┘
                    │      │      │
                    ▼      ▼      ▼
              [Term Dictionary (BlockTree - 有序前缀块)]
                    │      │      │
                    ▼      ▼      ▼
            [Posting Lists (跳表/FOR编码/RBM位图)]
```

- **Term Index**：用 FST 存储所有 Term 的前缀，快速定位到 Term Dictionary 对应的 block
- **Term Dictionary**：用 BlockTree 存储，每块包含一些按字典序排序的 Term，块内二分查找
- **Posting List**：倒排列表，存储文档 ID 列表，用 FOR 编码/RBM 位图压缩

FST 极大地压缩了 Term Index 的内存占用——通常可以把一个 10 亿 Term 的索引前缀压缩到几 GB 甚至几百 MB，**全部加载到内存中**，实现微秒级的 Term 定位。

---

## 六、堆 (Heap)

### 6.1 定义与实现

堆是一棵**完全二叉树**，且满足堆序性质：每个节点的值 ≥ (大顶堆) 或 ≤ (小顶堆) 其子节点的值。

```
小顶堆 (数组实现):
           [2]        ← arr[0]
         /     \
      [5]       [7]   ← arr[1], arr[2]
     /   \     /
   [8]  [11] [10]     ← arr[3], arr[4], arr[5]

数组表示: [2, 5, 7, 8, 11, 10, ...]
父 → 子: left=2*i+1, right=2*i+2
子 → 父: parent=(i-1)/2
```

堆的底层用**数组**实现，利用完全二叉树的性质避免了指针开销。

**核心操作：**
- `siftUp`（上滤）：新元素插入数组末尾，不断与父节点比较并上移，O(log n)
- `siftDown`（下滤）：弹出堆顶（arr[0]）后，把数组最后一个元素移到堆顶，不断与较小的子节点比较并下移，O(log n)
- `heapify`：对无序数组从最后一个非叶子节点往前逐个 siftDown，O(n)

**Java PriorityQueue 的 heapify 时间复杂度是 O(n)，不是 O(n log n)。** 这个经常被考到——因为越靠近叶子节点的元素下沉距离越短：
- 底层（约 n/2 个元素）下沉 0 层 → n/2 × 0 = 0
- 往上每层元素减半、下沉距离 +1
- 总比较次数 ∑(i·n/2^i) → 上限 2n → O(n)

### 6.2 TopK 问题

维护一个大小为 K 的**小顶堆**，遍历全量数据：

```
TopK 流式处理 (K=3):
数据流: 5, 9, 2, 15, 8, 1, 20, ...

[2]
 |      遍历过程：堆大小 < K 时直接插入；堆大小 = K 时，新元素与堆顶比较
[5]     如果 > 堆顶，弹出堆顶、插入新元素；如果 ≤ 堆顶，丢弃
 |
[9]     ← 当前堆内是 Top-3 候选
```

时间复杂度 O(n log K)，空间复杂度 O(K)。对比全量排序 O(n log n) → 当 n 特别大而 K 很小时，这个优化是数量级的差距。

**后端实际场景：**
- 排行榜服务（游戏积分 Top-100）
- 热门搜索词 Top-N（实时流式统计）
- 日志分析（TopK 错误信息、调用量最高的 API）
- 监控系统（CPU 占用 Top-10 的进程）

### 6.3 Java GC Finalizer 优先级队列

在 Java 8 及之前，`java.lang.ref.Finalizer` 使用一个单独的后台线程处理需要执行 `finalize()` 方法的对象。这些对象被放到一个优先级队列中，GC 按照优先级处理。虽然 Java 9+ 已经废弃了 `finalize()`，但堆作为优先级调度器的思想仍然广泛使用。

### 6.4 时间轮 (Timing Wheel) 与堆的对比

传统定时任务用堆 (PriorityQueue)，每次取出最近任务，复杂度 O(log n)。在连接数极高的场景（如 Netty 管理百万级连接的超时检测），堆的 log n 插入成为瓶颈。

**时间轮**把时间维度映射到数组桶：

```
时间轮 (tick = 1s, wheelSize = 8):
          ┌───┐
          │ 0 │ → [taskA] → [taskB]  ← 第 0 个 bucket
          ├───┤
          │ 1 │ → [taskC]
   ┌──────┼───┼──────┐
   │      │ 2 │      │ ← 指针 (cursor) 在每个 tick 前进一格
   │      ├───┤      │
   │ 7    │ 3 │      │
   │      ├───┤      │
   │      │ 4 │      │
   └──────┼───┼──────┘
          │ 5 │ → [taskD]
          ├───┤
          │ 6 │ → [taskE] → [taskF]
          └───┘
```

- **插入**：O(1)——直接挂到对应 bucket 的链表
- **删除**：O(1)——从链表移除
- **触发**：指针走到 bucket 时，取出所有该 bucket 的任务并执行

**多层时间轮**处理跨度很大的延迟——比如 3600 秒后的任务，放在"小时轮"里，等快到时间了再下沉到"分钟轮"。Netty 的 `HashedWheelTimer` 和 Kafka 的 `TimingWheel` 都采用类似设计。

---

## 七、图

### 7.1 图在后端系统中无处不在

图数据结构在后端系统中的几个经典应用：

```
微服务依赖图:                  消息流转图:
[A] → [B] → [D]              Producer → [Partition_0] → Consumer_1
  ↘   ↗                         ↘      [Partition_1] ↗
   [C] → [E]                        → [Partition_2]

网络拓扑图:                   Maven/Gradle 构建依赖图:
[DB] ── [Cache]
  │  \    /                    [common] ← [module-a]
[API] ── [MQ] ── [Worker]         ↑     ↘
                            [module-b] [module-c]
```

### 7.2 拓扑排序：依赖解析

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

### 7.3 Dijkstra 最短路径：链路追踪与路由

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

### 7.4 最小生成树

**场景：数据中心网络布线优化**——给定 N 台服务器和它们之间的距离，如何用最少的网线把所有服务器连起来？这就是最小生成树问题。

Kruskal 算法：按边权重从小到大加边，如果不形成环就加入（用并查集判环），直到包含所有节点。

**并查集 (Union-Find)** 是 MST 的关键组件：

```java
class UnionFind {
    int[] parent, rank;
    int find(int x) { return parent[x] == x ? x : (parent[x] = find(parent[x])); } // 路径压缩
    void union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return;
        if (rank[ra] < rank[rb]) parent[ra] = rb;  // 按秩合并
        else { parent[rb] = ra; if (rank[ra] == rank[rb]) rank[ra]++; }
    }
}
```

---

## 八、经典算法

### 8.1 排序算法：JDK Arrays.sort 中的工程智慧

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

### 8.2 二分查找及变体

```java
// 查找到确切的值（或返回插入位置）
int idx = Arrays.binarySearch(arr, key);
if (idx >= 0) { /* 找到了 */ }
else { int insertPos = -(idx + 1); }  // 没找到，取反减一得到应插入位置
```

**四个经典变体（后端面试高频）：**

1. **查找第一个等于 target 的位置**（左边界）：log 搜索时 `arr[mid] >= target` 时 `right = mid - 1`，记录 `ans = mid`
2. **查找最后一个等于 target 的位置**（右边界）：`arr[mid] <= target` 时 `left = mid + 1`，记录 `ans = mid`
3. **查找第一个 ≥ target 的位置**（lower_bound）：`arr[mid] >= target` 时往左缩
4. **查找第一个 > target 的位置**（upper_bound）：`arr[mid] > target` 时往左缩

**实际场景：** MySQL 页内二分查找记录、Java `TreeMap.ceilingEntry()` / `floorEntry()`、Redis ZSet 的 `ZRANGEBYSCORE` 需要先二分找到起始位置再顺序遍历。

### 8.3 递归与分治：MapReduce / GFS 的外排序

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

### 8.4 滑动窗口：TCP 流控到分布式限流

**TCP 滑动窗口 (流控)：**

```
TCP 发送窗口:
                  ← 已发送已确认 → ← 已发送未确认 → ← 未发送可发送 → ← 不可发送 →
                  |_______________|_________________|_________________|_____________|
                  ↑               ↑                 ↑                 ↑
                 left          SND.UNA           SND.NXT       SND.UNA + snd_wnd
```

TCP 的流量控制核心就是滑动窗口——接收方通过 ACK 告诉发送方 `rwnd` (接收窗口大小)，发送方控制未确认的字节数不超过这个窗口。

**分布式限流滑动窗口：**

固定窗口计数器最大的问题是**边界突刺**——09:59:59 和 10:00:01 两秒内可能发送两倍限流量的请求。滑动窗口解决这个问题：

```
滑动窗口限流 (窗口 = 60s, 粒度 = 6 格每 10s):
        ┌───┬───┬───┬───┬───┬───┐
  now → │g0 │g1 │g2 │g3 │g4 │g5 │  ← 每格记录该 time slot 内的请求数
        └───┴───┴───┴───┴───┴───┘
        ↑ 窗口覆盖范围 (过去 60 秒)

当前窗口计数 = g0+g1+g2+g3+g4+g5
下一时刻: 指针前进 → 覆盖范围右移 → 最左边格清零 → 新格开始计数
```

实现可以用数组 + 指针循环移动，也可以用 Redis `ZSET`（score 为时间戳，`ZREMRANGEBYSCORE` 删除过期元素，`ZCARD` 获取窗口内计数）。Sentinel/Gateway 的限流模块广泛使用滑动窗口算法。

### 8.5 位图 (BitMap) 与布隆过滤器

**位图 (BitMap)：** 用一个 bit 表示一个元素是否出现。

```
用户签到 BitMap (31 天):
     1   2   3   4   5   6   7   8   ...   31
   ┌───┬───┬───┬───┬───┬───┬───┬───┬─────┬───┐
   │ 1 │ 1 │ 0 │ 1 │ 1 │ 0 │ 0 │ 1 │ ... │ 1 │   ← 1 = 已签到
   └───┴───┴───┴───┴───┴───┴───┴───┴─────┴───┘

统计签到天数: BITCOUNT key
查看某日是否签到: GETBIT key offset
```

Redis BitMap 的命令（`SETBIT`/`GETBIT`/`BITCOUNT`/`BITOP`）底层用 String 类型的 SDS 存储，每个 char 型可以存 8 个 bit。一年 365 天只需要 46 字节 per user，100 万用户约 44MB——比用 `SET` 存签到日期（每个日期一个元素、每个元素几十 byte）节省几十倍。

**布隆过滤器 (Bloom Filter)：** BitMap 的加强版——用 K 个哈希函数映射到同一个位数组上，判断"一定不存在" vs "可能存在"。

```
布隆过滤器原理:
插入 "user_123":
  hash1("user_123") % m = 3  →  bit[3] = 1
  hash2("user_123") % m = 9  →  bit[9] = 1
  hash3("user_123") % m = 14 →  bit[14] = 1

查询 "user_456":
  hash1("user_456") % m = 3  →  bit[3] = 1 ✓
  hash2("user_456") % m = 7  →  bit[7] = 0 ✗ → "一定不存在"
```

它的核心用途是**缓存穿透防护**——大量查询不存在的 key 打到数据库，布隆过滤器可以精准拦截 99%+。

> 布隆过滤器的完整数学推导、Java 实现、Redis Bloom 模块、Counting Bloom Filter、布谷鸟过滤器等变种，详见 [布隆过滤器](..\data\bloomfilter.md)。

**HyperLogLog：** 解决"基数统计"问题（UV 去重计数）。12KB 内存可以统计 2^64 个不同元素，误差约 0.81%。原理基于伯努利试验的概率模型 + 调和平均 + 分桶。

> HyperLogLog 的数学推导和应用参见 [Redis 深度解析](..\data\redis.md)。

---

## 九、算法在中间件中的映射速查表

| 数据结构 / 算法 | 产品 / 中间件 | 具体用途 |
|-----------------|--------------|----------|
| **B+Tree** | MySQL InnoDB | 聚簇索引、二级索引、页内组织 |
| **B+Tree** | PostgreSQL | 默认索引类型（与 MySQL 稍有差异） |
| **B+Tree** | MongoDB (WiredTiger) | 默认存储引擎的索引结构 |
| **LSM-Tree** | RocksDB, LevelDB | 存储引擎（LSM + SSTable + Compaction） |
| **LSM-Tree** | Cassandra, HBase | 写入优化的分布式存储引擎 |
| **哈希表 (dict)** | Redis | 全局 key-value Dict、Hash 类型、Set 类型 |
| **哈希表 (渐进式 rehash)** | Redis | 单线程下的无阻塞扩容 |
| **跳表 (Skip List)** | Redis ZSet | 有序集合（score 排序 + 范围查询） |
| **跳表** | LevelDB MemTable | 内存表的有序数据结构 |
| **快速列表 (quicklist)** | Redis List | 链表 + ziplist 混合结构 |
| **压缩数组 (ziplist/listpack)** | Redis | 小数据量时的紧凑存储 |
| **FST** | Elasticsearch/Lucene | Term Index（术语索引前缀压缩） |
| **Trie 树** | ES Completion Suggester | 搜索建议/自动补全 |
| **Trie + AC 自动机** | 内容审核系统 | 多模式敏感词匹配 |
| **倒排索引 + 跳表** | Elasticsearch/Lucene | 倒排列表合并 (Posting List Intersection) |
| **位图 (BitMap)** | Redis BitMap | 签到统计、在线状态、权限标记 |
| **布隆过滤器** | RedisBloom, Guava BloomFilter | 缓存穿透防护、去重 |
| **布隆过滤器** | HBase, Cassandra | LSM-Tree 读取加速（SSTable 预过滤） |
| **布隆过滤器** | LevelDB, RocksDB | SSTable 查询预排除 |
| **HyperLogLog** | Redis HyperLogLog | UV 统计、基数估算 |
| **一致性哈希** | Redis Cluster, Dubbo | 数据/请求分布式路由 |
| **一致性哈希** | CDN, Nginx upstream | 缓存服务器选路 |
| **时间轮** | Netty HashedWheelTimer | 百万级连接超时管理 |
| **时间轮** | Kafka | 延迟操作、请求超时 |
| **小顶堆** | ScheduledThreadPoolExecutor | 定时任务调度 |
| **小顶堆** | Java GC Finalizer | 对象终结队列 |
| **红黑树** | Java HashMap, TreeMap, TreeSet | 哈希桶树化、有序映射 |
| **红黑树** | Linux CFS 调度器 | 进程的 vruntime 排序 |
| **红黑树** | Nginx 定时器 | 事件超时管理 |
| **滑动窗口** | TCP 协议 | 流量控制 |
| **滑动窗口** | Sentinel, Gateway | 分布式限流 |
| **滑动窗口** | Kafka Consumer | 流式窗口聚合 |
| **Dijkstra / 最短路径** | 服务网格, API 网关 | 延迟最优路由 |
| **拓扑排序** | Maven, Gradle, AirFlow | DAG 依赖解析 |
| **二分查找** | MySQL Page Directory | B+Tree 页内记录快速定位 |
| **归并排序 (外排)** | MapReduce, Spark | 分布式数据排序 |
| **哈希扰动函数** | Java HashMap | 减少低位碰撞 |
| **开放寻址** | ThreadLocalMap, Netty FastThreadLocal | 小规模哈希表 |

---

## 十、生产实践

### 10.1 为什么"避免 O(n²)"在后端极其重要

O(n²) 不是面试题里的理论——它在生产环境中可以直接把系统打挂：

**场景 1：嵌套循环 Join**

```sql
-- 这种写法在业务扩展后是灾难
SELECT * FROM orders o, order_items i WHERE o.id = i.order_id;
-- 如果 orders 10万行，order_items 100万行，Nested Loop Join = 10万×100万 = 1000亿次比较
```

正确的做法是：建立适当索引 → MySQL 使用 Index Nested Loop / Hash Join (8.0) → O(n log n) 或 O(n)。数据库优化器很聪明，但前提是 DBA 提供了好索引。

**场景 2：批量插入中的 O(n²) 遍历**

```java
// 坏代码：每插一条过滤一遍
List<Long> blacklist = loadBlacklist(); // 1万条
for (User u : batchUsers) {             // 10万条
    if (blacklist.contains(u.id)) {     // contains 在 ArrayList 里是 O(n)
        continue;
    }
    insert(u);
}
// 实际复杂度：10万 × 1万 = 10亿次比较

// 好代码：用 Set
Set<Long> blacklist = new HashSet<>(loadBlacklist());
for (User u : batchUsers) {
    if (blacklist.contains(u.id)) {     // O(1)
        continue;
    }
    insert(u);
}
```

在日常代码评审中，这类"数据结构选错导致 O(n²)"的问题非常常见。

### 10.2 算法复杂度与 QPS 的关系

每个请求处理时间的常数因子和算法复杂度的叠加，直接影响你能支撑的 QPS：

```
假设：单服务器 CPU 时间 = 100%，每个请求占用 t 秒
QPS ≈ 1/t

如果 t = 1ms  → QPS ≈ 1000
如果 t = 10μs → QPS ≈ 100,000
```

**实际案例：**
- 一个 O(n) 的遍历请求，n=1000 时 t=100μs → 10,000 QPS 没问题
- 同一个接口，n 增长到 100,000 时 t=10ms → 只剩下 100 QPS
- 如果算法不小心写成了 O(n²)，n=100,000 时 t 直接爆炸

所以在设计 API 时，预估数据量的增长对算法复杂度的放大效应，是后端工程师的基本素养。

### 10.3 算法优化三原则

**原则 1：时空权衡 (Space-Time Tradeoff)**

```java
// 用空间换时间：
// 不用每次都重新计算，提前存起来
Map<Long, User> cache = new HashMap<>();  // O(1) 查找
// vs
for (User u : users) { if (u.id == id) ... }  // O(n) 查找
```

Redis 缓存、布隆过滤器、倒排索引——本质上都是"用空间换时间"。

**原则 2：缓存友好 (Cache-Friendly)**

数据结构的理论复杂度和实际跑分可能天差地别——差别就是 CPU 缓存。同一份数据：
- 数组遍历：元素连续存储，每个 cache line (64B) 命中多个元素，L1/L2/L3 缓存层层加速
- 链表遍历：每个节点可能在不同的 cache line（甚至不同的 page），几乎每步都是 cache miss

**以 100 万个整数为例**：ArrayList 遍历约 0.5ms，LinkedList 遍历约 8ms——16 倍差距，不是因为复杂度不同（都是 O(n)），而是内存布局决定了一切。

> CPU 缓存的详细分析参见 [操作系统](os.md) 内存管理章节。

**原则 3：分治思想 (Divide and Conquer)**

- **MapReduce** 的 Map → Shuffle → Reduce 是分治思想的分布式实现
- **数据库分库分表** 是按 shard key 把数据分治到 N 个库
- **LSM-Tree 的 Compaction** 是分治——把多个小 SSTable 合并成大的
- **Kafka 的 Partition** 是把一个 Topic 的数据分治到多个分区，并行处理
- **Redis Cluster 的分片** 是把 key 空间分治到 16384 个 slot，再分给不同节点

遇到大规模问题时，先想：能不能把问题分解？能不能并行处理每个子问题？能不能在子问题层面做局部优化？

---

## 总结

数据结构与算法从来不是脱离工程的纯理论。从一行 `HashMap.get()` 到底层的链表→红黑树树化，从一个 `ZADD` 命令到底层的跳表 + dict 双结构，从一个 `SELECT` 到底层的 B+Tree 3 层索引——每一个你每天调用的 API 背后，都是前辈工程师在数据结构和算法之间做的精确取舍。

理解它们不是为了面试，而是为了看懂你用的每一个系统。当你真正理解了 InnoDB 为什么选 B+Tree、Redis 为什么选跳表、ES 为什么选 FST、LSM-Tree 为什么用顺序写换随机写——你就从一个"会用中间件"的工程师，变成了一个"理解中间件为什么这样设计"的工程师。这两者之间的差距，就是高级工程师与初中级工程师之间的分水岭。
