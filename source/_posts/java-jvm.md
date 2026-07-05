---
title: JVM 基础
date: 2020-01-05 00:00:00
updated: 2020-01-05 00:00:00
tags:
  - Java
  - JVM
  - GC
  - 类加载
  - 内存模型
categories:
  - Java
---

本文大部分内容通过收集网上文章总结而来，有些标注了出处，有些由于忘记等原因没有标注出处，若发现文中内容存在原出处并且没有标注，请联系本人。

[TOC]

# JVM 基础

先大致了解一下JVM的运行时内存结构，运行时内存结构包含了大部分需要了解甚至深入了解的JVM基础内容。

![jvm](/java/jvm/jvm-7365577.png)

JVM运行时的内存结构分为6个区域，分别是

- PC寄存器，即程序计数器
- 虚拟机栈
- 本地方法栈
- Heap，即堆
- 方法区，方法区的实现取决于虚拟机可以放在堆的永久代，或者Metaspace(java 8)。

上述 6 个区域，除了 PC Register 区不会抛出 StackOverflowError 或 OutOfMemoryError ，其它 5 个区域，当请求分配的内存不足时，均会抛出 OutOfMemoryError（即：OOM），其中 thread 独立的 JVM Stack 区及 Native Method Stack 区还会抛出 StackOverflowError。

还有一类不受 JVM 虚拟机管控的内存区，这里也提一下，即：堆外内存。

![directbuffer](/java/jvm/directbuffer.png)

可以通过 Unsafe 和 NIO 包下的 DirectByteBuffer 来操作堆外内存。如上图，虽然堆外内存不受 JVM 管控，但是堆内存中会持有对它的引用，以便进行 GC。


## 一、ClassLoader

说到jvm内存不得不先说一下类加载器，类都是通过类加载器加载到jvm中。

###  1. 双亲委派模型

![classloader](/java/jvm/classloader.png)

JVM默认采用双亲委派模型实现类加载机制。双亲委派模型中，类加载器是分层级的结构，一个类加载器有其parent类加载器，JVM自带的Bootstrap ClassLoader没有parent，可作为其他类加载器的parent。

采用双亲委派模型的主要原因是：

https://blog.csdn.net/21aspnet/article/details/88412989

https://www.ibm.com/developerworks/cn/java/j-lo-classloader/



### 2. 线程上下文类加载器 ContextClassLoader

![context_classloader](/java/jvm/context_classloader.png)

线程上下文类加载器是从 JDK 1.2 开始引入的。针对双亲委派模型无法解决的情况，可以采用线程上下文类加载器实现类加载，具体如下：


java.lang.Thread中的方法 getContextClassLoader()和 setContextClassLoader(ClassLoader cl)用来获取和设置线程的上下文类加载器。如果没有通过 setContextClassLoader(ClassLoader cl)方法进行设置的话，线程将继承其父线程的上下文类加载器。线程的初始上下文类加载器是系统类加载器。在线程中运行的代码可以通过此类加载器来加载类和资源。



#### SPI (Service Provider Interface)

SPI的全名为Service Provider Interface，主要是应用于厂商自定义组件或插件中。在java.util.ServiceLoader的文档里有比较详细的介绍。

简单的总结下java SPI机制的思想：我们系统里抽象的各个模块，往往有很多不同的实现方案，比如日志模块、xml解析模块、jdbc模块等方案。面向的对象的设计里，我们一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。为了实现在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制。 Java SPI就是提供这样的一个机制：为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。

Java SPI的具体约定为：当服务的提供者提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。

jdk提供服务实现查找的一个工具类：java.util.ServiceLoader。

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

### 3. Tomcat的类加载机制

![tomcat_classloader](/java/jvm/tomcat_classloader.png)

Tomcat目录下有4组目录：

/common目录：类库可以被Tomcat和Web应用程序共同使用；由 Common ClassLoader类加载器加载目录下的类库；

/server目录：类库只能被Tomcat可见；由 Catalina ClassLoader类加载器加载目录下的类库；

/shared目录：类库对所有Web应用程序可见，但对Tomcat不可见；由 Shared ClassLoader类加载器加载目录下的类库；

/WebApp/WEB-INF目录：仅仅对当前web应用程序可见。由 WebApp ClassLoader类加载器加载目录下的类库；

每一个JSP文件对应一个JSP类加载器。

## 二、堆

### 1. 从Java Object说起[^6]

Java中Object的组成：

> Object = Header + Primitive Fields + Reference Fields + Alignment & Padding

#### a. Java Object Header

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
- ...

#### b. 偏向锁 biased locking[^3]

![Synchronization.gif](/java/jvm/Synchronization.gif)

