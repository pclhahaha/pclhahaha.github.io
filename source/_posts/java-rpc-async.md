---
title: RPC 与异步编程
date: 2020-03-08 00:00:00
updated: 2020-03-08 00:00:00
tags:
  - Java
  - RPC
  - gRPC
  - CompletableFuture
  - RxJava
  - Reactor
  - 异步编程
categories:
  - Java
---

# RPC 与异步编程
---

## 第一部分：从 Protocol Buffers 到 gRPC

### 一、Protocol Buffers 基础

Protocol Buffers 的介绍文档：

https://developers.google.com/protocol-buffers/docs/overview

#### 1.1 proto3

gRPC 使用的是 Protocol Buffers 的 proto3 版本。

https://developers.google.com/protocol-buffers/docs/proto3

Java 语言使用 Protocol Buffers 的教程如下：

https://developers.google.com/protocol-buffers/docs/javatutorial#compiling-your-protocol-buffers

### 二、gRPC 服务定义与四种 RPC 类型

#### 2.1 服务定义

```protobuf
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

#### 2.2 四种 RPC 类型

- **Unary RPC**：一元调用，客户端发送一个请求，服务端返回一个响应。

  ```protobuf
  rpc SayHello(HelloRequest) returns (HelloResponse) {
  }
  ```

- **Server streaming RPC**：服务端流式 RPC，客户端发送一个请求，服务端返回一个流。

  ```proto
  rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse) {
  }
  ```

- **Client streaming RPC**：客户端流式 RPC，客户端发送一个流，服务端返回一个响应。

  ```proto
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) {
  }
  ```

- **Bidirectional streaming RPC**：双向流式 RPC，客户端和服务端都可以发送流。

  ```proto
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse) {
  }
  ```

### 三、gRPC 认证

- **Credentials**：channel credentials / call credentials
- **SSL/TLS**
- **Token-based authentication with Google**

### 四、gRPC 核心特性

使用场景：

- 微服务架构
- 边缘计算：边缘设备连接到后端服务，移动设备、浏览器、IoT
- 生成高效的客户端

核心特性：

- Idiomatic client libraries in 10 languages
- Highly efficient on wire and with a simple service definition framework
- Bi-directional streaming with HTTP/2 based transport
- Pluggable auth, tracing, load balancing and health checking

#### 4.1 HTTP/2 与双向 streaming

gRPC 基于 HTTP/2 传输，天然支持双向流式通信。HTTP/2 的多路复用特性使得单个连接上可以同时进行多个请求和响应，避免了 HTTP/1.1 的 head-of-line blocking 问题。这为 gRPC 的四种 RPC 类型提供了高效的底层传输能力。

---

## 第二部分：从异步编程到响应式编程

Q：响应式编程带来了什么好处？

A：当并发连接增加，很多时候我们是通过多线程、线程池的方式来提升性能，然而线程数增加后系统开销也增加了，而且很多场景下线程大部分是执行 IO 操作，其大部分时间是处于 idle 状态的，资源利用率不高。响应式编程通过引入非阻塞 non-blocking 的特性，实现了线程资源的充分利用。

这是一幅盗来的图[^1]，展示了响应式编程的发展历程，尤其是在 JVM 或者说 JAVA 领域。这里面比较核心的就是 Reactive Streams 规范、ReactiveX 项目及其包含的 RxJava、Project Reactor 项目、JAVA方面的 CompletableFuture API 和 Flow API。

<img src="/java/rpc-async/image-blog-body-completablefuture3.jpg" alt="CompletableFuture API and Reactive Programming Timeline" style="zoom: 67%;" />

### 五、CompletableFuture

首先，CompletableFuture 是 Future，意味着它支持任务异步执行。在此基础上，它还支持异步任务之间的依赖关系、异步任务的回调等。对于日常业务逻辑，使用 CompletableFuture 能够通过将业务逻辑异步化，来提升 cpu 利用率、缩短响应时间，达到提升性能的目的。

那么 CompletableFuture 和响应式编程的区别是什么呢？

关键区别在于 **Stream** 这个概念上，CompletableFuture 描述的是单次执行的结果，尽管可以通过异步任务之间的依赖关系构建一串异步任务组成的执行图，本质上面向的是单次操作，例如一次 rpc 请求的处理[^1]。

<img src="/java/rpc-async/image-blog-body-completablefuture2.jpg" alt="Java CompletableFuture API Example Invoice Path" style="zoom:67%;" />

响应式编程则面向的是 Stream，或者说叫"流"，也即是数据或事件的序列，有点类似 JAVA 的 Stream API。

当然用 CompletableFuture 也能够实现对数据/事件流的处理，但是有一个关键的特性是响应式编程具备而 CompletableFuture 则需要重新实现的，并且实现起来很复杂，那就是 **back pressure**。back pressure 就是在下游无法承载上游的压力时，采取一些措施。

至于 Java 中提供的 CallBack、Future 机制在实现响应式编程中的问题和缺点，例如"callback hell"、不支持 lazy computation 等，可以从 https://projectreactor.io/docs/core/release/reference/#getting-started 中找到答案。

所以，我们需要一套框架或者说类库来更好的实现响应式编程，需要具备以下特性：

- 支持异步任务的封装及组装，需要 API 来对异步任务进行包装，并且需要很多种算子来对异步任务进行组装，包括过滤、组装、异常处理等算子
- 减少异步任务的嵌套，减少代码的复杂性，增加可读性，避免"callback hell"等复杂度很高的代码
- 支持背压 back pressure，也就是下游和上游之间能协商数据流的速度

### 六、响应式编程——从 ReactiveX 到 RxJava

项目主页：http://reactivex.io/

github：https://github.com/ReactiveX?page=2

官网定义如下：

> [ReactiveX](http://reactivex.io/) is a collection of open source projects. The content of this page is licensed under Creative Commons Attribution 3.0 License, and code samples are licensed under the BSD License.

> ReactiveX is a combination of the best ideas from the Observer pattern, the Iterator pattern, and functional programming

> ReactiveX is a library for composing asynchronous and event-based programs by using observable sequences.

ReactiveX 基于观察者模式、迭代者模式和函数式编程的组合，为构建异步的基于事件的应用提供了一套类库，并且它是一个多语言的项目，例如，JAVA 中对应的就是 RxJava 类库。

#### 6.1 核心概念

##### 6.1.1 Observable[^2]

<img src="/java/rpc-async/legend.png" alt="img" style="zoom: 50%;" />

如上图所示，一个 Observable 其实就是一个产生数据流的对象，它能够转换为另一个 Observable。

Observable 可以被 Observer 订阅（subscribe），一个完整的 Observer 订阅 Observable 的例子的伪代码如下：

```groovy
def myOnNext     = { item -> /* do something useful with item */ };
def myError      = { throwable -> /* react sensibly to a failed call */ };
def myComplete   = { /* clean up after the final response */ };
def myObservable = someMethod(itsParameters);
myObservable.subscribe(myOnNext, myError, myComplete);
// go on about my business
```

其中 onNext、onError、onComplete 三个方法是 Observer 需要实现的，分别用于对数据流的正常处理、异常处理、完成操作。

##### 6.1.2 Observable 的各种算子（Operator）

算子可用于连接 Observable，或者改变 Observable 的行为。Operator 能够将 Observable 串联起来，形成一条 chain。

ReactiveX 支持的算子数量较多，这里就不展开了，可以到 http://reactivex.io/documentation/operators.html 这里查看。

##### 6.1.3 Single

Single 是 RxJava 中引进的类似 Observable，但只产生一个数据的对象，因此订阅它的 Observer 只需要实现 onSuccess、onError 方法即可。

Single 也有很多对应的算子，可用于创建 Single、对 Single 进行转换、组装、连接、延时等，具体可查看：http://reactivex.io/documentation/single.html

##### 6.1.4 Subject

http://reactivex.io/documentation/subject.html

一个 Subject 既是 Observable，又是 Observer，表明它既可以订阅一个或多个 Observable，又可以作为 Observable 将其订阅到的数据流重新发出，因此它类似一种桥梁或者说代理。

##### 6.1.5 Scheduler

http://reactivex.io/documentation/scheduler.html

Rx 默认是单线程的，整条链路上的调用都在 subscribe 方法被调用的线程中执行。不过一些算子接收 Scheduler 作为参数，控制算子在哪个 Scheduler 上执行，这里可以认为 Scheduler 是某种线程池，提供了多线程资源。

- **SubscribeOn** 算子指定 Observable 开始执行的线程，而 SubscribeOn 算子在链路的什么地方被调用并不重要。
- **ObserveOn** 算子指定了从该调用开始往后的算子所在的执行线程，如果要改变线程，则需要重新调用 ObserveOn，指定一个新的 Scheduler。

#### 6.2 RxJava

github：https://github.com/ReactiveX/RxJava

github wiki：https://github.com/ReactiveX/RxJava/wiki

RxJava 是 ReactiveX 项目的一部分，它也遵守 Reactive Streams 规范。不要以为 RxJava 是只针对 Java 语言的，RxJava 本身也是支持多语言的，支持一堆基于 JVM 的语言。下面是 RxJava 的最最最入门级 demo，包括 Observable 的创建、组装、订阅：

```java
public class RxJavaDemo {

