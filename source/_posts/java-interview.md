---
title: 后端面试 FAQ
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - 面试
  - Java
  - Spring
  - 数据库
  - 分布式
categories:
  - Java
---
## 一、Java 基础

### 1. String / StringBuilder / StringBuffer 区别？

- **String**：不可变类，用 `final` 修饰，底层 `byte[]`（JDK 9 前是 `char[]`）。每次拼接都创建新对象，频繁拼接会产生大量临时对象，加重 GC 压力。常量字符串存放在字符串常量池中（JDK 7 后位于堆中），相同字面量复用同一对象。
- **StringBuilder**：可变，线程不安全，方法没有同步，单线程下性能最高。扩容时按 `原长度 * 2 + 2` 增长如果不够则直接扩到目标长度。
- **StringBuffer**：可变，线程安全，所有公开方法用 `synchronized` 修饰，多线程环境下保证安全但性能低于 StringBuilder。
- `intern()` 方法（native）：将字符串实例放入常量池并返回引用。JDK 7 后常量池在堆中，可直接存储对象引用而非拷贝。慎用：可能导致常量池过大，且早期的 `PermGen OOM` 问题。
- **面试延伸**：为什么 String 要设计成不可变？—— 安全性（类加载器、网络连接参数）、字符串常量池复用、HashMap 等容器 key 稳定性、多线程天然安全。

### 2. == 和 equals 的区别？hashCode 约定？

- `==`：比较栈内存中的值。基本类型比实际值，引用类型比较的是对象的内存地址（引用是否指向同一对象）。
- `equals()`：定义在 Object 类中，默认实现就是用 `==` 比较地址。子类可重写来实现自定义的"相等"逻辑（如 String 比较字符序列、Integer 比较数值）。
- **hashCode 约定（三原则）**：
  1. 同一对象多次调用 `hashCode()` 必须返回相同值（前提是 equals 比较用到的信息没变）。
  2. 两个对象 `equals` 返回 true → `hashCode` 必须相等。
  3. 两个对象 `equals` 返回 false → `hashCode` 不要求不同，但不同则能提升哈希表性能。
- 重写 equals 必须同步重写 hashCode，否则会导致 HashMap/HashSet 出现逻辑 bug：同一对象存进去，按同样 key 却取不出来。
- HashMap 查找流程：`hashCode()` → 寻址 → `==` 比较引用 → 若不同再调用 `equals()`。两个 hashCode 冲突的 key 在同一个桶中以链表/红黑树存储。

### 3. Java 异常体系（受检异常 vs 非受检异常）

- Throwable 是根类，其下分为两个分支：
  - **Error**：JVM 层面严重错误，程序无法处理也不应捕获。如 `OutOfMemoryError`、`StackOverflowError`、`NoClassDefFoundError`。
  - **Exception**：程序可以处理的异常，又细分为两类：
- **受检异常（Checked Exception）**：继承 `Exception` 但不继承 `RuntimeException`。编译器强制要求 `try-catch` 或 `throws` 声明，否则编译不通过。典型如 `IOException`、`SQLException`、`ClassNotFoundException`。
- **非受检异常（Unchecked Exception）**：继承 `RuntimeException`，编译器不做检查。典型如 `NullPointerException`、`IndexOutOfBoundsException`、`IllegalArgumentException`。
- **实际最佳实践**：业务异常建议继承 `RuntimeException`，然后用全局异常处理器（`@ControllerAdvice` + `@ExceptionHandler`）统一拦截、记录日志、返回规范错误码，避免到处 try-catch。
- try-catch-finally 中，finally 块一定执行（除非调用了 `System.exit(0)` 或 JVM 崩溃），return 也无法阻止。finally 中有 return 会覆盖 try 中的 return（不推荐）。

### 4. 反射的原理与使用场景？

- **原理**：JVM 在类加载阶段将类的结构信息（字段、方法、构造器、注解）存入方法区，并为每个类生成唯一的一个 `Class` 对象。反射 API 通过这些元数据进行动态操作。
- **获取 Class 对象三种方式**：`Class.forName("全限定名")` 会触发类加载（初始化）；`对象.getClass()`；`类名.class`（只加载不初始化）。
- **核心操作**：
  - 创建实例：`clazz.newInstance()`（JDK 9 已废弃）→ `clazz.getDeclaredConstructor().newInstance()`。
  - 获取字段：`getField` vs `getDeclaredField`（前者只拿 public，后者拿全部）。私有字段需 `setAccessible(true)`。
  - 获取方法：`getMethod` vs `getDeclaredMethod`，调用 `method.invoke(target, args)`。
  - 性能开销来源：`Class.getMethod()` 遍历方法数组需要时间 + 安全检查 + 参数装箱拆箱 + 方法内联受限。
- **优化**：`setAccessible(true)` 关闭访问检查可提速约 10-20 倍；缓存 Method/Field 对象避免反复查找。
- **使用场景**：Spring 的依赖注入、AOP 动态代理、MyBatis 结果映射、Lombok 注解处理、JUnit 测试框架、序列化框架（Jackson/Gson）。

### 5. 泛型类型擦除

- Java 泛型是**编译期**概念，编译后泛型信息被擦除，字节码中不存在泛型类型。
- 擦除规则：
  - 无限制的 `<T>` → 擦除为 `Object`。
  - 有限定的 `<T extends Number>` → 擦除为 `Number`。
  - 编译器在需要的地方自动插入强制类型转换（`checkcast` 字节码指令）。
- 带来的限制：
  - 不能 `new T()` 或 `new T[]`（编译期不知道 T 的具体类型）。
  - 不能 `instanceof List<String>`（运行时无此信息，只能检查到 List 级别）。
  - static 方法/字段不能引用类的泛型参数（泛型参数与实例绑定）。
  - 方法签名冲突（`void foo(List<String>)` 和 `void foo(List<Integer>)` 编译后参数类型都是 List，视为同一方法，编译报错）。
- **泛型信息残留**：虽然类型变量被擦除，但类、字段、方法的签名中保留了泛型声明信息，可通过反射获取：`((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()`。这也是 Gson 的 `TypeToken` 和 Spring `RestTemplate` 能工作的原理。

### 6. Java 是值传递还是引用传递？

- **Java 只有值传递，没有引用传递。**
- 传递基本类型时，传递的是值的拷贝。
- 传递对象时，传递的是**对象引用地址的拷贝**（不是对象本身）。通过这个拷贝可以修改对象的成员变量，但无法改变外部变量引用的指向。
- 区分关键：如果支持引用传递，则 `swap(objA, objB)` 交换形参的引用后，外部的两个引用也能交换指向。Java 做不到这一点，证明是值传递。
- 常被混淆的场景：
  ```java
  public static void change(User u) { u.setName("new"); }  // 修改的是引用指向的对象内容，可行
  public static void swap(User a, User b) { User t = a; a = b; b = t; } // 交换的是形参引用，外部不会变
  ```
- JVM 层面：Java 操作对象一律通过引用（reference），这个 reference 在方法调用时被复制一份传入。

### 7. final / finally / finalize

| 关键字 | 作用 | 使用位置 | 备注 |
|--------|------|---------|------|
| **final** | 不可变/不可继承 | 类/方法/变量 | 修饰基本类型值不变；修饰引用类型引用不变、对象内容可变；搭配 static 即为编译期常量 |
| **finally** | 异常处理必执行的代码块 | try-catch-finally | 通常用于释放资源（JDK 7+ 建议用 try-with-resources 替代，需实现 AutoCloseable） |
| **finalize** | GC 回收前最后调用的方法 | Object 类方法 | JDK 9 已 Deprecated，执行时机不确定，优先级低，可能导致对象迟迟不能回收，绝对不推荐使用 |

- **final 修饰局部变量与成员变量的区别**：局部变量可先声明后赋值一次；成员变量（非 static）必须在构造器结束前完成赋值；static final 必须在静态初始化块或声明时赋值。
- final 方法在编译期可能被 JIT 内联优化。
- **finally 不执行的唯一情况**：`System.exit(0)` 退出 JVM 或线程被强制 kill。return 之前会先执行 finally 代码。

### 8. 深拷贝与浅拷贝

- **浅拷贝**（Shallow Copy）：只复制对象本身（基本类型字段复制值，引用类型字段复制引用地址），新旧对象共享内部引用对象。`Object.clone()` 默认实现就是浅拷贝。
- **深拷贝**（Deep Copy）：递归复制所有引用类型字段，新旧对象完全独立，不共享任何内部对象。
- **实现方式**：
  1. 实现 `Cloneable` 接口，重写 `clone()` 方法，递归对每个引用字段调用 clone。
  2. 利用序列化/反序列化：`ByteArrayOutputStream` → `ObjectOutputStream.writeObject(obj)` → `ObjectInputStream.readObject()`（需实现 Serializable，性能较差但实现简单、保证完整深拷贝）。
  3. 第三方工具：Apache Commons `SerializationUtils.clone()`，Spring `BeanUtils.copyProperties()`（只做浅拷贝）。
- `clone()` 不会调用构造器，而反序列化会调用第一个无参构造器（非 Serializable 父类）。
- **实际开发中的考量**：深拷贝开销大，能避免则避免（用不可变对象、共享只读对象）。

### 9. HashMap 的实现原理（JDK 8）

- **底层结构**：数组（Node[] table）+ 链表 + 红黑树。
- **put 流程**：
  1. 对 key 的 hashCode 做高 16 位与低 16 位异或（扰动函数），减小高位不相同但低位相同导致的碰撞。
  2. 计算桶位置：`(n - 1) & hash`，n 为数组长度（始终保持 2 的幂，便于位运算取模）。
  3. 桶为空 → 直接放入；桶非空 → 链表遍历，key 相同则替换，到达链表末尾后在尾部插入（JDK 7 是头插，死循环数据丢失问题）。
  4. 链表长度 ≥ 8 且数组长度 ≥ 64 → 转换为红黑树。树节点 ≤ 6 时退化为链表（差值 2 是为了防止反复转换）。
  5. 插入后判断 size > threshold（capacity × loadFactor）→ 扩容。
- **扩容**（resize）：
  - 容量翻倍 → 新数组 → 遍历旧数组迁移数据。
  - JDK 8 优化：不再重新计算 hash。利用扩容后容量的二进制特征，旧位置元素分两类：`hash & oldCap == 0` 留在原位，`== 1` 则移到 `原索引 + oldCap`。避免了 JDK 7 每次 rehash 的性能开销和死循环。
  - 负载因子 0.75：在时间和空间上的折中。太小 → 空间浪费，太大 → 碰撞增多降低查询效率。
- `get` 流程：同样 hash 找桶 → 先判断头节点是否匹配 → 是红黑树则调用 `getTreeNode` → 否则遍历链表。
- **线程不安全的体现**：JDK 7 并发扩容可能形成环形链表导致 CPU 100%；JDK 8 解决了死循环但仍存在数据覆盖问题。并发场景务必使用 ConcurrentHashMap。

### 10. ArrayList 和 LinkedList 的区别？

- **ArrayList**：底层 `Object[]`，随机访问 O(1)（内存连续 + CPU 缓存友好）。add 操作均摊 O(1)，扩容时 1.5 倍（`newCapacity = oldCapacity + (oldCapacity >> 1)`），需要复制数组 O(n)。中间插入/删除需移动元素 O(n)。实现了 `RandomAccess` 标记接口，遍历时 `for (int i)` 比 `for-each` 通过迭代器略快。
- **LinkedList**：底层双向链表（`Node {E item; Node prev; next}`）。头尾操作 O(1)（使用 first/last 指针），随机访问 O(n)（需要从头/尾开始遍历）。实现了 `Deque` 接口，可作队列、双端队列、栈使用。
- **实际选择**：绝大多数场景用 ArrayList（内存更紧凑、遍历快、现代 CPU 缓存预取友好）。只有需要频繁在头部插入或删除时才考虑 LinkedList（实际上此类场景很少）。
- ArrayList 初始容量可在构造时指定，避免频繁扩容；默认容量 10。`ensureCapacity` 可预分配空间。

---

## 二、Java 并发

### 1. synchronized 锁升级过程（JDK 8+）

- Mark Word 是对象头的一部分（64 位 JVM 中占 64 bit），记录了锁状态、hashCode、GC 年龄。
- 锁状态流转：**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**。只能升级不能降级（降级仅在 GC safepoint 可能发生）。
- **偏向锁**：
  - 目的：减少同一线程不断获取同一把锁的开销（CAS 替换 Mark Word 变为仅检查 thread ID）。
  - Mark Word 记录持有线程 ID，该线程再次进入时检查 ID 一致即可，无需 CAS。
  - 撤销：另一线程尝试获取锁时，在安全点暂停偏向线程，检查其是否仍在同步块 → 撤销偏向 → 升级为轻量级锁。
  - JDK 15 默认关闭偏向锁，JDK 18 已废弃（高并发下撤销成本高于收益）。