![synchronized](/java/jvm/synchronized.png)

jvm利用对象头中存储的信息实现了偏向锁.

1. 偏向锁：偏向锁假定将来只有第一个申请锁的线程会使用锁（不会有任何线程再来申请锁），对象头中存线程id，通过CAS操作加锁，无法通过自旋锁优化

2. 轻量级锁：对象头中Mark Word中的部分字节指向线程栈中的Lock Record，通过CAS操作加锁/解锁，可通过自旋锁优化

3. 重量级锁：对应操作系统的mutex，成本非常高，包括系统调用引起的内核态与用户态切换、线程阻塞造成的线程切换等，可通过自旋锁优化，避免线程切换

4. 锁升级/锁膨胀：当没有竞争时（只有一个线程使用锁），采用偏向锁，当有第二个线程尝试获取锁时，升级为轻量级锁，当竞争较多时，升级为重量级锁

#### c. 对齐（Alignment）和补齐（Padding）

- 对齐：任何对象都是以8 bytes的粒度来对齐的
  怎么理解这句话呢？请看一个例子，`new Object()`产生的对象的大小是多少呢？12 bytes的header，但对齐必须是8的倍数，还有4 bytes的alignment，所以对象的大小是16 bytes
  
- 补齐：补齐的粒度是4 bytes
  可以简单理解为，JVM分配内存空间一次最少分配8 bytes，对象中字段对齐的最小粒度为4 bytes

### 2. GC

![gc1](/java/jvm/gc1.png)



#### a. GC Roots



可达性分析



a.虚拟机栈(栈桢中的本地变量表)中的引用的对象

b.方法区中的类静态属性引用的对象

c.方法区中的常量引用的对象

d.本地方法栈中JNI的引用的对象



1.System Class ----------Class loaded by bootstrap/system class loader. For example, everything from the rt.jar like java.util.* . 

2.JNI Local ----------Local variable in native code, such as user defined JNI code or JVM internal code. 

3.JNI Global ----------Global variable in native code, such as user defined JNI code or JVM internal code. 4.Thread Block ----------Object referred to from a currently active thread block. Thread ----------A started, but not stopped, thread. 

5.Busy Monitor ----------Everything that has called wait() or notify() or that is synchronized. For example, by calling synchronized(Object) or by entering a synchronized method. Static method means class, non-static method means object. 

6.Java Local ----------Local variable. For example, input parameters or locally created objects of methods that are still in the stack of a thread. 

7.Native Stack ----------In or out parameters in native code, such as user defined JNI code or JVM internal code. This is often the case as many methods have native parts and the objects handled as method parameters become GC roots. For example, parameters used for file/network I/O methods or reflection. 7.Finalizable ----------An object which is in a queue awaiting its finalizer to be run. 

8.Unfinalized ----------An object which has a finalize method, but has not been finalized and is not yet on the finalizer queue. 

9.Unreachable ----------An object which is unreachable from any other root, but has been marked as a root by MAT to retain objects which otherwise would not be included in the analysis. 

10.Java Stack Frame ----------A Java stack frame, holding local variables. Only generated when the dump is parsed with the preference set to treat Java stack frames as objects. 

11.Unknown ----------An object of unknown root type. Some dumps, such as IBM Portable Heap Dump files, do not have root information. For these dumps the MAT parser marks objects which are have no inbound references or are unreachable from any other root as roots of this type. This ensures that MAT retains all the objects in the dump.

#### b. 常用GC算法

##### 1. mark-sweep 标记清除法

简单快速，易产生内存碎片

##### 2. mark-copy 标记复制法

避免内存碎片，浪费50%内存

##### 3. mark-compact 标记 - 整理（也称标记 - 压缩）法

效率低

##### 4. generation-collect 分代收集算法

以 Hotspot 为例（JDK 7）：

![generation_collect](/java/jvm/generation_collect.png)

将内存分成了三大块：年青代（Young Genaration），老年代（Old Generation）, 永久代（Permanent Generation），其中 Young Genaration 更是又细为分 eden，S0，S1 三个区。

结合我们经常使用的一些 jvm 调优参数后，一些参数能影响的各区域内存大小值，示意图如下：

![generation_collect1](/java/jvm/generation_collect1.png)

注：jdk8 开始，用 MetaSpace 区取代了 Perm 区（永久代），所以相应的 jvm 参数变成 -XX:MetaspaceSize 及 -XX:MaxMetaspaceSize。

#### c. GC回收器的各种实现

![gc_collectors](/java/jvm/gc_collectors.png)

##### 1. Serial 收集器

  单线程用标记 - 复制算法，快刀斩乱麻，单线程的好处避免上下文切换，早期的机器，大多是单核，也比较实用。但执行期间，会发生 STW（Stop The World）。

