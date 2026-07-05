---
title: ConcurrentHashMap 深度解析
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - ConcurrentHashMap
  - CAS
  - 并发扩容
categories:
  - Java
  - 集合
---

### 3.1 JDK 7: Segment 分段锁

```
ConcurrentHashMap
  └── Segment[] (固定数量, 默认 16, 不支持扩容)
        ├── Segment<K,V> extends ReentrantLock
        │     ├── count (volatile)
        │     └── HashEntry<K,V>[] table (segment 内部小 HashMap)
        └── Segment<K,V> ...
```

每个 Segment 持有一把独立的 ReentrantLock, 写操作只需锁对应 Segment, 并发度 = segment 数量:

```java
public V put(K key, V value) {
    int j = (hash >>> segmentShift) & segmentMask;
    Segment<K,V> s = segments[j];
    return s.put(key, hash, value, false);   // Segment.put 内 tryLock() 或 lock()
}
```

**size() 妥协**: 先无锁尝试 3 次 (通过 modCount 检查并发修改), 失败后才锁全部 Segment 统计。

**缺陷**: Segment 数量固定, 粒度有限; 跨 Segment 操作需全锁。

### 3.2 JDK 8: CAS + synchronized — 为什么重构?

| 维度 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | Segment[] + HashEntry[] | Node[] + 链表 + 红黑树 |
| 锁粒度 | Segment 级 | **桶级** (synchronized 锁桶头节点) |
| 锁机制 | ReentrantLock | CAS(无冲突) + synchronized(有冲突) |
| 扩容 | Segment 独立扩容 | **多线程协同扩容** |
| 计数 | Segment.count | CounterCell[] + baseCount (LongAdder 方案) |

**为什么弃用 ReentrantLock?**

1. JDK 6 起 synchronized 做了偏向锁→轻量级→重量级的逐步升级, 低竞争场景性能接近 ReentrantLock
2. 锁粒度缩小到桶级后, 如果用 ReentrantLock, 最坏情况每个桶一个 AQS 队列, 内存不可接受
3. CAS 用于无锁插入空桶, synchronized 仅用于冲突时锁桶头, 两种机制互补

### 3.3 put 流程

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());   // (h ^ (h>>>16)) & HASH_BITS
    for (Node<K,V>[] tab = table;;) {    // ******* 自旋 CAS 循环
        Node<K,V> f; int n, i, fh;
        // (1) table 未初始化 → CAS 初始化 (sizeCtl=-1 表示初始化中)
        if (tab == null || (n = tab.length) == 0) tab = initTable();

        // (2) 桶为空 → CAS 插入 (无锁)
        else if ((f = tabAt(tab, i = (n-1)&hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash,key,value,null))) break;
        }
        // (3) ForwardingNode (hash==MOVED) → 当前在扩容, 协助扩容
        else if ((fh = f.hash) == MOVED) tab = helpTransfer(tab, f);

        // (4) 桶非空且不在扩容 → synchronized 锁桶头节点
        else {
            synchronized (f) {           // 只锁当前桶的头节点!
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {       // 链表遍历/尾插
                        for (Node<K,V> e = f;; ++binCount) { /* 查找 key, 尾插 */ }
                    } else if (f instanceof TreeBin) {  // 红黑树
                        ((TreeBin<K,V>)f).putTreeVal(hash, key, value);
                    }
                }
            }
            if (binCount >= TREEIFY_THRESHOLD) treeifyBin(tab, i);
        }
    }
    addCount(1L, binCount);    // 计数 +1, 可能触发扩容
}
```

**sizeCtl 状态机** — 一个字段控制所有状态:

```
sizeCtl =  0       : 默认初始容量
sizeCtl = 正数      : threshold (扩容阈值 = capacity * 0.75)
sizeCtl = -1       : 正在初始化 table
sizeCtl = -(1+n)   : 正在扩容, 低16位表示 (n-1) 个线程参与
                     例如 -3 → 2 个线程正在协助扩容