- **轻量级锁**：
  - 线程在自己的栈帧中创建 Lock Record，通过 CAS 尝试将对象的 Mark Word 替换为指向 Lock Record 的指针。
  - 成功 → 获取锁；失败 → 自旋重试（自适应自旋：根据上次自旋是否成功和时间动态调整自旋次数）。
  - 适合**锁持有时间短、线程竞争不激烈**的场景。
- **重量级锁**：
  - 轻量级锁自旋超过一定次数（或竞争较多）后膨胀。基于操作系统的 Monitor（每个 Java 对象内置）。
  - Monitor 基于 `mutex` + `condition variable`，涉及用户态到内核态的切换，线程阻塞/唤醒开销大。
  - 但线程阻塞时不占 CPU，适合**锁持有时间长、竞争激烈**的场景。

### 2. volatile 的底层原理与应用

- **保证可见性**：对 volatile 变量的写操作，JVM 发送 Lock 前缀指令给 CPU：
  1. 将当前处理器的缓存行写回系统内存。
  2. 该写回操作会使其他 CPU 缓存了该内存地址的数据失效（MESI 协议的 Invalidate 消息）。
  3. 其他处理器再次读取该值时必须从主内存重新加载。
- **禁止指令重排序**：JVM 通过**内存屏障**实现：
  - `LoadLoad` / `StoreStore` / `LoadStore` / `StoreLoad` 四种屏障。
  - volatile 写之前加 StoreStore 屏障（禁止前面的普通写与 volatile 写重排），写之后加 StoreLoad 屏障（禁止后面的读与 volatile 写重排）。
  - volatile 读之后加 LoadLoad 和 LoadStore 屏障。
- **不保证原子性**：`count++`（读-改-写）即使 count 是 volatile，多线程下仍有问题。需要用 `synchronized` 或 `AtomicInteger`。
- **经典应用——DCL 单例**：
  ```java
  class Singleton {
      private static volatile Singleton instance;
      public static Singleton getInstance() {
          if (instance == null) {          // 第一重检查
              synchronized (Singleton.class) {
                  if (instance == null)    // 第二重检查
                      instance = new Singleton(); // 1.分配内存 2.初始化对象 3.赋值给 instance
              }                            // volatile 防止 2 和 3 重排序导致的半初始化对象
          }
          return instance;
     }
  }
  ```
  此外 volatile 还用于：**状态标志**（如 shutdown flag）、**CAS 原子操作中的变量**（Atomic 类内部 value 是 volatile 的）。

### 3. AQS（AbstractQueuedSynchronizer）原理

- AQS 是 JUC（java.util.concurrent）包的核心基石。ReentrantLock、Semaphore、CountDownLatch、ReentrantReadWriteLock、ThreadPoolExecutor（Worker）都基于它。
- **核心数据结构**：
  - **state（int）**：同步状态，通过 CAS + volatile 保证可见性和原子性。
  - **CLH 变种队列**：FIFO 双向链表，每个节点（Node）对应一个等待线程，包含 `waitStatus`（0, SIGNAL, CANCELLED, CONDITION, PROPAGATE）。
- **独占式获取流程**（ReentrantLock 非公平锁）：
  1. CAS 尝试将 state 从 0 设为 1，成功则设当前线程为独占线程，直接获取锁。
  2. 失败则调用 `acquire(1)` → `tryAcquire`（由子类实现，先检查 state==0 则再次 CAS；如果当前线程已是独占线程则重入 state++）→ 失败则新建 Node 入队尾（CAS 设置）。
  3. 入队后，若前驱节点是 head 且 tryAcquire 成功 → 出队，设为 head。
  4. 否则判断是否需要 park：前驱节点 waitStatus==SIGNAL → `LockSupport.park(this)` 阻塞。
- **共享式获取流程**（Semaphore）：
  - `tryAcquireShared` 返回剩余资源数，≥0 成功，<0 失败。成功后会尝试唤醒后继共享节点（连锁唤醒）。
- **释放**：`tryRelease` 将 state 还原 → `unparkSuccessor` 唤醒后继节点（从 tail 往前找最前面一个非 CANCELLED 的节点）。
- **Condition**：内部维护单向条件等待队列。`await()`：加入条件队列 → 释放锁 → park 等待；`signal()`：把条件队列的第一个节点移到 CLH 队列中等待获取锁。

### 4. ThreadLocal 的原理与内存泄漏

- **核心实现**：每个 Thread 内部持有一个 `ThreadLocalMap`（Entry[] 数组），key 是对 ThreadLocal 的**弱引用**，value 是实际存储的值。采用**线性探测法**解决冲突（而非链地址法）。
- 弱引用原因是：如果 key 是强引用，只要 ThreadLocal 对象被回收不了（因为 ThreadLocalMap 持有强引用 key），可能导致 Entry 无法回收。弱引用 key 允许 ThreadLocal 对象在外部没有引用时被 GC 回收。
- **内存泄漏原因**：虽然 key 是弱引用可以被回收，但 value 是强引用不会被回收。在线程池环境下，线程存活时间长，ThreadLocalMap 不会被销毁，key==null 的 Entry 永远存在，value 无法被释放 → 内存泄漏。
- **JDK 的应对**：
  - `set()`、`get()`、`remove()` 方法在操作过程中遇到 key==null 的 Entry 会做启发式清理（清理当前桶及附近几个桶）和 expungeStaleEntry（清理并 rehash 后续未过期的 Entry）。
  - 但依赖于调用 get/set 的时机，不够可靠。
- **最佳实践**：**用完必须手动调用 `remove()`**，建议放在 finally 块中。阿里规范强制要求：ThreadLocal 必须回收。
- **InheritableThreadLocal**：子线程可继承父线程的 ThreadLocal 值（构造子线程时浅拷贝）。线程池中慎用，因为线程复用导致值不会被重新拷贝。

### 5. 线程池的 7 个核心参数与执行流程

- **构造函数（7 个参数）**：
  ```java
  ThreadPoolExecutor(
      int corePoolSize,        // 常驻核心线程数
      int maximumPoolSize,     // 最大线程数
      long keepAliveTime,      // 非核心线程空闲存活时间
      TimeUnit unit,
      BlockingQueue<Runnable> workQueue,   // 任务缓冲队列
      ThreadFactory threadFactory,         // 工厂用于创建线程可自定义名称
      RejectedExecutionHandler handler     // 拒绝策略
  );
  ```
- **详细执行流程**：
  1. 当前运行的线程数 < corePoolSize → **直接创建新线程**执行当前任务（即使有空闲核心线程也创建，这保证了核心线程先满）。
  2. 核心线程数满了 → 任务进入 **workQueue** 排队等待。
  3. workQueue 也满了 & 线程数 < maximumPoolSize → **创建临时非核心线程**执行（此时新提交的任务直接分配给新线程而非队列中的任务，即"插队"）。
  4. workQueue 满 & 线程数 = maximumPoolSize → 执行 **拒绝策略**。
- **四种拒绝策略**：
  - **AbortPolicy**（默认）：抛 `RejectedExecutionException` 异常，调用方需自行处理。
  - **CallerRunsPolicy**：由提交任务的调用线程执行该任务。可以降低新任务的提交速度，起到反压效果。
  - **DiscardPolicy**：静默丢弃该任务。
  - **DiscardOldestPolicy**：丢弃队列中最老的任务，然后重新提交当前任务。
- **常用线程池快捷创建（Executors 静态方法）——阿里规范不推荐使用**：
  - `newFixedThreadPool`：corePoolSize==maxPoolSize，LinkedBlockingQueue（无界！可能导致 OOM）。
  - `newCachedThreadPool`：core=0，max=Integer.MAX_VALUE，SynchronousQueue，60s keepAlive。无限创建线程导致系统资源耗尽。
  - `newSingleThreadExecutor`：core=max=1，LinkedBlockingQueue 无界。
- **线程数估算**：CPU 密集型 = `N_CPU + 1`；IO 密集型 = `N_CPU * 2` 或 `N_CPU * (1 + 平均等待时间 / 平均计算时间)`。实际还需压测确定。

### 6. 线程六种状态及转换

- **NEW**：尚未调用 `start()`，Thread 对象刚创建。
- **RUNNABLE**：包含 JVM 层面的 Ready（就绪，等待 CPU 调度）和 Running（正在执行）。Java 将两者合并为 RUNNABLE。
- **BLOCKED**：等待获取 monitor 锁（synchronized），等待期间不响应 `interrupt()`。
- **WAITING**：无超时的等待，需其他线程显式通知唤醒：
  - `Object.wait()` → 等待 `Object.notify()/notifyAll()`
  - `Thread.join()` → 等待目标线程结束
  - `LockSupport.park()` → 等待 `LockSupport.unpark()` 或中断
- **TIMED_WAITING**：有限时间等待，到期自动唤醒：
  - `Thread.sleep(timeout)`（不释放锁！）
  - `Object.wait(timeout)`（释放锁）
  - `Thread.join(timeout)`
  - `LockSupport.parkNanos(timeout)`
- **TERMINATED**：`run()` 方法执行完毕或异常退出。
- 易混淆点：
  - BLOCKED vs WAITING：BLOCKED 是被动等待（等待进入 synchronized），WAITING 是主动等待（调用了 wait/join/park）。
  - `sleep()` 不会释放锁，`wait()` 会释放锁。
  - `Object.wait()` 必须在 synchronized 块中调用，否则抛 `IllegalMonitorStateException`。
  - `LockSupport.park()` 不需要配合 synchronized 使用。

### 7. 死锁的四个必要条件、排查与避免

- **四个必要条件**（缺一不可）：
  1. **互斥条件**：资源同时只能被一个线程持有。
  2. **持有并等待**：线程已经持有一个资源，又在等待获取另一个资源。
  3. **不可剥夺**：资源不能强行从线程手中夺走，只能由持有的线程主动释放。
  4. **环路等待**：存在线程-资源的环形等待链（T1 等 T2 的锁，T2 等 T1 的锁）。
- **排查命令与工具**：
  - `jps -l`：列出所有 Java 进程和主类名。
  - `jstack <pid>`：打印线程堆栈，末尾会直接打印 `Found one Java-level deadlock` 及死锁线程详细信息。
  - 可视化工具：JConsole → 线程 tab → 检测死锁按钮；VisualVM、Arthas（`thread -b` 一键定位死锁线程）。
- **避免死锁**：
  - **固定加锁顺序**：所有线程按相同的全局顺序获取锁（如对两个对象的 hash 排序后获取）。
  - **尝试获取锁**：`tryLock(long timeout, TimeUnit unit)` 超时后放弃持有的锁，回退重试。
  - **避免锁的嵌套**：减少一个线程同时持有多个锁的情况。
  - **使用无锁数据结构**：如 ConcurrentHashMap、Atomic 类、无锁队列（Disruptor）。

### 8. CompletableFuture 对比 Future 的优势

- **Future 的痛点**：
  - `get()` 阻塞等待，性能差，无法设置回调。
  - 多个 Future 之间难以编排：A 的结果作为 B 的输入、A 和 B 一起等完成后处理 C。
  - 缺乏异常处理机制（get 阻塞时要么一直等待要么超时抛异常）。
- **CompletableFuture 核心能力**：
  - **异步回调链**：`thenApply`（有参有返回值）、`thenAccept`（有参无返回）、`thenRun`（无参无返回）。纯异步非阻塞。
  - **组合编排**：`thenCombine(a, b, bifunc)` 等两个结果都完成；`thenCompose(f, fn)` 等第一个完成后再触发第二个异步任务（类似 flatMap，避免嵌套 Future<Future<T>>）。
  - **多任务聚合**：`allOf(cf1, cf2, ...)` 全部完成；`anyOf(cf1, cf2, ...)` 第一个完成。
  - **异常处理**：`exceptionally`（捕获异常转默认值）、`handle`（无论是否异常，都能拿到结果和异常）。
  - **自定义线程池**：`supplyAsync(supplier, executor)` 必须指定线程池，避免默认使用公共的 `ForkJoinPool.commonPool()`。
- **内部实现**：
  - 基于栈结构（Treiber Stack）存储依赖操作。每个 CF 保存一个 `Completion` 栈链表，完成任务后 CAS 的方式执行后续依赖。
  - `complete` / `completeExceptionally` 可将外部结果手动注入到 CF 中（用于包装已有回调式 API）。

### 9. ConcurrentHashMap 实现原理（JDK 8）

- **与 JDK 7 完全不同**：
  - JDK 7：分段锁（Segment × 16个，每个 Segment 内一小 HashMap + ReentrantLock），并发度 16。
  - JDK 8：抛弃 Segment，改用 **CAS + synchronized + 节点锁**，锁粒度细化到桶级别，并发度 = 桶数量。