##### 2. ParNew 收集器

  Serial 的多线程版本，同样会 STW，在多核机器上会更适用。

##### 3. Parallel Scavenge 收集器

  ParNew 的升级版本，主要区别在于提供了两个参数：-XX:MaxGCPauseMillis 最大垃圾回收停顿时间；-XX:GCTimeRatio 垃圾回收时间与总时间占比，通过这 2 个参数，可以适当控制回收的节奏，更关注于吞吐率，即总时间与垃圾回收时间的比例。

##### 4. Serial Old 收集器

  因为老年代的对象通常比较多，占用的空间通常也会更大，如果采用复制算法，得留 50% 的空间用于复制，相当不划算，而且因为对象多，从 1 个区，复制到另 1 个区，耗时也会比较长，所以老年代的收集，通常会采用"标记 - 整理"法。从名字就可以看出来，这是单线程（串行）的， 依然会有 STW。

##### 5. Parallel Old 收集器

  一句话：Serial Old 的多线程版本。

##### 6. CMS 收集器

   ![cms_collector](/java/jvm/cms_collector-7367423.png)


##### 7. G1回收器

![g1_collector1](/java/jvm/g1_collector1.png)

**G1 Young GC**
young GC 前：

![g1_collector2](/java/jvm/g1_collector2.png)

young GC 后：

![g1_collector3](/java/jvm/g1_collector3.png)

由于 region 与 region 之间并不要求连续，而使用 G1 的场景通常是大内存，比如 64G 甚至更大，为了提高扫描根对象和标记的效率，G1 使用了二个新的辅助存储结构：
Remembered Sets：简称 RSets，用于根据每个 region 里的对象，是从哪指向过来的（即：谁引用了我），每个 Region 都有独立的 RSets。（Other Region -> Self Region）。
Collection Sets ：简称 CSets，记录了等待回收的 Region 集合，GC 时这些 Region 中的对象会被回收（copied or moved）。

![collection_sets](/java/jvm/collection_sets.png)

RSets 的引入，在 YGC 时，将年青代 Region 的 RSets 做为根对象，可以避免扫描老年代的 region，能大大减轻 GC 的负担。注：在老年代收集 Mixed GC 时，RSets 记录了 Old->Old 的引用，也可以避免扫描所有 Old 区。

**Old Generation Collection（也称为 Mixed GC）**

按 oracle 官网文档描述分为 5 个阶段：Initial Mark(STW) -> Root Region Scan -> Cocurrent Marking -> Remark(STW) -> Copying/Cleanup(STW && Concurrent)

注：也有很多文章会把 Root Region Scan 省略掉，合并到 Initial Mark 里，变成 4 个阶段。

![old_gc1](/java/jvm/old_gc1.png)

存活对象的"初始标记"依赖于 Young GC，GC 日志中会记录成 young 字样。

![old_gc2](/java/jvm/old_gc2.png)

并发标记过程中，如果发现某些 region 全是空的，会被直接清除。

![old_gc3](/java/jvm/old_gc3.png)

![old_gc4](/java/jvm/old_gc4.png)

并发复制 / 清查阶段。这个阶段，Young 区和 Old 区的对象有可能会被同时清理。GC 日志中，会记录为 mixed 字段，这也是 G1 的老年代收集，也称为 Mixed GC 的原因。

![old_gc5](/java/jvm/old_gc5.png)

上图是，老年代收集完后的示意图。

通过这几个阶段的分析，虽然看上去很多阶段仍然会发生 STW，但是 G1 提供了一个预测模型，通过统计方法，根据历史数据来预测本次收集，需要选择多少个 Region 来回收，尽量满足用户的预期停顿值（-XX:MaxGCPauseMillis 参数可指定预期停顿值）。

注：如果 Mixed GC 仍然效果不理想，跟不上新对象分配内存的需求，会使用 Serial Old GC（Full GC）强制收集整个 Heap。

小结：与 CMS 相比，G1 有内存整理过程（标记 - 压缩），避免了内存碎片；STW 时间可控（能预测 GC 停顿时间）。



##### 8. ZGC[^5]

Java11引进了ZGC，可通过-XX:+UnlockExperimentalVMOptions -XX:+UseZGC参数打开。ZGC分为3个阶段：

1. 第一阶段stop-the-world，标记GC roots。由于GC roots一般数量较少，这个阶段耗时较少。
2. 第二阶段并发从GC roots出发标记可达对象。
3. 最后一个阶段stop-the-world，处理边界情况，例如weak reference。

###### 指针染色 reference coloring

![zgc-pointer](/java/jvm/zgc-pointer的副本.png)

