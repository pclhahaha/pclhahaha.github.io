---
title: Java 并发编程
date: 2020-02-09 00:00:00
updated: 2020-02-09 00:00:00
tags:
  - Java
  - 并发
  - JMM
  - AQS
  - 线程池
categories:
  - Java
---
# Java 并发编程

## 一、多线程基础

### 1. 多线程的风险
- 安全性问题：竞态条件race condition、指令重排序等
- 活跃性问题：死锁、饥饿、活锁
- 性能问题：上下文切换、同步机制抑制编译器优化，使内存缓存数据无效，增加共享内存总线的同步流量

### 2. 线程安全性

线程安全本质上就是对共享且可变的状态进行安全的访问，共享指可以被多个线程访问，可变指状态可以被修改。线程安全有以下特性：

- 无状态的对象一定是线程安全的
- 不可变对象一定是线程安全的
- 有状态的可变对象在并发情况下存在竞态条件，即多个线程可能同时修改同一个变量，可通过原子操作或锁实现线程安全
- **原子性**：对状态的原子性操作可以保证线程安全，例如jdk自带的原子类，或者加锁
- 锁：通过加锁/解锁操作避免竞态条件，保证原子性，java中的锁有：内部锁/监视器锁（synchronized关键字），监视器锁是可重入的，通过在锁对象上的一个计数器实现；显式锁

#### 2.1 原子性、可见性、顺序性

- 原子性上面已经提到过，主要是解决了并发情况下的竞态条件
- 可见性：同步带来的另一个好处是内存可见性，可以避免并发情况下的过期数据、非原子的64位操作等问题
- 过期数据问题：例如，指令指令重排序会造成一个线程读取到的共享变量是过期的，指令重排序本质上是一个**顺序性**问题；还有一种可能是CPU多级缓存和内存间的不一致造成读到过期数据
- 非原子的64位操作问题：当读写非volatile的double或long时，若处理器不支持64位算数原子操作，JVM允许将其分为两个32位的操作，如果没有通过加volatile关键字或没有加同步锁保护，那么可能得到一个值的高32位和另一个值的低32位
- **volatile只能保证可见性，加锁可以保证可见性与原子性**
- 以上提到的原子性、可见性、顺序性问题，会在下一节对JMM的描述中展开来说

#### 2.2 对象的发布publish与逸出escape

一个对象能被外部代码使用称为发布，未准备好的对象被发布称为逸出，例如，未构造完毕的对象被传递给外部引用，对象的发布需要注意以下问题：

- 不要让this引用在构造期间逸出，this引用在构造期间逸出可能导致对象未构造完毕即能被使用

- 局部创建对象：例如，多线程共享的对象，构造时由于可见性的问题，可能只完成了部分创建，即其他线程看见的是未完成构造的对象，例如：在double-check-locking(DCL)机制实现的懒汉单例模式中，如果instance变量不加volatile关键字，那么由于步骤(5)对象初始化并赋值给引用过程的指令重排序，可能造成instance指向的是未创建完成的对象时被另一个线程使用，代码如下[^5]：

```java
  public class LazySingleton {
      private int id;
      private static LazySingleton instance;
      private LazySingleton() {
          this.id= new Random().nextInt(200)+1;                 // (1)
      }
      public static LazySingleton getInstance() {
          if (instance == null) {                               // (2)
              synchronized(LazySingleton.class) {               // (3)
                  if (instance == null) {                       // (4)
                      instance = new LazySingleton();           // (5)
                  }
              }
          }
          return instance;                                      // (6)
      }
      public int getId() {
          return id;                                            // (7)
      }
  }
```

#### 2.3 变量的线程封闭

不需要共享的变量可通过线程封闭来实现线程安全，这样对象不会被其他线程访问到，即不发布对象。

- 规范的方式是使用ThreadLocal：线程首次调用ThreadLocal.get方法时，会请求initialValue提供一个初始值