- **put 详细流程**：
  1. 计算 hash，若 table 未初始化 → CAS + 自旋 + `initTable()` 完成初始化（只允许一个线程执行，通过 `sizeCtl` 状态控制）。
  2. 若目标桶位置为空 → CAS 尝试直接放入（`casTabAt`），成功即返回。
  3. 若目标位置节点的 hash == MOVED（-1）→ 当前正在扩容，当前线程帮助扩容（`helpTransfer` 协同迁移数据）。
  4. 否则 → `synchronized` 锁定桶的头节点（锁定的是 Node 对象本身，不是整个 Segment），插入链表尾部（`binCount` 累加）。
  5. 若 binCount ≥ 8 → 调用 `treeifyBin`（内部会再检查数组长度 ≥ 64 才树化，否则优先扩容）。
  6. `addCount(1L, binCount)` → 增加计数并检查是否需要扩容。
- **多线程协同扩容（transfer）**：
  - 分配 stride（每个线程负责迁移的桶数，最少 16）。
  - 通过 `transferIndex`（CAS 递减）从右往左分配。
  - 迁移完成的桶设置 ForwardingNode（hash == MOVED），其他线程 see 到 ForwardingNode 就跳过。
- **size 计数原理**：
  - `baseCount` + `CounterCell[]` 数组分散计数（类似 LongAdder 的 Striped64 方案），减少单变量 CAS 并发冲突。
  - `sumCount()` 时累加 baseCount + 所有 CounterCell 的值。

---

## 三、JVM

### 1. JVM 内存结构（JDK 8 运行时数据区）

- **堆（Heap）**：所有线程共享，存放对象实例和数组。GC 主要工作区。
  - 分代结构（G1/ZGC 使用的是 Region 非传统分代）：
    - **年轻代** = Eden（默认占年轻代 8/10）+ Survivor0 + Survivor1（各 1/10）。对象优先在 Eden 分配。
    - **老年代**：经过一定次数 Minor GC 仍存活的对象，或大对象直接进入。
- **方法区 / 元空间（Metaspace）**：所有线程共享，存储类的元信息（类型信息、常量、静态变量、即时编译后的代码缓存）。
  - JDK 7 及以前：永久代（PermGen，在堆内，`-XX:MaxPermSize` 配置）。
  - JDK 8+：**元空间**（使用本地内存，默认不限制大小，`-XX:MaxMetaspaceSize` 可选限制）。好处是避免了 `OOM: PermGen`，且元空间在内存紧张时可被操作系统回收。
  - 运行时常量池：Class 文件常量池的运行时表示，JDK 7 后从永久代移到堆。
- **虚拟机栈**：线程私有，每个方法对应一个栈帧。栈帧内容包括：
  - **局部变量表**：存储方法参数和局部变量。
  - **操作数栈**：字节码指令执行的计算栈。
  - **动态链接**：指向运行时常量池中该方法的引用。
  - **返回地址**：方法调用的下一条字节码指令地址。
- **本地方法栈**：线程私有，为 Native 方法（C/C++ 实现）服务。
- **程序计数器（PC 寄存器）**：线程私有，指向当前线程执行的字节码指令地址。JVM 中唯一不会 OOM 的区域。
- **直接内存**（不属于 JVM 运行时数据区）：NIO 的 `ByteBuffer.allocateDirect()` 分配在堆外内存。受 `-XX:MaxDirectMemorySize` 限制，默认随 MaxHeapSize。回收依赖 GC 时 Cleaner 虚引用机制。

### 2. 类加载过程与双亲委派

- **类加载的五个阶段（加载/验证/准备/解析/初始化）**：
  1. **加载**：通过全限定名获取二进制字节流 → 转为方法区运行数据结构 → 在堆中生成 Class 对象。加载源：class 文件、jar、网络、动态生成（Proxy、CGLIB）。
  2. **验证**：确保 Class 文件字节流符合规范（文件格式、元数据、字节码、符号引用验证），保障 JVM 安全。可开启 `-Xverify:none` 跳过以加速（不推荐）。
  3. **准备**：为静态变量分配内存并赋零值（`static final` 修饰的常量除外，直接赋值编译时确定的值）。如 `static int a = 123` 准备阶段 a = 0，初始化阶段才 = 123。
  4. **解析**：将常量池中的符号引用替换为直接引用（类/接口/字段/方法/方法类型的符号引用）。
  5. **初始化**：执行类的初始化方法 `<clinit>()`，按代码顺序执行静态变量赋值和静态代码块。父类先于子类。
- **双亲委派模型**：
  - 启动类加载器（Bootstrap）：加载 `<JAVA_HOME>/lib` 下的核心类库，C++ 实现，无法在 Java 中直接引用。
  - 扩展类加载器（Extension）：加载 `<JAVA_HOME>/lib/ext`，JDK 9 被模块化加载替代。
  - 应用程序类加载器（Application）：加载 `classpath` 下的用户类。
  - 自定义类加载器：继承 ClassLoader，重写 `findClass()` 而非 `loadClass()`。
- **双亲委派执行流程**：类加载器收到请求 → 检查是否已加载 → 未加载则委托给父加载器 → 父类无法加载时，子加载器自己 `findClass()` 加载。
- **破坏双亲委派的场景**：
  - JDBC（SPI 机制）：`java.sql.DriverManager` 由 Bootstrap 加载，但其 SPI 实现的 Driver 由 App ClassLoader 加载。通过**线程上下文类加载器（TCCL）**解决：Bootstrap 通过 `Thread.getContextClassLoader()` 获取应用类加载器去加载实现类。
  - Tomcat：每个 Web App 有自己的 WebAppClassLoader 隔离开来，先自己加载再委派。
  - OSGi、热部署、模块化场景。

### 3. 判断对象是否可回收

- JVM 使用**可达性分析算法**（而非引用计数法，引用计数无法解决循环引用）。
- 以一系列 GC Roots 对象为起点，从这些根对象出发沿着引用链搜索，所走过的路径称为**引用链（Reference Chain）**。从 GC Roots 到这个对象不可达，则对象可回收。
- **可作为 GC Roots 的对象**：
  1. 虚拟机栈（栈帧的局部变量表）中引用的对象。
  2. 方法区中类静态变量引用的对象。
  3. 方法区中常量引用的对象（字符串常量池中的引用）。
  4. 本地方法栈中 JNI 引用的对象。
  5. Java 虚拟机内部的引用（基本数据类型对应的 Class 对象、常驻的异常对象、系统类加载器）。
  6. 所有被同步锁（synchronized）持有的对象。
  7. 反映 Java 虚拟机内部情况的 JMXBean、JVMTI 中注册的回调等。
- **两次标记过程**：
  1. 第一次标记：不可达且未覆盖 `finalize()` 方法或 `finalize()` 已被调用过 → 直接回收。
  2. 若覆盖了 `finalize()` 且未执行过 → 放入 F-Queue，由低优先级 Finalizer 线程执行 `finalize()`。若在 `finalize()` 中重新与引用链建立关联（如将自己赋给某个 GC Root 的静态变量），则可逃脱本次回收。**但绝不建议依赖此机制**。
- 不可达的对象不一定会被立刻回收，处于"缓刑"阶段。真正宣告一个对象死亡需要经过至少两次标记。

### 4. CMS / G1 / ZGC 三大收集器对比

- **CMS（Concurrent Mark Sweep）**：JDK 8 及以前常用，JDK 14 已移除。
  - 目标：最短停顿时间，适合对响应时间敏感的应用。
  - 四阶段：
    1. **初始标记**（STW）：标记 GC Roots 能直接关联到的对象。
    2. **并发标记**：从直接关联对象遍历整个对象图，与用户线程并发执行。
    3. **重新标记**（STW）：修正并发标记期间因用户线程运行导致的标记变动（使用增量更新算法）。
    4. **并发清除**：并发清理死亡对象，不需要 STW。
  - **缺点**：采用标记清除算法 → 产生内存碎片 → 碎片严重时触发 Serial Old 全堆收集（单线程，STW 长）；无法处理"浮动垃圾"（并发清除过程中新产生的垃圾），可能在 CMS 期间又触发 Full GC（Concurrent Mode Failure，通过 `-XX:CMSInitiatingOccupancyFraction` 预留空间）；对 CPU 敏感（并发阶段会占用部分线程）。

- **G1（Garbage First）**：JDK 9 默认 GC，JDK 8 开始可用（-XX:+UseG1GC）。
  - 核心思想：将堆分为大小相等的 Region（1~32MB），不再物理离散分代。每个 Region 可以扮演 Eden/Survivor/Old/Humongous（大对象）。
  - 回收策略：**优先回收垃圾最多的 Region（Garbage First）**，根据用户设定的预期停顿时间（`-XX:MaxGCPauseMillis`）选择性回收。
  - 回收类型：
    - **Young GC**：STW，回收 Eden + Survivor，存活对象晋升或复制到新 Survivor。
    - **Mixed GC**：STW，回收所有 Young Region + 部分 Old Region（按回收价值排序取前 N 个，保证在用户设定的停顿时间限制内完成）。
    - **Full GC**：退化到 Serial Old 单线程收集，出现说明 GC 调优不够好。
  - 关键技术：**写屏障**记录跨 Region 引用（Remember Set/RSet 避免全堆扫描）；SATB（Snapshot-At-The-Beginning）算法维护并发标记阶段的引用一致性。

- **ZGC**：JDK 11 实验性，JDK 15 正式发布，目标停顿 < 1ms。
  - 核心技术：**染色指针（Colored Pointers）** + **读屏障（Load Barrier）** + **并发重映射**。
  - 染色指针：在 64 位指针中利用高位来存储对象状态标记（Marked0/Marked1/Remapped/Finalizable），省去了额外元数据存储和内存开销。
  - 回收阶段全部并发：标记 → 预备重分配 → 并发重分配（并发转移对象内存）→ 并发重映射（修改被转移对象的引用）。唯一 STW 只在初始标记和最终标记极短时间内。
  - 缺点：依赖操作系统物理内存的地址映射支持（目前主要是 Linux）；由于染色指针需要拦截指针访问，有一定吞吐量损耗。

| 对比维度 | CMS | G1 | ZGC |
|---------|-----|----|-----|
| 算法 | 标记清除 | 标记整理（Region） | 标记整理（Region + 染色指针） |
| 分数代 | 物理分代 | 逻辑分代（Region） | 逻辑分代 |
| STW 时长 | 数十~数百 ms | 可预测（毫秒级） | < 1ms |
| 碎片问题 | 有 | 无（整理回收） | 无 |
| 目标堆大小 | 4~8G | 4~TBs | TB 级别 |
| 内存开销 | 低 | 中等（RSet） | 低（染色指针复用指针位） |
| 适用场景 | 低延迟中小堆 | 大内存可预测停顿 | 超低延迟大内存 |

### 5. Full GC 触发条件与排查手段

- **Full GC 触发条件**（年轻代 GC + 老年代 GC 一起发生，或单独老年代 GC）：
  1. **空间分配担保失败**：年轻代存活对象要晋升到老年代，但老年代剩余空间 < 晋升对象总大小（或小于历次晋升的平均值）→ 触发 Full GC。JDK 6u24 之后只要老年代连续空间大于晋升对象总和或历次平均即可，不要求总空间够。
  2. **老年代空间不足**：大对象直接要进老年代，老年代放不下。
  3. **元空间不足**：Metaspace 使用量达到 `MaxMetaspaceSize`。
  4. **显式调用 System.gc()**：只建议出发，JVM 不一定执行。`-XX:+DisableExplicitGC` 可禁用。
  5. **CMS 中 Concurrent Mode Failure**：并发清除时老年代空间增加到阈值，触发 Full GC 兜底。
  6. **晋升失败（Promotion Failed）**：Minor GC 时 Survivor 空间不够且老年代空间不够。
- **排查工具与步骤**：
  1. **`jstat -gcutil <pid> 1000`**：每秒输出各区使用比例、GC 次数和累计时间。关注 FGC（Full GC 次数）和 FGCT（Full GC 耗时）。
  2. **`jmap -heap <pid>`**：查看堆内存整体配置和当前各代使用情况。
  3. **`jmap -histo:live <pid>`**：触发一次 Full GC 后（慎用线上），输出存活对象的统计排行，看到底哪些对象占内存多。
  4. **生成 dump 文件**：`jmap -dump:format=b,file=heap.hprof <pid>` 或添加参数 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path`。
  5. **MAT / JProfiler 分析 dump**：Histogram 看大对象 → Dominator Tree 找到最大持有者 → GC Roots 引用链分析找到 GC Root 路径 → 确定是内存泄漏还是正常大对象。
  6. **Arthas**：在线排查，不用 dump 全堆，`heapdump` + OQL，`vmtool` 强制 GC。
  7. **GC 日志**：`-Xlog:gc*`（JDK 11+）或 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps`，分析 GCViewer/gceasy.io 可视化。

### 6. OOM 常见原因与排查思路