指针指向OS虚拟内存中的位置，由于32位指针只能寻址4G的堆内存，而现在的应用一般都需要更多的堆内存，因此ZGC必须使用64位指针。

ZGC指针使用42 bit进行寻址，因此可以寻址4TB的堆内存。除此之外，还有4bit用于存储指针的状态。

- **finalizable** bit – 说明该对象只能通过finalizer可达
- **remap** bit – 指针指向当前对象存储的位置并处于最新状态
- **marked0** and **marked1** bits – 用于标记可达的对象

这些bits也称为metadata bits。ZGC中这些metadata bits中有且仅有一个为1。

###### Relocation 内存块迁移

ZGC中，relocation分为以下阶段:

1. 并发阶段：寻找需要迁移的内存块，保存到relocation set。
2. stop-the-world阶段：迁移在relocation set中的所有GC roots并更新他们的引用，即指针指向的地址。
3. 并发阶段：迁移relocation set中剩余的对象，并保存旧的和新的地址之间的映射关系到forwarding table中。注意，此时并没有完成新的地址更新到指针中。 
4. 剩余的引用的新的地址的重写发生在下一个标记阶段。这样，我们就不用遍历两遍对象树。load barriers也可以实现这个步骤。

###### Remapping and Load Barriers

在迁移阶段，并没有将新的迁移后的地址更新到指针/引用中。因此，通过那些引用是无法访问到正确的对象的，甚至可能访问到垃圾。ZGC 通过使用load barriers来解决这个问题. **Load barriers通过remapping技术实现将指针/引用指向迁移后的新的地址 。**

当应用加载一个引用时，触发load barrier通过以下步骤返回正确的地址：

1. 检查*remap* bit是否设为1，如果是说明已经指向最新地址，可直接返回。
2. 然后检查引用的对象是否在relocation set中，如果不在说明不需要迁移它，为防止下次再检查，将该引用/指针的*remap* bit设为1，并返回该引用。
3. 到这一步就知道引用的对象需要迁移，如果迁移是否已完成，则跳到下一步，如果没完成则在forwarding table中创建一个entry，保存迁移后的新地址。
4. 到这一步目标对象已经迁移完成。我们更新引用使其指向新的地址，设置*remap* bit为1，并返回该引用。

由于每次加载一个引用都会触发load barrier，因此会有一点性能上的损失，尤其是第一次访问一个迁移的对象时，这是为了获得更短的暂停时间而付出的代价。不过由于这些步骤都很快，因此对应用的性能不会有太大影响。



#### d. GC日志, 参数与优化

GC优化相关经验不多，可以参考下面这篇文章。

https://tech.meituan.com/2017/12/29/jvm-optimize.html

#### e. Object.finalize() 方法与 Finalizer 类

这部分内容属于GC相关的内容，可以参考 https://www.jianshu.com/p/9d2788fffd5f [^9]，此文写的比较深入，涉及到了hotspot的源码，强烈推荐。

##### finalize() 方法[^8]

```java
public class FinalizerDemo {

    static AtomicInteger aliveCount = new AtomicInteger(0);

    FinalizerDemo() {
        aliveCount.incrementAndGet();
    }

    @Override
    protected void finalize() throws Throwable {
        FinalizerDemo.aliveCount.decrementAndGet();
    }

    public static void main(String args[]) {
        for (int i = 0; ; i++) {
            FinalizerDemo f = new FinalizerDemo();
            if ((i % 100_000) == 0) {
                System.out.format("After creating %d objects, %d are still alive.%n", new Object[]{i, FinalizerDemo.aliveCount.get()});
            }
        }
    }
}
```

运行这段代码，加上-XX:+PrintGCDetails参数查看GC详情，可以发现很快就开始full gc，而去掉对finalize()方法的重写则不会触发full gc。这是为什么呢？这就涉及到重写了finalize()方法的类的回收了，下面详细说一下，或者也可以直接阅读上面推荐的文章。

列一下finalize()方法相关的特性：

- 当GC判断没有其他对该对象的引用时，会调用该方法，一般用于执行资源释放或其他清理工作
- finalize()方法抛出的异常会被忽略
- finalize()方法中可以使该对象重新关联到引用链上，也即是"复活"该对象，例如，让一个静态全局变量指向该对象，使其免于被回收
- finalize()方法最多执行一次

##### Finalizer 类

阅读上面推荐的文章就知道finalize()方法与Finalizer类脱不开关系。下面分析一下这个类的源码。

Finalizer类有3个静态变量，2个成员变量，分别是：

