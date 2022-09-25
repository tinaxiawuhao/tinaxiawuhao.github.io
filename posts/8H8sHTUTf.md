---
title: 'java中静态代码块，非静态代码块，构造函数之间的执行顺序'
date: 2021-05-01 16:34:46
tags: [java]
published: true
hideInList: false
feature: /post-images/8H8sHTUTf.png
isTop: false
---

它们之间的执行顺序为：静态代码块—>非静态代码块—>构造方法。
静态代码块只在第一次加载类的时候执行一次，之后不再执行；而非静态代码块和构造函数都是在每new一次就执行一次，只不过非静态代码块在构造函数之前执行而已。
如果存在子类，则加载顺序为先父类后子类。
看如下的代码：
```java
package com.ykp.test;
 
class ClassA {
    public ClassA() {
        System.out.println("父类构造函数");
    }
 
    {
        System.out.println("父类非静态代码块1");
    }
    {
        System.out.println("父类非静态代码块2");
    }
    static {
        System.out.println("父类静态代码块 1");
    }
    static {
        System.out.println("父类静态代码块 2");
    }
}
 
public class ClassB extends ClassA {
    public ClassB() {
        System.out.println("子类构造函数");
    }
 
    {
        System.out.println("子类非静态代码块2");
    }
    {
        System.out.println("子类非静态代码块1");
    }
    static {
        System.out.println("子类静态代码块 2");
    }
    static {
        System.out.println("子类静态代码块 1");
    }
 
    public static void main(String[] args) {
        System.out.println("....主方法开始....");
        new ClassB();
        System.out.println("************");
        new ClassB();
        System.out.println("....主方法结束....");
    }
}
```

执行结果：
```java
父类静态代码块 1
父类静态代码块 2
子类静态代码块 2
子类静态代码块 1
....主方法开始....
父类非静态代码块1
父类非静态代码块2
父类构造函数
子类非静态代码块2
子类非静态代码块1
子类构造函数
************
父类非静态代码块1
父类非静态代码块2
父类构造函数
子类非静态代码块2
子类非静态代码块1
子类构造函数
....主方法结束....
```

从结果可以看出，首先加载类，加载的时候先加载父类，然后子类，类加载的时候就执行静态代码快，也就是说静态代码块是在类加载的时候就加载的，而且只加载一次。如果存在多个执行顺序按照代码的先后来。
对于非静态代码块，是在new的时候加载的，只是在构造函数之前加载而已。如果存在多个执行顺序按照代码的先后来。