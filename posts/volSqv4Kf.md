---
title: '阻塞队列BlockingQueue'
date: 2021-09-08 15:40:12
tags: [java]
published: true
hideInList: false
feature: /post-images/volSqv4Kf.png
isTop: false
---
## 为什么要使用阻塞队列

之前，介绍了一下 ThreadPoolExecutor 的各参数的含义（[线程池ThreadPoolExecutor](https://tianxiawuhao.github.io/post/y8VkHlUlp)），其中有一个 BlockingQueue，它是一个阻塞队列。那么，小伙伴们有没有想过，为什么此处的线程池要用阻塞队列呢？

我们知道队列是先进先出的。当放入一个元素的时候，会放在队列的末尾，取出元素的时候，会从队头取。那么，当队列为空或者队列满的时候怎么办呢。

这时，阻塞队列，会自动帮我们处理这种情况。

当阻塞队列为空的时候，从队列中取元素的操作就会被阻塞。当阻塞队列满的时候，往队列中放入元素的操作就会被阻塞。

而后，一旦空队列有数据了，或者满队列有空余位置时，被阻塞的线程就会被自动唤醒。

这就是阻塞队列的好处，你不需要关心线程何时被阻塞，也不需要关心线程何时被唤醒，一切都由阻塞队列自动帮我们完成。我们只需要关注具体的业务逻辑就可以了。

而这种阻塞队列经常用在生产者消费者模式中。（可参看：[手写一个生产者消费者模式](https://tianxiawuhao.github.io/post/5QxJ3UtmK)）

## 常用的阻塞队列

那么，一般我们用到的阻塞队列有哪些呢。下面，通过idea的类图，列出来常用的阻塞队列，然后一个一个讲解（不懂怎么用的，可以参考这篇文章：[怎么用IDEA快速查看类图关系](https://blog.csdn.net/qq_26542493/article/details/104512954)）。

![](https://tinaxiawuhao.github.io/post-images/1631086948386.jpg)

阻塞队列中，所有常用的方法都在 BlockingQueue 接口中定义。如

插入元素的方法： put，offer，add。移除元素的方法： remove，poll，take。

它们有四种不同的处理方式，第一种是在失败时抛出异常，第二种是在失败时返回特殊值，第三种是一直阻塞当前线程，最后一种是在指定时间内阻塞，否则返回特殊值。（以上特殊值，是指在插入元素时，失败返回false，在取出元素时，失败返回null）

|      | 抛异常   | 特殊值   | 阻塞   | 超时               |
| ---- | -------- | -------- | ------ | ------------------ |
| 插入 | add(e)   | offer(e) | put(e) | offer(e,time,unit) |
| 移除 | remove() | poll()   | take() | poll(time,unit)    |


**1） ArrayBlockingQueue**

这是一个由数组结构组成的有界阻塞队列。首先看下它的构造方法，有三个。

![](https://tinaxiawuhao.github.io/post-images/1631086962267.jpg)

第一个可以指定队列的大小，第二个还可以指定队列是否公平，不指定的话，默认是非公平。它是使用 ReentrantLock 的公平锁和非公平锁实现的（后续讲解AQS时，会详细说明）。

简单理解就是，ReentrantLock 内部会维护一个有先后顺序的等待队列，假如有五个任务一起过来，都被阻塞了。如果是公平的，则等待队列中等待最久的任务就会先进入阻塞队列。如果是非公平的，那么这五个线程就需要抢锁，谁先抢到，谁就先进入阻塞队列。

第三个构造方法，是把一个集合的元素初始化到阻塞队列中。

另外，ArrayBlockingQueue 没有实现读写分离，也就是说，读和写是不能同时进行的。因为，它读写时用的是同一把锁，如下图所示：

![](https://tinaxiawuhao.github.io/post-images/1631086978381.jpg)

**2) LinkedBlockingQueue**

这是一个由链表结构组成的有界阻塞队列。它的构造方法有三个。

![](https://tinaxiawuhao.github.io/post-images/1631086987611.jpg)

可以看到和 ArrayBlockingQueue 的构造方法大同小异，不过是，LinkedBlockingQueue 可以不指定队列的大小，默认值是 Integer.MAX_VALUE 。

但是，最好不要这样做，建议指定一个固定大小。因为，如果生产者的速度比消费者的速度大的多的情况下，这会导致阻塞队列一直膨胀，直到系统内存被耗尽（此时，还没达到队列容量的最大值）。

此外，LinkedBlockingQueue 实现了读写分离，可以实现数据的读和写互不影响，这在高并发的场景下，对于效率的提高无疑是非常巨大的。

![](https://tinaxiawuhao.github.io/post-images/1631086997560.jpg)

**3） SynchronousQueue**

这是一个没有缓冲的无界队列。什么意思，看一下它的 size 方法：

![](https://tinaxiawuhao.github.io/post-images/1631087010961.jpg)

总是返回 0 ，因为它是一个没有容量的队列。

当执行插入元素的操作时，必须等待一个取出操作。也就是说，put元素的时候，必须等待 take 操作。

那么，有的同学就好奇了，这没有容量，还叫什么队列啊，这有什么意义呢。

我的理解是，这适用于并发任务不大，而且生产者和消费者的速度相差不多的场景下，直接把生产者和消费者对接，不用经过队列的入队出队这一系列操作。所以，效率上会高一些。

可以去查看一下 Excutors.newCachedThreadPool 方法用的就是这种队列。

这个队列有两个构造方法，用于传入是公平还是非公平，默认是非公平。

![](https://tinaxiawuhao.github.io/post-images/1631087023429.jpg)

**4）PriorityBlockingQueue**

这是一个支持优先级排序的无界队列。有四个构造方法：

![](https://tinaxiawuhao.github.io/post-images/1631087030940.jpg)

可以指定初始容量大小（注意初始容量并不代表最大容量），或者不指定，默认大小为 11。也可以传入一个比较器，把元素按一定的规则排序，不指定比较器的话，默认是自然顺序。

PriorityBlockingQueue 是基于二叉树最小堆实现的，每当取元素的时候，就会把优先级最高的元素取出来。我们测试一下：

```java
public class Person {
    private int id;
    private String name;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Person{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }

    public Person(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public Person() {
    }
}

public class QueueTest {
    public static void main(String[] args) throws InterruptedException {

        PriorityBlockingQueue<Person> priorityBlockingQueue = new PriorityBlockingQueue<>(1, new Comparator<Person>() {
            @Override
            public int compare(Person o1, Person o2) {
                return o1.getId() - o2.getId();
            }
        });

        Person p2 = new Person(7, "李四");
        Person p1 = new Person(9, "张三");
        Person p3 = new Person(6, "王五");
        Person p4 = new Person(2, "赵六");
        priorityBlockingQueue.add(p1);
        priorityBlockingQueue.add(p2);
        priorityBlockingQueue.add(p3);
        priorityBlockingQueue.add(p4);

		//由于二叉树最小堆实现，用这种方式直接打印元素，不能保证有序
        System.out.println(priorityBlockingQueue);
        System.out.println(priorityBlockingQueue.take());
        System.out.println(priorityBlockingQueue);
        System.out.println(priorityBlockingQueue.take());
        System.out.println(priorityBlockingQueue);

    }
}
```

打印结果：

```json
[Person{id=2, name='赵六'}, Person{id=6, name='王五'}, Person{id=7, name='李四'}, Person{id=9, name='张三'}]
Person{id=2, name='赵六'}
[Person{id=6, name='王五'}, Person{id=9, name='张三'}, Person{id=7, name='李四'}]
Person{id=6, name='王五'}
[Person{id=7, name='李四'}, Person{id=9, name='张三'}]
```

可以看到，第一次取出的是 id 最小值 2， 第二次取出的是 6 。

**5）DelayQueue**

这是一个带有延迟时间的无界阻塞队列。队列中的元素，只有等延时时间到了，才能取出来。此队列一般用于过期数据的删除，或任务调度。以下，模拟一下定长时间的数据删除。

首先定义数据元素，需要实现 Delayed 接口，实现 getDelay 方法用于计算剩余时间，和 CompareTo方法用于优先级排序。

```java
public class DelayData implements Delayed {

    private int id;
    private String name;
    //数据到期时间
    private long endTime;
    private TimeUnit timeUnit = TimeUnit.MILLISECONDS;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public long getEndTime() {
        return endTime;
    }

    public void setEndTime(long endTime) {
        this.endTime = endTime;
    }

    public DelayData(int id, String name, long endTime) {
        this.id = id;
        this.name = name;
        //需要把传入的时间endTime 加上当前系统时间，作为数据的到期时间
        this.endTime = endTime + System.currentTimeMillis();
    }

    public DelayData() {
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return this.endTime - System.currentTimeMillis();
    }

    @Override
    public int compareTo(Delayed o) {
        return o.getDelay(this.timeUnit) - this.getDelay(this.timeUnit) < 0 ? 1: -1;
    }

}
```

模拟三条数据，分别设置不同的过期时间：

```java
public class ProcessData {
    public static void main(String[] args) throws InterruptedException {
        DelayQueue<DelayData> delayQueue = new DelayQueue<>();

        DelayData a = new DelayData(5, "A", 5000);
        DelayData b = new DelayData(8, "B", 8000);
        DelayData c = new DelayData(2, "C", 2000);

        delayQueue.add(a);
        delayQueue.add(b);
        delayQueue.add(c);

        System.out.println("开始计时时间:" + System.currentTimeMillis());
        for (int i = 0; i < 3; i++) {
            DelayData data = delayQueue.take();
            System.out.println("id:"+data.getId()+"，数据:"+data.getName()+"被移除，当前时间:"+System.currentTimeMillis());
        }
    }
}
```

最后结果：

```json
开始计时时间:1583333583216
id:2，数据:C被移除，当前时间:1583333585216
id:5，数据:A被移除，当前时间:1583333588216
id:8，数据:B被移除，当前时间:1583333591216
```

可以看到，数据是按过期时间长短，按顺序移除的。C的时间最短 2 秒，然后过了 3 秒 A 也过期，再过 3 秒，B 过期。