#### 2.4 不可变性

- 不可变对象的条件：不可修改的状态，所有field都是final类型的，正确的构造（没有在构造时发生this引用的逸出）。不可变对象天生是线程安全的。
- 不可变对象保证了初始化安全性，因为final关键字确保了初始化安全性，即对象发布时不会发生局部创建对象的问题，即保证了可见性
- 使用volatile发布不可变对象，可实现线程安全，即使用一个不可变对象保存所有变量，而用一个volatile引用指向不可变对象，当需要更新时，直接替换不可变对象，保证了可见性与原子性

#### 2.5 对象的安全发布

非不可变对象需要被安全的发布，对象的安全发布意味着对象的状态必须在被其他线程（除发布线程）引用的同时可见，一个正确创建的对象可以通过以下方式安全发布：

- 通过静态初始化器初始化对该对象的引用，利用JVM在类加载时的同步机制保证安全发布
- 将该对象的引用存储到volatile域或AutomicReference：volatile保证了对象的可见性，AutomicReference保证对该对象的操作满足原子性，即存在同步机制，这里说下个人理解：原子性保证了可见性，如果可见性无法保证，那么原子性也无法实现
- 将该对象的引用存储到正确创建的对象的final域中：final保证了不可变性
- 将该对象的引用存储到由锁正确保护的域中，例如线程安全的容器类，其他线程在获取放入容器的对象时可以通过同步机制保证对象的安全发布

### 3. 基本组件

#### 3.1 同步容器

- Vector、HashTable、Collections.synchronizedXxx
- 对同步容器的复合操作，例如，put-if-abscent语义的手动实现是不保证线程安全的
- 迭代器使用过程中如果容器被修改会抛出ConcurrentModificationException，它是fail-fast的，通过维护一个计数器，当计数器被修改则抛出异常
- 隐藏迭代器：容器的toString、hashCode、equals的方法，容器本身作为元素或作为另一个容器的key时，containsAll、removeAll、retainAll方法以及把容器做为构造函数参数，都会隐式迭代，可能抛出异常

#### 3.2 并发容器

- ConcurrentHashMap：提供了不会抛出ConcurrentModificationException的迭代器；附加了put-if-abscent、remove-if-equal、replace-if-equal的原子操作
- ConcurrentSkipListMap, ConcurrentSkipListSet
- CopyOnWriteArrayList, CopyOnWriteArraySet：copy-on-write容器在每次需要修改时创建并重新发布一个新的容器拷贝
- ConcurrentLinkedQueue

#### 3.3 阻塞队列与生产者-消费者模式

- LinkedBlockingQueue, ArrayBlockingQueue
- PriorityBlockingQueue
- SynchronousQueue：不存储队列元素，适合消费者充足的场景

#### 3.4 双端队列与工作窃取

双端队列在工作窃取（work-stealing）算法中起着关键作用——被窃取线程从队列头部消费任务，窃取线程从队列尾部获取任务，从而减少竞争。这一设计在 Fork/Join 框架中得到了充分运用，详见第六节。

#### 3.5 阻塞和可中断的方法

线程阻塞挂起时，被设置成某个状态（BLOCKED, WAITING, TIMED_WAITING）。

BlockingQueue的put和take方法会抛出一个InterruptedException，Thread.sleep也会抛出这个异常，当一个方法能够抛出这个异常，说明这是一个可阻塞的方法，并且如果被中断，可以提前结束阻塞状态。Thread提供interrupt方法，用来中断一个线程或者查询某线程是否已被中断，具体实现是：线程维护一个bool类型属性，代表中断状态，中断时设置这个值。

#### 3.6 Synchronizer