    public static void main(String[] args) {
        RxJavaDemo demo = new RxJavaDemo();
        demo.hello("pcl", "j");
        demo.customObservableBlocking().subscribe(s -> System.out.println(s));
        demo.customObservableNonBlocking().subscribe(s -> System.out.println(s));
        demo.simpleComposition();
    }
    /**
     * hello world
     *
     * @param args
     */
    private void hello(String... args) {
        Flowable.fromArray(args).subscribe(s -> System.out.println("Hello " + s + "!"));
    }
    /**
     * Creating an Observable from an Existing Data Structure
     */
    private void createObservables() {
        Observable<String> o1 = Observable.fromArray(new String[]{"a", "b", "c"});

        Integer[] list = {5, 6, 7, 8};
        Observable<Integer> o2 = Observable.fromArray(list);

        Observable<String> o3 = Observable.just("one object");
    }
    /**
     * This example shows a custom Observable that blocks
     * when subscribed to (does not spawn an extra thread).
     *
     * @return
     */
    private Observable<String> customObservableBlocking() {
        return Observable.<String>create(emitter -> {
            for (int i = 0; i < 50; i++) {
                emitter.onNext("count_" + i);
            }
            emitter.onComplete();
        });
    }
    /**
     * 创建一个非阻塞的Observable
     *
     * @return
     */
    private Observable<String> customObservableNonBlocking() {
        return Observable.<String>create(emitter -> {
            new Thread() {
                @Override
                public void run() {
                    for (int i = 0; i < 50; i++) {
                        emitter.onNext("count_" + i);
                    }
                    emitter.onComplete();
                }
            }.start();
        });
    }
    /**
     * 用算子对Observable进行组装，或者转变
     */
    private void simpleComposition() {
        customObservableNonBlocking().skip(10).take(5)
                .map(stringValue -> {return stringValue + "_xform";})
                .subscribe(s -> System.out.println("onNext => " + s));
    }
}
```

其中 simpleComposition() 方法的 marble diagram 如下图所示[^3]，在 reactive 编程模型中，常常用 marble diagram 表示数据流的流转。

<img src="/java/rpc-async/Composition.1.png" alt="img" style="zoom:25%;" />

##### 6.2.1 异常处理

Observable 一般不抛出异常，而是发出一个 onError 通知，不过有些情况下，例如调用 onError() 方法本身失败的话，会抛出 RuntimeException、OnErrorFailedException、OnErrorNotImplementedException 异常。

可以使用异常处理算子来处理 onError() 方法通知的异常。subscribe 的时候可以传入第二个方法处理异常。

##### 6.2.2 RxJava 的算子

只能说 RxJava 支持的算子很多，就不多说了，当然你也可以自己写新的算子，这就属于比较难的操作了。

##### 6.2.3 Back Pressure

其实从上面一路写下来，有没有发现 RxJava 和 Stream API 非常相似，我是这么觉得的。但是这里就要说一说 RxJava 所具备的背压这一特性，是响应式编程的非常重要的一个特性。

back pressure 就是在下游无法承载上游的压力时，采取一些措施，通知上游放慢速度。

```java
PublishProcessor<Integer> source = PublishProcessor.create();

