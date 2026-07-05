---
title: AQS 原理与应用
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - AQS
  - ReentrantLock
  - Semaphore
  - CountDownLatch
categories:
  - Java
  - 并发
---

### 1. AQS原理
<img src="/java/concurrency/aqsuml.png" alt="aqs" style="zoom: 50%;" />

上面在讲ReentrantLock等的过程中说到它是基于AQS实现的。AQS提供了一个框架，该框架可用于实现基于FIFO队列的阻塞锁或相关同步器(semaphores, events, etc)。由于只是提供了一个框架，其子类需要提供具体实现，一般来说子类应该被定义为non-public的内部辅助类（例如ReentrantLock类内部的Sync类，如上面类图所示），用于实现其外部类的同步性质。AQS框架支持独占模式和共享模式，供具体实现来选择。下面列出其子类需要具体实现的方法列表。

| 需要子类实现的方法 | 作用                    |
| ------------------ | ----------------------- |
| tryAcquire         | 尝试在互斥模式下acquire |
| tryRelease         | 尝试在互斥模式下release |
| tryAcquireShared   | 尝试在共享模式下acquire |
| tryReleaseShared   | 尝试在共享模式下acquire |
| isHeldExclusively  | 判断是否被当前线程独占  |

子类通过实现上面这些方法决定了同步器在获取同步时的行为。

下面来解释一下AQS的运行机制。

#### 1.1 Node类
从上面的类图可以看到AQS类内部有一个Node类，该类用于实现一个双向链表，表示等待获取的线程队列，该类有5个成员变量
| 变量               | 含义                   |
| ----------------- | ----------------------- |
| waitStatus        | 该节点的状态：CANCELLED(acquire取消，在锁的场景下可以理解为取消加锁), SIGNAL(等待唤醒), CONDITION(等待一个condition的唤醒), PROPAGATE(共享模式下), 0(初始状态) |
| thread            | 该等待节点对应的线程 |
| prev  | 等待队列的前一个节点 |
| next  | 等待队列的后一个节点 |
| nextWaiter | 若该节点在等待一个condition，则nextWaiter指向等待该condition的下一个节点 |

#### 1.2 ConditionObject类
ConditionObject类实现了Condition接口，Condition一般是配合Lock使用，这里ConditionObject用于配合AQS实现类似的效果，例如，可以创建多个ConditionObject类表示不同的条件，满足某一个条件则唤醒该ConditionObject对应的等待队列中的节点，并将其加入AQS的等待队列，去尝试获取锁。

| 变量  | 含义                    |
| ----------------- | ----------------------- |
| firstWaiter        | 该ConditionObject的等待队列的头节点 |
| nextWaiter        | 该ConditionObject的等待队列的尾节点 |

#### 1.3 AQS类

有3个成员变量

| 变量 | 含义                    |
| ---------------- | ----------------------- |
| state            | 保存状态的变量，在锁的场景下可以是锁的状态，如0表示未加锁，1表示加锁；或者可重入锁的情况下保存锁重入的次数，0表示未加锁，3表示已加锁并重入了3次 |
| head             | 等待获取锁的队列的头节点 |
| tail             | 等待获取锁的队列的尾节点 |


通过上面对3个类以及AQS源码的分析，我们可以得出AQS的运行时数据结构，当尝试获取锁时，将对应线程加入等待队列，释放锁时，将其移出队列。若要支持Condition，则可以利用ConditionObject，ConditionObject实现了条件队列。

<img src="/java/concurrency/aqs.png" alt="aqs" style="zoom: 50%;" />



#### 1.4 基于AQS实现的锁及其他同步器

基于AQS实现的锁及其他同步器如下：

| 使用了AQS的同步器实现  | 使用场景                               |
| ---------------------- | -------------------------------------- |
| ReentrantLock          | 可重入锁，类似synchronized             |
| ReentrantReadWriteLock | 读写锁，适用于需要加锁的读多写少的场景 |
| Semaphore              | 信号量，用于控制并发量                 |
| CountdownLatch         | 闭锁，让线程等待其他线程的完成         |

### 2. ReentrantLock

ReentrantLock的特性如下：

- 可重入：同一个线程最大重入次数2147483647

- 支持公平锁与非公平锁：公平锁是指线程获取锁时要先判断当前排队的线程队列是否为空，为空则直接通过CAS机制尝试获取锁，不为空则排队；非公平锁是指线程获取锁时先尝试获取锁，失败则再次尝试获取锁（自旋），再次失败则进入排队队列，进入排队队列后，所有线程都排队，死循环至获取锁成功或中断。

ReentrantLock是利用AQS实现的，具体的分析可以查看美团技术团队的这篇文章

https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html[^2]

非常详细，示意图清晰。