- latch闭锁：用于保证特定活动直到其他活动结束后才开始。CountDownLatch是一个实现。
- FutureTask：描述一个可携带结果的计算，通过Callable执行计算，通过Future.get获取结果或者异常，异常统一封装为ExecutionException，如果计算被取消，返回CancellationException。
- 信号量Semaphore：用来控制能够同时访问某资源的并发数量，可用于实现资源池或者给容器限定边界
- 关卡barrier：用于限制所有线程同时到达关卡点。CyclicBarrier，Exchanger是具体实现。

CountDownLatch基于AQS实现，CyclicBarrier基于ReentrantLock，而ReentrantLock基于AQS实现。AQS提供了一个基于队列的同步器框架，许多同步器可以基于AQS实现，AQS的原理在第五节详细展开描述。


> 📖 独立文章：[JMM (Java Memory Model)](/java-memory-model/)

## 三、原子变量类

### 1. CAS
Java内部很多机制以及很多标准类库中都用到了CAS机制，Java的CAS操作依赖硬件对CAS的支持，主流处理器基本都有自己的CAS实现。使用CAS相比于使用锁，可以减少线程上下文切换，减小竞争的颗粒度，一般来说性能优于锁，但是基于CAS的无锁算法实现上会更复杂，相关例子可以参考ConcurrentLinkedQueue的算法。
### 2. 原子变量类
原子变量类保证了可见性与原子性，相比volatile只能保证可见性，功能更为强大。以下是一些常用的原子变量类。

| 原子变量类                                      | 详情                                                         |
| ----------------------------------------------- | ------------------------------------------------------------ |
| AtomicBoolean                                   | 原子化的boolean                                              |
| AtomicInteger、AtomicLong                       | 原子化的int、long                                            |
| AtomicIntegerArray、AtomicLongArray             | 数组内的元素可以原子化的更新                                 |
| AtomicReference                                 | 可以被原子化更新的对象引用                                   |
| AtomicReferenceArray                            |                                                              |
| AtomicStampedReference、AtomicMarkableReference | 支持原子化的更新引用及附带的stamp integer或mark bit，相当于版本号，可防止ABA问题 |
| DoubleAccumulator、LongAccumulator              |                                                              |
| DoubleAdder、LongAdder                          |                                                              |

另外还要介绍一下原子化的域更新器：

| 原子化的域更新器            | 作用                                    |
| --------------------------- | --------------------------------------- |
| AtomicIntegerFieldUpdater   | 原子化的更新指定类的volatile int 变量   |
| AtomicLongFieldUpdater      | 原子化的更新指定类的volatile long 变量  |
| AtomicReferenceFieldUpdater | 原子化的更新指定类的volatile引用的field |

## 四、锁
### 1. synchronized关键字    

关于synchronized关键字的文章已经比较多了，可参考以下文章：

- 偏置锁

  https://blogs.oracle.com/dave/biased-locking-in-hotspot

  https://www.usenix.org/legacy/event/jvm01/full_papers/dice/dice.pdf

- synchronized关键字的全面描述，包括偏置锁相关的内容

  https://www.cnblogs.com/javaminer/p/3889023.html

  https://blog.csdn.net/u012465296/article/details/53022317

  https://blog.csdn.net/chenssy/article/details/54883355

### 2. Lock

#### 2.1 ReentrantLock

ReentrantLock是Lock接口的实现，ReentrantLock支持与synchronized一样的语义，包括可重入性，之所以创建ReentrantLock这么一个显式锁机制，主要是synchronized存在一些局限性，例如：无法在获取锁时取消或设置超时或获取失败立即返回，不支持公平锁（虽然绝大多数情况下出于性能考虑使用非公平锁）等。需要注意的是ReentrantLock和synchronized的性能上差距很小，因此出于简化程序的目的，应尽量避免使用ReentrantLock。

Lock的使用规范如下，一定要在finally块中释放锁，否则可能由于异常导致锁无法释放。

```java
class ReentrantLockDemo {
   private final ReentrantLock lock = new ReentrantLock();
   public void demoMethod() { 
     lock.lock();  // block until condition holds
     try {
       // ... method body
     } finally {
       lock.unlock()
     }
   }
}
```

