---
title: Java 语言基础
date: 2020-01-01 00:00:00
updated: 2020-01-01 00:00:00
tags:
  - Java
  - Lambda
  - Stream API
  - 函数式编程
  - 泛型
categories:
  - Java
---
# Java 语言基础

## 一、基础数据类型

| **Data Type** | **Default Value** | **Default size** | Range |
| :------------ | :---------------- | :--------------- | ----- |
| boolean       | false             | 1 bit            |       |
| char          | '\u0000'          | 2 byte           |       |
| byte          | 0                 | 1 byte           |       |
| short         | 0                 | 2 byte           |       |
| int           | 0                 | 4 byte           |       |
| long          | 0L                | 8 byte           |       |
| float         | 0.0f              | 4 byte           |       |
| double        | 0.0d              | 8 byte           |       |

## 二、内部类

### 1. 非静态成员内部类

这种情况下，内部类需要创建实例时，必须存在一个外部类的实例对象，通过 `OutterClass.new` 来创建。并且内部类拥有指向外部实例对象的指针，可以通过 `OutterClass.this` 访问外部类的属性和方法，包括 `private` 修饰的属性和方法。

### 2. 局部内部类

局部内部类存在方法体或作用域之中，超出方法体或作用域就会失效。

```java
public class LocalInnerCalss {
    interface Destionation {
        String getName();
    }

    public Destionation destionation(String str) {
        class DomesticDestination implements Destionation {
            private String name;

            private DomesticDestination(String whereTo) {
                name = whereTo;
            }

            public String getName() {
                return name;
            }
        }
        return new DomesticDestination(str);
    }
//    Destionation dest = new DomesticDestination("");//超出方法体无法访问

    private void foreign(boolean b) {
        if (b) {
            class ForeignDestination implements Destionation {
                private String name;

                ForeignDestination(String s) {
                    name = s;
                }

                public String getName() {
                    return name;
                }
            }
            ForeignDestination ts = new ForeignDestination("paris");
            String string = ts.getName();
            System.out.println(string);
        }
//        Destionation ts = new ForeignDestination("chenssy");//出了作用域无法创建对象
    }

    public static void main(String[] args) {
        LocalInnerCalss localInnerCalss = new LocalInnerCalss();
        Destionation d = localInnerCalss.destionation("shanghai");
        System.out.println(d.getName());
        localInnerCalss.foreign(true);
    }
}
```


```

## 四、接口

### 1. 接口成员变量

接口成员变量默认 `public static final`，所以可以省去这些修饰符。它的定位是公共静态不可修改的常量，是高级的抽象。

1. 如果不是 `static`，那么当一个类实现两个接口 `X`, `Y`，而两个接口都有同名变量 `a` 时，会无法区分变量；如果是 `static` 变量，则可以通过 `X.a`, `Y.a` 来区分。
2. 如果不是 `final`，则该接口的实现类可修改该值，则修改后的值对所有该接口的实现类可见，违反了开闭原则。

开闭原则：对修改关闭，对扩展（不同的实现 `implements`）开放。

### 2. 接口默认方法

在 Java 8 之前，接口与其实现类之间的**耦合度**太高了（**tightly coupled**），当一个接口添加方法时，所有的实现类都必须随之修改。默认方法解决了这个问题，它可以为接口添加新的方法，而不会破坏已有的接口的实现。这在 lambda 表达式作为 Java 8 语言的重要特性而出现之际，为升级旧接口且保持向后兼容（backward compatibility）提供了途径。[^1]

需要注意的是接口默认方法这个特性并不能使得接口取代抽象类的作用，接口和抽象类的区别如下：

- 接口可以被类多实现（被其他接口多继承），抽象类只能被单继承。
- 接口中没有 `this`，没有构造函数，不能拥有实例字段（实例变量，只能有 `public static final` 变量）或实例方法，无法保存**状态**（**state**），抽象类中可以。
- 抽象类不能在 Java 8 的 lambda 表达式中使用。
- 接口的 default methods 只能是 `public`，抽象类的 abstract method 可以是 `protected`。
- 从设计理念上，接口反映的是 **"like-a"** 关系，抽象类反映的是 **"is-a"** 关系。

**写法：**

```java
public interface DemoInterface {

