---
title: Java 集合框架深度解析
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - Java
  - 集合
  - HashMap
  - ConcurrentHashMap
  - 源码分析
categories:
  - Java
---
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


---


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