- **常见 OOM 类型及排查方向**：
  1. **`java.lang.OutOfMemoryError: Java heap space`**：
     - 堆内存不足或存在内存泄漏（大对象无法回收）。
     - 排查：dump 堆 → MAT Histogram 按 Retained Heap 排序 → 找 GC Root 路径 → 找到未释放引用的代码。
     - 也可能是堆本身就配小了，正常对象就接近堆上限。
  2. **`java.lang.OutOfMemoryError: GC overhead limit exceeded`**：
     - GC 占用超过 98% CPU 但每次仅回收 < 2% 的堆内存。通常是堆配太小或内存泄漏严重。
     - 解决方案：增大堆、排查内存泄漏、优化 GC 策略。`-XX:-UseGCOverheadLimit` 仅临时关闭，不是根本解决。
  3. **`java.lang.OutOfMemoryError: Metaspace`**：
     - 动态生成的类太多，如 CGLIB 动态代理过多、反射不断生成代理类、JSP 太多、Lambda 过多。JDK 8 元空间默认无上限，但操作系统物理内存有限。
     - 排查：`jcmd <pid> VM.classloader_stats` 查看各 ClassLoader 加载类数量。可配置 `-XX:MaxMetaspaceSize` 限制。
  4. **`java.lang.OutOfMemoryError: Direct buffer memory`**：
     - NIO 的 `ByteBuffer.allocateDirect()` 分配堆外内存超限（默认 = Xmx 值）。
     - 排查：检查是否有没关闭的 DirectByteBuffer（调用了 reserveMemory 未释放）。使用 Unsafe 或 Netty 的池化分配器。
  5. **`java.lang.OutOfMemoryError: unable to create new native thread`**：
     - 线程数达到系统或进程上限，无法分配新线程的栈空间（-Xss 设置过大时每个线程占用多 → 可创建的线程少）。
     - 排查：`ps -eLf` 看线程数；检查线程池是否正确释放线程。
- **预防策略**：
  - `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump 事故现场。
  - 监控 JVM 内存和 GC Metrics，设置告警。
  - 压测时也开启 dump，确认内存 GC 正常。

### 7. 对象创建过程（从 new 到对象生成全链路）

1. **类加载检查**：遇到 `new` 指令，先检查该指令的参数能否在常量池中定位到一个类的符号引用，接着检查该类是否已加载、解析和初始化。未加载则先执行类加载。
2. **内存分配**：类加载完成后为新生对象分配内存，所需内存在类加载完成后确定。
   - **指针碰撞**（Bump the Pointer）：内存规整时，已用和空闲内存分界线通过指针标记，分配只需移动指针（Serial/ParNew 等带有 compact 过程的收集器可用）。
   - **空闲列表**（Free List）：内存不规整时，JVM 维护可用内存块的列表（CMS 这种基于标记-清除算法的收集器适用）。
   - **线程安全问题**：多线程下分配内存可能冲突。解决：
     - **CAS + 失败重试**：保证指针更新操作的原子性。
     - **TLAB（Thread Local Allocation Buffer）**：每个线程在 Eden 区预先分配一小块私有缓冲区（默认占 Eden 1%），在该区域内分配不需要 CAS。TLAB 不够时再在共享区分配或申请新的 TLAB。
3. **零值初始化**：内存分配完成后，JVM 将分配到的内存空间（除对象头外）全部初始化为**零值**。这一步保证了对象的实例字段在无显式赋初始值的情况下可以直接使用（默认值：0/0L/null/false）。
4. **设置对象头**：
   - **Mark Word**：存储哈希码、GC 分代年龄、锁状态标志、偏向线程 ID、偏向时间戳。32/64 bit 依 JVM 位数而定。
   - **类型指针**（Klass Pointer）：指向方法区中该类的元数据，确定该对象是哪个类的实例。启用压缩指针时（默认）占 32 bit。
   - 如果是数组，还需额外记录数组长度。
5. **执行 `<init>()` 方法**：调用对象构造器代码（`<init>` 方法），按照程序员编写的构造器进行初始化。至此，一个真正可用的对象创建完毕。

---

## 四、Spring

### 1. IOC 原理与 Bean 完整生命周期

- **IOC（控制反转）核心**：将对象创建和依赖关系的管理权交给 Spring 容器，通过依赖注入实现松耦合。容器是 `BeanFactory` 和其高级实现 `ApplicationContext`。
- **依赖注入方式**：
  - **构造器注入**（推荐）：依赖不可变、便于单元测试、防止 NPE（编译器要求提供注入项）。
  - **Setter 注入**：可选依赖时可使用，对象可变。
  - **字段注入**（`@Autowired` 在字段上）：简洁但难以测试（需反射注入），不利于检测依赖项（可能 NPE 后构造）。
- **Bean 的完整生命周期（详细版）**：
  1. **实例化**：Spring 通过反射调用构造器（或工厂方法）创建 Bean 实例。
  2. **设置属性值**：依据配置（XML/@Autowired/@Value），填充 Bean 的属性，包括依赖注入。
  3. **调用 Aware 接口方法**（按顺序）：
     - `BeanNameAware.setBeanName()` → `BeanClassLoaderAware.setBeanClassLoader()` → `BeanFactoryAware.setBeanFactory()`
  4. **BeanPostProcessor 前置处理**：`postProcessBeforeInitialization(Object bean, String beanName)`，遍历所有容器中的 BeanPostProcessor。
  5. **检查 InitializingBean**：若实现了 `InitializingBean`，调用 `afterPropertiesSet()`。
  6. **调用 init-method**：若配置了 `@Bean(initMethod = "xxx")` 或 XML `init-method`，执行自定义初始化方法。
  7. **BeanPostProcessor 后置处理**：`postProcessAfterInitialization(Object bean, String beanName)`。**AOP 代理对象就是在这里创建的**（AbstractAutoProxyCreator 在此阶段判断是否需要生成代理并替换）。
  8. **Bean 就绪**，放入单例池（singletonObjects 一级缓存）中。
  9. **容器关闭时销毁**：`@PreDestroy` → `DisposableBean.destroy()` → `destroy-method`。
- **BeanFactoryPostProcessor vs BeanPostProcessor**：
  - BFPP：修改 Bean 定义（Bean 的配置元数据）的后处理器，在 Bean 实例化之前执行，可以修改 beanDefinition。例如 `PropertySourcesPlaceholderConfigurer` 解析 `${}` 占位符。
  - BPP：修改 Bean 实例（在初始化前后拦截），在每个 Bean 初始化阶段执行。

### 2. AOP 的原理，JDK 动态代理 vs CGLIB

- **AOP（面向切面编程）核心概念**：
  - **JoinPoint（连接点）**：程序中可以插入切面的点（方法调用、异常抛出等）。
  - **Pointcut（切点）**：匹配连接点的表达式，定义了哪些连接点需要被拦截。
  - **Advice（通知）**：在切点上执行的动作。Before/After/Around/AfterReturning/AfterThrowing。
  - **Aspect（切面）**：切点 + 通知的组合。
  - **Weaving（织入）**：将切面应用到目标对象并创建代理对象的过程。
- **JDK 动态代理**：
  - 要求目标类必须实现至少一个接口。生成的是实现了相同接口的代理类（`$Proxy` 后缀），继承 `java.lang.reflect.Proxy`。
  - 原理：通过 `Proxy.newProxyInstance(classLoader, interfaces, invocationHandler)` 动态生成代理类的字节码，所有方法调用经由 `InvocationHandler.invoke()`。
  - 方法调用流程：代理对象调用方法 → 调用 `InvocationHandler.invoke()` → 执行 Advice 链 → 反射调用目标对象方法。
  - 性能：低版本 JDK 比 CGLIB 慢，JDK 8+ 经过大量优化（缓存生成的代理类、优化反射调用），差距不明显。
- **CGLIB（Code Generation Library）代理**：
  - 通过继承目标类，生成子类代理对象。使用 ASM 操作字节码生成新类。
  - 限制：不能代理 `final` 类/方法；被代理方法 `static/private` 不代理。
  - 原理：通过 `Enhancer` 创建子类，拦截方法调用并通过 `MethodInterceptor.intercept()` 执行增强逻辑。
- **Spring 代理选择策略**：
  - Spring Boot 2.x 开始默认：目标类实现了接口 → JDK 代理；无接口 → CGLIB 代理。
  - `@EnableAspectJAutoProxy(proxyTargetClass = true)` 或 `spring.aop.proxy-target-class=true` 强制使用 CGLIB。
  - `@Transactional`、`@Async`、`@Cacheable` 等 Spring 内置切面也被此配置影响。
- **AOP 失效的常见情况**：
  - 同类内部方法调用（自调用），this 调用不走代理。
  - 非 Spring 管理的 Bean（自己 new 的）。
  - 私有方法、final 方法。

### 3. @Autowired 和 @Resource 的区别

- **@Autowired（Spring）**：
  - 默认按类型（byType）注入，配合 `@Qualifier` 可实现按名称注入（配合 Bean 名称）。
  - `required` 属性，默认为 true（找不到 Bean 抛异常），设为 false 注入前需判空。
  - 可标注在构造器（构造器注入时自动生效无需注解，Java 规范和 Spring 检测到唯一构造器自动装配）、setter、字段上。
- **@Resource（JSR-250/Jakarta）**：
  - 默认按名称（byName）注入：先根据 `name` 属性找，若未指定 name 默认使用字段名/setter 的属性名。
  - 如果按 name 找不到且未显式指定 name → 退回按类型（byType）匹配。
  - 不支持 `required` 可选参数。
- **@Inject（JSR-330）**：
  - 需要额外依赖（javax/jakarta.inject），Spring 也支持。类似 `@Autowired`，默认 byType，配合 `@Named` 按名称。
- 推荐使用构造器注入，优点：字段可声明为 final → 不可变；测试时直接传 mock 依赖；容器启动时就能发现依赖缺失。

### 4. 循环依赖与三级缓存

- **什么是循环依赖**：
  - 构造器循环：A 构造器依赖 B，B 构造器依赖 A（包括间接循环）。**Spring 完全无法解决**。
  - Setter 循环：A 和 B 都通过 setter 注入对方。**Spring 通过三级缓存可解决**（仅限单例 Bean）。
- **Spring 的三级缓存**：
  - **singletonObjects（一级缓存）**：`ConcurrentHashMap<String, Object>`，完全创建好的单例 Bean（实例化 + 注入 + 初始化全完成）。
  - **earlySingletonObjects（二级缓存）**：`HashMap<String, Object>`，提前暴露的半成品 Bean（已实例化但未注入/初始化）。一般会存代理对象的引用。
  - **singletonFactories（三级缓存）**：`HashMap<String, ObjectFactory<?>>`，创建 Bean 的工厂，返回早期引用。
- **解决流程（以 A → B → A 为例）**：
  1. 容器创建 A → 实例化 A（构造器调用完）→ 将 A 的 ObjectFactory 放入三级缓存。
  2. 填充 A 的属性 → 发现 A 依赖 B → 开始创建 B。
  3. 实例化 B → 将 B 的 ObjectFactory 放入三级缓存。
  4. 填充 B 的属性 → 发现 B 依赖 A → 从缓存查找 A：
     - 三级缓存中找到 A 的工厂 → 调用工厂获取对象（若 A 需要代理则返回代理对象）→ 存入二级缓存，移出三级缓存。
  5. B 获取到 A（可能的代理对象） → B 注入完成 → B 创建完毕 → 放入一级缓存。
  6. 回到 A 的创建流程 → A 从一级缓存拿到 B → A 注入完成 → A 创建完毕 → 放入一级缓存 → 从二级缓存移除。
- **为什么需要三级缓存**（而非两级）：
  - 目的是解决**代理对象不一致**的问题：如果 A 需要被 AOP 代理，实际注入给 B 的应该是 A 的代理对象，而非 A 的原始实例。
  - ObjectFactory 可以持有原始 Bean 引用，调用 `getObject()` 时执行 `getEarlyBeanReference()` 逻辑，如果需要代理就在此时生成代理，保证所有地方拿到的是同一个代理对象。
- **不能解决的情况**：构造器注入循环（还没有放入缓存的时机，因为需要 B 的实例才能调用 A 的构造器）。原型 Bean 循环（原型 Bean 不放入缓存，每次获取都新建）。

### 5. Spring Boot 自动配置原理

- **入口**：`@SpringBootApplication` 包含 `@EnableAutoConfiguration`。
- `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入选择器：
  1. `AutoConfigurationImportSelector` 读取 classpath 下的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Spring Boot 3.x，类名一行一个）或 `META-INF/spring.factories` 中的 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 键（Spring Boot 2.x）。
  2. 返回所有自动配置类的全限定类名列表，准备加载。
- **条件注解过滤**：并非全部自动配置类都会生效，每个配置类上使用了条件注解：
  - `@ConditionalOnClass`：classpath 中存在指定类时才生效。例如 `DataSourceAutoConfiguration` 要求存在 `DataSource.class`。
  - `@ConditionalOnMissingBean`：容器中不存在指定类型的 Bean 时才生效。此即**用户自定义覆盖默认配置**的原理。
  - `@ConditionalOnProperty`：配置文件中存在特定值才生效。
  - `@ConditionalOnBean`：容器中有指定 Bean 时生效。
  - `@ConditionalOnExpression`：SpEL 表达式满足时生效。
