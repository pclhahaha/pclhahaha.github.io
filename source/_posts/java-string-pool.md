---
title: 字符串常量池与 String.intern()
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - Java
  - 字符串常量池
  - String.intern
  - JVM
categories:
  - Java
  - JVM
---

## 一、概述

Java 中 `String` 是最常用的类之一。为了提高内存利用率和减少重复对象，JVM 设计了字符串常量池（String Pool），让相同的字符串字面量共享同一个对象。

## 二、字符串的三种创建方式

```java
// 方式 1：字面量 — 自动放入常量池
String s1 = "hello";          // 常量池中没有"hello" → 创建并放入
String s2 = "hello";          // 常量池中已存在 → 直接引用同一个对象
System.out.println(s1 == s2); // true，同一个对象

// 方式 2：new String() — 在堆中创建新对象，不自动放入常量池
String s3 = new String("hello");
System.out.println(s1 == s3); // false，堆中的新对象 ≠ 常量池中的对象

// 方式 3：String.intern() — 手动放入常量池
String s4 = new String("world").intern();  // 检查常量池是否存在"world"
                                           // 不存在 → 放入常量池
                                           // 存在 → 返回常量池中的引用
```

## 三、String.intern() 的原理

`intern()` 方法的行为是：检查字符串常量池中是否已经存在内容相同的字符串，若存在则直接返回常量池中的引用；若不存在，则**在 JDK 7 之前**将当前字符串对象拷贝一份放入常量池，**在 JDK 7 之后**则将当前堆中对象的引用放入常量池。

```java
// JDK 7+ intern() 行为
String s1 = new String("a") + new String("b"); // s1 指向堆中的 "ab"
String s2 = s1.intern();                        // 常量池中没有 "ab"
                                                // → 把 s1 的引用存入常量池
String s3 = "ab";                               // 字面量，去常量池找 "ab"
                                                // → 找到的是 s1 的引用
System.out.println(s1 == s3); // JDK 7+: true    JDK 6: false
```

## 四、常量池的存储位置演变

字符串常量池的物理存储位置在不同 JDK 版本中有过重大变化：

| JDK 版本 | 常量池位置 | 特点 |
|----------|-----------|------|
| JDK 6 及以前 | PermGen（永久代） | 大小固定（-XX:MaxPermSize），大量 intern 可能导致 OutOfMemoryError: PermGen space |
| JDK 7 | Heap（堆） | 移入堆中以解决 PermGen OOM 问题，可获得 GC 管理的好处 |
| JDK 8+ | Heap（堆）的 MetaSpace 之外 | PermGen 被 MetaSpace 取代，但 String Pool 仍在堆中 |

由于常量池存储在堆中（JDK 7+），它的容量受堆内存大小限制。如果大量调用 `intern()`，HotSpot 会自动调整常量池的哈希表大小以避免性能退化。

## 五、实际案例：使用 intern() 优化内存

假设一个服务需要处理大量的字符串，其中存在大量重复值：

```java
// 不使用 intern()：100万个 "ERROR" 字符串 = 100万个对象 ≈ 40MB
List<String> list1 = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list1.add(new String("ERROR"));  // 每次创建新对象
}

// 使用 intern()：100万个 "ERROR" 字符串 = 1个对象 ≈ 40 bytes
List<String> list2 = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list2.add(new String("ERROR").intern());  // 都指向常量池中同一个对象
}
```

适用于：大量重复的枚举值、协议头、日志级别等。不适合：每个字符串都是唯一的场景（intern() 查找会增加 CPU 开销且无内存收益）。

## 六、包装类的内部缓存

与 String Pool 类似，Java 的基本类型包装类也有内部缓存：

```java
Integer a = 127;    Integer b = 127;
System.out.println(a == b);  // true，[-128, 127] 的 Integer 被缓存

Integer c = 128;    Integer d = 128;
System.out.println(c == d);  // false，超出缓存范围，每次创建新对象

// Byte、Short、Long 也缓存 [-128, 127]
// Character 缓存 [0, 127]
// Boolean 缓存 TRUE 和 FALSE
```

这个缓存机制是 JVM 规范明确规定的一部分，其范围可以通过 JVM 参数调整（例如 `-XX:AutoBoxCacheMax=2000` 可以将 Integer 缓存上限扩展到 2000）。

## 七、小结

字符串常量池的核心价值在于"共享"——相同内容的字符串只存一份，节省内存。`intern()` 方法提供了手动控制这种共享的能力，配合 JDK 7 之后常量池移入堆的变更，使得 intern 成为大规模重复字符串场景下的有效优化工具。但需注意过多调用 `intern()` 会增加常量池的维护开销，应该在明确的内存优化场景下使用。
