---
title: JMM (Java Memory Model)
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - JMM
  - happens-before
  - volatile
  - 内存屏障
categories:
  - Java
  - 并发
---

在可共享内存的多处理器架构中，存在CPU多级缓存与内存的缓存一致性问题，不同的架构由不同的缓存一致性协议，本质上是通过内存屏障来实现。另外还有指令重排序问题，指令重排序提升了性能，然而在多线程环境中如果无法确认代码的执行顺序，就无法确认代码的正确性。

JMM(Java Memory Model)通过提供自己的存储模型，屏蔽了java虚拟机与底层硬件存储模型的差异化，在语言层面定义了**内存屏障**，用来屏蔽不同硬件存储模型的内存屏障的不同实现。

### 1. 从硬件平台的存储模型到Java存储模型

#### 1.1 缓存一致性问题 cache coherence

![处理器Cache模型](/java/concurrency/d69cecab903313c776b50de1c43050bc31123.png)

CPU一般有多级缓存，与主内存之间通过同步协议保证一致性，比较经典的是MESI协议，参考https://blog.csdn.net/muxiqingyang/article/details/6615199、https://www.cnblogs.com/yanlong300/p/8986041.html。

#### 1.2 指令重排序

指令重排序可能是编译器指令重排序（编译器级别）或CPU指令重排序（处理器级别，out-of-order execution），指令重排序可以使计算性能得到提升。
即使指令没有重排序，由于CPU缓存的存在，缓存刷新至内存的时许不同也会导致重排序问题。

#### 1.3 memory barrier 内存屏障

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。Java编译器也会根据内存屏障的规则禁止重排序。

内存屏障可以被分为以下几种类型：

- LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

![内存屏障示意表](/java/concurrency/bde75d1129494bf77b8b8b1ade546cd276768.png)


以上关于指令重排序、内存屏障的描述参考了https://tech.meituan.com/2014/09/23/java-memory-reordering.html，希望深入的了解的可直接阅读原文。

#### 1.4 Java存储模型的happens-before法则[^4]

JMM为程序中的所有动作定义了一种happens-before关系，两个操作如果满足happens-before关系，则前者的结果一定对后者可见，保证了顺序性及可见性，而不满足happens-before关系的动作之间可以任意重排序。

- 程序次序法则：同一线程中，代码中先出现的动作happens-before代码中后出现的动作，这只保证最终执行结果与顺序执行一致，并不能保证指令不重排序，只是结果上表现为happens-before。这个可以解释为，同一线程中前面的写操作对后面操作可见。
- 监视器锁法则：对同一个监视器锁的解锁happens-before后续对该锁的加锁，显式锁同样适用。
- volatile法则：对volatile修饰的域的写happens-before后续对其的读操作，原子变量同样适用。
- 线程启动法则：一个线程内，对Thread.start的调用happens-before每一个被启动线程中的动作。
- 线程终结法则：线程中的任何动作happens-before其他线程监测到这个线程已经终结，或者从Thread.join调用中成功返回，或者Thread.isAlive返回false
- 中断法则：一个线程调用另一个线程的interrupt happens-before被中断的线程发现中断
- 终结法则：一个对象的构造函数的结束happens-before这个对象的finalizer的开始
- 传递性：如果A happens-before B，且B happens-before C，则A happens-before C

基于happens-before可以推断出多线程情况下代码的执行顺序，当然如果正确没有使用相应的同步机制，大部分操作是无法推断的😣。

#### 1.5 volatile、synchorized、final[^3]

- volatile关键字：volatile关键字可以保证直接从主存中读取一个变量，如果这个变量被修改后，总是会被写回到主存中去。普通变量与volatile变量的区别是：volatile的特殊规则保证了新值能立即同步到主内存，以及每个线程在每次使用volatile变量前都立即从主内存刷新。因此我们可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。
- synchorized关键字：同步块的可见性是由以下机制保证的：
  "如果对一个变量执行lock操作，将会清空工作内存中变量的值，在执行引擎使用这个变量前需要重新执行load或assign操作初始化变量的值"
  "对一个变量执行unlock操作之前，必须先把此变量的值同步到主内存中（执行store和write操作）
- final关键字：final关键字的可见性是指，被final修饰的字段在构造器中一旦被初始化完成，并且构造器没有把"this"的引用传递出去（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到"初始化了一半"的对象），那么在其他线程就能看见final字段的值（无须同步）


### 2. CPU缓存的伪共享问题

CPU缓存的最小单位是缓存行 (Cache Line) ，一个缓存行的大小通常是 64 字节（取决于 CPU），它有效地引用主内存中的一块地址。一个 Java 的 long 类型是 8 字节，因此在一个缓存行中可以存 8 个 long 类型的变量。

假设两个线程A和B运行在两个CPU上，每个CPU都有一个缓存行中存放了两个volatile long类型的变量X和Y，A更新X后，由于X是volatile的，B所在CPU的缓存行就失效了，需要重新加载，即使B想要更新的是Y，两者逻辑上不存在竞争关系，但在缓存行这个层次上发生了冲突。这是一个伪共享问题的典型场景。

上述场景中，假如A和B交替执行，那么伪共享问题一直发生，对性能影响会很大。

#### 2.1 @sun.misc.Contended注解

在Java 7之前，可以在属性的前后进行padding来避免伪共享问题。

在Java 8中，提供了@sun.misc.Contended注解来避免伪共享，原理是在使用此注解的对象或字段的前后各增加128字节大小的padding，使用2倍于大多数硬件缓存行的大小来避免相邻扇区预取导致的伪共享冲突。



以上关于伪共享问题的内容参考了https://www.jianshu.com/p/c3c108c3dcfd。

### 3. 重新理解对象的安全发布与初始化安全性

在了解了JMM之后，可以回顾一下之前提到的对象的安全发布以及初始化安全性。

理解对象安全发布的一个很好的例子就是懒汉单例模式（当然现在已经不推荐使用懒汉单例模式了，它复杂且节约的性能有限），理解为何instance变量必须被volatile修饰才能保证安全。推荐思考一下。
