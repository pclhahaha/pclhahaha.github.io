---
title: Java 集合框架深度解析
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - Java
  - 集合
  - HashMap
  - ConcurrentHashMap
  - 源码分析
categories:
  - Java
---

[TOC]

## 一、集合框架概览

Java 集合框架从两个顶层接口出发：`Collection` 和 `Map`。

```
                    Iterable
                       |
                   Collection
              /       |       \
           List     Queue     Set
           /          \       /  \
     ArrayList      Deque   HashSet TreeSet
     LinkedList    /    \
     Vector   ArrayDeque  LinkedList
           /             \
  ArrayBlockingQueue  PriorityQueue
  LinkedBlockingQueue


                      Map
                 /     |      \
            HashMap  TreeMap  Hashtable
              |                  |
       LinkedHashMap     ConcurrentHashMap
```

### 1.1 List / Set / Queue / Map 实现类一览

| 实现类 | 底层结构 | 线程安全 | 关键特性 |
|--------|---------|---------|---------|
| `ArrayList` | Object[] | 否 | 随机 O(1), 扩容 1.5x |
| `LinkedList` | 双向链表 | 否 | 头尾 O(1), 随机 O(n) |
| `Vector` | Object[] | 是 (synchronized) | JDK 1.0 遗留 |
| `CopyOnWriteArrayList` | Object[] + 写时复制 | 是 (ReentrantLock) | 读无锁, 写 O(n) |
| `HashSet` | HashMap (PRESENT 占位) | 否 | O(1) 查删 |
| `TreeSet` | TreeMap | 否 | 红黑树有序 |
| `LinkedHashSet` | LinkedHashMap | 否 | 维护插入顺序 |
| `ArrayDeque` | 循环数组 | 否 | 栈/双端队列首选 |
| `PriorityQueue` | 二叉堆 (Object[]) | 否 | 按优先级出队 |
| `ArrayBlockingQueue` | Object[] + 单锁 | 是 | 有界阻塞 |
| `LinkedBlockingQueue` | 单向链表 + 双锁 | 是 | 可选边界, 默认无界 |
| `HashMap` | 数组+链表+红黑树(8+) | 否 | O(1) 平均 |
| `TreeMap` | 红黑树 | 否 | key 有序 |
| `Hashtable` | 数组+链表 | 是 (synchronized) | JDK 1.0 遗留 |
| `ConcurrentHashMap` | Node[]+CAS+synchronized(8) | 是 | 桶级锁, 读无锁 |

### 1.2 线程安全集合速览

```java
// 传统 — 全局锁
Map<K,V> m1 = new Hashtable<>();
Map<K,V> m2 = Collections.synchronizedMap(new HashMap<>());
List<E>  l2 = Collections.synchronizedList(new ArrayList<>());

// JUC — 高并发方案
Map<K,V>   m3 = new ConcurrentHashMap<>();          // CAS + synchronized(桶)
List<E>    l3 = new CopyOnWriteArrayList<>();        // 写时复制, 读无锁
Queue<E>   q1 = new ArrayBlockingQueue<>(100);       // 单锁 + Condition
Queue<E>   q2 = new LinkedBlockingQueue<>();         // 双锁 takeLock/putLock
Queue<E>   q3 = new ConcurrentLinkedQueue<>();       // 无锁 CAS
Set<E>    set = ConcurrentHashMap.newKeySet();
```

---

## 二、HashMap 深度解析

### 2.1 数据结构演进

```
JDK 7:  数组 + 单向链表
JDK 8:  数组 + 单向链表 + 红黑树

Bucket Array (Node[] table)
  index i  → [k1,v1] → [k2,v2] → null          // 链表 (长度<8)
  index j  →   ROOT                                  // 红黑树
               /   \
            LEFT  RIGHT  ...                     // TreeNode (长度≥8)
```

核心 Node 结构：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    public final int hashCode() { return Objects.hashCode(key) ^ Objects.hashCode(value); }
}
```

TreeNode 继承链：`Node` → `LinkedHashMap.Entry` → `TreeNode`, 同时具备红黑树指针和双向链表指针（用于树退化 `untreeify` 时 O(n) 遍历）：

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent, left, right, prev;   // 红黑树: parent/left/right; 链表: prev/next
    boolean red;
}
```

