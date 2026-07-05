---
title: Java Stream API 深度解析
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - Stream API
  - 函数式编程
  - 并行流
categories:
  - Java
---

### 1. Stream 接口概述

Stream 表示支持串行或并行的聚合操作的元素序列，除了 `Stream` 接口之外，还有 `IntStream`、`LongStream`、`DoubleStream`，都符合 Stream 的定义。

Stream 的计算操作可以被组装为 Stream 管道操作，如下代码示例：

```java
int sum = widgets.stream()
                      .filter(w -> w.getColor() == RED)
                      .mapToInt(w -> w.getWeight())
                      .sum();
```

包括一个 source（可以是数组 array、集合 collection、生成器函数 generator function、I/O channel 等），>=0 个中间操作（将一个 Stream 转换为另一个 Stream，例如 `filter`），以及一个结束操作（产生一个结果或副作用 side-effect，例如 `count()` 或 `forEach()`）。只有当结束操作存在，对 Stream 的计算才会开始，source 的元素只有在需要时才会被消费。

Stream 和 Collection 的实现目的是不同的，Collections 主要关心的是如何管理和访问其中的元素，Streams 不提供直接访问元素的方式，而是关注如何声明式地描述其 source 以及针对 source 的计算。不过 Stream 也提供了遍历元素的操作：`iterator()`、`spliterator()`。

一般来说，针对 source 的操作不能修改 source，除非 source 被明确设计为能够支持并发操作，否则会产生无法预测的甚至错误的后果。大部分操作接受的参数是用户指定的行为，例如 lambda 表达式，对这些表示行为的参数，一般的要求是：1. 不修改 source；2. 无状态。一般来说，这样的参数是函数式接口的实现，例如：lambda 表达式、方法引用，并且一般不能为 `null`。

**注意事项：**

1. 一个 Stream 只能被使用一次。
2. Stream 实现了 `AutoCloseable` 接口并有一个 `close()` 方法，一般只有当 source 是 I/O channel 的 Streams 才需要关闭，这种情况下可放在 `try-with-resources` 中声明。
3. Stream 管道操作可以串行或并行执行。

### 2. Collection 接口的 stream() 与 parallelStream() 方法

`Collection` 接口，即各种集合类的最上层接口，支持通过 `stream()`、`parallelStream()` 方法产生 Stream。源码如下：

```java
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}

default Stream<E> parallelStream() {
    return StreamSupport.stream(spliterator(), true);
}
```

其中 `spliterator()` 返回 `Spliterator` 接口，该接口用于对 Collection 进行遍历或分片（`parallelStream` 的情况下）。

### 3. Stream 接口及其实现 ReferencePipeline

`Stream` 接口支持一系列操作，包括 `filter`、`map`、`distinct`、`sorted`、`peek`、`limit`、`skip` 等方法，这些方法仍然返回 `Stream` 接口；还支持 `reduce`、`forEach`、`min`、`max`、`collect`、`count`、`toArray` 等方法，这些方法不再返回 `Stream` 接口，是对流式操作的结束操作，是 TerminalOp。

`Stream` 接口的具体实现是 `ReferencePipeline` 类，执行具体的操作。

对于 `parallelStream`，不同的操作有不同的实现，各自的实现会决定是否真正并行操作。例如，`reduce` 操作会调用 `ReduceOps` 执行具体操作，`ReduceOps` 中 `ReduceOp` 类的部分源码如下：

```java
@Override
public <P_IN> R evaluateParallel(PipelineHelper<T> helper,
                                 Spliterator<P_IN> spliterator) {
    return new ReduceTask<>(this, helper, spliterator).invoke().get();
}
```

其中 `ReduceTask` 实现了 `ForkJoinTask`，表明它是通过 fork/join 框架实现的并行处理。

> 关于 Stream API 原理的更多细节可参考文末参考文献。[^3][^4]

### 4. 生成器

Stream 为生成器的创建提供了便捷，生成器的好处是只有在你需要的时候才生成数据或对象，不需要提前生成。

例如，`IntStream` 的 `generate()` 方法就生成一个流，可以无限调用生成整数：

```java
public static IntStream generate(IntSupplier s) {
    Objects.requireNonNull(s);
    return StreamSupport.intStream(
            new StreamSpliterators.InfiniteSupplyingSpliterator.OfInt(Long.MAX_VALUE, s), false);
}
```