| 变量                         | 作用                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| ReferenceQueue\<Object\> queue | 全局静态变量，是一个引用队列，保存Finalizer对象      |
| Finalizer unfinalized        | 全局静态变量，Finalizer对象以双向链表的形式保存，该变量指向当前链表的头部，每次新创建一个Finalizer对象会将其放在链表头部 |
| Object lock                  | 全局静态变量，用于在对双向链表操作时加锁                     |
| next                         | 链表中当前对象的下一个                                       |
| prev                         | 链表中当前对象的前一个                                       |

注意到Finalizer类的源码中有一个register方法，注释是该方法由VM调用，推测是当重写了finalize()的对象初始化时会调用register方法（推荐文章中有对这部分的hotspot源码分析），该方法会将该对象作为构造器参数创建一个Finalizer对象，Finalizer继承了FinalReference\<Object\>，因此构造器会将finalizee（即实现了finalize()方法的对象）和全局的ReferenceQueue作为构造参数构造Reference\<Object\>。

```java
    private Finalizer(Object finalizee) {
        super(finalizee, queue);
        add();
    }

    /* Invoked by VM */
    static void register(Object finalizee) {
        new Finalizer(finalizee);
    }
```

Finalizer类的runFinalizer方法就是最终用来调用对象的finalize()方法的。先判断是否已执行过finalize()方法，若没有，则从双向链表中移除自身，然后调用finalizee的finalize()方法，最后finalizee = null;这一步注释说的是清理包含这个变量的栈帧，减少false retention概率，关于false retention后面单开文章说。

```java
private void runFinalizer(JavaLangAccess jla) {
        synchronized (this) {
            if (hasBeenFinalized()) return;
            remove();//不去掉的话，会导致对象仍然在引用链上，无法回收
        }
        try {
            Object finalizee = this.get();
            if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
                jla.invokeFinalize(finalizee);

                /* Clear stack slot containing this variable, to decrease
                   the chances of false retention with a conservative GC */
                finalizee = null;
            }
        } catch (Throwable x) { }
        super.clear();
    }
```

###### FinalizerThread

Finalizer类通过静态初始化代码块启动一个FinalizerThread守护线程，该线程优先级较低。注意线程优先级较低这一点，正是这一点导致在上面的示例中来不及执行对象的finalize()方法，导致垃圾回收很慢，内存泄漏，触发full gc，甚至OOM。

```java
static {
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
         tgn != null;
         tg = tgn, tgn = tg.getParent());
    Thread finalizer = new FinalizerThread(tg);
    finalizer.setPriority(Thread.MAX_PRIORITY - 2);
    finalizer.setDaemon(true);
    finalizer.start();
}
```

下面看下FinalizerThread的逻辑：

```java
private static class FinalizerThread extends Thread {
    private volatile boolean running;
    FinalizerThread(ThreadGroup g) {
        super(g, "Finalizer");
    }
    public void run() {
        if (running)
            return;

        // Finalizer thread starts before System.initializeSystemClass
        // is called.  Wait until JavaLangAccess is available
        while (!VM.isBooted()) {
            // delay until VM completes initialization
            try {
                VM.awaitBooted();
            } catch (InterruptedException x) {
                // ignore and continue
            }
        }
        final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
        running = true;
        for (;;) {
            try {
                Finalizer f = (Finalizer)queue.remove();
                f.runFinalizer(jla);
            } catch (InterruptedException x) {
                // ignore and continue
            }
        }
    }
}
```

主要关注最后的死循环部分，调用queue的remove()方法，取出队列中的Finalizer对象，执行其runFinalizer方法。

到这里就剩下一个疑问点，Finalizer对象是什么时候加入queue队列的？这就要说到Reference\<T\>类了。

##### Reference 与 ReferenceHandler

前面说到Finalizer继承了FinalReference\<Object\>，而FinalReference类继承了Reference。看一下这个类的静态初始化代码块，里面启动了一个ReferenceHandler守护线程，该线程优先级较高。

```java
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();

        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean tryHandlePendingReference() {
                return tryHandlePending(false);
            }
        });
    }
```

ReferenceHandle线程死循环执行一个方法tryHandlePending，源码如下：

```java
static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }
```

主要逻辑就是将pending链表的头节点取出，并将其加入队列中（最后3行代码），其中pending链表是GC扫描出的pending状态的Reference对象链表，Finalizer实现了Reference，也会被扫描出来，至于具体逻辑就要查看jdk源码了，这个正好在参考文献 [^9] 也就是最开头推荐的文章中有提到。

##### 小结

回到最开始的例子，再分析一下。main函数死循环创建一个重写了finalize()方法的对象，JVM会为每一个对象创建一个Finalizer对象，ReferenceHandle线程负责将Finalizer对象加入ReferenceQueue，该线程优先级较高，而FinalizerThread线程负责消费ReferenceQueue，执行对象的finalize()方法，优先级较低。只有当对象的finalize()方法执行完，对象才能被回收。