source
.observeOn(Schedulers.computation())
.subscribe(v -> compute(v), Throwable::printStackTrace);

for (int i = 0; i < 1_000_000; i++) {
    source.onNext(i);
}

Thread.sleep(10_000);
```

这段代码会抛出 MissingBackpressureException 异常，因为 PublishProcessor 不支持背压，而 observeOn 算子内部的 buffer 是有边界的，当 PublishProcessor 产生数据的速度超过计算的速度时，数据会存在 observeOn 的内部 buffer 中，当溢出时就抛出 MissingBackpressureException 异常。

如果改成下面的代码则可以正常运行，因为 Flowable.range 支持背压，range 可以和 observeOn 之间通过类似协程的机制，协商应该以什么样的速度产生数据。具体的机制是，range 通过调用 observeOn 的 onSubscribe 方法发送一个 callback 方法（org.reactivestreams.Subscription 接口的实现）给订阅者，observeOn 回调 Subscription.request(n) 方法告诉 range 产生 n 个数据。

```java
Flowable.range(1, 1_000_000)
.observeOn(Schedulers.computation())
.subscribe(v -> compute(v), Throwable::printStackTrace);

Thread.sleep(10_000);
```

下面通过一个更显式的例子来说明，先调用 onStart 方法要求 range 发送一个数据，然后异步调用 onNext 方法进行数据计算，并再发一个 request 要求一个数据。注意 onStart 中 request 执行完后，range 就会发送数据，会异步触发 onNext，若此时 onStart 中初始化操作尚未完成，则可能产生异常：

```java
Flowable.range(1, 1_000_000)
        .subscribe(new DisposableSubscriber<Integer>() {
            @Override
            public void onStart() {
                request(1);
            }
            public void onNext(Integer v) {
                v = v * 2;
                System.out.println(v);
                request(1);
            }
            @Override
            public void onError(Throwable ex) {
                ex.printStackTrace();
            }
            @Override
            public void onComplete() {
                System.out.println("Done!");
            }
        });