### 2.2 Hash 算法

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 扰动函数
}
// 定位桶: tab[(n - 1) & hash]   ←  当 n 是 2 的幂时等价于 hash % n
```

扰动函数将 hashCode 高 16 位与低 16 位异或，让高位差异参与低位索引计算，降低碰撞。

**为什么 n 必须是 2 的幂?**  `(n-1) & hash` 要求 n-1 低位全是 1:

```
n=16  → n-1 = 0b1111   → 4 位全有效
n=15  → n-1 = 0b1110   → 最低位永远 0, 一半索引永不使用
```

`tableSizeFor` 将任意整数向上取整到 2 的幂:

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;  n |= n >>> 2;  n |= n >>> 4;
    n |= n >>> 8;  n |= n >>> 16;             // 最高位1后的所有位全变1
    return (n < 0) ? 1 : (n >= 1<<30) ? 1<<30 : n + 1;
}
```

### 2.3 put 流程 (源码走读)

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // (1) table 为空 → 首次 resize 初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // (2) 桶为空 → 直接新建 Node
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

    else {
        Node<K,V> e; K k;
        // (3a) 头节点匹配 → 待覆盖
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // (3b) 桶内是红黑树 → putTreeVal
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // (3c) 链表 → 遍历查找/尾插
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1)  // binCount >= 7 → 可能树化
                        treeifyBin(tab, hash);               // 内部还检查 tab.length >= 64
                    break;
                }
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // (4) key 已存在 → 更新 value
        if (e != null) { V oldValue = e.value; e.value = value; return oldValue; }
    }
    // (5) size++ 后超过 threshold → 扩容
    if (++size > threshold) resize();
    return null;
}
```

关键阈值：

```java
static final int TREEIFY_THRESHOLD   = 8;     // 链表→树
static final int UNTREEIFY_THRESHOLD = 6;     // 树→链表 (7→8 留有缓冲, 避免频繁转化)
static final int MIN_TREEIFY_CAPACITY = 64;   // 树化前 table 至少 64 (否则只扩容)
```

### 2.4 扩容机制

#### JDK 7 — 头插法导致死循环

```java
// JDK 7 transfer() 伪代码
void transfer(Entry[] newTable) {
    for (Entry<K,V> e : table) {
        while (null != e) {
            Entry<K,V> next = e.next;  // ①
            int i = indexFor(e.hash, newTable.length);
            e.next = newTable[i];      // ② 头插: 新节点放链表头
            newTable[i] = e;           // ③
            e = next;                  // ④
        }
    }
}
```

**死循环场景**: 线程1 完成① (记录 next) 后挂起, 线程2 完成整个扩容 (链表被反转), 线程1 恢复后遍历已反转的链表, 最终形成 `A → B → A` 的环形引用。

**根因**: (1) 头插法导致链表反转; (2) 无同步保护。

#### JDK 8 — 尾插法 + 高低位拆分

```java
// 关键代码: 每个桶拆分为 lo / hi 两条链表, 保持原有顺序
for (int j = 0; j < oldCap; ++j) {
    Node<K,V> e; if ((e = oldTab[j]) != null) {
        // ...单节点/TreeNode 略...
        else {
            Node<K,V> loHead = null, loTail = null;   // 低位链表
            Node<K,V> hiHead = null, hiTail = null;   // 高位链表
            do {
                next = e.next;
                if ((e.hash & oldCap) == 0) {    // 新增位=0 → 索引不变
                    if (loTail == null) loHead = e; else loTail.next = e;
                    loTail = e;
                } else {                          // 新增位=1 → 索引 + oldCap
                    if (hiTail == null) hiHead = e; else hiTail.next = e;
                    hiTail = e;
                }
            } while ((e = next) != null);
            newTab[j] = loHead;
            newTab[j + oldCap] = hiHead;
        }
    }
}
```

**高低位拆分原理**:

```
扩容前 n=16 (2^4), 索引 = hash & (n-1) = hash & 0b1111  (用低 4 位)
扩容后 2n=32 (2^5), 索引 = hash & (2n-1) = hash & 0b11111 (用低 5 位)