那么当FinalizerThread线程由于优先级低而抢不到CPU资源，或者finalize()方法执行较慢等原因来不及消费ReferenceQueue时，就会出现GC无法回收垃圾，从而导致full gc。

finalize()方法最多执行一次的原理也在此说一下我个人的理解，不一定正确。上面说到，当对象创建时，如果该对象重写了finalize()方法，JVM会调用Finalizer对象的register方法，即该对象对应的Finalizer对象只会被创建一次，而一旦这个Finalizer对象被移出ReferenceQueue就不会再回到ReferenceQueue。

### 3. JVM 压缩指针 compressed oops

oop（ordinary object pointer），即普通对象指针，是JVM中用于代表引用对象的句柄。

对于Java进程，在oop只有32位时，只能引用4G堆内存。因此，如果需要使用更大的堆内存，需要部署64位JVM。这样，oop为64位，可引用的堆内存就更大了。32位的对象引用占4个字节，而64位的对象引用占8个字节。也就是说，64位的对象引用大小是32位的2倍。带来的负面效果就是，对象引用占用了更多内存空间，占用更多内存空间意味着gc的次数更多，占用更多cpu时间。因此，使用64位jvm并不一定能立刻带来性能上的提升。

那么难道要放弃使用64位机器吗？答案是可以通过压缩指针解决这个问题，实现在64位机器上使用32bit的对象指针来引用超过4G（事实上是32G）的堆内存空间。设置参数`-XX:-UseCompressedOops`即可开启压缩指针，从java7开始当最大堆内存小于32G时即默认开启压缩指针。

指针压缩的原理：由于jvm对所有对象都进行了padding处理，所以对象的大小都是8 bytes的倍数，这也就意味着对象地址都是8的倍数，即oop的最后3位总是0，这给指针压缩带来了空间。oop可以增加3位至35bit，其最后3位为0，当保存oop时将其右移3位，只保存32bit信息，取出来时左移3位恢复，这样就可以引用**2^(32+3)=2^35=32 GB**的堆内存空间。位移操作带来的消耗可以忽略不计。[^4]

![img](/java/jvm/Untitled-Diagram-2-e1554193172973.jpg)

**PS:**

- ZGC中由于需要使用64bit colored pointers，因而不支持指针压缩，因此需要权衡它带来的更低的gc延迟与更多内存消耗。
- 在堆内存超出32GB的情况下，可以通过修改-XX:ObjectAlignmentInBytes参数改变对象padding对齐的大小，这个值必须是8到256之间的一个2的指数，来使用压缩指针。例如当设为按16bytes对齐时，可通过压缩指针管理64GB的堆内存。带来的问题时，对象之间的空闲空间增多，因而实际可能并没有性能上的提升。


## 三、方法区

方法区，主要存放类信息、静态变量、运行时常量池、字段数据、方法数据、方法代码等。

- Classloader reference
- 运行时常量池 — 数字常量, field references, method references, attributes; As well as the constants of each class and interface, it contains all references for methods and fields. When a method or field is referred to, the JVM searches the actual address of the method or field on the memory by using the runtime constant pool.
- Field data — Per field: name, type, modifiers, attributes
- Method data — Per method: name, return type, parameter types (in order), modifiers, attributes
- Method code — Per method: bytecodes, operand stack size, local variable size, local variable table, exception table; Per exception handler in exception table: start point, end point, PC offset for handler code, constant pool index for exception class being caught

《Java虚拟机规范》规定了方法区的概念和作用，并没有规定如何具体实现。JDK7及之前，HotSpot虚拟机使用永久代来实现方法区，JDK8中去掉了永久代，用Metaspace实现方法区。

### 1. 运行时常量池 runtime constant pool

#### 静态常量池(class文件常量池)

编译生成的 class 字节码文件结构中有一项是常量池（Constant Pool Table），用于存放编译期生成的各种字面量(Literal)和符号引用(Symbolic References)，这部分内容将在类加载后进入方法区的运行时常量池中存放。运行时常量池可以在运行期间将符号引用解析为直接引用。

- 字面量：字符串字面量和声明为 final 的（基本数据类型）常量值。字符串字面量除了类中所有双引号括起来的字符串(包括方法体内的)，还包括所有用到的类名、方法的名字和这些类与方法的字符串描述、字段(成员变量)的名称和描述符；声明为final的常量值指的是成员变量，不包含本地变量，本地变量是属于方法的。这些都在常量池的 UTF-8 表中(逻辑上的划分)；