ReentrantLock底层是基于AQS实现的，具体在第五节中描述。

#### 2.2 ReadWriteLock与ReentrantReadWriteLock

ReentrantReadWriteLock是ReadWriteLock接口的实现，是一个读写锁，在读多写少的并发场景下，使用读写锁可以提升性能。ReentrantReadWriteLock有以下特性：

- 写锁可以降级为读锁，读锁不能升级为写锁
- 写锁是互斥的，读锁是共享的

ReentrantReadWriteLock底层是基于AQS实现的，它使用AQS的state变量的高16位用作读锁，低16位用作写锁。

### 3. 条件队列

线程在某个条件不满足的情况下进入条件队列并释放锁，由另一个线程在某个条件满足的情况下唤醒处于条件队列中的线程。类似于提供了synchronized和Lock两种锁的实现，Java也提供了两种条件队列的实现。

| Object的内部条件队列                                         | Lock的Condition                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Object的wait、notify、notifyAll方法                          | Condition的await、signal、signalAll方法                      |
| 一个对象只有一个内部条件队列，多个条件的情况下使用一个对象进行wait、notify、notifyAll操作 | 一个Lock可以new多个Condition，对应不同的条件，分别进行await、signal、signalAll操作 |

注意Condition只是一个接口，需要Lock的具体实现类的newCondition方法提供实现。ReentrantLock的newCondition方法返回的是一个AQS中的ConditionObject类型的对象，第五节中会对ConditionObject的原理有解释，可以看到是通过一个链表实现的条件队列。

Condition的官方使用示例如下：

```java
class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = <b>lock.newCondition(); 
   final Condition notEmpty = <b>lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length)
         <b>notFull.await();
       items[putptr] = x;
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }

   public Object take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0)
         notEmpty.await();
       Object x = items[takeptr];
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
   }
 }
```

### 4. 关于锁的总结

参考文献【1】[^1]做了非常好的总结，推荐直接阅读该文章。 


## 六、多线程框架
### 1. Executor框架(JAVA 5)

<img src="/java/concurrency/executor.png" alt="executor" style="zoom: 33%;" />

Executor接口提供了execute方法，该方法将任务提交，并在之后的某个时间点执行该任务，具体执行策略取决于其具体实现。ExecutorService在Executor的基础上提供了管理Executor生命周期的方法，如shutDown, shutDownNow方法。ThreadPoolExecutor是Executor的实现类，实现了基于线程池的Executor框架。

#### 1.1 线程池

Executors类提供了生成ThreadPoolExecutor，ScheduledThreadPoolExecutor，ForkJoinPool的工厂方法：

| 生成线程池方法                   | 描述                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| newFixedThreadPool               | `new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());` |
| newSingleThreadExecutor          | `new FinalizableDelegatedExecutorService (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));` |
| newCachedThreadPool              | `new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());` |
| newScheduledThreadPool           | `new ScheduledThreadPoolExecutor(corePoolSize);`             |
| newSingleThreadScheduledExecutor | `new DelegatedScheduledExecutorService (new ScheduledThreadPoolExecutor(1));` |
| newWorkStealingPool              | `new ForkJoinPool (Runtime.getRuntime().availableProcessors(),      ForkJoinPool.defaultForkJoinWorkerThreadFactory,      null, true);` |

可以看到newFixedThreadPool，newSingleThreadExecutor，newCachedThreadPool方法返回的都是ThreadPoolExecutor对象，只不过配置不同；newScheduledThreadPool，newSingleThreadScheduledExecutor方法返回的都是ScheduledThreadPoolExecutor对象；newWorkStealingPool返回的是ForkJoinPool。

#### 1.2 线程池配置

##### 1.2.1 ThreadPoolExecutor