    int code = 0;
    String msg = "hello";

    default void hello(){
        System.out.println(msg);
    }

}
```

**注意事项：**

默认方法不能和 `Object` 对象的方法重名，原因如下：a. 接口不能拥有状态，这意味着它在实现一些 `Object` 方法中存在一定的局限性；b. Java 中如果父类和接口拥有相同的方法，子类会去优先实现父类中的方法而不是接口中的方法，这样的话会使得接口默认方法没有意义。

## 五、函数式接口

### 1. @FunctionalInterface 注解

`@FunctionalInterface` 注解作用于接口，表示该接口是函数式接口（functional interface）。

函数式接口有以下几个特点：

1. 函数式接口有且只有一个抽象方法
2. 默认方法不计入抽象方法的个数，静态方法也不计入抽象方法个数
3. 覆盖 `java.lang.Object` 的方法的方法也不计入抽象方法的个数
4. 函数式接口可以通过 lambda 表达式、方法引用、构造器引用来创建实例
5. 对于编译器来说，只要一个接口符合函数式接口的定义就会将其当作函数式接口，不一定需要 `@FunctionalInterface` 注解标注，注解用于检查该接口是否只包含一个抽象方法

代码示例如下：

```java
@FunctionalInterface
public interface HelloFuncInterface {

    String hello();

    default String helloWorld() {
        return "hello world";
    }

    static void printHello() {
        System.out.println("Hello");
    }
}
```

### 2. JDK 中的函数式接口

- `java.lang.Runnable`
- `java.awt.event.ActionListener`
- `java.util.Comparator`
- `java.util.concurrent.Callable`
- `java.util.function` 包下的接口，如 `Consumer`、`Predicate`、`Supplier` 等

## 六、lambda 表达式

lambda 表达式用来实现函数式接口。

### 1. lambda 表达式形式

`(params) -> expression`
`(params) -> statement`
`(params) -> { statements }`

```java
/**
 * 通过lambda表达式实现自定义的函数式接口（注：Java 8 之前一般是用匿名类实现的）
 */
private void hello() {
    HelloFuncInterface helloFuncInterface = () -> {
        System.out.println("hello");
        return "hello";
    };
    helloFuncInterface.hello();
}
```

### 2. lambda 表达式替换匿名类，减少冗余代码

```java
new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + ":runnable by lambda expression");
}).start();

new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + ":runnable by anonymous class");
    }
}).start();
```

## 七、方法引用

方法引用由 `::` 双冒号操作符标示，看起来像 C++ 的作用域解析运算符。**实现抽象方法的参数列表，必须与方法引用方法的参数列表保持一致！至于返回值就不作要求。**

```java
/**
 * 方法引用
 */
private void methodReference() {
    List<String> pets = Arrays.asList("cat", "dog", "snake", "squirrel");
    pets.forEach((a) -> System.out.println(a));//lambda表达式
    pets.forEach(System.out::println);//方法引用
}
```

### 1. 引用方法[^2]

- **对象引用::实例方法名**

`System.out` 是一个静态成员变量对象：

```java
Consumer<String> consumer = System.out::println;
consumer.accept("hello");
```

- **类名::静态方法名**

```java
Function<Long, Long> abs = Math::abs;
System.out.println(abs.apply(-3L));
```

- **类名::实例方法名**

```java
BiPredicate<String, String> b = String::equals;
System.out.println(b.test("abc", "abcd"));
```

### 2. 引用构造器

在引用构造器的时候，构造器参数列表要与接口中抽象方法的参数列表一致，格式为 `类名::new`。如：

```java
Function<Integer, StringBuffer> fun = StringBuffer::new;
StringBuffer buffer = fun.apply(10);
```

`Function` 接口的 `apply` 方法接收一个参数，并且有返回值。在这里接收的参数是 `Integer` 类型，与 `StringBuffer` 类的一个构造方法 `StringBuffer(int capacity)` 对应，而返回值就是 `StringBuffer` 类型。上面这段代码的功能就是创建一个 `Function` 实例，并把它的 `apply` 方法实现为创建一个指定初始大小的 `StringBuffer` 对象。

### 3. 引用数组

引用数组和引用构造器很像，格式为 `类型[]::new`，其中类型可以为基本类型也可以是类。如：

```java
Function<Integer, int[]> fun = int[]::new;
int[] arr = fun.apply(10);