新增的第 5 位由 (hash & oldCap) == 0 ? 判断:
  oldCap = 16 = 0b1_0000
  hash & 16 == 0  → 新增位=0 → 索引不变 (lo)
  hash & 16 != 0  → 新增位=1 → 索引 +16  (hi)

例: hash=5  → 5&15=5,  5&31=5,    hash&16=0 → lo
    hash=21 → 21&15=5, 21&31=21,  hash&16=16 → hi → newIdx = 5+16=21
```

尾插法保持原有顺序，从根本上杜绝环形引用。

### 2.5 红黑树操作

**treeifyBin — 不是无条件树化**:

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    // table 长度 < 64 → 只扩容不树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else {
        // (1) 先将链表转为 TreeNode 双向链表
        // (2) hd.treeify(tab): 基于 compareTo / identityHashCode 比较建红黑树
        //     通过 balanceInsertion(root, x) 保持平衡
    }
}
```

`balanceInsertion` 处理四种场景:

```
情况1: 父节点为黑 → 无需调整
情况2: 父红, 叔红    → 父叔变黑, 祖父变红, 向上递归
情况3: 父红, 叔黑, 当前节点是父的右子 → 左旋父, 转为情况4
情况4: 父红, 叔黑, 当前节点是父的左子 → 父变黑, 祖父变红, 右旋祖父
```

**退化**: resize 时 TreeNode 通过 `split` 拆分, 如果拆分后节点数 ≤ `UNTREEIFY_THRESHOLD`(6), 调用 `untreeify()` — 利用 TreeNode 的双向链表 (prev/next) 在 O(n) 时间内退化为普通链表。

### 2.6 设计中的数学

#### 为什么树化阈值是 8?

假设 hashCode 理想分布, 每个桶元素数服从 λ≈0.5 的泊松分布:

```
k=0: 0.60653066
k=1: 0.30326533
k=2: 0.07581633
k=3: 0.01263606
k=4: 0.00157952
k=5: 0.00015795
k=6: 0.00001316
k=7: 0.00000094
k=8: 0.00000006     ← 概率低于千万分之一
```

**结论**: 桶长度达 8 的概率不足千万分之一, 如果真出现, 几可认定为故意构造的哈希碰撞攻击. 此时 O(n) 链表退化为 O(log n) 红黑树, 防止性能降级。

#### 为什么负载因子 0.75?

| 负载因子 | 优点 | 缺点 |
|---------|------|------|
| 1.0 | 空间利用率最高 | 碰撞严重, 查询退化 |
| 0.5 | 碰撞极少 | 空间浪费 50%, 频繁扩容 |
| **0.75** | **时间/空间平衡** | — |

在 λ≈0.5 (负载 0.75 条件下平均每桶 0.5) 时，泊松分布显示 K=8 概率已极低 —— 数学验证的设计。

### 2.7 线程不安全的表现

| 场景 | 描述 |
|------|------|
| put 覆盖丢失 | 两线程同时 tab[i]=newNode, 后者覆盖前者 |
| resize 读到 null | 线程1 `oldTab[j]=null` 后, 线程2 get 该桶返回 null |
| JDK 7 死循环 | 头插法导致环, 多线程 resize 时 get 死循环 |
| size 不准确 | `++size` 非原子, 可能远超 threshold 而不扩容 |

---

## 三、ConcurrentHashMap 深度解析

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

## 四、ArrayList vs LinkedList

### 4.1 ArrayList 扩容

底层 `Object[] elementData`, 默认容量 10, 扩容 1.5 倍:

```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    elementData = Arrays.copyOf(elementData, newCapacity); // System.arraycopy(native)
}
```

`modCount` 用于 fail-fast: 迭代期间检测到 `modCount` 变化立即抛 `ConcurrentModificationException`。

### 4.2 LinkedList 结构

```java
private static class Node<E> { E item; Node<E> next; Node<E> prev; }
transient Node<E> first;
transient Node<E> last;
```

