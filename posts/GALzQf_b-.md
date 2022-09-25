---
title: 'list、set、map等集合类线程不安全的问题及解决方法'
date: 2021-09-03 16:34:53
tags: [java]
published: true
hideInList: false
feature: /post-images/GALzQf_b-.png
isTop: false
---

### List

ArrayList不是线程安全类，在多线程同时写的情况下，会抛出java.util.ConcurrentModificationException异常。

```java
private static void listNotSafe() {
    List<String> list=new ArrayList<>();
    for (int i = 1; i <= 30; i++) {
        new Thread(() -> {
            list.add(UUID.randomUUID().toString().substring(0, 8));
            System.out.println(Thread.currentThread().getName() + "\t" + list);
        }, String.valueOf(i)).start();
    }
}
```

解决方法：

1. 使用Vector（ArrayList所有方法加synchronized，太重）。
2. 使用Collections.synchronizedList()转换成线程安全类。
3. 使用java.concurrent.CopyOnWriteArrayList（推荐）。
   CopyOnWriteArrayList这是JUC的类，通过写时复制来实现读写分离。比如其add()方法，就是先复制一个新数组，长度为原数组长度+1，然后将新数组最后一个元素设为添加的元素。

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //得到旧数组
        Object[] elements = getArray();
        int len = elements.length;
        //复制新数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //设置新元素
        newElements[len] = e;
        //设置新数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```


### Set
跟List类似，HashSet和TreeSet都不是线程安全的，与之对应的有CopyOnWriteSet这个线程安全类。这个类底层维护了一个CopyOnWriteArrayList数组。

```java
private final CopyOnWriteArrayList<E> al;
public CopyOnWriteArraySet() {
    al = new CopyOnWriteArrayList<E>();
}
```

使用Collections.synchronizedList()转换成线程安全类。

HashSet和HashMap
HashSet底层是用HashMap实现的。既然是用HashMap实现的，那HashMap.put()需要传两个参数，而HashSet.add()只传一个参数，这是为什么？实际上HashSet.add()就是调用的HashMap.put()，只不过Value被写死了，是一个private static final Object对象。

### Map
HashMap不是线程安全的，Hashtable是线程安全的，但是跟Vector类似，太重量级。所以也有类似CopyOnWriteMap，只不过叫ConcurrentHashMap。

关于集合安全类

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CopyOnWriteArraySet;

public class ContainerNotSafeDemo {
    public static void main(String[] args) {
        listNotSafe();
        setNoSafe();
        mapNotSafe();
    }

    private static void mapNotSafe() {
        //Map<String,String> map=new HashMap<>();
        Map<String, String> map = new ConcurrentHashMap<>();
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                map.put(Thread.currentThread().getName(), UUID.randomUUID().toString().substring(0, 8));
                System.out.println(Thread.currentThread().getName() + "\t" + map);
            }, String.valueOf(i)).start();
        }
    }

    private static void setNoSafe() {
        //Set<String> set=new HashSet<>();
        Set<String> set = new CopyOnWriteArraySet<>();
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                set.add(UUID.randomUUID().toString().substring(0, 8));
                System.out.println(Thread.currentThread().getName() + "\t" + set);
            }, String.valueOf(i)).start();
        }
    }

    private static void listNotSafe() {
        //List<String> list=new ArrayList<>();
        List<String> list = new CopyOnWriteArrayList<>();
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0, 8));
                System.out.println(Thread.currentThread().getName() + "\t" + list);
            }, String.valueOf(i)).start();
        }
    }
}
```