```

`initTable` 源码关键部分:

```java
while ((tab = table) == null || tab.length == 0) {
    if ((sc = sizeCtl) < 0) Thread.yield();     // 其他线程在初始化
    else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  // CAS 抢初始化权
        try {
            // ...创建 table, 设置 sizeCtl = n - (n>>>2) = 0.75n
        } finally { sizeCtl = sc; }
    }
}
```

### 3.4 多线程协同扩容 (transfer)

JDK 8 最大亮点 — 多个写线程一起参与扩容:

```java
// 核心: CAS 抢迁移任务区间 [transferIndex-stride, transferIndex)
int stride = Math.max((n >>> 3) / NCPU, MIN_TRANSFER_STRIDE); // 每线程步长最小 16

// transferIndex 从 n 开始递减 (从旧表末尾向前推进)
// 线程 CAS 抢区间:
if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,
         nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
    bound = nextBound;
    i = nextIndex - 1;   // 从区间末尾开始迁移
}
```

**扩容过程**:

```
旧 table (n=64), stride=16:
+-----------------------------------------------------------+
| 0..15 | 16..31 | 32..47 | 48..63 |
+-----------------------------------------------------------+
   ↑        ↑        ↑        ↑
  线程C    线程B    线程A    (transferIndex 从 64→0)

线程迁移完每个桶后放置 ForwardingNode (hash==MOVED):
  其他线程访问到此桶 → 调用 helpTransfer 协助扩容
  其他线程 get 此桶   → ForwardingNode.find() 去 nextTable 查找

所有桶迁移完成 → 替换 table = nextTable, sizeCtl = 1.5n
```

**线程退出机制**: 每个线程完成所分配区间后 CAS 递减 sizeCtl 中记录的参与线程数, 最后一个退出线程负责 `finishing` 收尾 (替换 table, 更新 sizeCtl)。

### 3.5 CounterCell 计数方案

基于 `Striped64` (LongAdder 模板):

```java
@sun.misc.Contended
static final class CounterCell { volatile long value; }

private transient volatile long baseCount;
private transient volatile CounterCell[] counterCells;

// size ≈ baseCount + sum(counterCells[i].value)
```

`addCount` 流程: 先 CAS `baseCount`, 失败则 CAS 更新某个 `CounterCell`, 多次失败后扩容 `counterCells` (2倍), 最终使用 `ThreadLocalRandom.getProbe()` 分散到不同 Cell。

`size()` 返回的是**近似快照**, 非精确值 — 统计期间可能有并发修改。

### 3.6 get 为什么不需要加锁?

```java
public V get(Object key) {
    int h = spread(key.hashCode());
    if ((e = tabAt(tab, (n - 1) & h)) != null) {
        if (e.hash == h && key.equals(e.key)) return e.val;     // 头匹配
        else if (e.hash < 0) return e.find(h, key).val; // TreeBin / ForwardingNode
        else { while ((e = e.next) != null) { /* 链表遍历 */ } }
    }
    return null;
}
```

无锁读的保证:

1. `Node.val` 和 `Node.next` 都是 `volatile` 修饰
2. `table` 数组 `volatile` 修饰, `tabAt` 通过 `U.getObjectVolatile` 读取保证每次读主存最新值
3. 扩容用尾插法, 已存在节点不被修改; `ForwardingNode.find()` 去 `nextTable` 查找
4. 迭代器是**弱一致性** (weakly consistent), 不抛 `ConcurrentModificationException`

### 3.7 线程安全 Map 对比

| 特性 | Hashtable | synchronizedMap | ConcurrentHashMap |
|------|-----------|-----------------|-------------------|
| 锁机制 | 全方法 synchronized | mutex 全表锁 | CAS + volatile + synchronized(桶级) |
| 读并发度 | 1 | 1 | 极高 (几乎无锁) |
| 写并发度 | 1 | 1 | 高 (桶级锁) |
| null 键/值 | 不支持 | 委托底层 Map | 不支持 |
| 迭代器 | fail-fast | fail-fast | 弱一致性 |
| 性能(高并发) | 极差 | 极差 | 极好 |

---
