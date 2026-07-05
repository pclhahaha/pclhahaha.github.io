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


> 📖 独立文章：[堆 (Heap) 与 TopK](/heap-and-priority-queue/)

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


> InnoDB 索引的详细分析（包括页内结构、联合索引、索引优化策略）参见 [MySQL 深度解析](..\data\mysql.md)。


- **HBase**：基于 HDFS 的 LSM-Tree 变体


```


在实际的敏感词过滤系统中，一个包含 10 万个敏感词的 Trie，内存约占 10-50MB，可以实现在一篇 5000 字的文章中在几毫秒内找出所有敏感词。


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


> 📖 独立文章：[分治思想与外部归并排序](/external-merge-sort/)
### 8.3 递归与分治：MapReduce / GFS 的外排序

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
