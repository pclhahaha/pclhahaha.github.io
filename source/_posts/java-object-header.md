---
title: Java 对象头与锁升级
date: 2026-07-05
updated: 2026-07-05
tags:
  - JVM
  - 对象头
  - Mark Word
categories:
  - Java
  - JVM
---

先来看下在不同jvm下对象头的格式：

![32位jvm](/java/jvm/image-20191203131237845.png)

![64位jvm（未开启指针压缩）](/java/jvm/image-20191203131353286.png)

![64位jvm（开启指针压缩）](/java/jvm/image-20191203131439467.png)

以上3幅图分别是32位、64位（未开启指针压缩）、64位（开启指针压缩）jvm中对象头的定义。[^2]

可以看到一个对象头由Mark Word和Klass Word组成，其中Klass Word保存指向元数据的指针，而Mark Word分为多个组成部分，存储对象自身的运行时数据，具体如下：

- 哈希码（HashCode）
- GC分代年龄：用于在GC标记过程中存储对象的GC年龄
- 锁状态标志
- 持有锁的线程ID：处于偏向锁状态时，持有锁的线程ID
- 持有锁的线程栈中的锁记录：处于轻量级锁状态时，持有锁的线程的锁记录的指针
- 偏向时间戳