- **自定义 Starter**：
  1. 创建 autoconfigure 模块，编写自动配置类，用 `@Configuration` + 条件注解。
  2. 创建 Starter 模块，只放依赖和 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。
  3. 配合 `@ConfigurationProperties(prefix = "xxx")` 让用户可配置。
- 排除自动配置：`@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` 或配置 `spring.autoconfigure.exclude=xxx`。

### 6. @Transactional 失效场景全集

1. **非 public 方法**：Spring 默认使用 JDK/CGLIB 代理实现事务，代理只能拦截 public 方法。非 public（private/protected）方法上即使加了注解也不会生效。
2. **同类方法自调用（最常出问题）**：没有本类方法调用事务方法，调用的是 `this.methodB()`，不经过 Spring 的代理对象 → 事务不生效。
   - 解决方案 1：`AopContext.currentProxy()` 获取代理对象再调用（需配置 `@EnableAspectJAutoProxy(exposeProxy = true)`）。
   - 解决方案 2：将事务方法提取到另一个 Service 中。
   - 解决方案 3：自己注入自己（`@Autowired private XxxService self`），通过 `self.methodB()` 调用（本质还是通过代理）。
3. **异常被 catch 且未重新抛出**：方法内有 try-catch 且捕获了 RuntimeException 后未重新抛出，Spring 认为方法正常执行完毕 → 提交事务。
4. **回滚的异常类型不匹配**：默认只回滚 `RuntimeException` 和 `Error`。受检异常（Checked Exception，如 IOException）抛出时事务不会回滚。
   - 解决：`@Transactional(rollbackFor = Exception.class)` 明确指定所有异常都回滚。
5. **多线程**：事务信息被绑定到 ThreadLocal 中（`TransactionSynchronizationManager` -> resources），新线程内不在同一个事务上下文。
6. **数据库引擎不支持事务**：如 MySQL 的 MyISAM 引擎不支持事务，加了 `@Transactional` 也不生效。
7. **传播行为设置错误**：`propagation = Propagation.NOT_SUPPORTED`（不用事务，有事务挂起）、`NEVER`（不适用事务，有事务抛异常）等也会导致当前不启用事务。

### 7. Spring Bean 作用域

| 作用域 | 说明 | 适用范围 |
|--------|------|---------|
| **singleton**（默认） | 在 Spring IoC 容器中，一个 Bean 定义只产生一个实例 | 所有 |
| **prototype** | 每次请求（注入/调用 getBean）都创建新实例 | 所有 |
| **request** | 每个 HTTP 请求创建一个新实例 | Web |
| **session** | 每个 HTTP Session 创建一个实例 | Web |
| **application** | 在整个 ServletContext 生命周期内只有一个实例 | Web（ServletContext） |
| **websocket** | 每个 WebSocket 连接一个实例 | WebSocket |

- **prototype 陷阱**：将一个 prototype Bean 注入到 singleton Bean 中（最常见场景），期望每次调用都拿到新实例。但 singleton Bean 只会在创建时注入一次 prototype Bean 的引用，之后不会再改变。
- 解决 prototype 丢失问题的方法：
  - 使用 `@Lookup` 注解标记方法，Spring 通过 CGLIB 代理在每次调用时执行 `getBean()`。
  - 注入 `ObjectFactory<PrototypeBean>` 或 `Provider<PrototypeBean>`。
  - 使用 ApplicationContext 主动 `getBean()`（不推荐）。
- Spring 对 prototype 不管理完整生命周期：负责创建但不负责销毁。销毁逻辑需要调用方自己处理。
- 自定义作用域：实现 `org.springframework.beans.factory.config.Scope` 接口。

---

## 五、MySQL

### 1. B+Tree 索引结构详解（为什么用它而不用 B-Tree 或 Hash）

- **B+Tree 结构特点**：
  - 非叶子节点只存键值和子节点指针，不存数据（data 只存在叶子节点）。这使得非叶子节点可以容纳更多的键值 → 树的高度更低 → **磁盘 IO 次数更少**。
  - 叶子节点之间通过**双向链表**串联 → 天然支持范围查询（`BETWEEN`、`>、<、ORDER BY`）和全表扫描，无需回到非叶子节点。
  - 每个节点大小等于一个磁盘页（InnoDB 默认 16KB），一次 IO 读取整页。
- **与 B-Tree 的对比**：
  - B-Tree 的非叶子节点中也存储 data，同样的页容量能存的关键字更少 → 树可能更高 → IO 更多。
  - B-Tree 范围查询需要回到上层节点，不如 B+Tree 链表直通高效。
- **与 Hash 的对比**：
  - Hash 索引仅支持 = / IN 精确匹配，无法范围查询和排序。
  - 存在 hash 冲突时需要遍历链表，效率下降。
  - Memory 引擎支持 Hash 索引，InnoDB 内部有自适应哈希索引（AHI），将高频访问的 B+Tree 数据页 hash 化。
- **容量估算**：三层 B+Tree，bigint 主键（8B）+ 指针（6B），非叶子节点一页可存约 16KB/14B ≈ 1170 个键值。叶子节点一条记录假设 1KB → 一页 16 条。三层树总容量约 `1170 * 1170 * 16 ≈ 2000万` 条。

### 2. 聚簇索引和非聚簇索引（回表与覆盖索引）

- **聚簇索引（Clustered Index）**：
  - InnoDB 的主键索引，数据行实际上就存储在叶子结点中（索引即数据）。一张表只有一个聚簇索引。
  - 若建表时无显式主键 → InnoDB 选择第一个 Unique NOT NULL 列作为聚簇索引；若也没有 → 自动生成 6 字节的 `row_id` 隐藏列作为聚簇索引。
- **非聚簇索引（Secondary Index，二级索引/辅助索引）**：
  - 叶子节点不存数据行，只存该行的主键值（而非行指针）。这样当主键行位置变化时，只需修改聚簇索引，二级索引不需要动（在主键上做到物理无关）。
- **回表**：通过二级索引查到主键值 → 再到聚簇索引（主键）中查找完整行 → 完成一次**回表**。
  - 用 EXPLAIN 查看，`Extra` 中 `Using index condition` 说明发生了回表（但使用了 ICP 优化下推）。
- **覆盖索引（Covering Index）**：
  - 查询所需的列全部在二级索引中，不需要回表。EXPLAIN 中 `Extra` = `Using index`。
  - 实践：`SELECT column FROM table WHERE indexed_col = x` → 若 column 也在此索引中（联合索引），最快。
- **联合索引的最左前缀法则**：
  - 对于 `INDEX(a, b, c)`：查询条件中有 a → 用索引；有 a,b → 用索引（只用前缀两列，c 不算）；无 a 但有 b → 不用索引。
  - 范围查询中 `a > 1 AND b = 2` — a 走索引，b 不走（索引在第一个范围条件后停用）。

### 3. 事务四大特性（ACID）和 InnoDB 隔离级别实现

- **ACID 含义及 InnoDB 实现机制**：
  - **原子性（A）**：事务要么全部执行，要么全部不执行。通过 **Undo Log** 实现（记录旧值，需要回滚时通过 Undo 恢复）。
  - **一致性（C）**：事务执行前后数据库一致性状态不变。由原子性+隔离性+持久性共同保证。
  - **隔离性（I）**：并发事务互不干扰。通过 **MVCC + 锁（行锁/间隙锁/临键锁）** 实现。
  - **持久性（D）**：事务提交后数据永久保存。通过 **Redo Log** 实现（WAL 预写式日志，先写日志再刷脏页）。
- **四种隔离级别及实现**：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现机制 |
|---------|------|----------|------|---------------|
| **READ UNCOMMITTED** | ✓ | ✓ | ✓ | 不加锁（几乎没有使用场景） |
| **READ COMMITTED** | ✗ | ✓ | ✓ | MVCC：**每次**快照读生成新 ReadView |
| **REPEATABLE READ**（InnoDB 默认） | ✗ | ✗ | ✗(部分解决) | MVCC：**事务开始**生成一个 ReadView + Next-Key Lock 解决当前读幻读 |
| **SERIALIZABLE** | ✗ | ✗ | ✗ | 所有普通查询隐式加 `LOCK IN SHARE MODE`（共享锁），读写串行化 |

- 幻读的解释：同一事务内，两次读取同一个范围，第二次多出（或少了一些）行。RR 级别下：
  - 快照读（普通 SELECT）通过 MVCC 可避免幻读（ReadView 在事务开始就固定了）。
  - 当前读（SELECT ... FOR UPDATE/UPDATE/INSERT/DELETE）通过 Next-Key Lock 锁住间隙防止插入 → 避免幻读。

### 4. MVCC（多版本并发控制）机制深度

- 核心组件：**Undo Log（版本链）+ ReadView（读视图）+ 隐藏字段（trx_id, roll_pointer）**。
- 每行都有隐藏列：
  - **DB_TRX_ID**（6 字节）：最近一次插入或更新该行的事务 ID。
  - **DB_ROLL_PTR**（7 字节）：回滚指针，指向该行对应的 Undo Log 记录（历史版本）。
  - **DB_ROW_ID**（6 字节）：隐含自增行 ID（无主键无唯一键时），单调递增。
- Undo Log 版本链：
  - 每次行更新都会生成一条 Undo Log → 包含修改前的值 + roll_pointer 指向更早的 Undo Log → 形成**版本链**。
  - 这条链为不同事务的快照读提供不同版本的数据。
- ReadView 内部关键字段：
  - `m_ids`：生成 ReadView 时，当前系统中活跃（已启动但未提交）的事务 ID 列表。
  - `min_trx_id`：活跃事务中最小的事务 ID。
  - `max_trx_id`：下一个将要分配的事务 ID（即当前已完成事务的最大 ID + 1）。
  - `creator_trx_id`：创建该 ReadView 的当前事务的 ID。
- **可见性判断（顺着版本链找第一个可见版本）**：
  - `trx_id == creator_trx_id` → 自己的修改，可见。
  - `trx_id < min_trx_id` → 该版本修改者已提交，可见。
  - `trx_id >= max_trx_id` → 该版本修改者是在 ReadView 生成后才开始的 → 不可见。
  - `min_trx_id ≤ trx_id < max_trx_id` → 看在不在 `m_ids` 中：不在 → 已提交，可见；在 → 未提交，不可见。
- **RC vs RR 的创建时机区别**：
  - RC：每次执行快照读时都生成一个新的 ReadView（使用最新提交），能读到其他事务已提交的更新 → 不可重复读。
  - RR：只在事务第一次快照读时生成一个 ReadView，后续都用同一个 → 读不到别的事务的提交 → 可重复读。

### 5. InnoDB 锁机制详解

- **按粒度分类**：表锁（Table Lock）、行锁（Row Lock）、页锁（极少用）。
- **行锁的类型**：
  - **Record Lock**（记录锁，也叫行锁）：锁定单个行记录。锁的是索引记录而非数据行（若无索引则会锁住所有记录的隐藏聚簇索引）。
  - **Gap Lock**（间隙锁）：锁定索引记录之间的间隙（开区间），但不包含记录本身。目的是防止幻读。**只在 RR 级别下存在**。
  - **Next-Key Lock**（临键锁）：**Record Lock + Gap Lock** 的组合，锁定索引记录 + 该记录前面的间隙，产生一个**左开右闭**的区间 `(prev, current]`。
  - **Insert Intention Lock**（插入意向锁）：一种特殊的 Gap Lock，表示一个事务想在间隙中执行插入。多个事务可以在同一间隙持有插入意向锁（互不阻塞），只要它们插入的位置不同。
- **加锁规则**（RR 级别，等值查询）：
  - 等值查询且命中**唯一索引**：Next-Key Lock 退化为 Record Lock（不需 Gap Lock）。
  - 等值查询未命中：Next-Key Lock 退化为 Gap Lock，锁定找到的匹配范围。
  - 范围查询（`> / < / BETWEEN`）：使用 Next-Key Lock，并且范围覆盖最后一个符合条件的记录继续向后锁定一个 gap（防止该位置插入符合条件的数据）。
- **意向锁（Intention Lock）**：表级锁，IS/IX 两种。当事务要对某行加 S 锁之前，必须先获得 IS 锁（表级共享意向锁）；加 X 锁之前必须获得 IX 锁（表级独占意向锁）。作用是让表锁（LOCK TABLES）能快速判断表中是否有行锁，而不必遍历所有行。

### 6. 索引失效场景汇总