```
first                                              last
  ↓                                                  ↓
  [null|A|→]  ⇄  [←|B|→]  ⇄  [←|C|→]  ⇄  [←|D|null]
```

### 4.3 性能对比

| 操作 | ArrayList | LinkedList | 原因 |
|------|-----------|------------|------|
| `get(i)` | **O(1)** | O(n) | 数组寻址 vs 指针遍历 |
| `add(E)` | 均摊 O(1) | **O(1)** | 扩容均摊 vs 直接尾插 |
| `add(0,E)` | O(n) | **O(1)** | 后移元素 vs 改头指针 |
| `add(i,E)` | O(n) | **O(n)** | 元素移位 vs 遍历定位+改指针 |
| `remove(i)` | O(n) | **O(n)** | 元素移位 vs 遍历定位+改指针 |
| 遍历 | **O(n) 缓存友好** | O(n) 缓存差 | 连续内存 vs 随机指针 |
| 内存 (100万元素) | ~4 MB | ~40 MB | 纯数据 vs 每个 3 个引用 + 对象头 |

**LinkedList 随机插入也 O(n)** — 虽然 `add(index,E)` 定位后改指针是 O(1), 但 `node(index)` 遍历到目标位置已是 O(n), 定位开销主导。因此频繁中间插入/删除**两者都不适合**，应重新考虑数据结构。

### 4.4 选型指南

| 场景 | 推荐 |
|------|------|
| 频繁随机读取 | `ArrayList` |
| 频繁尾插入 | `ArrayList` (均摊 O(1), 内存省) |
| 频繁头插入/删除 | `LinkedList` 或 `ArrayDeque` |
| 栈 | `ArrayDeque` (比 LinkedList 快, 比 Stack 好) |
| 队列 | `ArrayDeque` (循环数组) |
| 大量删除 | `ArrayList` — 从后往前删, 避免重复移位 |

---

## 五、其他集合

### 5.1 LinkedHashMap — 维护顺序

继承 `HashMap`, 额外维护一条双向链表保持迭代顺序:

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;  // ← 新增: 双向链表指针
}
transient LinkedHashMap.Entry<K,V> head;
transient LinkedHashMap.Entry<K,V> tail;
```

```
HashMap 桶数组:                LinkedHashMap 双向链表:
  bucket[0] → {A} ⇄ {C}           head ⇄ A ⇄ B ⇄ C ⇄ tail
  bucket[1] → {B}
```

**accessOrder — LRU 缓存核心**:

```java
// accessOrder = false (默认): 迭代顺序 = 插入顺序
// accessOrder = true:  迭代顺序 = 访问顺序 (最近访问的移至末尾)

LinkedHashMap<K,V> cache = new LinkedHashMap<K,V>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > MAX;   // 最老元素自动淘汰
    }
};
```

当 `accessOrder=true` 时, `afterNodeAccess(e)` 将访问节点移到链表末尾 —— 最久未访问的自然停留在 head。

### 5.2 TreeMap / TreeSet

`TreeMap` 底层红黑树:

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key; V value; Entry<K,V> left, right, parent; boolean color = BLACK;
}
private final Comparator<? super K> comparator;  // 外部比较器
```

排序规则: `Comparator` (构造参数) > `Comparable` (key 自身实现) > 否则抛 `ClassCastException`。

`TreeSet` 内就是一个 `NavigableMap` (即 TreeMap), 所有操作委托给 Map, 用 `PRESENT = new Object()` 占位 value。

### 5.3 HashSet

```java
public class HashSet<E> extends AbstractSet<E> {
    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();
    public boolean add(E e)   { return map.put(e, PRESENT)==null; }
    public boolean remove(Object o) { return map.remove(o)==PRESENT; }
    public boolean contains(Object o) { return map.containsKey(o); }
    public int size()              { return map.size(); }
}
```

全部委托给 HashMap, value 统一用 `PRESENT` 占位。`LinkedHashSet` 同理, 内部是 LinkedHashMap。

### 5.4 CopyOnWriteArrayList