- 符号引用：指向 UTF-8 表中字面量的引用

  1. 类和接口的全限定名：例如对于String这个类，它的全限定名就是java/lang/String。

  2. 字段的名称和描述符：这里的字段就是类或者接口中声明的变量。

  3. 方法的名称和描述符：这里的描述符是方法的参数类型+返回值类型。

PS : int -128~127 范围的值也在运行时常量池中。

#### 字符串常量池 String Pool

字符串常量池是全局的，JVM 中独此一份，因此也称为全局字符串常量池。运行时常量池中的字符串在从静态常量池加载时，会先去String Pool中查询此此字符串在String Pool中的引用，如果没有则在String Pool中创建此字符串然后返回其引用，用返回的引用替换运行时常量池的字符串。运行时常量池是全局共享的，多个类共用一个运行时常量池，class文件中常量池多个相同的字符串在运行时常量池只会存在一份。String 类的 intern() 方法可在运行期间把字符串放到字符串常量池中。

JVM 中除了字符串常量池，8种基本数据类型中除了两种浮点类型剩余的6种基本数据类型的包装类，都使用了缓冲池技术，但是 Byte、Short、Integer、Long、Character 这5种整型的包装类也只是在对应值在 [-128,127] 时才会使用缓冲池，超出此范围仍然会去创建新的对象。

**PS：**

jdk1.6（含）之前字符串常量池是方法区的一部分，并且其中存放的是字符串的实例。
jdk1.7中字符串常量池存储的是字符串对象的引用，字符串对象实例是在堆中。
jdk1.8 已移除永久代，字符串常量池存储的也只是引用。

#### String.intern()

https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html

### 2. Permgen vs Metaspace

区别：

1. 存储位置不同，永久代是堆的一部分，和新生代，老年代地址是连续的，而元空间属于本地内存；

2. 存储内容不同，元空间存储类的元信息，静态变量和常量池等并入堆中。相当于永久代的数据被分到了堆和元空间中。

   
   https://www.cnblogs.com/paddix/p/5309550.html

## 四、栈

### 1.  JVM栈帧 (Stack Frame)

![stack](/java/jvm/stack.png)

#### a. 线程

一个线程对应一个jvm stack，一个stack中包含一组stack frame。线程调用方法对应stack frame入栈，方法结束（正常返回或异常结束）对应出栈。

#### b. 栈帧

栈帧用于存储局部变量表、操作数栈、动态链接、方法返回等信息。 每个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。
在活动线程中，只有位于栈顶的栈帧才是有效的，称为当前栈帧，与这个栈帧相关联的方法称为当前方法。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作。

- 局部变量表：局部变量表存放了编译期可知的各种基本数据类型(boolean、byte、char、short、int、float、long、double)「String是引用类型」，对象引用(reference类型)和returnAddress类型（它指向了一条字节码指令的地址）。
- 操作数栈：
- 动态连接
  - 每个栈帧都包含一个指向**运行时常量池**中，该帧所属方法的引用，以支持动态连接
- 方法返回地址

#### c. StackOverflowError

当方法的递归调用太深时，会发生StackOverflowError错误。其他会产生StackOverflowError的极端情况还有：方法内部一直嵌套调用、方法的局部变量过多、类的构造函数间相互循环调用、类在实例化时将类的实例作为成员变量。




## 五、本地方法栈

本地 (原生) 方法栈，顾名思义就是调用操作系统原生本地方法时，所需要的内存区域。HotSpot虚拟机直接将本地方法栈和虚拟机栈合二为一。同虚拟机栈相同，Java虚拟机规范对这个区域也规定了两种异常情况：`StackOverflowError` 和 `OutOfMemoryError`。

## 六、程序计数器
PC (program counter) register 存储了当前正在被执行的JVM指令的地址。


## 七、执行引擎
有关jvm执行引擎的内容也看了不少文章，感觉不是简单能说清楚的，推荐可以先阅读以下文章：

http://wuzguo.com/blog/2017/06/23/jvm_execution_engine.html

https://blog.csdn.net/qq_33938256/article/details/52584658

https://blog.csdn.net/dd864140130/article/details/49515403

后续有机会，针对这个话题单独展开一下，下面就简单说一下之前提到的JIT编译器带来的逃逸分析。

### 1. Just-In-Time(JIT) compiler

#### JIT编译优化技术带来的内存分配变化

JIT编译除了具有缓存的功能外，还会对代码做各种优化，比如：逃逸分析、 锁消除、 锁膨胀、 方法内联、 空值检查消除、 类型检测消除、 公共子表达式消除等

#### 逃逸分析

从jdk 1.7开始已经默认开始逃逸分析，如需关闭，需要指定-XX:-DoEscapeAnalysis[^7]

