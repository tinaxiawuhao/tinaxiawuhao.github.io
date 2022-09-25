---
title: 'CAS/ABA/AtomicReference'
date: 2021-09-13 11:27:29
tags: [java]
published: true
hideInList: false
feature: /post-images/2nzKhQbUe.png
isTop: false
---
锁是用来做并发最简单的方式，当然代价也是最高的。

独占锁是一种悲观锁，synchronized就是一种独占锁；它假设最坏的情况，并且只有在确保其它线程不会造成干扰的情况下执行，会导致其它所有需要锁的线程挂起直到持有锁的线程释放锁。

所谓乐观锁就是每次不加锁,假设没有冲突而去完成某项操作;如果发生冲突了那就去重试，直到成功为止。

CAS(Compare And Swap)是一种有名的无锁算法。CAS算法是乐观锁的一种实现。CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B并返回true，否则返回false。

> 注:synchronized和ReentrantLock都是悲观锁。

> 注:什么时候使用悲观锁效率更高、什么使用使用乐观锁效率更高，要根据实际情况来判断选择。

 

提示:atomic中包下的类，采用的即为CAS乐观算法。

以AtomicInteger的public final int getAndSet(int newValue)方法，进行简单说明，
该方法是这样的:

![](https://tinaxiawuhao.github.io/post-images/1631512353549.png)

其调用了Unsafe类的public final int getAndSetInt(Object var1, long var2, int var4)方法:

![](https://tinaxiawuhao.github.io/post-images/1631512364454.png)

而该方法又do{…}while(…)循环调用了本地方法public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

![](https://tinaxiawuhao.github.io/post-images/1631512374405.png)

注:至于Windows/Linux下public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5)本地方法是如何实现的，推荐阅读https://blog.csdn.net/v123411739/article/details/79561458。

 

**CAS(Compare And Swap)原理简述：**


       某一线程执行一个CAS逻辑(如上图线程A),如果中途有其他线程修改了共享变量的值(如:上图中线程A执行到笑脸那一刻时),导致这个线程的CAS逻辑运算后得到的值与期望结果不一致，那么这个线程会再次执行CAS逻辑(这里是一个do while循环),直到成功为止。

 

**ABA问题：**


      就是说一个线程把数据A变为了B，然后又重新变成了A。此时另外一个线程读取的时候，发现A没有变化，就误以为是原来的那个A。这就是有名的ABA问题。

> 注:根据实际情况，判断是否处理ABA问题。如果ABA问题并不会影响我们的业务结果，可以选择性处理或不处理;如果
>      ABA会影响我们的业务结果的，这时就必须处理ABA问题了。
>      追注:对于AtomicInteger等,没有什么可修改的属性;且我们只在意其结果值，所以对于这些类来说，本身就算发生了ABA现象，也不会对原线程的结果造成什么影响。

AtomicReference原子应用类，可以保证你在修改对象引用时的线程安全性，比较时可以按照偏移量进行
怎样使用AtomicReference：

```java
AtomicReference<String> ar = new AtomicReference<String>();
ar.set("hello");
//CAS操作更新
ar.compareAndSet("hello", "hello1");
```


AtomicReference的成员变量：

```java
private static final long serialVersionUID = -1848883965231344442L;
//unsafe类,提供cas操作的功能
private static final Unsafe unsafe = Unsafe.getUnsafe();
//value变量的偏移地址,说的就是下面那个value,这个偏移地址在static块里初始化
private static final long valueOffset;
//实际传入需要原子操作的那个类实例
private volatile V value;
```
类装载的时候初始化偏移地址：

```java
static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicReference.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
```


compareAndSet方法：

```java
//就是调用Unsafe的cas操作,传入对象,expect值,偏移地址,需要更新的值,即可,如果更新成功,返回true,如果失败,返回false
public final boolean compareAndSet(V expect, V update) {
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }
```



对于String变量来说,必须是对象相同才视为相同,而不是字符串的内容相同就可以相同:

```java
AtomicReference<String> ar = new AtomicReference<String>();
ar.set("hello");
System.out.println(ar.compareAndSet(new String("hello"), "hello1"));//false
```
AtomicReference可解决volatile不具有原子性i++操作
AtomicReference可以保证结果是正确的.

```java
private static volatile Integer num1 = 0;
private static AtomicReference<Integer> ar=new AtomicReference<Integer>(num1);

public void dfasd111() throws InterruptedException{
    for (int i = 0; i < 1000; i++) {
        new Thread(new Runnable(){
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++)
                    while(true){
                        Integer temp=ar.get();
                        if(ar.compareAndSet(temp, temp+1))break;
                    }
            }       
        }).start();
    }
    Thread.sleep(10000);
    System.out.println(ar.get()); //10000000
}
```

类似i++这样的"读-改-写"复合操作(在一个操作序列中, 后一个操作依赖前一次操作的结果), 在多线程并发处理的时候会出现问题, 因为可能一个线程修改了变量, 而另一个线程没有察觉到这样变化, 当使用原子变量之后, 则将一系列的复合操作合并为一个原子操作,从而避免这种问题, i++=>i.incrementAndGet() 

这里的compareAndSet方法即cas操作本身是原子的，但是在某些场景下会出现异常场景

此处说一下ABA问题：

比如，我们只是简单得要做一个数值加法，即使在我取得期望值后，这个数字被不断的修改，只要它最终改回了我的期望值，我的加法计算就不会出错。也就是说，当你修改的对象没有过程的状态信息，所有的信息都只保存于对象的数值本身。

但是，在现实中，还可能存在另外一种场景。就是我们是否能修改对象的值，不仅取决于当前值，还和对象的过程变化有关，这时，AtomicReference就无能为力了。

**AtomicStampedReference与AtomicReference的区别：**
AtomicStampedReference它内部不仅维护了对象值，还维护了一个时间戳（我这里把它称为时间戳，实际上它可以使任何一个整数，它使用整数来表示状态值）。当AtomicStampedReference对应的数值被修改时，除了更新数据本身外，还必须要更新时间戳。当AtomicStampedReference设置对象值时，对象值以及时间戳都必须满足期望值，写入才会成功。因此，即使对象值被反复读写，写回原值，只要时间戳发生变化，就能防止不恰当的写入。

**解决ABA问题**
```java

public static void main(String[] args) {
 
        String str1 = "aaa";
        String str2 = "bbb";
        AtomicStampedReference<String> reference = new AtomicStampedReference<String>(str1,1);
        reference.compareAndSet(str1,str2,reference.getStamp(),reference.getStamp()+1);
        System.out.println("reference.getReference() = " + reference.getReference());
 
        boolean b = reference.attemptStamp(str2, reference.getStamp() + 1);
        System.out.println("b: "+b);
        System.out.println("reference.getStamp() = "+reference.getStamp());
 
        boolean c = reference.weakCompareAndSet(str2,"ccc",4, reference.getStamp()+1);
        System.out.println("reference.getReference() = "+reference.getReference());
        System.out.println("c = " + c);
    }
 
输出:
reference.getReference() = bbb
b: true
reference.getStamp() = 3
reference.getReference() = bbb
c = false
```