---
title: ' 多线程基础概念'
date: 2021-05-09 13:18:21
tags: [java]
published: true
hideInList: false
feature: /post-images/ddmZREDcW.png
isTop: false
---
## 什么是线程

> 是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。

### 线程安全的问题

**并发编程三要素是什么？在 Java 程序中怎么保证多线程的运行安全？**

并发编程三要素（线程的安全性问题体现在）：

原子性：原子，即一个不可再被分割的颗粒。原子性指的是一个或多个操作要么全部执行成功要么全部执行失败。(`synchronized`,`lock`,`unlock`)

可见性：一个线程对共享变量的修改,另一个线程能够立刻看到。（`synchronized`,`volatile`,`final`）

有序性：程序执行的顺序按照代码的先后顺序执行。（处理器可能会对指令进行重排序）（`synchronized`,`volatile`）

出现线程安全问题的原因：

- 线程切换带来的原子性问题
- 缓存导致的可见性问题
- 编译优化带来的有序性问题

解决办法：

- JDK Atomic开头的原子类、synchronized、LOCK，可以解决原子性问题
- synchronized、volatile、LOCK，可以解决可见性问题
- Happens-Before 规则可以解决有序性问题

### 并发与并行

- 并发：多个任务在同一个 CPU 核上，按细分的时间片轮流(交替)执行，从逻辑上来看那些任务是同时执行。
- 并行：单位时间内，多个处理器或多核处理器同时处理多个任务，是真正意义上的“同时进行”。
- 串行：有n个任务，由一个线程按顺序执行。由于任务、方法都在一个线程执行所以不存在线程不安全情况，也就不存在临界区的问题。

### 并发编程（多线程）的优点

1. 早期的CPU是单核的，为了提升计算能力，将多个计算单元整合到一起。形成了多核CPU。**多线程就是为了将多核CPU发挥到极致，一边提高性能**。

2. 方便进行业务拆分，提升系统并发能力和性能：在特殊的业务场景下，先天的就适合于并发编程。现在的系统动不动就要求百万级甚至千万级的并发量，而多线程并发编程正是开发高并发系统的基础，利用好多线程机制可以大大提高系统整体的并发能力以及性能。面对复杂业务模型，并行程序会比串行程序更适应业务需求，而并发编程更能吻合这种业务拆分 

### 并发编程（多线程）的缺点

上面说了多线程的优点是：为了提高计算性能。那么一定会提高？答案是不一定的。有时候多线程不一定比单线程计算快

多线程会带来额外的开销和复杂的问题

- 上下文切换
- 死锁
- 内存泄漏

#### 上下文切换

时间片是CPU分配给各个线程的时间，因为时间非常短，所以CPU不断通过切换线程，让我们觉得多个线程是同时执行的，时间片一般是几十毫秒。而每次切换时，需要保存当前的状态起来，以便能够进行恢复先前状态，而这个切换时非常损耗性能，过于频繁反而无法发挥出多线程编程的优势。
 减少上下文切换可以采用无锁并发编程，CAS算法，使用最少的线程和使用协程。

- 无锁并发编程：可以参照concurrentHashMap锁分段的思想，不同的线程处理不同段的数据，这样在多线程竞争的条件下，可以减少上下文切换的时间。

- CAS算法，利用Atomic下使用CAS算法来更新数据，使用了乐观锁，可以有效的减少一部分不必要的锁竞争带来的上下文切换

- 使用最少线程：避免创建不需要的线程，比如任务很少，但是创建了很多的线程，这样会造成大量的线程都处于等待状态

- 协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换

#### 线程死锁

> 死锁是指两个或两个以上的进程（线程）在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程（线程）称为死锁进程（线程）。

多个线程同时被阻塞，它们中的一个或者全部都在等待某个资源被释放。由于线程被无限期地阻塞，因此程序不可能正常终止。

