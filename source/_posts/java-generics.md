---
title: Java 泛型与类型擦除
date: 2026-07-05
updated: 2026-07-05
tags:
  - Java
  - 泛型
  - 类型擦除
  - 通配符
categories:
  - Java
---

### 1. 通配符

`<? extends T>` ：上界通配符（upper bounds wildcards）

`<? super T>` ：下界通配符（lower bounds wildcards）

```java
public class GenericDemo {

    interface Lang {
        String getName();
    }

    class Java implements Lang {

        @Override
        public String getName() {
            return "java";
        }
    }

    class Java8 extends Java {

        @Override
        public String getName() {
            return "java8";
        }
    }

    class Coder<T> {

        T lang;

        public Coder(T lang) {
            this.lang = lang;
        }

        public Coder() {
        }

        public T getLang() {
            return lang;
        }

        public void setLang(T lang) {
            this.lang = lang;
        }
    }

    private void testExtends() {
//        Coder<Lang> coder1 = new Coder<Java>(new Java());//编译器认为Coder<Lang> Coder<Java>类型不匹配
        Coder<? extends Lang> coder2 = new Coder<>(new Java());//通过上界通配符Coder<? extends Lang>获得了通配能力，可以匹配Lang及Lang的子类
        /**
         * 编译器无法判断<? extends Lang>的具体类型，只知道其中元素是Lang的子类，
         * 假如可以set，则任意Lang的子类都可以存入，泛型不再安全，因而java设计为使用extends时无法set
         */
//        coder2.setLang(new Java());
        System.out.println(coder2.getLang().getName());

        Coder<? extends Lang> coder3 = new Coder<>(new Java8());
        System.out.println(coder3.getLang().getName());
    }

    private void testSuper() {
        Coder<? super Java> coder = new Coder<>();//super使得可以匹配Java及Java的基类
        coder.setLang(new Java8());
        System.out.println(coder.getLang().getClass());
        coder.setLang(new Java());//两种类型都能存入，实际发生了向上转型，通过向上转型保证了泛型的安全，但丢失了子类的部分信息
        System.out.println(coder.getLang().getClass());//返回的类型只能是Object，丢失了类信息
    }

    public static void main(String[] args) {
        GenericDemo demo = new GenericDemo();
        demo.testExtends();
        demo.testSuper();

    }
}
```