写时复制: 写操作不修改原数组, 而是 `Arrays.copyOf` 复制新数组后再替换引用:

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);  // volatile 写, 对所有读线程立即可见
        return true;
    } finally { lock.unlock(); }
}

public E get(int index) { return get(getArray(), index); } // 完全无锁
```

`COWIterator` 持有创建时的数组快照, 遍历期间不感知后续修改, 不抛 `ConcurrentModificationException`。

**适用**: 读多写少 (配置、白名单、监听器列表)。写操作 O(n) 需复制全数组, 数据量大时不适合。

### 5.5 ArrayBlockingQueue vs LinkedBlockingQueue

| 维度 | ArrayBlockingQueue | LinkedBlockingQueue |
|------|--------------------|---------------------|
| 底层 | 循环数组 (定长) | 单向链表 (可选上限) |
| 锁 | **单锁** (put+take 共用) | **双锁** (putLock + takeLock) |
| 并发度 | put 和 take 互斥 | **put 和 take 可并发** |
| 内存 | 预分配数组 | 每个元素新建 Node |
| 吞吐 | 较低 | 较高 (生产消费不互斥) |
| 默认容量 | 必须指定 | `Integer.MAX_VALUE` |

双锁原理: 入队操作尾部, 出队操作头部, 无数据结构冲突, 两把锁让生产者和消费者并行。

---

## 六、线程安全方案总结

### 6.1 各集合安全方案对比

| 集合 | 安全方案 | 粒度 | 特殊限制 |
|------|---------|------|---------|
| Hashtable | 全方法 synchronized | 全局 | 不使用 (已过时) |
| synchronizedMap/List | mutex 包装器 synchronized | 全局 | 迭代需手动加锁 |
| Vector | 全方法 synchronized | 全局 | 不使用 |
| ConcurrentHashMap (7) | Segment[] + ReentrantLock | Segment 级 (16) | Segment 数固定 |
| ConcurrentHashMap (8) | CAS + synchronized(桶头) | **桶级** | key/value 不能为 null |
| CopyOnWriteArrayList | volatile 数组 + ReentrantLock | 写锁, 读无锁 | 写 O(n) |
| CopyOnWriteArraySet | 委托 CopyOnWriteArrayList | 同上 | 同上 |
| ArrayBlockingQueue | ReentrantLock(单锁) + Condition | 单锁 | 定长 |
| LinkedBlockingQueue | ReentrantLock(双锁) + Condition | 双锁 | 默认无界 (OOM 风险) |
| ConcurrentLinkedQueue | CAS 无锁 (Michael-Scott) | 无锁 | — |

### 6.2 常见误区

1. **`synchronizedMap` 迭代不是线程安全的** — 必须手动同步:
   ```java
   Map m = Collections.synchronizedMap(new HashMap<>());
   synchronized (m) { for (Object o : m.keySet()) { } }
   ```

2. **`ConcurrentHashMap` 复合操作** — `get`+`put` 非原子:
   ```java
   // ❌ 可能覆盖其他线程刚 put 的值
   if (map.get(key) == null) map.put(key, val);
   // ✅ 正确做法
   map.computeIfAbsent(key, k -> computeValue());
   ```

3. **`size()` 不精确** — `ConcurrentHashMap.size()` 和 `mappingCount()` 都返回近似快照。

4. **`CopyOnWriteArrayList` 写密集型场景** — 百万级数据每次 add 复制整个数组, 延迟毫秒级。

5. **`HashMap` 并发 `put`+`get`** — JDK 8 虽解决死循环, 但 resize 期间元素正在迁移, get 仍可能返回不一致结果。

---

> **参考文献**:
> [^1]: Doug Lea, *Concurrent Programming in Java* (JSR-166 理论基础)
> [^2]: JDK HashMap 源码注释 (泊松分布概率表)
> [^3]: Brian Goetz, *Java Concurrency in Practice*, 第 5 章
> [^4]: OpenJDK `java/util/HashMap.java` / `java/util/concurrent/ConcurrentHashMap.java`
> [^5]: CLRS *Introduction to Algorithms*, 第 13 章 (红黑树)