逃逸分析(Escape Analysis)是目前Java虚拟机中比较前沿的优化技术。这是一种可以有效减少Java 程序中同步负载和内存堆分配压力的跨函数全局数据流分析算法。通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围从而决定是否要将这个对象分配到堆上。

逃逸分析的基本行为就是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他地方中，称为方法逃逸。

逃逸分析的 JVM 参数如下：

- 开启逃逸分析：-XX:+DoEscapeAnalysis
- 关闭逃逸分析：-XX:-DoEscapeAnalysis
- 显示分析结果：-XX:+PrintEscapeAnalysis

使用逃逸分析，编译器可以对代码做如下优化：

一、同步省略。如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的操作可以不考虑同步。

二、将堆分配转化为栈分配。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会逃逸，对象可能是栈分配的候选，而不是堆分配。

三、分离对象或标量替换。有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。



By design, Java is slow due to dynamic linking and run-time interpreting.
JIT compiler compensate for the disadvantages of the interpreter for repeating operations by keeping a native code instead of bytecode.

## 八、内存分配优化

其实这部分属于比较不是那么基础的内容，之所以放在这里是因为在写这篇文章的过程中，发现坑越挖越多，越挖越深，只好把已经挖的坑先填一填，难受。

![img](/java/jvm/SouthEast.png)



### 1. 栈上分配与标量替换[^7]

这部分其实是jvm对内存分配做的优化工作，通过逃逸分析判断局部变量是否会逃逸出线程调用栈，若不回则进行栈上分配，如果是基本类型的变量则进行标量替换。

相关启动参数：见上面逃逸分析部分。

### 2. bump-the-pointer和TLAB allocation（Thread-Local Allocation Buffers）

应用程序的对象大部分是由线程创建并分配内存的。

HotSpot虚拟机使用了两种技术来加快内存分配。他们分别是是**bump-the-pointer**和TLABs（Thread-Local Allocation Buffers）

bump-the-pointer

TLAB allocation仍然是堆的一部分，从它的名字中的Thread-Local就可以看出，只不过这部分buffer是单独分配给线程进行内存分配的。

https://www.jianshu.com/p/8be816cbb5ed

https://blog.csdn.net/yangzl2008/article/details/43202969

https://dzone.com/articles/thread-local-allocation-buffers

https://blog.csdn.net/yangsnow_rain_wind/article/details/80434323

https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/

https://blog.csdn.net/zhou2s_101216/article/details/79221310

https://www.jianshu.com/p/580f17760f6e

https://juejin.im/post/5c151b266fb9a049ac790d9a

https://segmentfault.com/a/1190000016960388

## 九、JVM系统线程

除了开发者在java应用中创建的线程，jvm中还有许多系统创建的系统线程，这些线程完成许多jvm内部的任务。
### 1. main thread
对应`public static void main(String[])`方法的系统线程。jvm先启动一个main线程，然后去调用指定的类的main方法。

### 2. Compiler threads
Compiler线程在运行时将字节码编译成机器码。
### 3. GC threads
GC线程负责GC相关的操作。
### 4. Periodic task thread
The timer events (i.e. interrupts) to schedule execution of periodic operations are performed by this thread.
### 5. Signal dispatcher thread
这个线程接收发送给JVM的信号，并调用相应的JVM方法进行处理。
### 6. VM thread


As a pre-condition, some operations need the JVM to arrive at a safe point where modifications to the Heap area does no longer happen. Examples for such scenarios are "stop-the-world" garbage collections, thread stack dumps, thread suspension and biased locking revocation. These operations can be performed on a special thread called VM thread.

# 小结
本文是个人在查找阅读许多文章的基础上总结得来，其中部分内容经过个人实际试验，但大部分内容乃是前人的经验总结，在此深表感谢。在写本文的过程中，发现JVM体系庞大，越挖越深，颇有"吾生也有涯，而知也无涯"的感触，也总结出一个字，学！！！


# 参考文献

[^1]: https://juejin.im/post/5c4c8ad9f265da6179752b03
[^2]: https://gist.github.com/arturmkrtchyan/43d6135e8a15798cc46c#file-objectheader64-txt
[^3]: https://wiki.openjdk.java.net/display/HotSpot/Synchronization
[^4]:https://www.baeldung.com/jvm-compressed-oops
[^5]: https://www.baeldung.com/jvm-zgc-garbage-collector
[^6]: https://segmentfault.com/a/1190000012354736
[^7]:https://juejin.im/post/5c151b266fb9a049ac790d9a
[^8]: https://www.cnblogs.com/benwu/articles/5812903.html
[^9]: https://www.jianshu.com/p/9d2788fffd5f
