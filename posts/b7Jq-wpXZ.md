---
title: '多线程基础概念四'
date: 2021-05-12 14:59:14
tags: [java]
published: true
hideInList: false
feature: /post-images/b7Jq-wpXZ.png
isTop: false
---

###  Lock 接口(Lock interface)简介

>Lock 接口比同步方法和同步块提供了更具扩展性的锁操作。他们允许更灵活的结构，可以具有完全不同的性质，并且可以支持多个相关类的条件对象。

**优势**
它的优势有：

（1）可以使锁更公平

（2）可以使线程在等待锁的时候响应中断

（3）可以让线程尝试获取锁，并在无法获取锁的时候立即返回或者等待一段时间

（4）可以在不同的范围，以不同的顺序获取和释放锁

整体上来说 Lock 是 synchronized 的扩展版，Lock 提供了无条件的、可轮询的(tryLock 方法)、定时的(tryLock 带参方法)、可中断的(lockInterruptibly)、可多条件队列的(newCondition 方法)锁操作。另外 Lock 的实现类基本都支持非公平锁(默认)和公平锁，synchronized 只支持非公平锁，当然，在大部分情况下，非公平锁是高效的选择。

#### 乐观锁和悲观锁

悲观锁：总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。再比如 Java 里面的同步原语 `synchronized` 关键字的实现也是悲观锁。

乐观锁：顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库提供的类似于 write_condition 机制，其实都是提供的乐观锁。在 Java中 `java.util.concurrent.atomic` 包下面的原子变量类就是使用了乐观锁的一种实现方式 `CAS` 实现的。

##### 乐观锁的实现方式：

1、使用版本标识来确定读到的数据与提交时的数据是否一致。提交后修改版本标识，不一致时可以采取丢弃和再次尝试的策略。

2、java 中的 Compare and Swap 即 CAS ，当多个线程尝试使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。 CAS 操作中包含三个操作数 —— 需要读写的内存位置（V），进行比较的预期原值（A）和拟写入的新值(B)。如果内存位置 V 的值与预期原值 A 相匹配，那么处理器会自动将该位置值更新为新值 B。否则处理器不做任何操作。

#####  CAS

CAS 是 compare and swap 的缩写，即我们所说的比较交换。

CAS 是一种基于锁的操作，而且是乐观锁。在 java 中锁分为乐观锁和悲观锁。悲观锁是将资源锁住，等一个之前获得锁的线程释放锁之后，下一个线程才可以访问。而乐观锁采取了一种宽泛的态度，通过某种方式不加锁来处理资源，比如通过给记录加 version 来获取数据，性能较悲观锁有很大的提高。

CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存地址里面的值和 A 的值是一样的，那么就将内存里面的值更新成 B。CAS是通过无限循环来获取数据的，若果在第一轮循环中，a 线程获取地址里面的值被b 线程修改了，那么 a 线程需要自旋，到下次循环才有可能机会执行。

`java.util.concurrent.atomic` 包下的类大多是使用 CAS 操作来实现的(`AtomicInteger`,`AtomicBoolean`,`AtomicLong`)。

###### CAS产生的问题

1. ABA 问题：

   比如说一个线程 one 从内存位置 V 中取出 A，这时候另一个线程 two 也从内存中取出 A，并且 two 进行了一些操作变成了 B，然后 two 又将 V 位置的数据变成 A，这时候线程 one 进行 CAS 操作发现内存中仍然是 A，然后 one 操作成功。尽管线程 one 的 CAS 操作成功，但可能存在潜藏的问题。从 Java1.5 开始 JDK 的 atomic包里提供了一个类` AtomicStampedReference` 来解决 ABA 问题。

2. 循环时间长开销大：

   对于资源竞争严重（线程冲突严重）的情况，CAS 自旋的概率会比较大，从而浪费更多的 CPU 资源，效率低于 synchronized。

3. 只能保证一个共享变量的原子操作：

   当对一个共享变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁。

#### 公平锁锁和非公平锁

公平锁：多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远都是队列的第一位才能得到锁。

- 优点：所有的线程都能得到资源，不会饿死在队列中。
- 缺点：吞吐量会下降很多，队列里面除了第一个线程，其他的线程都会阻塞，cpu唤醒阻塞线程的开销会很大。