1. **查询条件包含 OR**：or 的条件里只要有一个字段没索引，整个查询都不会用索引。
2. **联合索引不满足最左前缀**：跳过第一列直接查后面的列。
3. **对索引列使用函数或运算**：`WHERE LEFT(name, 3) = 'abc'`、`WHERE amount + 100 > 500`。
4. **隐式类型转换**：`WHERE phone = 13800138000` 若 phone 为 vachar，MySQL 自动将 phone 转为数值 → 索引失效。反过来 `WHERE id = '123'` 若 id 为 int，MySQL 将 `'123'` 转换为数值 → 索引仍有效（因为 CAST 在常量侧）。
5. **LIKE 以 `%` 开头**：`LIKE '%abc'`（全模糊）无法使用索引，`LIKE 'abc%'`（右模糊）可以使用。
6. **索引列使用 IS NOT NULL / IS NULL**：部分情况走索引，取决于数据分布优化器的成本估计。
7. **负向查询**：`!=` / `<>` / `NOT IN` / `NOT EXISTS`（优化器在多数情况下选择全表扫描，除非覆盖索引或结果集很小）。
8. **复合索引范围查询后的列不会走索引**：`INDEX(a, b, c)` → `WHERE a = 1 AND b > 2 AND c = 3` 中只有 a、b 会走索引，c 不会。
9. **Mysql 优化器的选择性判断**：优化器会估算每种执行计划的花费，若走索引的行数接近全表行数的 30% 以上可能直接走全表扫描（基于统计信息和代价模型）。

### 7. SQL 优化套路（从定位到解决）

- **第一步：打开慢查询日志并分析**
  - 配置：`slow_query_log = ON`，`long_query_time = 1`（超过 1s 记录），`log-queries-not-using-indexes = ON`。
  - 用 `mysqldumpslow` 或 `pt-query-digest`（Percona Toolkit）汇总统计最耗时、执行次数最多的 SQL。
- **第二步：EXPLAIN 解读重点**
  - `type`：性能由好到差 system > const > eq_ref > ref > range > index > ALL。至少要达到 range，最好 ref/const。
  - `key`：实际使用的索引。`possible_keys`：候选索引。若前者为 NULL 则没有索引被使用或优化器放弃了索引。
  - `rows`：预估扫描行数，越小越好。
  - `Extra`：
    - `Using index` → 覆盖索引（不需要回表），最优。
    - `Using index condition` → 索引下推（ICP），在索引层面过滤部分条件，降低了回表次数。
    - `Using where` → 在 server 层做了过滤（一般正常）。
    - `Using filesort` → 额外排序操作，需要优化（加索引或改写 SQL）。
    - `Using temporary` → 使用临时表，有问题，常见于 GROUP BY + ORDER BY 不同字段。
    - `Using join buffer` → JOIN 的缓冲，看是否能在被驱动表上建索引。
- **第三步：索引优化**
  - 核心 SQL 必须要用上索引且为最优索引。
  - 用 `force index` 强制 SQL 走某个索引（验证用）。
  - 尽可能创造覆盖索引（减少回表）。
  - 对于已存在的慢 SQL 建联合索引满足最左原则。
- **第四步：SQL 改写**
  - `SELECT *` → 只查需要的列（配合覆盖索引）。
  - 大分页 `LIMIT 100000, 20` → 改为 `WHERE id > last_id LIMIT 20`（利用主键递增）。
  - JOIN 替代子查询（优化器 MySQL 5.7+ 已做的比较好，但子查询可能产生临时表）。
  - 分批处理大 UPDATE/DELETE（每次 LIMIT 200），避免长事务和锁持有时间，减少主从延迟。
- **第五步：schema 与参数调优**
  - 合理选择字段类型和长度，减少空间。
  - InnoDB buffer pool 设到物理内存 60~80%。
  - 连接池参数适量。
- 更多可参考 [MySQL 日志与复制机制——探索 MySQL 高可用护航体系](mysql-binlog.html)。

### 8. 分库分表方案与实践

- **垂直分库**：按业务领域把不同表拆分到不同数据库。如订单库、用户库、商品库。减少单库的数据量和并发请求。
- **垂直分表**：将一张大宽表按照列的使用频率拆成多张表（热点列和冷列分开），减少单条记录大小 → 提升缓存内存利用率和 IO 效率。
- **水平分库分表**：同一业务表的数据分散到多个数据库或多张表中，通过分片键路由。
- **分片算法**：
  - **取模/哈希**：`key % N` 或 `hash(key) % N`。数据分布均匀，但扩容时数据需要大量迁移（一般扩容一倍再重新分配）。
  - **范围路由**：id 1~1000万 → 表1，1000万~2000万 → 表2。扩容简单，但容易出现热点（最新的数据都在某一张表里）。
  - **一致性哈希**：环状分布，扩容时迁移数据少、但一致性 hash 数据可能不均匀（引入虚拟节点）。
  - **自定义路由表**：灵活但不通用，维护路由元数据的复杂度和可靠性。
- **分库分表带来的新问题**：
  - 分布式事务（Seata、TCC、MQ 最终一致性）。
  - 跨分片查询（字段冗余、全局表、ES/Hive 同步做聚合分析）。
  - 全局唯一 ID（雪花算法、Leaf 号段）。
  - SQL 下发和结果合并（数据库中间件如 ShardingSphere-Proxy、Mycat 负责）。
  - 扩容和数据迁移（ShardingSphere 的 Scaling 模块可平滑扩容，数据一致性检查和断点续迁）。
- **常见中间件**：Apache ShardingSphere（JDBC/Proxy）、Mycat、Vitess（YouTube 开源）、DRDS（阿里云）。

### 9. 主从延迟产生原因与解决方案

- **主从同步原理**：主库写 binlog → 从库的 IO 线程拉取 binlog 写入 relay log → 从库的 SQL 线程回放 relay log 中的事件。
- **延迟原因**：
  - 从库机器性能差于主库，而且从库往往还要承担读请求。
  - 主库大事务执行时间长（例如大批量更新/删除），导致主库 binlog 生成慢 → 传输到从库慢 → 从库回放更慢（大事务在从库也是串行回放，阻塞后续事务）。
  - 从库单线程回放（MySQL 5.5 及以前）。5.6 支持按库并行复制，5.7 LOGICAL_CLOCK 基于组提交（Group Commit）的并行复制（同一组提交的事务在从库并行回放，无冲突）。
  - 8.0 支持 WRITESET 方式并行复制（基于不同行的事务可并行回放）。
  - 网络延迟。
- **解决办法**：
  1. **并行复制**：`slave_parallel_type = LOGICAL_CLOCK` + `slave_parallel_workers = 8`（MySQL 5.7+）。
  2. **强制读主**：刚写入后的关键业务操作（如订单支付后立即查看结果）→ 强制路由到主库读。
  3. **半同步复制**（semi-sync replication）：至少有一个从库确认收到 binlog 后才向客户端返回提交成功。牺牲一定性能换取延迟可控。MySQL 5.7+ 的增强半同步（AFTER_SYNC）更高效。
  4. **缓存层**：写入后同步更新 Redis 缓存，读请求走缓存 → 即使主从有延迟，缓存已是最新数据。
  5. **业务层方案**：允许一定的延迟（非关键展示数据），核心数据写后立刻读强制走主。
- 监控：`SHOW SLAVE STATUS` 中 `Seconds_Behind_Master`（有误差，零值不代表零延迟）。

### 10. 为什么 InnoDB 推荐整型自增主键？

- 整型 int/bigint 而非 varchar：
  - 比较效率高（CPU 整数比较远快于字符串逐字节比较）。
  - 占用空间小（int 4B vs varchar 可能几十字节）。二级索引叶子节点存主键值 → 主键小 = 每个二级索引都小 = 同样的 16KB 页能存更多索引条目 → B+Tree 更低 → IO 更少。
- 自增而非 UUID 等随机值：
  - 自增数据按顺序插入到 B+Tree 最右侧 → 写操作是**顺序写入磁盘**（WAL → 刷盘），性能高。
  - 随机主键（如 UUID）→ 插入位置随机分布 → 频繁**页分裂**（已满页需要分裂出两个半满页）→ 很多页碎片、随机 I/O → 写性能大幅下降。
  - 页分裂导致页填充度低 → 缓存（buffer pool）利用率低 → 读性能也变差。
- UUID 的顺序版本（UUID v7，时间戳基数）也可以作为折中方案。
- 分布式环境中，自增主键确实存在单点问题，可考虑雪花算法等趋势递增的分布式 ID。

## 六、Redis

### 1. 缓存三大问题：穿透 / 击穿 / 雪崩

- **缓存穿透**：查询一个根本不存在的数据（缓存和 DB 都没有，常见于恶意攻击查询 id = -1，-2 等）。
  - **布隆过滤器**：位图 + 多个哈希函数，高效判断一个 key 是否**可能存在**。由于存在一定误判率（不存在会误判为可能存在），只用于过滤一定不存在的 key。Guava 和 Redisson 都有实现。
  - **空值缓存**：查询 DB 确实不存在，也缓存一个空结果（空对象或标识），设置较短过期时间（如 5 分钟）防止同一 key 反复打到 DB。注意空值缓存会占用内存。
  - 前端和网关层做合法性校验（如 ID 必须是正数）。
- **缓存击穿**：热点数据 key 过期的时间点，瞬间大量请求同时落到数据库。
  - **互斥锁**：只允许一个线程从 DB 查询并更新缓存，其余线程等待并重试从缓存获取。Redisson 的 `RLock` 或 `SETNX`。
  - **逻辑过期 + 异步更新**：缓存值存一个逻辑过期时间（真正的 Redis TTL 设置得更大，甚至不过期）。读取时发现逻辑过期 → 返回旧缓存 → 异步线程更新 DB 和缓存（不论更新成功与否都不影响当前请求）。常用读写锁在多读场景优化更好。
- **缓存雪崩**：大量 key 在同一时段集体过期，或者 Redis 宕机，瞬时所有请求打到后端数据库。
  - **过期时间加随机偏移**：`TTL = base + random(0, base*0.3)`，打散过期时间。
  - **Redis 集群高可用**：哨兵/集群模式 + 主从备份，单机宕机自动故障切换。
  - **服务降级与限流**：当发现 DB 压力过大时，快速返回默认值或者直接限流保护数据库。
  - **多级缓存**：本地缓存（Caffeine/Guava Cache）+ Redis + DB，Redis 雪崩时本地缓存仍有部分数据可兜底。

### 2. RDB 和 AOF 持久化对比

- **RDB（Redis Database）快照**：
  - 触发方式：
    - `save`：主线程执行，阻塞所有请求（生产禁用）。
    - `bgsave`：fork 子进程执行 RDB 写入。使用操作系统的**写时复制（Copy-on-Write）**机制。如果父进程有大量写操作，COW 会导致内存额外占用。
  - 优点：二进制紧凑，恢复大数据集时比 AOF 快。文件小，适合备份和灾备恢复。
  - 缺点：丢数据的窗口 = 两次快照间隔（`save 900 1` 等条件配置）。bgsave 的 fork 操作在大内存时可能很耗时（瞬时阻塞）。
- **AOF（Append Only File）追加日志**：
  - 原理：记录所有写命令（其实是 RESP 协议文本），Redis 启动时重放命令重建数据。
  - 三种 `appendfsync` 策略：
    - `always`：每个写命令都 fsync，最安全但性能极差。
    - `everysec`（默认推荐）：每秒把 AOF 缓冲区刷盘，最多丢失 1 秒数据。
    - `no`：扔给操作系统决定何时刷盘，最不安全。
  - **AOF 重写（Rewrite）**：随着 AOF 文件膨胀，Redis 后台 fork 子进程根据当前内存数据生成一份新的最小的 AOF（把冗余命令合并），完成后替换旧 AOF 文件。同样基于 COW。
- **混合持久化（Redis 4.0+）**：
  - `aof-use-rdb-preamble yes`：AOF 文件前半段是 RDB 二进制数据（快照），后半段是 AOF 增量命令。兼具 RDB 的快速恢复和 AOF 的低丢失率。

### 3. 缓存一致性方案（Cache Aside 及相关）

- **旁路缓存（Cache Aside Pattern）是最常用的模式**：
  - **读**：先读缓存 → 命中则返回 → 未命中则读 DB → 回写缓存（设 TTL）→ 返回数据。
  - **写**：**先更新数据库，后删除缓存**（或设置特殊标记令缓存失效）。
- **为什么不用 update（更新缓存）而用 delete（删除缓存）**？
  - 并发写可能导致脏缓存：线程 A 更新 DB，线程 B 也更新 DB → 可能 A 先写缓存、B 后写缓存 → 缓存中是 A 的旧数据，DB 中是 B 的新数据。
  - 删缓存是幂等操作（多次 delete 安全），更新缓存却不是（需要顺序保证）。
- **为什么先操作 DB 再删缓存（而非反序）**？
  - 先删缓存再更新 DB：删缓存 → 另一个线程并发读 → 缓存没命中但是读到了旧的 DB 数据 → 把旧数据写入缓存 → DB 更新完成 → 此时缓存中是旧值。需要**延迟双删**来补偿（删缓存→更新 DB→睡眠短暂时间→再删一次缓存，有开销和依旧存在极小概率的不一致窗口期）。
  - 先更新 DB 后删缓存：更新 DB → 删除缓存 → 在两者之间读请求可能读到缓存中的旧值 = 仅有一小段时间不一致。只有缓存删除失败时才会持续不一致。**