```

##### 6.2.4 解决背压问题

**方案一：增加 buffer size**

```java
PublishProcessor<Integer> source = PublishProcessor.create();

source.observeOn(Schedulers.computation(), 1024 * 1024)
      .subscribe(e -> { }, Throwable::printStackTrace);

for (int i = 0; i < 1_000_000; i++) {
    source.onNext(i);
}
```

仍然有机会产生 MissingBackpressureException。

**方案二：批量算子 / 采样算子**

```java
PublishProcessor<Integer> source = PublishProcessor.create();

source
      .buffer(1024)
      .observeOn(Schedulers.computation(), 1024)
      .subscribe(list -> {
          list.parallelStream().map(e -> e * e).first();
      }, Throwable::printStackTrace);

for (int i = 0; i < 1_000_000; i++) {
    source.onNext(i);
}

PublishProcessor<Integer> source = PublishProcessor.create();

source
      .sample(1, TimeUnit.MILLISECONDS)
      .observeOn(Schedulers.computation(), 1024)
      .subscribe(v -> compute(v), Throwable::printStackTrace);

for (int i = 0; i < 1_000_000; i++) {
    source.onNext(i);
}
```

仍然有机会产生 MissingBackpressureException。

**方案三：onBackpressureXXX 算子**

- `onBackpressureBuffer()`
- `onBackpressureBuffer(int capacity)`
- `onBackpressureBuffer(int capacity, Action onOverflow)`
- `onBackpressureBuffer(int capacity, Action onOverflow, BackpressureOverflowStrategy strategy)`
- `onBackpressureDrop()`
- `onBackpressureLatest()`

还有一个办法是创建支持背压的数据源。

### 七、响应式编程——从 Reactive Streams 到 Project Reactor

上文提到 RxJava 属于 ReactiveX 项目，也遵守 Reactive Streams 规范，这里就介绍一下 Reactive Streams 规范，以及在其基础上的 Project Reactor。

#### 7.1 Reactive Streams

主页：https://www.reactive-streams.org/

> Reactive Streams is an initiative to provide a standard for asynchronous stream processing with non-blocking back pressure. This encompasses efforts aimed at runtime environments (JVM and JavaScript) as well as network protocols.

Reactive Streams 是一套规范，为异步流处理及非阻塞的背压提供了标准。

对于 Java 程序员，Reactive Streams 是一个 API。Reactive Streams 为我们提供了 Java 中的 Reactive Programming 的通用 API。`Reactive Streams API` 中仅仅包含了如下四个接口[^4]：

```java
//发布者
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}

//订阅者
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}

//表示Subscriber消费Publisher发布的一个消息的生命周期
public interface Subscription {
    public void request(long n);
    public void cancel();
}