非公平锁：多个线程去获取锁的时候，会直接去尝试获取，获取不到，再去进入等待队列，如果能获取到，就直接获取到锁。

- 优点：可以减少CPU唤醒线程的开销，整体的吞吐效率会高点，CPU也不必取唤醒所有线程，会减少唤起线程的数量。
- 缺点：你们可能也发现了，这样可能导致队列中间的线程一直获取不到锁或者长时间获取不到锁，导致饿死。

#### 伪共享

**一、伪共享的定义：**

> 缓存系统中是以缓存行（cache line）为单位存储的，当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

**二、CPU缓存机制**

> CPU 缓存（Cache Memory）是位于 CPU 与内存之间的临时存储器，它的容量比内存小的多但是交换速度却比内存要快得多。
> 高速缓存的出现主要是为了解决 CPU 运算速度与内存读写速度不匹配的矛盾，因为 CPU 运算速度要比内存读写速度快很多，这样会使 CPU 花费很长时间等待数据到来或把数据写入内存。
> 在缓存中的数据是内存中的一小部分，但这一小部分是短时间内 CPU 即将访问的，当 CPU 调用大量数据时，就可避开内存直接从缓存中调用，从而加快读取速度。

CPU 和主内存之间有好几层缓存，因为即使直接访问主内存也是非常慢的。如果你正在多次对一块数据做相同的运算，那么在执行运算的时候把它加载到离 CPU 很近的地方就有意义了。

按照数据读取顺序和与 CPU 结合的紧密程度，CPU 缓存可以分为一级缓存，二级缓存，部分高端 CPU 还具有三级缓存。每一级缓存中所储存的全部数据都是下一级缓存的一部分，越靠近 CPU 的缓存越快也越小。所以 L1 缓存很小但很快(译注：L1 表示一级缓存)，并且紧靠着在使用它的 CPU 内核。L2 大一些，也慢一些，并且仍然只能被一个单独的 CPU 核使用。L3 在现代多核机器中更普遍，仍然更大，更慢，并且被单个插槽上的所有 CPU 核共享。最后，你拥有一块主存，由全部插槽上的所有 CPU 核共享。拥有三级缓存的的 CPU，到三级缓存时能够达到 95% 的命中率，只有不到 5% 的数据需要从内存中查询。

多核机器的存储结构如下图所示：