下面讲解ThreadPoolExecutor的配置参数，关于ForkJoinPool的配置详见下一小节。ThreadPoolExecutor的构造函数如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

| 参数            | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| corePoolSize    | 线程池中的最小线程数，线程池初始化后核心线程并不开始，除非调用了prestartAllCoreThreads |
| maximumPoolSize | 最大线程数                                                   |
| keepAliveTime   | 超过corePoolSize数量的线程的最长空闲时间                     |
| workQueue       | 任务的排队队列：有限队列、无限队列、同步移交                 |
| threadFactory   | 创建线程的工厂方法，一般直接采用默认的Executors类提供的DefaultThreadFactory |
| handler         | 当线程池已满且队列已满时的任务提交时的处理策略，有CallerRunsPolicy、AbortPolicy、DiscardPolicy、DiscardOldestPolicy |

##### 1.2.2 最佳的线程池大小

首先，如果存在不同类型的任务，且差别很大，比如计算密集型和I/O密集型任务，那么最好使用多个线程池。

最佳线程池大小的配置可以根据任务的类型大致如下计算：

- 计算密集型任务线程池大小：CPU个数+1
- I/O密集型任务线程池大小：CPU个数 * CPU目标使用率 * （1+等待时间与计算时间的比）

其他比如连接池大小、内存、文件句柄、套接字句柄等都会限制线程池大小

#### 1.3 Future, Callable

![future](/java/concurrency/future.png)

Runnable提供了run方法用于执行计算，Callable的call方法可以返回计算的执行结果。

Future描述了任务的生命周期，提供了方法获取任务执行的结果，取消任务、检查任务是否完成或取消。ExecutorService的submit方法接受Runnable或Callable并返回一个Future。FutureTask是Future的具体实现。

#### 1.4 CompletionService及其实现ExecutorCompletionService

CompletionService整合了Executor和BlockingQueue的功能，ExecutorCompletionService是其具体实现。提交到ExecutorCompletionService的任务被包装为一个QueueingFuture，覆盖了done方法，该方法在任务完成时将结果放入其BlockingQueue中。

#### 1.5 线程取消、线程池关闭、JVM关闭

##### 1.5.1 线程取消

线程取消有以下方式：

- 循环检查取消标志
- 中断：对于处于阻塞状态中的线程无法通过设置取消标志实现取消，中断机制提供了这种情况下的取消机制。每个线程有一个boolean类型的中断状态，中断时设置为true，即线程B调用线程A的interrupt方法时，线程A的中断状态被设置为true。阻塞库函数，例如，Thread.sleep或Object.wait通过native方法检测线程是否被中断，其对中断的响应表现为：清除中断状态并抛出异常InterruptedException，表示阻塞操作因中断提前结束
- 通过Future.cancel取消

##### 1.5.2 线程异常处理

- 可以在线程内部catch异常
- 线程API提供了UncaughtExceptionHandler，当线程因为未捕获异常退出时，该handler处理异常，如果handler不存在，默认行为是像System.err打印stack trace
- 只有通过execute方法提交的任务才能将抛出的异常传给异常处理器，通过submit方法提交的任务，只会被Future.get方法重新抛出为ExecutionException

##### 1.5.3 JVM的关闭

- Shutdown Hook

  Shutdown Hook是通过Runtime.addShutdownHook方法注册的尚未开始的线程。如果是通过调用Runtime.halt或者kill -9的方式强行关闭JVM，那么除了关闭JVM之外不需要完成任何其他动作，也不会运行Shutdown Hook；

  Shutdown Hook之间并发执行，不保证顺序，Shutdown Hook之行结束后，如果runFinalizersOnExit为true，JVM可以选择运行finalizer，之后停止；

  JVM不会停止或者中断应用线程，应用线程在JVM停止时强制退出。

- daemon线程

  当只有daemon线程时，JVM会发起退出，daemon线程会被抛弃，不会执行finally块，也不会释放栈