//处理器，表示一个处理阶段，它既是订阅者也是发布者，并且遵守两者的契约
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {

}
```

https://github.com/reactive-streams/reactive-streams-jvm 定义了 Reactive Streams 的 JVM 规范。

**Java 9 中 Flow 类下的内容与 Reactive Streams 完全一致，这部分留到后面研究。**

其他还有 AKKA Streams 支持 Reactive Streams。

Publisher-Subscriber 接口既有观察者模式，又包含了迭代器模式。Subscriber 通过订阅的方式实现了观察者模式。由 Publisher 通过 push 元素实现对元素的迭代，而相应的 Java 中的 Iterator 是通过 pull 的方式实现迭代。push 的方式是响应式编程的关键，Publisher 通过调用 Subscriber 的 onNext 方法通知 Subscriber 下一个元素，通过 onError 通知异常，通过 onComplete 通知完成。

#### 7.2 Project Reactor

主页：https://projectreactor.io/

> Reactor is a fourth-generation reactive library, based on the [Reactive Streams](https://github.com/reactive-streams/reactive-streams-jvm) specification, for building non-blocking applications on the JVM

Reactor 文档：

https://projectreactor.io/docs/core/release/reference/

Reactor Core 运行需要 Java 8+ 环境，可以直接和 Java 8 的函数式 API 交互，包括 CompletableFuture、Stream 和 Duration。它实现了 Reactive Streams 规范，并提供 **Flux** 和 **Mono** 两个异步序列 API——Flux 是支持 N 个元素的序列，Mono 是支持 0 个或 1 个的序列（类似 RxJava 中的 Single）。**reactor-netty** 项目支持非阻塞的 IPC (inter-process communication)，它还提供了支持 HTTP (包括 WebSockets)、TCP、UDP 协议的带有背压特性的网络引擎，支持响应式的编解码，适合微服务框架。

Reactor 实现了 Reactive Streams 规范，因此实现了 Publisher、Subscriber 接口。需要注意的一点是 Publisher 创建后，并不开始产生数据，各种算子对 Publisher 的包装也不会导致数据开始产生，只有当调用链由 Subscriber 订阅后，内部调用 request 方法才会触发数据开始流动。

Reactor 背压的实现机制和 RxJava 类似，也是通过上面提到的 request 机制来进行协商。

##### 7.2.1 Flux 与 Mono

Flux 与 Mono 是 Publisher 的实现，其创建代码示例如下：

```java
private void createFluxMono() {
    Flux<String> seq1 = Flux.just("foo", "bar", "foobar");

    List<String> iterable = Arrays.asList("foo", "bar", "foobar");
    Flux<String> seq2 = Flux.fromIterable(iterable);

    Mono<String> noData = Mono.empty();

    Mono<String> data = Mono.just("foo");

    Flux<Integer> numbersFromFiveToSeven = Flux.range(5, 3);
}
```

其 subscribe 代码示例如下：

```java
private void subscribe() {
    //最后一个函数表示对每一个元素执行的操作
    Flux<Integer> ints1 = Flux.range(1, 3);
    ints1.subscribe(i -> System.out.println(i));

    //最后一个函数表示异常时执行的操作
    Flux<Integer> ints2 = Flux.range(1, 4)
            .map(i -> {
                if (i <= 3) return i;
                throw new RuntimeException("Got to 4");
            });
    ints2.subscribe(i -> System.out.println(i),
            error -> System.err.println("Error: " + error));

    //最后一个函数表示完成时执行的操作
    Flux<Integer> ints3 = Flux.range(1, 4);
    ints3.subscribe(i -> System.out.println(i),
            error -> System.err.println("Error " + error),
            () -> System.out.println("Done"));

    //最后一个函数表示订阅时执行的操作
    Flux<Integer> ints4 = Flux.range(1, 4);
    ints4.subscribe(i -> System.out.println(i),
            error -> System.err.println("Error " + error),
            () -> System.out.println("Done"),
            sub -> sub.request(2));
}
```

以上是 Reactor 的一些基操，其他部分的内容包括算子的使用、背压的实现、异步的实现、异常处理、测试、Debug 以及一些高级特性等，建议还是看官方文档，不是一篇文章能讲完的。

总的来说，Reactor 的实现和 RxJava 在很多方面是相似的。

### 八、Spring WebFlux

<img src="/java/rpc-async/diagram-reactive-1290533f3f01ec9c57baf2cc9ea9fa2f-20200306130014908.svg" alt="Spring Reactive diagram"  />

在 Project Reactor 提供的类库的基础上，Spring 构建了自己的 reactive 技术栈。Spring WebFlux 可以运行在 Netty、Undertow 服务器上或者 Servlet 3.1+ 的 web 容器中。

再一次推荐官方文档：

https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux

---

## 参考文献

[^1]: https://www.jrebel.com/blog/java-completablefuture-api
[^2]: http://reactivex.io/documentation/observable.html
[^3]: https://github.com/ReactiveX/RxJava/wiki/How-To-Use-RxJava
[^4]: https://zhuanlan.zhihu.com/p/95966853