如下图所示，线程 A 持有资源 2，线程 B 持有资源 1，他们同时都想申请对方的资源，所以这两个线程就会互相等待而进入死锁状态。

![](https://tinaxiawuhao.github.io/post-images/1620451228802.png)

下面通过一个例子来说明线程死锁：

```java
public class DeadLockDemo {
    private static Object resource1 = new Object();//资源 1
    private static Object resource2 = new Object();//资源 2

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (resource1) {
                System.out.println(Thread.currentThread() + "get resource1");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resource2");
                synchronized (resource2) {
                    System.out.println(Thread.currentThread() + "get resource2");
                }
            }
        }, "线程 1").start();

        new Thread(() -> {
            synchronized (resource2) {
                System.out.println(Thread.currentThread() + "get resource2");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resource1");
                synchronized (resource1) {
                    System.out.println(Thread.currentThread() + "get resource1");
                }
            }
        }, "线程 2").start();
    }
}
```



输出结果

```java
Thread[线程 1,5,main]get resource1
Thread[线程 2,5,main]get resource2
Thread[线程 1,5,main]waiting get resource2
Thread[线程 2,5,main]waiting get resource1
```

线程 A 通过 synchronized (resource1) 获得 resource1 的监视器锁，然后通过Thread.sleep(1000)；让线程 A 休眠 1s 为的是让线程 B 得到CPU执行权，然后获取到 resource2 的监视器锁。线程 A 和线程 B 休眠结束了都开始企图请求获取对方的资源，然后这两个线程就会陷入互相等待的状态，这也就产生了死锁。上面的例子符合产生死锁的四个必要条件。

**形成死锁的四个必要条件是什么**

1. 互斥条件：线程(进程)对于所分配到的资源具有排它性，即一个资源只能被一个线程(进程)占用，直到被该线程(进程)释放
2. 请求与保持条件：一个线程(进程)因请求被占用资源而发生阻塞时，对已获得的资源保持不放。
3. 不剥夺条件：线程(进程)已获得的资源在末使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。
4. 循环等待条件：当发生死锁时，所等待的线程(进程)必定会形成一个环路（类似于死循环），造成永久阻塞

**如何避免线程死锁**

我们只要破坏产生死锁的四个条件中的其中一个就可以了。

1. **破坏互斥条件**

   这个条件我们没有办法破坏，因为我们用锁本来就是想让他们互斥的（临界资源需要互斥访问）。

2. **破坏请求与保持条件**

   一次性申请所有的资源。

3. **破坏不剥夺条件**

   占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

4. **破坏循环等待条件**

   靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。

### 创建线程的四种方式

创建线程有四种方式：

- 继承 Thread 类；
- 实现 Runnable 接口；
- 实现 Callable 接口；
- 使用 Executors 工具类创建线程池

#### 继承 Thread 类

步骤

1. 定义Thread 类的子类，并重写该类的run() 方法，该run() 方法的方法体就代表类线程需要完成的任务。因此把run() 方法称为线程执行体。
2. 创建 Thread 子类的实例，即创建线程对象
3. 调用子类实例的star()方法来启动线程

```java
/**
   * 继承 thread 的内部类，以买票例子
   */
public class FirstThread extends Thread{
    private int i;
    private int ticket = 10;

    @Override
    public void run() {
        for (;i<20;i++) {
			//当继承thread 时，直接使用this 可以获取当前的线程,getName()获取当前线程的名字
            if(this.ticket>0){
               	Log.e(TAG, getName() + ", 卖票：ticket=" + ticket--);
            }
        }

    }
}

private void starTicketThread(){
    Log.d(TAG,"starTicketThread, "+Thread.currentThread().getName());

    FirstThread thread1 = new FirstThread();
    FirstThread thread2 = new FirstThread();
    FirstThread thread3 = new FirstThread();

    thread1.start();
    thread2.start();
    thread3.start();

    //开启3个线程进行买票，每个线程都卖了10张，总共就30张票

}
```

运行结果

![](https://tinaxiawuhao.github.io/post-images/1620451264035.png)

**注意** ：可以看到 3 个线程输入的 票数变量不连续，注意：ticket 是 FirstThread 的实例属性，而不是局部变量，但是因为程序每次创建线程对象都需要创建一个FirstThread 的对象，所有多个线程不共享该实例的属性。

```java
//使用Lambda表达式，实现多线程
  new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + "新线程创建了！");
  }
  ).start();
```

#### 实现 Runnable 接口

步骤

1. 定义Runnable接口实现类SecondThread，并重写run()方法
2. 创建SecondThread实例secondThread，以secondThread作为target创建Thead对象，**该Thread对象才是真正的线程对象**
3. 调用线程对象的start()方法

```java
/**
   * 实现 runnable 接口，创建线程类
   */
public class SecondThread implements Runnable{

    private int i;
    private int ticket = 100;

    @Override
    public void run() {
        for (;i<20;i++) {
            //如果线程类实现 runnable 接口
            //获取当前的线程，只能用 Thread.currentThread() 获取当前的线程名
            Log.d(TAG,Thread.currentThread().getName()+" "+i);

            if(this.ticket>0){
                Log.e(TAG, Thread.currentThread().getName() + ", 卖票：ticket=" + ticket--);
            }
        }
    }
}


private void starTicketThread2(){
    Log.d(TAG,"starTicketThread2, "+Thread.currentThread().getName());

    SecondThread secondThread = new SecondThread();

    //通过new Thread(target,name)创建新的线程
    new Thread(secondThread,"买票人1").start();
    new Thread(secondThread,"买票人2").start();
    new Thread(secondThread,"买票人3").start();

    //虽然是开启了3个线程，但是一共只买了100张票
}
```

执行结果

![](https://tinaxiawuhao.github.io/post-images/1620451273746.png)

**注意**：可以看到 3 个线程输入的 票数变量是连续的，采用 Runnable 接口的方式创建多个线程可以共享线程类的实例的属性。这是因为在这种方式下，程序所创建的Runnable 对象只是线程的 target ,而多个线程可以共享同一个 target,所以多个线程可以共享同一个线程类（实际上应该是该线程的target 类）的实例属性。

```java
//使用匿名内部类的方式，实现多线程
  new Thread(new Runnable() {
    @Override
    public void run() {
      System.out.println(Thread.currentThread().getName() + "新线程创建了！");
    }
  }).start();
```

#### 实现 Callable 接口

步骤

1. 创建callable接口的实现类，并实现call() 方法，该call() 方法将作为线程的执行体，且该call() 方法是有返回值的。
2. 创建 callable实现类的实例，使用 FutureTask 类来包装Callable对象，该FutureTask 对象封装 call() 方法的返回值。
3. 使用FutureTask 对象作为Thread对象的target创建并启动新线程。
4. 调用FutureTask对象的get()方法来获取子线程执行结束后的返回值。

```java
/**
       * 使用callable 来实现线程类
       */
public class ThirdThread implements Callable<Integer>{

    private int ticket = 20;

    @Override
    public Integer call(){

        for ( int i = 0;i<10;i++) {
            //获取当前的线程，只能用 Thread.currentThread() 获取当前的线程名
            //        Log.d(TAG,Thread.currentThread().getName()+" "+i);

            if(this.ticket>0){
                Log.e(TAG, Thread.currentThread().getName() + ", 卖票：ticket=" + ticket--);
            }
        }


        return ticket;
    }
}

private void starCallableThread(){
    ThirdThread thirdThread = new ThirdThread();
    FutureTask<Integer> task = new FutureTask<Integer>(thirdThread);

    new Thread(task,"有返回值的线程").start();

    try {
        Integer integer = task.get();
        Log.d(TAG,"starCallableThread, 子线程的返回值="+integer);

    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }

}
```

执行结果

![](https://tinaxiawuhao.github.io/post-images/1620451283904.png)

**注意**：注意：Callable的call() 方法允许声明抛出异常，并且允许带有返回值。 

程序最后调用FutureTask 对象的get()方法来返回Call(）方法的返回值，导致主线程被阻塞，直到call()方法结束并返回为止。

#### 使用 Executors 工具类创建线程池

Executors提供了一系列工厂方法用于创先线程池，返回的线程池都实现了`ExecutorService`接口。

主要有`newFixedThreadPool`，`newCachedThreadPool`，`newSingleThreadExecutor`，`newScheduledThreadPool`，后续详细介绍这四种线程池

```java
public class MyRunnable implements Runnable {

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " run()方法执行中...");
    }

}
public class SingleThreadExecutorTest {
    
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        MyRunnable runnableTest = new MyRunnable();
        for (int i = 0; i < 5; i++) {
            executorService.execute(runnableTest);
        }

        System.out.println("线程任务开始执行");
        executorService.shutdown();
    }

}
```



执行结果

![](https://tinaxiawuhao.github.io/post-images/1620451292701.png)

#### 说一下 runnable 和 callable 有什么区别？

相同点

- 都是接口
- 都可以编写多线程程序
- 都采用Thread.start()启动线程

主要区别

- Runnable 接口 run 方法无返回值；Callable 接口 call 方法有返回值，是个泛型，和Future、FutureTask配合可以用来获取异步执行的结果
- Runnable 接口 run 方法只能抛出运行时异常，且无法捕获处理；Callable 接口 call 方法允许抛出异常，可以获取异常信息

**注**：Callalbe接口支持返回执行结果，需要调用FutureTask.get()得到，此方法会阻塞主进程的继续往下执行，如果不调用不会阻塞。

#### 线程的 run()和 start()有什么区别？

每个线程都是通过某个特定Thread对象所对应的方法run()来完成其操作的，run()方法称为线程体。通过调用Thread类的start()方法来启动一个线程。

start() 方法用于启动线程，run() 方法用于执行线程的运行时代码。run() 可以重复调用，而 start() 只能调用一次。

#### 为什么我们调用 start() 方法时会执行 run() 方法，为什么我们不能直接调用 run() 方法？

new 一个 Thread，线程进入了新建状态。调用 start() 方法，会启动一个线程并使线程进入了就绪状态，当分配到时间片后就可以开始运行了。 start() 会执行线程的相应准备工作，然后自动执行 run() 方法的内容，这是真正的多线程工作。

而直接执行 run() 方法，会把 run 方法当成一个 main 线程下的普通方法去执行，并不会在某个线程中执行它，所以这并不是多线程工作。

总结： 调用 start 方法方可启动线程并使线程进入就绪状态，而 run 方法只是 thread 的一个普通方法调用，还是在主线程里执行。

#### 什么是 Callable 和 Future?

Callable 接口类似于 Runnable，从名字就可以看出来了，但是 Runnable 不会返回结果，并且无法抛出返回结果的异常，而 Callable 功能更强大一些，被线程执行后，可以返回值，这个返回值可以被 Future 拿到，也就是说，Future 可以拿到异步执行任务的返回值。

Future 接口表示异步任务，是一个可能还没有完成的异步任务的结果。所以说 Callable用于产生结果，Future 用于获取结果。

#### 什么是 FutureTask

FutureTask 表示一个异步运算的任务。FutureTask 里面可以传入一个 Callable 的具体实现类，可以对这个异步运算的任务的结果进行等待获取、判断是否已经完成、取消任务等操作。只有当运算完成的时候结果才能取回，如果运算尚未完成 get 方法将会阻塞。一个 FutureTask 对象可以对调用了 Callable 和 Runnable 的对象进行包装，由于 FutureTask 也是Runnable 接口的实现类，所以 FutureTask 也可以放入线程池中。