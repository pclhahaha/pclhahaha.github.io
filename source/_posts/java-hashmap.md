---
title: HashMap 深度解析 (JDK8)
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - HashMap
  - 哈希算法
  - 红黑树
  - 扩容
categories:
  - Java
  - 集合
---

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