- 兜底方案**：
  - **Canal + MQ 异步最大努力删除**：监听 MySQL binlog 变更 → 发送缓存删除消息 → 重试机制保证删除成功。解决了缓存删除失败的问题。
  - **消息队列**（如 RocketMQ/RabbitMQ）确保缓存删除消息不丢失。
  - **定时任务**对账：定期扫 DB 和 Redis 数据做一致性校验。
- 更多可参考 [缓存基础：缓存模式与最佳实践](cache-basics.html)。

### 4. 基于 Redis 的分布式锁（演进与最佳实践）

- **V1：SETNX + EXPIRE（两步非原子）**：`SETNX lock_key value`，成功再 `EXPIRE lock_key 30`。SETNX 和 EXPIRE 之间如果进程宕机 → 死锁（锁永远不释放）。
- **V2：SET NX + EX（原子）**：`SET lock_key unique_value NX EX 30`。获取锁和设过期时间一体化。**这是核心命令**。
- **关键环节**：
  1. value 存什么？—— 存唯一的 `UUID + 线程ID`，释放锁时要判断是不是自己持有（避免误删别人的锁）。
  2. 释放锁必须用 **Lua 脚本原子执行**：
     ```lua
     if redis.call('get', KEYS[1]) == ARGV[1] then
         return redis.call('del', KEYS[1])
     else
         return 0
     end
     ```
     没有原子性 → 先判断后删除 → 在判断和删除之间锁过期另一线程获取了锁 → 误删。
  3. **锁续期（watch dog）**：如果任务执行时间 > 锁过期时间 → 锁自动释放 → 另一线程抢到 → 任务重叠。需要一个后台线程定期重置过期时间（如每 10 秒续期 30 秒），即看门狗机制。Redisson 内置此机制（默认锁时间 30s，每 10s 检查并自动续期）。
- **可重入锁**：需要记录当前持有锁的线程和重入次数 → hash 结构：`hset lock_key thread_id 1`。加锁次数增，解锁次数减，归零时释放。Redisson 也是基于 hash + Lua。
- **单点问题**：RedLock 算法：N 个独立 Redis Master 实例，客户端依次向 N 个实例获取锁（SET NX），成功数 ≥ N/2+1 且总时间 < 锁 TTL → 获锁成功。但网络分区场景下 RedLock 也不保证绝对安全（Martin Kleppmann 2016 对此有激烈讨论）。实践中多数场景单实例 + 哨兵就足够了。

### 5. Redis 集群方案选型

| 特征 | 主从（无哨兵） | 哨兵（Sentinel） | 集群（Cluster） |
|------|--------------|----------------|---------------|
| 自动故障转移 | 否 | 是 | 是（集群内自带 Sentinel） |
| 水平分片 | 否 | 否 | 是，16384 个哈希槽（slot） |
| 客户端复杂度 | 低 | 低（需感知哨兵地址） | 高（需支持 MOVED/ASK 重定向） |
| 扩容/缩容 | 手动 | 手动 | 在线 reshared slot + migrate data |
| 多 Key 操作限制 | 无限制 | 无限制 | 跨 slot 的 mget/mset/pipeline 不允许（需 hash tag 强制同一 slot） |
| 适用规模 | 小（< 几百G） | 大（被动故障切换） | 超大（TB+） |

- **Hash 槽（slot）**：`CRC16(key) % 16384` 确定 key 归属的 slot。节点迁移 slot 是异步的：迁移过程中槽处于 importing/migrating 状态，客户端请求重定向（MOVED = 永久重定向，ASK = 临时重定向到迁入节点）。
- **Hash Tag**：用 `{` 和 `}` 包裹 key 的部分使得只有花括号内的部分决定 slot，例如 `{user:1001}:name` 和 `{user:1001}:age` 划入同一个 slot，允许这组 key 在集群中做 mget。

### 6. 缓存淘汰策略

- **LRU（Least Recently Used）**：只看最近被访问的时间。
  - `allkeys-lru`：所有 key 参与淘汰，适合请求分布呈幂律分布的场景（大部分请求集中在少数 key 上）。推荐首选。
  - `volatile-lru`：只在设置了过期时间的 key 中进行淘汰，其余 key 即便很久没访问也不会淘汰。
- **LFU（Least Frequently Used）**：看最近使用的频率（4.0+）。LRU 的不足是一个 key 偶尔被大量访问，另一个 key 长期被频繁低频访问，前者很容易把后者顶掉。
  - `allkeys-lfu` / `volatile-lfu`。
- **Random**：随机淘汰。O(1) 极简，但在特定场景下效果不错。
- **TTL**：在带过期时间的 key 中淘汰即将过期的（最接近 expiration time）。
- **noeviction**：默认策略（3.0 以前，之后建议主动选择），内存满则写入报错。
- **Redis LRU 的实现是近似的**：从所有键中随机抽样 N 个（`maxmemory-samples`，默认 5），淘汰其中最久未使用的。不是标准 LRU（维护双向链表每次访问都要移动到头部，有锁成本）。

### 7. Redis 常用数据结构及底层实现

- **String（SDS）**：
  - 用**简单动态字符串**，结构字段：len（已用长度）、alloc（分配长度）、flags（类型标识）、buf[]（字节数组）。
  - 优点：O(1) 获取长度、二进制安全、自动扩容、杜绝缓冲区溢出。
  - 应用：缓存字符串/数字、分布式锁、计数器、session 共享。
- **List（QuickList）**：
  - Redis 3.2 后用 QuickList（ziplist 和 linkedlist 的组合），链表每个节点是一个 ziplist（压缩列表）或 listpack（7.0+），减少内存开销和节点数量。
  - 应用：消息队列（BRPOP/BLPOP 阻塞弹出）、最新动态列表。
- **Hash**：
  - 默认使用 listpack（7.0+）或 ziplist（早期），当数据量/长度超过 `hash-max-listpack-entries` 等阈值时转为哈希表（hashtable），渐进式 rehash。
  - 应用：缓存对象（比 String 存 JSON 更灵活，可部分更新单个字段）。
- **Set**：
  - 当元素都是整数组且数量 < 512 时使用 intset（连续内存，二分查找），否则用哈希表。
  - 应用：标签、去重、交集并集（共同关注/推荐朋友）、随机抽取。
- **Sorted Set（ZSet）**：
  - 核心：listpack + **跳跃表（skiplist）** + 字典（dict，从 member 到 score 的映射）。
  - 跳表使用多层链表，查找和插入 O(log N) 且比平衡树简单高效。
  - 应用：排行榜（实时按积分排名）、带权重的延时队列。
- **Redis 7.0 用 ListPack 替代 Ziplist**：Listpack 去除了 Ziplist 的级联更新问题（连锁更新可能 O(N²) 导致性能劣化）。结构更简单：总字节数 + 元素数量 + 元素 1 长度+数据 + 元素 2 + ... + 结尾标志。

### 8. 为什么 Redis 单线程还这么快？

- **核心原因不是"单线程"而是"设计特性"**：
  - 纯内存操作，带宽通常在 10 万 QPS 以上。
  - IO 多路复用（Linux epoll）：单线程同时监听多个连接的就绪事件，不用为每个连接创建线程。
  - 避免线程切换和锁竞争：单线程处理命令的模式消除了上下文切换开销和自旋/等待锁的损耗。
  - 精巧紧凑的数据结构（SDS、listpack、跳表——针对缓存局部性和内存效率深度优化）。
- **单线程的局限与应对**：
  - 耗时的单个命令会阻塞整个 Redis（如 `KEYS *`、`HGETALL large_hash`、`SORT` 等）。解决方案：用 `SCAN`（逐步迭代，光标式）代替 `KEYS`；将大集合拆分成小集合。
  - 网络 IO 瓶颈：6.0 引入了 IO 多线程处理网络包的读取/解析和写回，但命令执行仍由主线程串行处理（保持无锁）。
- **删除大 key 的阻塞问题**：Redis 4.0 引入 `UNLINK` 命令和懒惰释放（lazy free）。`UNLINK key` 将 key 的释放操作交给后台线程执行，主线程不阻塞。
- 从 Redis 7.0 开始，AOF 写入、module 等操作也引入了多线程优化。

## 七、分布式系统

### 1. CAP 定理

- **C（Consistency 强一致性）**：每次读操作都能读取到最新的写操作结果（所有节点在同一时刻数据一致）。
- **A（Availability 可用性）**：每一个非失败的节点必须在限定的时间内对每个请求返回非错误的响应，即便出现网络分区故障。只要节点没有宕机就必须要正常响应。
- **P（Partition Tolerance 分区容错性）**：节点之间网络通信出现分区（消息丢弃/延迟）后，系统整体仍能继续提供服务。在分布式系统中 P 是既定的前提（网络分区不可抗拒），因此实践中在 C 和 A 中二选一。
- **实际系统分类**：
  - CP 系统（弃A保C）：ZooKeeper（选择 leader 期间不提供服务）、Consul 默认模式。适用于配置管理、服务协调等对一致性要求极高的场景。
  - AP 系统（弃C保A）：Eureka、Cassandra、DynamoDB。适用于服务注册发现（优先保证服务可用，准许短暂不一致）等场景。通过最终一致性修复数据。
  - Nacos 可以在这两者间通过配置切换（AP 模式 vs CP 模式）。
- **BASE 理论（CAP 中 AP 的延伸）**：
  - Basically Available 基本可用（系统出现故障时允许损失部分可用性——响应时间增加或功能降级）。
  - Soft State 软状态（允许系统存在中间状态，且该中间状态不会影响系统整体可用性）。
  - Eventually Consistent 最终一致（经过一段时间后，所有数据副本最终达到一致状态）。

### 2. 分布式事务的六种方案

| 方案 | 原理与步骤 | 优点 | 缺点 | 适用场景 |
|------|----------|------|------|---------|
| **2PC（XA）** | 协调者 → 所有参与者 Prepare → 所有返回 OK → Commit；任一返回 No → Rollback。依赖数据库的 XA 接口 | 强一致保证 | 同步阻塞、协调者单点、性能低、锁资源时间长 | 单体应用内需严格强一致的短事务 |
| **TCC** | Try（预留资源/检查）→ Confirm（提交业务，幂等）→ Cancel（释放预留资源，幂等）。由业务代码实现三个接口 | 不阻塞、性能高、细粒度控制 | 代码侵入大、需实现幂等和空回滚、Cancel 失败也需处理 | 资金交易、支付、库存（有明确预留语义的场景） |
| **本地消息表**（ebay 经典方案） | 在本地事务中同时写业务数据和消息表 → 定时任务扫描消息表发送 MQ → 消费者处理 → 更新消息状态或回调 | 高可靠性、最终一致性 | 消息表耦合到同一数据库中、需定时扫描 | 内部系统间无需实时一致的场景 |
| **事务消息（RocketMQ）** | 先发送 half 消息 → 执行本地事务 → 根据本地结果 Commit/Rollback half 消息 → 若未收到二次确认，MQ 回调 check 接口 | 不侵入数据库、高性能 | 依赖特定的 MQ（RocketMQ、Kafka 的事务消息较弱） | 高并发在线业务（交易、秒杀） |
| **Seata AT** | 数据源代理自动生成 Undo SQL（前镜像和后镜像），全局事务管理器协调各分支事务的注册和提交/回滚 | 业务代码无侵入，开箱即用 | 隔离性天生弱（写隔离需全局锁）、性能开销 | 无侵入、能接受最终一致的内部服务 |
| **Saga** | 长事务拆分为多个本地事务 T1, T2, ..., Tn，每个 Ti 有对应的补偿操作 Ci。若某 Ti 失败，逆序执行 C_{i-1}, C_{i-2}, ..., C1 | 适合长时间、多步骤的业务流 | 实现复杂、缺乏隔离性（中间状态有窗口暴露） | 微服务编排、订单流程、旅行社预订等长链路 |

- 选型建议：大多数场景用 **本地消息表 + MQ** 或 **RocketMQ 事务消息** 达到最终一致性就足够了。需要强一致性的资金/库存场景考虑 **TCC**。追求无侵入可考虑 **Seata AT**。

### 3. 分布式 ID 生成方案对比

- **UUID**：`java.util.UUID.randomUUID()`，128 位，本地生成无网络开销。缺点是无序 + 字符串存储空间大 → B+Tree 插入时频繁页分裂，严重影响 MySQL InnoDB 写入性能。一般不建议用在数据库主键上。
- **数据库自增 ID（单点）**：直接 `auto_increment`。优点是简单有序。缺点：单点性能瓶颈和单点故障。改进：多台 DB 不同的 `auto_increment_increment` 和 `auto_increment_offset`（例如 DB1 只产生 1,3,5...，DB2 只产生 2,4,6...），当 DB 数量变化时需要重新配置。
- **号段模式（Leaf-Segment）**：从数据库一次拉取一个 ID 号段（如 1~1000）缓存在本地，本地用完再批量拉取下一号段 → DB 压力降为号段拉取频率。优点：趋势递增、节省 DB 资源、支持多 client 高性能。缺点：号段用完需要重新拉，DB 宕机影响新号段获取（但可配置多 DB 容灾）。美团 Leaf 主要用此方案。
- **雪花算法（Snowflake）**：`1 bit 未使用 + 41 bit 时间戳（毫秒，可用 69 年）+ 10 bit 工作机器 ID（5 bit 机房 + 5 bit 机器）+ 12 bit 序列号（同毫秒内递增）`。本地内存生成，趋势递增，QPS 可达百万。问题与解决方案：
  - **时钟回拨**（机器时间被 NTP 或人工回拨）会导致 ID 重复。解决方案：等待时钟同步追上（抛异常）；预留更大的序列号位；使用投递 MQ 异步生成（美团 Leaf-snowflake）；用 ZK 持久化当前时间戳（发现时钟回拨则抛异常或等待）。