![img](https://images2015.cnblogs.com/blog/897247/201608/897247-20160823201221667-1059026925.png)

当 CPU 执行运算的时候，它先去 L1 查找所需的数据，再去 L2，然后是 L3，最后如果这些缓存中都没有，所需的数据就要去主内存拿。走得越远，运算耗费的时间就越长。所以如果你在做一些很频繁的事，你要确保数据在 L1 缓存中。

Martin Thompson 给出了一些缓存未命中的消耗数据，如下所示：

![img](https://images2015.cnblogs.com/blog/897247/201608/897247-20160823201305464-895015043.png)

**三、缓存行**

缓存系统中是以缓存行（cache line）为单位存储的。缓存行通常是 64 字节（译注：本文基于 64 字节，其他长度的如 32 字节等不适本文讨论的重点），并且它有效地引用主内存中的一块地址。例如一个 的 long 类型是 8 字节，因此在一个缓存行中可以存 8 个 long 类型的变量。所以，如果你访问一个 long 数组，当数组中的一个值被加载到缓存中，它会额外加载另外 7 个，以致你能非常快地遍历这个数组。事实上，你可以非常快速的遍历在连续的内存块中分配的任意数据结构。而如果你在数据结构中的项在内存中不是彼此相邻的（如链表），你将得不到免费缓存加载所带来的优势，并且在这些数据结构中的每一个项都可能会出现缓存未命中。

如果存在这样的场景，有多个线程操作不同的成员变量，但是相同的缓存行，这个时候会发生什么？。没错，伪共享（False Sharing）问题就发生了！有张 Disruptor 项目的经典示例图，如下：

![img](https://images2015.cnblogs.com/blog/897247/201608/897247-20160823202002573-736704844.png)

 

上图中，一个运行在处理器 core1上的线程想要更新变量 X 的值，同时另外一个运行在处理器 core2 上的线程想要更新变量 Y 的值。但是，这两个频繁改动的变量都处于同一条缓存行。两个线程就会轮番发送 RFO 消息，占得此缓存行的拥有权。当 core1 取得了拥有权开始更新 X，则 core2 对应的缓存行需要设为 I 状态。当 core2 取得了拥有权开始更新 Y，则 core1 对应的缓存行需要设为 I 状态(失效态)。轮番夺取拥有权不但带来大量的 RFO 消息，而且如果某个线程需要读此行数据时，L1 和 L2 缓存上都是失效数据，只有 L3 缓存上是同步好的数据。从前一篇我们知道，读 L3 的数据非常影响性能。更坏的情况是跨槽读取，L3 都要 miss，只能从内存上加载。

表面上 X 和 Y 都是被独立线程操作的，而且两操作之间也没有任何关系。只不过它们共享了一个缓存行，但所有竞争冲突都是来源于共享。

因此，当两个以上CPU都要访问同一个缓存行大小的内存区域时，就会引起冲突，这种情况就叫“共享”。但是，这种情况里面又包含了“其实不是共享”的“伪共享”情况。比如，两个处理器各要访问一个word，这两个word却存在于同一个cache line大小的区域里，这时，从应用逻辑层面说，这两个处理器并没有共享内存，因为他们访问的是不同的内容（不同的word）。但是因为cache line的存在和限制，这两个CPU要访问这两个不同的word时，却一定要访问同一个cache line块，产生了事实上的“共享”。显然，由于cache line大小限制带来的这种“伪共享”是我们不想要的，会浪费系统资源。

**四、如何避免伪共享**？ 

1）让不同线程操作的对象处于不同的缓存行。

可以进行缓存行填充（Padding） 。例如，如果一条缓存行有 64 字节，而 Java 程序的对象头固定占 8 字节(32位系统)或 12 字节( 64 位系统默认开启压缩, 不开压缩为 16 字节)，所以我们只需要填 6 个无用的长整型补上6*8=48字节，让不同的 VolatileLong 对象处于不同的缓存行，就避免了伪共享( 64 位系统超过缓存行的 64 字节也无所谓，只要保证不同线程不操作同一缓存行就可以)。

2）使用编译指示，强制使每一个变量对齐。

强制使对象按照缓存行的边界对齐。例如可以使数据按64位对齐，那么一个缓存行只有一个可操作对象，这样发生伪共享之后，也只是对应缓存行的数据变化，并不影响其他的对象。

#### 无锁队列

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReferenceArray;
 
 
/**
 * 用数组实现无锁有界队列
 */
 
public class LockFreeQueue {
 
    private AtomicReferenceArray atomicReferenceArray;
    //代表为空，没有元素
    private static final  Integer EMPTY = null;
    //头指针,尾指针
    AtomicInteger head,tail;
 
 
    public LockFreeQueue(int size){
        atomicReferenceArray = new AtomicReferenceArray(new Integer[size + 1]);
        head = new AtomicInteger(0);
        tail = new AtomicInteger(0);
    }
 
    /**
     * 入队
     * @param element
     * @return
     */
    public boolean add(Integer element){
        int index = (tail.get() + 1) % atomicReferenceArray.length();
        if( index == head.get() % atomicReferenceArray.length()){
            System.out.println("当前队列已满,"+ element+"无法入队!");
            return false;
        }
        while(!atomicReferenceArray.compareAndSet(index,EMPTY,element)){
            return add(element);
        }
        tail.incrementAndGet(); //移动尾指针
        System.out.println("入队成功!" + element);
        return true;
    }
 
    /**
     * 出队
     * @return
     */
    public Integer poll(){
        if(head.get() == tail.get()){
            System.out.println("当前队列为空");
            return null;
        }
        int index = (head.get() + 1) % atomicReferenceArray.length();
        Integer ele = (Integer) atomicReferenceArray.get(index);
        if(ele == null){ //有可能其它线程也在出队
            return poll();
        }
        while(!atomicReferenceArray.compareAndSet(index,ele,EMPTY)){
            return poll();
        }
        head.incrementAndGet();
        System.out.println("出队成功!" + ele);
        return ele;
    }
 
    public void print(){
       StringBuffer buffer = new StringBuffer("[");
       for(int i = 0; i < atomicReferenceArray.length() ; i++){
           if(i == head.get() || atomicReferenceArray.get(i) == null){
               continue;
           }
           buffer.append(atomicReferenceArray.get(i) + ",");
       }
       buffer.deleteCharAt(buffer.length() - 1);
       buffer.append("]");
       System.out.println("队列内容:"    +buffer.toString());
 
    }
 
}
```

#### 死锁

当线程 A 持有独占锁a，并尝试去获取独占锁 b 的同时，线程 B 持有独占锁 b，并尝试获取独占锁 a 的情况下，就会发生 AB 两个线程由于互相持有对方需要的锁，而发生的阻塞现象，我们称为死锁。

##### 1. 产生死锁的条件和防止死锁

**互斥 请求与保持 不可剥夺 循环等待**

产生死锁的必要条件：

1. 互斥条件：所谓互斥就是进程在某一时间内独占资源。

2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。

3. 不可剥夺条件：进程已获得资源，在末使用完之前，不能强行剥夺。

4. 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之 一不满足，就不会发生死锁。

理解了死锁的原因，尤其是产生死锁的四个必要条件，就可以最大可能地避免、预防和 解除死锁。

防止死锁可以采用以下的方法：

- 尽量使用 `tryLock(long timeout, TimeUnit unit)`的方法`(ReentrantLock、ReentrantReadWriteLock)`，设置超时时间，超时可以退出防止死锁。
- 尽量使用 `Java. util. concurrent` 并发类代替自己手写锁。
- 尽量降低锁的使用粒度，尽量不要几个功能用同一把锁。
- 尽量减少同步的代码块。

```java
import lombok.SneakyThrows;
import java.util.concurrent.TimeUnit;

class HoldLockThread implements Runnable{
    private String lockOne;
    private String lockTwo;

    public HoldLockThread(String lockOne, String lockTwo) {
        this.lockOne = lockOne;
        this.lockTwo = lockTwo;
    }

    @Override
    @SneakyThrows
    public void run(){
        synchronized (lockOne){
            System.out.println(Thread.currentThread().getName()+"自己获取"+lockOne+"尝试获取"+lockTwo);
            TimeUnit.SECONDS.sleep(2L);
            synchronized (lockTwo){
                System.out.println(Thread.currentThread().getName()+"自己获取"+lockTwo+"尝试获取"+lockOne);
            }
        }
    }
}
public class DeadLockDemo {
    public static void main(String[] args) {
        String lockA="lockA";
        String lockB="lockB";

        new Thread(new HoldLockThread(lockA,lockB),"ThreadAAA").start();
        new Thread(new HoldLockThread(lockB,lockA),"ThreadBBB").start();
    }
}
```

##### 2. 死锁与活锁与饥饿的区别

死锁：是指两个或两个以上的进程（或线程）在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。

活锁：任务或者执行者没有被阻塞，由于某些条件没有满足，导致一直重复尝试，失败，尝试，失败。

活锁和死锁的区别在于，处于活锁的实体是在不断的改变状态，这就是所谓的“活”， 而处于死锁的实体表现为等待；活锁有可能自行解开，死锁则不能。

饥饿：一个或者多个线程因为种种原因无法获得所需要的资源，导致一直无法执行的状态。

Java 中导致饥饿的原因：

1、高优先级线程吞噬所有的低优先级线程的 CPU 时间。

2、线程被永久堵塞在一个等待进入同步块的状态，因为其他线程总是能在它之前持续地对该同步块进行访问。

3、线程在等待一个本身也处于永久等待完成的对象(比如调用这个对象的 wait 方法)，因为其他线程总是被持续地获得唤醒。


### ReentrantLock(重入锁)

`ReentrantLock`重入锁，是实现Lock接口的一个类，也是在实际编程中使用频率很高的一个锁，支持重入性，表示能够对共享资源能够重复加锁，即当前线程获取该锁再次获取不会被阻塞。

在java关键字synchronized隐式支持重入性，synchronized通过获取自增，释放自减的方式实现重入。与此同时，ReentrantLock还支持公平锁和非公平锁两种方式。那么，要想完完全全的弄懂ReentrantLock的话，主要也就是ReentrantLock同步语义的学习：

1. 重入性的实现原理；
2. 公平锁和非公平锁

#### 1. 重入性的实现原理

要想支持重入性，就要解决两个问题：

**1. 在线程获取锁的时候，如果已经获取锁的线程是当前线程的话则直接再次获取成功；**

**2. 由于锁会被获取n次，那么只有锁在被释放同样的n次之后，该锁才算是完全释放成功**。

ReentrantLock支持两种锁：**公平锁**和**非公平锁**。**何谓公平性，是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求上的绝对时间顺序，满足FIFO**。

#### 2. ReentrantLock使用

```java
class Bank{
      /**
       * volatile实现
       */
      private  int count=0;
      /**
       * 使用可重入锁
       */
      private Lock lock=new ReentrantLock();

      public void getCount(){
            System.out.println("账户余额为："+count);
      }
      /**
       * 同步方法实现存钱
       * @param money
       */
      public void save(int money){
            lock.lock();
            try {
                  count+=money;
                  System.out.println(System.currentTimeMillis()+"存进："+money);
            } catch (Exception e) {
                  // TODO Auto-generated catch block
                  e.printStackTrace();
            }finally {
                  lock.unlock();//释放锁
            }
      }
      /**
       * 同步代码块实现取钱
       * @param money
       */
      public  void remove(int money){
            if (count-money<0) {
                  System.err.println("余额不足。");
                  return;
            }
                lock.lock();
                  try {
                        count-=money;
                        System.err.println(System.currentTimeMillis()+"取出："+money);
                  } catch (Exception e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                  }finally {
                        lock.unlock();
                  }

      }
}
```



#### 读写锁ReentrantReadWriteLock

首先明确一下，不是说 ReentrantLock 不好，只是 ReentrantLock 某些时候有局限。如果使用 ReentrantLock，可能本身是为了防止线程 A 在写数据、线程 B 在读数据造成的数据不一致，但这样，如果线程 C 在读数据、线程 D 也在读数据，读数据是不会改变数据的，没有必要加锁，但是还是加锁了，降低了程序的性能。因为这个，才诞生了读写锁 ReadWriteLock。

ReadWriteLock 是一个读写锁接口，读写锁是用来提升并发程序性能的锁分离技术，ReentrantReadWriteLock 是 ReadWriteLock 接口的一个具体实现，实现了读写的分离，读锁是共享的，写锁是独占的，读和读之间不会互斥，读和写、写和读、写和写之间才会互斥，提升了读写的性能。

而读写锁有以下三个重要的特性：

（1）公平选择性：支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平。

（2）重进入：读锁和写锁都支持线程重进入。



### 并发容器

#### 1. ConcurrentHashMap

`ConcurrentHashMap`是Java中的一个**线程安全且高效的HashMap实现**。平时涉及高并发如果要用map结构，那第一时间想到的就是它。相对于hashmap来说，`ConcurrentHashMap`就是线程安全的map，其中利用了锁分段的思想提高了并发度。

那么它到底是如何实现线程安全的？

JDK 1.6版本关键要素：

- segment继承了`ReentrantLock`充当锁的角色，为每一个segment提供了线程安全的保障；
- segment维护了哈希散列表的若干个桶，每个桶由`HashEntry`构成的链表。

JDK1.8后，`ConcurrentHashMap`抛弃了原有的**Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性**。

##### 1.1 ConcurrentHashMap 的并发度

`ConcurrentHashMap` 把实际 map 划分成若干部分来实现它的可扩展性和线程安全。这种划分是使用并发度获得的，它是 `ConcurrentHashMap` 类构造函数的一个可选参数，默认值为 16，这样在多线程情况下就能避免争用。

在 JDK8 后，它摒弃了 Segment（锁段）的概念，而是启用了一种全新的方式实现,利用 CAS 算法。同时加入了更多的辅助变量来提高并发度，

##### 1.2 并发容器的实现

何为同步容器：可以简单地理解为通过 synchronized 来实现同步的容器，如果有多个线程调用同步容器的方法，它们将会串行执行。比如` Vector`，`Hashtable`，以及 `Collections.synchronizedSet`，`synchronizedList` 等方法返回的容器。可以通过查看 `Vector`，`Hashtable `等这些同步容器的实现代码，可以看到这些容器实现线程安全的方式就是将它们的状态封装起来，并在需要同步的方法上加上关键字 synchronized。

并发容器使用了与同步容器完全不同的加锁策略来提供更高的并发性和伸缩性，例如在 `ConcurrentHashMap` 中采用了一种粒度更细的加锁机制，可以称为分段锁，在这种锁机制下，允许任意数量的读线程并发地访问 map，并且执行读操作的线程和写操作的线程也可以并发的访问 map，同时允许一定数量的写操作线程并发地修改 map，所以它可以在并发环境下实现更高的吞吐量。

##### 1.3 同步集合与并发集合区别

同步集合与并发集合都为多线程和并发提供了合适的线程安全的集合，不过并发集合的可扩展性更高。在 Java1.5 之前程序员们只有同步集合来用且在多线程并发的时候会导致争用，阻碍了系统的扩展性。Java5 介绍了并发集合像`ConcurrentHashMap`，不仅提供线程安全还用锁分离和内部分区等现代技术提高了可扩展性。

##### 1.4 SynchronizedMap 和 ConcurrentHashMap 比较

`SynchronizedMap` 一次锁住整张表来保证线程安全，所以每次只能有一个线程来访为 map。

`ConcurrentHashMap` 使用分段锁来保证在多线程下的性能。

`ConcurrentHashMap` 中则是一次锁住一个桶。`ConcurrentHashMap` 默认将hash 表分为 16 个桶，诸如 get，put，remove 等常用操作只锁当前需要用到的桶。

这样，原来只能一个线程进入，现在却能同时有 16 个写线程执行，并发性能的提升是显而易见的。

另外 `ConcurrentHashMap` 使用了一种不同的迭代方式。在这种迭代方式中，当iterator 被创建后集合再发生改变就不再是抛出`ConcurrentModificationException`，取而代之的是在改变时 new 新的数据从而不影响原有的数据，iterator 完成后再将头指针替换为新的数据 ，这样 iterator线程可以使用原来老的数据，而写线程也可以并发的完成改变。

#### 2. CopyOnWriteArrayList

`CopyOnWriteArrayList` 是一个并发容器。有很多人称它是线程安全的，我认为这句话不严谨，缺少一个前提条件，那就是非复合场景下操作它是线程安全的。

`CopyOnWriteArrayList`(免锁容器)的好处之一是当多个迭代器同时遍历和修改这个列表时，不会抛出` ConcurrentModificationException`。在`CopyOnWriteArrayList` 中，写入将导致创建整个底层数组的副本，而源数组将保留在原地，使得复制的数组在被修改时，读取操作可以安全地执行。

`CopyOnWriteArrayList` 的使用场景

通过源码分析，我们看出它的优缺点比较明显，所以使用场景也就比较明显。就是合适读多写少的场景。

`CopyOnWriteArrayList` 的缺点

1. 由于写操作的时候，需要拷贝数组，会消耗内存，如果原数组的内容比较多的情况下，可能导致 young gc 或者 full gc。
2. 不能用于实时读的场景，像拷贝数组、新增元素都需要时间，所以调用一个 set 操作后，读取到数据可能还是旧的，虽然`CopyOnWriteArrayList` 能做到最终一致性,但是还是没法满足实时性要求。
3. 由于实际使用中可能没法保证 `CopyOnWriteArrayList` 到底要放置多少数据，万一数据稍微有点多，每次 add/set 都要重新复制数组，这个代价实在太高昂了。在高性能的互联网应用中，这种操作分分钟引起故障。

`CopyOnWriteArrayList` 的设计思想

1. 读写分离，读和写分开
2. 最终一致性
3. 使用另外开辟空间的思路，来解决并发冲突

#### 3. ThreadLocal 

ThreadLocal 是一个本地线程副本变量工具类，在每个线程中都创建了一个 ThreadLocalMap 对象，简单说 ThreadLocal 就是一种以空间换时间的做法，每个线程可以访问自己内部 ThreadLocalMap 对象内的 value。通过这种方式，避免资源在多线程间共享。

原理：线程局部变量是局限于线程内部的变量，属于线程自身所有，不在多个线程间共享。Java提供ThreadLocal类来支持线程局部变量，是一种实现线程安全的方式。但是在管理环境下（如 web 服务器）使用线程局部变量的时候要特别小心，在这种情况下，工作线程的生命周期比任何应用变量的生命周期都要长。任何线程局部变量一旦在工作完成后没有释放，Java 应用就存在内存泄露的风险。

经典的使用场景是为每个线程分配一个 JDBC 连接 Connection。这样就可以保证每个线程的都在各自的 Connection 上进行数据库的操作，不会出现 A 线程关了 B线程正在使用的 Connection； 还有 Session 管理 等问题。

ThreadLocal 使用例子：

```java
public class TestThreadLocal {
    
    //线程本地存储变量
    private static final ThreadLocal<Integer> THREAD_LOCAL_NUM 
        = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };
 
    public static void main(String[] args) {
        for (int i = 0; i <3; i++) {//启动三个线程
            Thread t = new Thread() {
                @Override
                public void run() {
                    add10ByThreadLocal();
                }
            };
            t.start();
        }
    }
    
    /**
     * 线程本地存储变量加 5
     */
    private static void add10ByThreadLocal() {
        for (int i = 0; i <5; i++) {
            Integer n = THREAD_LOCAL_NUM.get();
            n += 1;
            THREAD_LOCAL_NUM.set(n);
            System.out.println(Thread.currentThread().getName() + " : ThreadLocal num=" + n);
        }
    }
    
}
```

打印结果：启动了 3 个线程，每个线程最后都打印到 “ThreadLocal num=5”，而不是 num 一直在累加直到值等于 15

![](https://tianxiawuhao.github.io/post-images/1620459942353.png)

##### 3.1什么是线程局部变量？

线程局部变量是局限于线程内部的变量，属于线程自身所有，不在多个线程间共享。Java 提供 ThreadLocal 类来支持线程局部变量，是一种实现线程安全的方式。但是在管理环境下（如 web 服务器）使用线程局部变量的时候要特别小心，在这种情况下，工作线程的生命周期比任何应用变量的生命周期都要长。任何线程局部变量一旦在工作完成后没有释放，Java 应用就存在内存泄露的风险。

##### 3.2 ThreadLocal内存泄漏分析与解决方案

**ThreadLocal造成内存泄漏的原因？**

ThreadLocalMap 中使用的 key 为 ThreadLocal 的弱引用,而 value 是强引用。所以，如果 ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。这样一来，ThreadLocalMap 中就会出现key为null的Entry。假如我们不做任何措施的话，value 永远无法被GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap实现中已经考虑了这种情况，在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录。使用完 ThreadLocal方法后 最好手动调用remove()方法

**ThreadLocal内存泄漏解决方案？**

- 每次使用完ThreadLocal，都调用它的remove()方法，清除数据。
- 在使用线程池的情况下，没有及时清理ThreadLocal，不仅是内存泄漏的问题，更严重的是可能导致业务逻辑出现问题。所以，使用ThreadLocal就跟加锁完要解锁一样，用完就清理。

#### 4. BlockingQueue

`阻塞队列（BlockingQueue）`是一个支持两个附加操作的队列。

这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。

阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

JDK7 提供了 7 个阻塞队列。分别是：

`ArrayBlockingQueue` ：一个由数组结构组成的有界阻塞队列。

`LinkedBlockingQueue` ：一个由链表结构组成的有界阻塞队列。

`PriorityBlockingQueue` ：一个支持优先级排序的无界阻塞队列。

`DelayQueue`：一个使用优先级队列实现的无界阻塞队列。

`SynchronousQueue`：一个不存储元素的阻塞队列。

`LinkedTransferQueue`：一个由链表结构组成的无界阻塞队列。

`LinkedBlockingDeque`：一个由链表结构组成的双向阻塞队列。

Java 5 之前实现同步存取时，可以使用普通的一个集合，然后在使用线程的协作和线程同步可以实现生产者，消费者模式，主要的技术就是用好，`wait`,`notify`,`notifyAll`,`sychronized` 这些关键字。而在 java 5 之后，可以使用阻塞队列来实现，此方式大大简少了代码量，使得多线程编程更加容易，安全方面也有保障。

`BlockingQueue` 接口是 Queue 的子接口，它的主要用途并不是作为容器，而是作为线程同步的的工具，因此他具有一个很明显的特性，当生产者线程试图向 `BlockingQueue` 放入元素时，如果队列已满，则线程被阻塞，当消费者线程试图从中取出一个元素时，如果队列为空，则该线程会被阻塞，正是因为它所具有这个特性，所以在程序中多个线程交替向 BlockingQueue 中放入元素，取出元素，它可以很好的控制线程之间的通信。

阻塞队列使用最经典的场景就是 socket 客户端数据的读取和解析，读取数据的线程不断将数据放入队列，然后解析线程不断从队列取数据解析。