Function<Integer, Integer[]> fun2 = Integer[]::new;
Integer[] arr2 = fun2.apply(10);
```

## 八、java.util.function

### 1. Predicate 接口

```java
@FunctionalInterface
public interface Predicate<T> {

    /**
     * Evaluates this predicate on the given argument.
     *
     * @param t the input argument
     * @return {@code true} if the input argument matches the predicate,
     * otherwise {@code false}
     */
    boolean test(T t);
}
```

`Predicate` 是一个函数式接口，也是一个泛型接口，作用是对传入的一个泛型参数进行判断，并返回一个 `boolean` 值，任何符合这一条件的 lambda 表达式都可以转为 `Predicate` 接口，该接口的实现一般用于集合过滤等。下面提供一些例子：

```java
/**
 * lambda表达式加Predicate接口
 */
private void predicateDemo() {
    List<String> pets = Arrays.asList("cat", "dog", "snake", "squirrel");
    //简单的Predicate接口示例
    pets.stream().filter(s -> s.startsWith("s")).forEach(System.out::println);

    Predicate<String> startWith = s -> s.startsWith("s");
    Predicate<String> lengthPredicate = s -> s.length() > 2;
    //Predicate支持and()操作将两个Predicate接口的逻辑取与
    System.out.println("Predicate and() demo:");
    pets.stream().filter(startWith.and(lengthPredicate)).forEach(System.out::println);
    //Predicate支持or()操作将两个Predicate接口的逻辑取或
    System.out.println("Predicate or() demo:");
    pets.stream().filter(startWith.or(lengthPredicate)).forEach(System.out::println);
    //Predicate支持negate()操作将Predicate接口的逻辑取反
    System.out.println("Predicate negate() demo:");
    pets.stream().filter(startWith.negate()).forEach(System.out::println);
    //Predicate静态方法isEqual()
    System.out.println("Predicate isEqual() demo:");
    pets.stream().filter(Predicate.isEqual("cat")).forEach(System.out::println);
}
```

从示例中可以看到，`Predicate` 接口支持 `and()`, `or()`, `negate()` 等，并且有一个静态方法 `isEqual`。

### 2. Consumer 接口

```java
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * Returns a composed {@code Consumer} that performs, in sequence, this
     * operation followed by the {@code after} operation. If performing either
     * operation throws an exception, it is relayed to the caller of the
     * composed operation.  If performing this operation throws an exception,
     * the {@code after} operation will not be performed.
     *
     * @param after the operation to perform after this operation
     * @return a composed {@code Consumer} that performs in sequence this
     * operation followed by the {@code after} operation
     * @throws NullPointerException if {@code after} is null
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

`Consumer` 是一个函数式接口，也是一个泛型接口，作用是对传入的一个泛型参数进行操作，可能会对原数据产生 side effect，即改变数据。常见的例子有：

`Iterable` 接口的 `forEach` 方法接受一个 `Consumer` 接口类型的操作，对集合中的元素进行操作，例如打印等。

### 3. Function 接口

```java
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);
}
```

该接口代表接受一个参数并返回一个参数的操作类型。

### 4. Supplier 接口

```java
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

该接口表示调用后返回一个参数的操作类型，返回结果不要求一定是新创建的或者不同的。


```

## 参考文献

[^1]: https://www.zhihu.com/question/68474140/answer/263792395
[^2]: https://blog.csdn.net/TimHeath/article/details/71194938
[^3]: https://objcoding.com/2019/03/04/lambda/#top
[^4]: https://github.com/CarpenterLee/JavaLambdaInternals
[^5]: https://juejin.im/post/5abc9ccc6fb9a028d6643eea#heading-13