- Finalizer

  参见文章《Object.finalize()方法与Finalizer类浅析》，推荐的操作是：不要使用Finalier


ForkJoinTask 是 Fork/Join 框架中任务的抽象基类，RecursiveTask 和 RecursiveAction 均继承自它。ForkJoinTask 提供了 `fork()` 方法用于异步执行任务（将任务放入当前线程的 WorkQueue 中排队），以及 `join()` 方法用于等待任务执行完成并获取结果。

### 3. CompletableFuture (JAVA 8)


CompletableFuture 相关可先参考以下文章：

https://colobu.com/2016/02/29/Java-CompletableFuture/

https://www.nurkiewicz.com/2013/05/java-8-definitive-guide-to.html


## 七、活跃性问题、性能问题

### 1. 死锁

死锁最常见的场景是出现了环路的锁依赖关系。在数据库系统中，一般设计了死锁监测，通过检查表示锁依赖关系的有向图上是否存在环路，如果存在死锁，会选择一个事务退出。

不光是锁的使用会造成死锁，资源的使用也可能造成死锁，例如有两个数据库D1、D2的连接池，线程A持有到数据库D1的连接，等待D2的连接，线程B持有到数据库D2的连接，等待D1的连接，这就有可能造成死锁，若连接池大小为1，则一定发生死锁。

通过使用显示Lock的tryLock方法的带有timeout的版本，能够一定程度上避免死锁，至少在死锁发生的情况下能够通过超时进行回退。

thread dump能够进行死锁检测，可用于线上诊断。

### 2. 饥饿

饥饿问题是指当线程访问它需要的资源时被拒绝，不能继续进行，常见的是CPU资源的饥饿问题。线程优先级的使用不当、死循环都会造成CPU资源的饥饿问题

非公平锁也会造成线程的饥饿问题，特殊情况下，先尝试获取锁的线程反而没法抢到锁。

### 3. 其他

- 弱响应性问题：即响应时间较长

- 活锁：活锁问题一般发生在错误恢复机制中，例如，在消息处理应用程序中，如果对某种特定类型的消息处理存在bug，每次处理都会失败，失败后又被放回队首，下次还是处理这个消息，形成了死循环。可以通过在错误恢复机制中引入一定的随机性来避免着问题。

### 4. 性能

- 线程上下文切换：包括线程调度的花费、线程换入后CPU缓存数据的加载都是线程上下文切换带来的花费。

- 内存同步：synchronized、volatile等提供的可见性保证是通过使用内存屏障使CPU缓存无效化实现的，不能使用CPU缓存使得性能下降，并且内存屏障还能防止指令重排序，这就导致了编译器不能对代码执行进行优化。

  现在JVM中的JIT编译器通过逃逸分析能够实现锁消除的优化，如果一个变量不从线程内逸出，对其的加锁操作会被省略，或者通过锁粗化，即将邻近的synchronized块用相同的锁合并起来。这些JVM的优化机制表明对于没有竞争的同步代码，其开销已经经过很好的优化了，真正影响性能的是真正发生了锁竞争的代码。

- 阻塞

  获取锁的时候如果存在竞争会发生阻塞，这时候可以选择自旋或者线程挂起，取决于具体的场景。

- 如何减少锁的竞争
  - 缩小锁的范围：避免很大的同步块
  - 减小锁的粒度：通过锁的分拆或分离，将一个粗粒度的锁拆分为多个锁，例如ConcurrentHashMap相比HashTable通过更细粒度的锁提升了性能
  - 使用非独占锁：读写锁相比独占锁带来了性能提升

# 参考文献

[^1]:https://tech.meituan.com/2018/11/15/java-lock.html
[^2]:https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html
[^3]:https://blog.csdn.net/chenxiaoti/article/details/82776128
[^4]:https://www.jianshu.com/p/b9186dbebe8e
[^5]:https://www.jianshu.com/p/ca19c22e02f4
