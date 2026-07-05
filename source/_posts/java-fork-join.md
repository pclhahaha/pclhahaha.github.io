---
title: Fork/Join 框架与工作窃取
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - ForkJoin
  - 工作窃取
  - ForkJoinPool
categories:
  - Java
  - 并发
---

Java 7 引入了 Fork/Join 框架，通过 RecursiveTask 和 RecursiveAction 这两个类可以方便地实现 Fork/Join 模式：

- **RecursiveTask**：用于有返回值的计算任务，继承自 ForkJoinTask，需要实现 `compute()` 方法并返回结果
- **RecursiveAction**：用于无返回值的计算任务，同样继承自 ForkJoinTask，需要实现 `compute()` 方法，无返回值

Fork/Join 框架的异常处理：任务在执行过程中如果抛出异常，可以通过 ForkJoinTask 的 `get()` 方法获取到（`get()` 会将异常包装为 ExecutionException 抛出），也可以通过 `isCompletedAbnormally()` 和 `getException()` 方法来检查。

使用 Fork/Join 框架时需要特别注意任务切分的方式。JDK 用来执行 Fork/Join 任务的工作线程池大小默认等于 CPU 核心数，在一个 4 核 CPU 上，最多可以同时执行 4 个子任务。但是如果切分不当，执行时间可能会超出预期。

这是因为执行 `compute()` 方法的线程本身也是一个 Worker 线程，当对两个子任务分别调用 `fork()` 时，这个 Worker 线程会把两个子任务都分派给其他 Worker，而自己则停下来等待结果——白白浪费了 Fork/Join 线程池中的一个 Worker 线程。这导致原本 4 个子任务至少需要 7 个线程才能并发执行，反而增加了额外开销。

正确的做法是使用 `invokeAll()` 方法：查看 JDK 的 `invokeAll()` 方法源码可以发现，对于 N 个任务，其中 N-1 个任务会使用 `fork()` 交给其他线程异步执行，而 `invokeAll()` 会保留一个任务由当前线程直接调用 `compute()` 同步执行。这样充分利用了线程池，确保没有空闲的线程。

## 一、ForkJoinPool

ForkJoinPool 是专门为运行 ForkJoinTask 任务而实现的 ExecutorService，同样提供了管理和监控功能。与一般的 ExecutorService（如 ThreadPoolExecutor）相比，ForkJoinPool 的核心区别在于采用了 **work-stealing（工作窃取）** 算法，这正是它能够高效执行 Fork/Join 类型任务的关键所在。

##### 2.1.1 ForkJoinWorkerThreadFactory

ForkJoinWorkerThreadFactory 是 ForkJoinPool 的静态成员内部接口，用于为指定的 ForkJoinPool 创建 ForkJoinWorkerThread 工作线程。默认实现是 DefaultForkJoinWorkerThreadFactory，一般情况下直接使用默认实现即可。

##### 2.1.2 work-stealing 工作窃取算法

工作窃取算法的核心思想是：池中的每个工作线程都拥有一个双端队列（WorkQueue），用于存放待执行的任务。当线程完成自身队列中的所有任务后，会尝试从其他繁忙线程的队列尾部"窃取"任务来执行，而不是空闲等待。

具体来说：

- 每个 Worker 线程维护一个双端队列，新任务添加到队列头部
- Worker 线程从自身队列头部取出任务执行（LIFO，后进先出），这样有利于局部性和缓存命中
- 当 Worker 线程的队列为空时，它会随机选择一个其他 Worker 的队列，从队列的**尾部**窃取任务（FIFO，先进先出）
- 被窃取的通常是较大的、拆分较早的任务，这有助于保持工作负载的均衡

这种"自己从头取、别人从尾偷"的双端队列设计，最大程度地减少了窃取线程与被窃取线程之间的竞争，因为两者操作的是双端队列的不同端。

对应于第一节 3.4 中提到的双端队列概念，Fork/Join 框架正是工作窃取算法最经典的应用场景。

##### 2.1.3 ForkJoinWorkerThread

ForkJoinWorkerThread 是由 ForkJoinWorkerThreadFactory 创建的线程，每个线程都绑定到 ForkJoinPool 中的一个 WorkQueue。线程在运行时会不断地从自己的队列或者通过工作窃取从其他队列获取并执行 ForkJoinTask。

##### 2.1.4 WorkQueue

WorkQueue 是 ForkJoinPool 内部的双端队列实现，用于存放 ForkJoinTask。Pool 中所有线程都会尝试从各自队列或其他队列中寻找并执行任务，当找不到任何可执行的任务时，线程会阻塞等待新的任务到来。

##### 2.1.5 ForkJoinTask

ForkJoinTask 是 Fork/Join 框架中任务的抽象基类，RecursiveTask 和 RecursiveAction 均继承自它。ForkJoinTask 提供了 `fork()` 方法用于异步执行任务（将任务放入当前线程的 WorkQueue 中排队），以及 `join()` 方法用于等待任务执行完成并获取结果。
