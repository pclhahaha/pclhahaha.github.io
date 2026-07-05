---
title: LinkedHashMap 与 LRU 缓存
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - LinkedHashMap
  - LRU
  - accessOrder
categories:
  - Java
  - 集合
---

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