- **Redis 自增**：`INCR key` / `INCRBY key N` 一步递增返回原子值。优势是简单，每次取一批 ID（`INCRBY 1000`）减少 Redis 请求。缺点是依赖 Redis 持久化策略避免宕机后 ID 回滚，也需要高可用架构（哨兵或集群）。
- 更多可参考 [分布式 ID 服务：Twitter Snowflake 的技术演进](distribution-id.html)。

### 4. MQ 选型与消息不丢失全流程保障

- **Kafka**：超高的吞吐量（百万 TPS），顺序写磁盘（append-only log）+ 零拷贝（sendfile 实现）。适合大数据日志/监控/埋点流式处理（与 Flink/Spark 集成）。局限性：消息量小时延迟偏高，不支持定时/事务消息（Kafka 事务与 RocketMQ 不是一个概念），topic 过多时性能下降。
- **RocketMQ**：阿里开源的在线业务型 MQ。低延迟 + 高并发 + 事务消息 + 定时消息 + 消息重试 + 死信队列。适合电商交易、订单、支付等在线业务场景。
- **RabbitMQ**：AMQP 标准协议实现，Erlang 语言。灵活的路由（Exchange + Binding + Queue），可靠性高。适合企业级低频稳定消息，中小规模。
- **消息零丢失三环节**：
  1. **生产端**：使用同步发送 + 失败重试。配置 Broker 应答机制：Kafka `acks=all`（所有 ISR 副本确认）；RocketMQ 的 `SYNC_MASTER`（同步刷盘 + 同步复制）。
  2. **Broker 端**：Kafka 的多个 ISR 副本 + `min.insync.replicas` 最小确认数（默认 1，建议 2）+ `unclean.leader.election=false`（禁止落后太多副本当选）。RocketMQ 同步刷盘 + 异步刷盘 + 主从 HA 同步。
  3. **消费端**：处理完业务成功后再手动提交 offset（`enable.auto.commit=false`或 RocketMQ 的 `ConsumeConcurrentlyStatus.CONSUME_SUCCESS`）；确保幂等（记录已消费的消息 ID 在 Redis/DB 中）。

### 5. 服务注册与发现

- **核心角色**：
  - 服务提供者：启动时向注册中心注册（IP+端口+元数据），心跳维持注册。
  - 服务消费者：从注册中心获取服务列表并缓存到本地，负载均衡调用。订阅注册中心变更以实时更新。
  - 注册中心：存储服务注册表并将变更推送给订阅的消费者。
- **实现方式**：
  - **客户端发现**（主流）：消费者自己从注册中心拉取/订阅列表 + 客户端负载均衡。如 Spring Cloud + Eureka/Nacos + Ribbon/LoadBalancer。
  - **服务端发现**：消费者请求经过统一的负载均衡器（如 Nginx/Envoy/Kubernetes Service），由均衡器查注册表并转发。无需客户端感知服务实例变化，但多一跳。
- **健康检查**：
  - 客户端主动上报：Eureka（心跳延续租约）、Nacos（临时实例 heart beat）。
  - 服务端主动探测：Consul（agent 探测本地进程 + 跨节点探测）、Nacos（永久实例服务端检查）。
- **CAP 取舍**：
  - Eureka（AP）：优先可用性。当网络分区时宁愿保留过期实例也不下线 → 注册中心集群所有节点正常响应 → 可能查到失效实例。
  - Zookeeper（CP）：主节点选举期间整个集群不可用（写被拒绝）。不适合大规模服务发现。
  - Nacos：支持切换 AP/CP，推荐临时实例用 AP、持久实例用 CP，兼顾不同场景。
  - Consul：CP（默认）。

### 6. 限流算法

- **固定窗口计数器**：
  - 原理：`(timestamp / window_size)` 作为窗口 key，当前窗口内计数器累加超过阈值 → 拒绝。简单、O(1)。
  - **临界问题**：在窗口末尾和下一个窗口开头各放行阈值流量 → 实际同一时间点承受 2 倍阈值压力。
- **滑动窗口计数器**：
  - 将固定窗口再拆分为更小粒度的子窗口。例如 1s 拆 10 个 100ms 窗口，限流时统计当前和过去 N-1 个窗口的计数和。
  - Redis 实现：`ZSET` 元素 score 为时间戳，`zremrangeByScore` 移除窗口外数据，`zcard` 统计窗口内数量 → 超过阈值去限流。
- **漏桶算法**：
  - 请求进入漏桶，桶以恒定速率漏出请求到服务器。桶满 → 请求丢弃/排队。强迫平滑输出 → 无法应对流量突发（但能保护后端）。
- **令牌桶算法（推荐）**：
  - 以恒定速率向桶中放入令牌，桶满后令牌不再增加。每个请求需消费一个令牌才能处理。若桶中有积攒的令牌 → 可瞬时处理积攒的突发请求（只要令牌够）。`Guava RateLimiter` 默认即此实现。
  - Sentinel 扩展了：预热限流（冷启动时限制流量逐步增大，避免系统被瞬时大量请求压垮）和排队等待（请求不均匀时排队延后处理）。
- **实际场景**：阿里 Sentinel 广泛使用。更多细节可参考 [系统容量预估、限流、降级实践](overcapacity.html)。

### 7. 最终一致性的实现手段

- **消息队列 + 本地消息表**（最通用）：写本地业务数据和消息表在同一个本地事务内 → 定时任务将处理中的消息投递到 MQ → 消费者处理 → 回调更新消息状态。保证消息不丢、不乱序和至少投递一次。
- **幂等性保障**：MQ 可能投递多次，消费端必须幂等。方案：
  - 数据库唯一约束（如 `order_id` + `user_id` 唯一索引）。
  - 分布式锁 + 业务状态机（如状态从 `待付款` → `已付款`，更新时加 `WHERE status = '待付款'`，影响行数为 0 说明已处理过）。
  - 消费记录表（用 msg_id 做唯一键，成功记录）。
- **定时任务补偿**：定时扫描流水表/状态表，对处理失败或未完成的状态的流水重试处理直到达到终态。
- **Saga 补偿模式**：长事务拆成 T1, T2, ..., Tn，若 Ti 失败逆序执行 C_{i-1}, ..., C1 回滚。
- **TCC 的 Confirm/Cancel**：也是最终一致性的实现（虽然 TCC 的目标是强一致，但确认/取消阶段也可能最终一致重试多次）。
- **对账系统**：T+1 日终生成差异报表并自动补偿或人工介入（银行/支付领域必须）。
- **事件溯源**：保存所有事件而非最终状态，通过 replay 事件得到当前数据 → CQRS + Event store。

---

## 八、系统设计

### 1. 设计一个秒杀系统

秒杀系统的核心挑战是**写入热点**——同一时刻数十万/百万请求抢占极少数库存。设计的核心思路是**逐层削减流量，异步化下单**。

- **前端层**（削减 99%+）：秒杀按钮置灰/防抖防重复提交 + 动态验证码/答题（减轻瞬时并行度）+ 秒杀页面完全静态化（不经过后端动态渲染，推到 CDN）。
- **网关层**（限流与预校验）：Nginx/OpenResty 做**粗粒度库存检查**（读取 Redis 中的库存值，若 ≤ 0 则直接返回失败，不再进入后端）。令牌桶/漏桶对接口和用户维度限流。
- **服务层**：
  - Redis **原子预减库存**：`DECR` + 判 >= 0（或用 Lua：`if get(key) > 0 then decr(key) return 1 else return 0 end`）。库存为 0 后所有后续请求直接返回已售罄。
  - **异步下单**：库存预减成功后，将下单请求投递到 Kafka/RocketMQ，立即返回客户端"排队中"或"下单处理中"（用 WebSocket/长轮询 push 结果）。
  - 消费端从 MQ 消费 → 真正写 DB 扣库存 → 生成订单 → 更新缓存中的结果（成功/失败）。
- **数据库层**：
  - **防超卖**：`UPDATE product_stock SET stock = stock - 1 WHERE stock > 0 AND product_id = ?`（利用数据库行锁 + 条件更新）。判断影响行数 = 0 回滚。
  - **热点库存分桶**：将热点商品的库存拆分成多个桶（如 10 个桶各 100 个），请求按用户 ID hash 分配到不同桶减库存。竞争被分散，总的锁冲突大大降低。
  - 读写分离，订单表按用户 ID 分库分表。
- **降级与容灾**：各层有限流、熔断、兜底逻辑。大促前全链路压测找到瓶颈提前扩容。更多可参考 [秒杀系统设计原理：高并发下的极速响应](seckill.html)。

### 2. 设计一个短链系统

- **核心流程**：用户提交长 URL → 生成唯一短码 → 存储映射 → 返回短链。用户访问短链 → 查映射 → 302 重定向到长 URL。
- **短码生成方式**：
  - **发号器 + Base62 编码**（推荐）：采用自增 ID 生成器（如 Snowflake 或号段模式）生成唯一 ID → 转 62 进制（0-9 A-Z a-z, 共 62 个字符）。无冲突、可逆。10 亿以内的 ID 只需约 6 位短码。性能极高。
  - **哈希算法**：`MurmurHash64(longUrl)` → 取部分位 → Base62 编码。可能冲突，冲突时添加随机字符/自增后缀再哈希或拉取下一个未用的短码。不需要全局发号器但需要冲突处理。MD5/SHA 也可以但更长性能较差（且 128bit/256bit 安全哈希没必要在短链场景用）。
- **存储**：K-V 表，`short_code` → `long_url` + `create_time` + `expire_time`（可选）。短码作为主键（或分片键），Redis 缓存访问频繁的短链映射。
- **跳转**：访问短链 → 查 Redis → 存在则 302 返回长 URL（浏览器自动跳转），不存在则回源 DB（若 DB 仍不存在则返回 404）。302 为临时重定向（支持统计但不传递 SEO 权重），301 为永久重定向（浏览器缓存不重复访问服务端）。
- **扩展功能**：
  - 同一长 URL 返回相同短码：`long_url` 建唯一索引，先查后生成。
  - 访问统计：通过异步日志 + 流处理系统（Kafka + Flink）对 PV/UV/地理位置/设备做实时统计。
  - 自定义后缀：增加 `custom_code` 字段，生成时检查冲突。

### 3. 如何设计一个高并发系统

- **架构原则**：分而治之、异步解耦、缓存为王、无状态、可水平扩展。
- **分层架构**：
  - 接入层（DNS 负载/CDN/反向代理 Nginx）：DNS 智能解析和就近接入，静态资源推送至 CDN→ 大部分请求在边缘节点被拦截。
  - 网关层（API Gateway）：统一鉴权、限流、日志、协议转换、灰度路由。性能要求高，通常基于 Netty/OpenResty 异步非阻塞实现。
  - 业务服务层：微服务化，服务之间 RPC（Dubbo/gRPC）或 HTTP Restful 通信。每个服务均设计为无状态，Store 状态外置（Session 存 Redis / JWT 自包含）。配置横向自动扩缩容（K8s HPA + Nacos 服务上下线）。
  - 数据层：分层存储。热数据在分布式缓存（Redis Cluster + Caffeine 本地缓存多级缓存），温数据在 MySQL（读写分离，分库分表），冷数据归档至 HBase/Hive/对象存储。
- **异步处理**：非核心主链路的流程全部异步化——消息队列投递。主要的请求路径上只处理关键的即时性的步骤，其余通过事件异步驱动。
- **容量与保护**：
  - 全链路压测和容量规划。估算各层 QPS → 寻找瓶颈 → 扩容。
  - **限流**（令牌桶/Sentinel）、**熔断**（Hystrix/Resilience4j/Sentinel）、**降级**（开关 + 默认值/静态页面）、**隔离**（线程池隔离/信号量隔离）。
- **可观测性**：分布式链路追踪（SkyWalking, Jaeger + OpenTelemetry）用于跟踪请求在全链路的耗时分布。指标监控（Prometheus + Grafana）+ 日志（ELK/Loki）。
- **数据库高并发特化**：
  - 缓存优先（Cache Aside）：绝大多数读请求走 Redis → 缓存时效和一致性控制。
  - 预热（warmup）：系统启动后预加载热点数据到缓存（避免冷启动雪崩）。
  - 异步批量写：短时间内大量写操作合并为批量 INSERT（如缓存在 Redis 缓冲中，到一定量或时间窗口后批量写入 DB）。
