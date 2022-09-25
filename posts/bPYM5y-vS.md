---
title: 'CountDownLatch,CyclicBarrier,Semaphore的用法和区别'
date: 2022-04-20 19:55:53
tags: [java]
published: true
hideInList: false
feature: /post-images/bPYM5y-vS.png
isTop: false
---
### CountDownLatch
> CountDownLatch（也叫闭锁）是一个同步协助类，允许一个或多个线程等待，直到其他线程完成操作集。

CountDownLatch 使用给定的计数值（count）初始化。await 方法会阻塞直到当前的计数值（count）由于 countDown 方法的调用达到 0，count 为 0 之后所有等待的线程都会被释放，并且随后对await方法的调用都会立即返回。

#### 构造方法
```java
//参数count为计数值
public CountDownLatch(int count) {}; 
```

#### 常用方法
```java
// 调用 await() 方法的线程会被挂起，它会等待直到 count 值为 0 才继续执行
public void await() throws InterruptedException {};

// 和 await() 类似，若等待 timeout 时长后，count 值还是没有变为 0，不再等待，继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException {};

// 会将 count 减 1，直至为 0
public void countDown() {};
```


#### 使用案例
> 首先是创建实例 CountDownLatch countDown = new CountDownLatch(2)；
> 需要同步的线程执行完之后，计数 -1， countDown.countDown()；
> 需要等待其他线程执行完毕之后，再运行的线程，调用 countDown.await()实现阻塞同步。

**应用场景**
CountDownLatch 一般用作多线程倒计时计数器，强制它们等待其他一组（CountDownLatch的初始化决定）任务执行完成。

CountDownLatch的两种使用场景：

> 让多个线程等待，模拟并发。
> 让单个线程等待，多个线程（任务）完成后，进行汇总合并。

**场景 1：模拟并发**

```java
import java.util.concurrent.CountDownLatch;

/**
 * 让多个线程等待：模拟并发，让并发线程一起执行
 */
public class CountDownLatchTest {
    public static void main(String[] args) throws InterruptedException {

        CountDownLatch countDownLatch = new CountDownLatch(1);
        
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    // 等待
                    countDownLatch.await();
                    String parter = "【" + Thread.currentThread().getName() + "】";
                    System.out.println(parter + "开始执行……");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }

        Thread.sleep(2000);
       
        countDownLatch.countDown();
    }
}
```

**场景 2：多个线程完成后，进行汇总合并**

很多时候，我们的并发任务，存在前后依赖关系；比如数据详情页需要同时调用多个接口获取数据，并发请求获取到数据后、需要进行结果合并；或者多个数据操作完成后，需要数据 check；这其实都是：在多个线程(任务)完成后，进行汇总合并的场景。

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;

/**
 * 让单个线程等待：多个线程(任务)完成后，进行汇总合并
 */
public class CountDownLatchTest3 {

    //用于聚合所有的统计指标
    private static Map map = new ConcurrentHashMap();
    //创建计数器，这里需要统计4个指标
    private static CountDownLatch countDownLatch = new CountDownLatch(4);

    public static void main(String[] args) throws Exception {

        //记录开始时间
        long startTime = System.currentTimeMillis();

        Thread countUserThread = new Thread(() -> {
            try {
                System.out.println("正在统计新增用户数量");
                Thread.sleep(3000);//任务执行需要3秒
                map.put("userNumber", 100);//保存结果值
                System.out.println("统计新增用户数量完毕");
                countDownLatch.countDown();//标记已经完成一个任务
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        Thread countOrderThread = new Thread(() -> {
            try {
                System.out.println("正在统计订单数量");
                Thread.sleep(3000);//任务执行需要3秒
                map.put("countOrder", 20);//保存结果值
                System.out.println("统计订单数量完毕");
                countDownLatch.countDown();//标记已经完成一个任务
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        Thread countGoodsThread = new Thread(() -> {
            try {
                System.out.println("正在商品销量");
                Thread.sleep(3000);//任务执行需要3秒
                map.put("countGoods", 300);//保存结果值
                System.out.println("统计商品销量完毕");
                countDownLatch.countDown();//标记已经完成一个任务
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        Thread countmoneyThread = new Thread(() -> {
            try {
                System.out.println("正在总销售额");
                Thread.sleep(3000);//任务执行需要3秒
                map.put("countMoney", 40000);//保存结果值
                System.out.println("统计销售额完毕");
                countDownLatch.countDown();//标记已经完成一个任务
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        //启动子线程执行任务
        countUserThread.start();
        countGoodsThread.start();
        countOrderThread.start();
        countmoneyThread.start();

        try {
            //主线程等待所有统计指标执行完毕
            countDownLatch.await();
            long endTime = System.currentTimeMillis();//记录结束时间
            System.out.println("------统计指标全部完成--------");
            System.out.println("统计结果为：" + map);
            System.out.println("任务总执行时间为" + (endTime - startTime) + "ms");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```

### CylicBarrier
从字面上的意思可以知道，这个类的中文意思是“循环栅栏”。大概的意思就是一个可循环利用的屏障。

它的作用就是会让所有线程都等待完成后才会继续下一步行动。

现实生活中我们经常会遇到这样的情景，在进行某个活动前需要等待人全部都齐了才开始。例如吃饭时要等全家人都上座了才动筷子，旅游时要等全部人都到齐了才出发，比赛时要等运动员都上场后才开始。

在`JUC`包中为我们提供了一个同步工具类能够很好的模拟这类场景，它就是`CyclicBarrier`类。利用`CyclicBarrier`类可以实现一组线程相互等待，当所有线程都到达某个屏障点后再进行后续的操作。

`CyclicBarrier`字面意思是“可重复使用的栅栏”，`CyclicBarrier`相比 `CountDownLatch `来说，要简单很多，其源码没有什么高深的地方，它是 `ReentrantLock` 和 `Condition` 的组合使用。

看如下示意图，`CyclicBarrier` 和 `CountDownLatch` 是不是很像，只是 `CyclicBarrier` 可以有不止一个栅栏，因为它的栅栏（Barrier）可以重复使用（Cyclic）。

![](https://tianxiawuhao.github.io/post-images/1656504599018.png)

就好比以前的那种客车一样，当第一轮车坐满之后发车，然后接着等第二辆车坐满之后在发车。

#### 构造方法
```java
// parties表示屏障拦截的线程数量，每个线程调用 await 方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。
public CyclicBarrier(int parties)
    
// 用于在线程到达屏障时，优先执行 barrierAction，方便处理更复杂的业务场景(该线程的执行时机是在到达屏障之后再执行)
public CyclicBarrier(int parties, Runnable barrierAction)
```


#### 常用方法
```java
//屏障 指定数量的线程全部调用await()方法时，这些线程不再阻塞
// BrokenBarrierException 表示栅栏已经被破坏，破坏的原因可能是其中一个线程 await() 时被中断或者超时
public int await() throws InterruptedException, BrokenBarrierException
    
public int await(long timeout, TimeUnit unit) throws InterruptedException, B
rokenBarrierException, TimeoutException
    
//循环 通过reset()方法可以进行重置
public void reset()
```
#### 使用案例
```java
import java.util.concurrent.CyclicBarrier;

/**
 * CyclicBarrier(回环栅栏)允许一组线程互相等待，直到到达某个公共屏障点 (Common Barrier Point)
 * CountDownLatch 用于等待countDown事件，而栅栏用于等待其他线程。
 */
public class CyclicBarrierTest {

    public static void main(String[] args) {

        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);

        for (int i = 0; i < 5; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println(Thread.currentThread().getName()
                                + "开始等待其他线程");
                        cyclicBarrier.await();
                        System.out.println(Thread.currentThread().getName() + "开始执行");
                        //TODO 模拟业务处理
                        Thread.sleep(5000);
                        System.out.println(Thread.currentThread().getName() + "执行完毕");

                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }
}
```



**应用场景**
可以用于多线程计算数据，最后合并计算结果的场景。

```java
import java.util.Set;
import java.util.concurrent.*;

public class CyclicBarrierTest2 {

    //保存每个学生的平均成绩
    private ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<String, Integer>();

    private ExecutorService threadPool = Executors.newFixedThreadPool(3);

    private CyclicBarrier cb = new CyclicBarrier(3, () -> {
        int result = 0;
        Set<String> set = map.keySet();
        for (String s : set) {
            result += map.get(s);
        }
        System.out.println("三人平均成绩为:" + (result / 3) + "分");
    });


    public void count() {
        for (int i = 0; i < 3; i++) {
            threadPool.execute(new Runnable() {

                @Override
                public void run() {
                    //获取学生平均成绩
                    int score = (int) (Math.random() * 40 + 60);
                    map.put(Thread.currentThread().getName(), score);
                    System.out.println(Thread.currentThread().getName() + "同学的平均成绩为：" + score);
                    try {
                        //执行完运行await(),等待所有学生平均成绩都计算完毕
                        cb.await();
                    } catch (InterruptedException | BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }


    public static void main(String[] args) {
        CyclicBarrierTest2 cb = new CyclicBarrierTest2();
        cb.count();
    }
}
```

测试结果：

![](https://tianxiawuhao.github.io/post-images/1656504516218.png)

### Semaphore
Semaphore，俗称信号量，基于 AbstractQueuedSynchronizer 实现。使用 Semaphore 可以控制同时访问资源的线程个数。

比如：停车场入口立着的那个显示屏，每有一辆车进入停车场显示屏就会显示剩余车位减 1，每有一辆车从停车场出去，显示屏上显示的剩余车辆就会加 1，当显示屏上的剩余车位为 0 时，停车场入口的栏杆就不会再打开，车辆就无法进入停车场了，直到有一辆车从停车场出去为止。

比如：在学生时代都去餐厅打过饭，假如有 3 个窗口可以打饭，同一时刻也只能有 3 名同学打饭。第 4 个人来了之后就必须在外面等着，只要有打饭的同学好了，就可以去相应的窗口了 。

#### 构造方法
```java
//创建具有给定的许可数和非公平的公平设置的 Semaphore。  
Semaphore(int permits)   

//创建具有给定的许可数和给定的公平设置的 Semaphore。  
Semaphore(int permits, boolean fair)   
```

> permits 表示许可证的数量（资源数），就好比一个学生可以占用 3 个打饭窗口。
> fair 表示公平性，如果这个设为 true 的话，下次执行的线程会是等待最久的线程。

#### 常用方法
```java
public void acquire() throws InterruptedException
public boolean tryAcquire()
public void release()
public int availablePermits()
public final int getQueueLength()
public final boolean hasQueuedThreads()
protected void reducePermits(int reduction)
protected Collection<Thread> getQueuedThreads()
```

>acquire()：表示阻塞并获取许可。
>tryAcquire()：方法在没有许可的情况下会立即返回 false，要获取许可的线程不会阻塞。
>release()：表示释放许可。
>int availablePermits()：返回此信号量中当前可用的许可证数。
>int getQueueLength()：返回正在等待获取许可证的线程数。
>boolean hasQueuedThreads()：是否有线程正在等待获取许可证。
>void reducePermit(int reduction)：减少 reduction 个许可证。
>Collection getQueuedThreads()：返回所有等待获取许可证的线程集合。

#### 使用案例
我们可以模拟车站买票，假如车站有 3 个窗口售票，那么同一时刻每个窗口只能存在一个人买票，其他人则等待前面的人完成后才可以去买票。

```java
import java.util.concurrent.Semaphore;

public class SemaphoreTest {

    public static void main(String[] args) {
        // 3 个窗口
        Semaphore windows = new Semaphore(3);
        // 模拟 5 个人购票
        for (int i = 0; i < 5; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    // 占用窗口，加锁
                    try {
                        windows.acquire();
                        System.out.println(Thread.currentThread().getName() + "：开始购票");
                        // 买票
                        Thread.sleep(5000);
                        System.out.println(Thread.currentThread().getName() + "：购票成功");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        // 释放许可，释放窗口
                        windows.release();
                    }
                }
            }, "Thread" + i).start();
        }
    }
}
```

测试结果如下：

![](https://tianxiawuhao.github.io/post-images/1656504535515.png)

很明显可以看到当前面 3 个线程购票成功之后，剩余的线程再开始购票。

**应用场景**
可以用于做流量控制，特别是公用资源有限的应用场景。

如我们实现一个同时只能处理 5 个请求的限流器。

```java
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class SemaphoneTest2 {

    /**
     * 实现一个同时只能处理5个请求的限流器
     */
    private static Semaphore semaphore = new Semaphore(5);

    /**
     * 定义一个线程池
     * 0
     */
    private static ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 50, 1
            , TimeUnit.SECONDS, new LinkedBlockingDeque<>(200));

    /**
     * 模拟执行方法
     */
    public static void exec() {
        try {
            semaphore.acquire(1);
            // 模拟真实方法执行
            System.out.println("执行exec方法");
            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            semaphore.release(1);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        {
            for (;;) {
                Thread.sleep(100);
                // 模拟请求以10个/s的速度
                executor.execute(() -> exec());
            }
        }
    }
}
```

### 总结
#### 1、CountDownLatch、CyclicBarrier、Semaphore的区别

`CountDownLatch` 和 `CyclicBarrier` 都能够实现线程之间的等待，只不过它们侧重点不同：

`CountDownLatch` 一般用于某个线程 A 等待若干个其他线程执行完任务之后，它才执行；
而`CyclicBarrier`一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；
另外，`CountDownLatch`是不能够重用的，而 `CyclicBarrier` 是可以重用的（reset）。
`Semaphore`和锁有点类似，它一般用于控制对某组资源的访问权限。

#### 2、CountDownLatch 与 Thread.join 的区别

`CountDownLatch` 的作用就是允许一个或多个线程等待其他线程完成操作，看起来有点类似 join() 方法，但其提供了比 `join()` 更加灵活的API。
`CountDownLatch` 可以手动控制在n个线程里调用 n 次 `countDown() `方法使计数器进行减一操作，也可以在一个线程里调用 n 次执行减一操作。
而 join() 的实现原理是不停检查 join 线程是否存活，如果 join 线程存活则让当前线程永远等待。所以两者之间相对来说还是 `CountDownLatch` 使用起来较为灵活。

#### 3、CyclicBarrier 与 CountDownLatch 区别

`CountDownLatch`的计数器只能使用一次，而`CyclicBarrier`的计数器可以使用reset()方法重置。所以`CyclicBarrier`能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。
`CyclicBarrier`还提供`getNumberWaiting`(可以获得CyclicBarrier阻塞的线程数量)、`isBroken`(用来知道阻塞的线程是否被中断)等方法。
`CountDownLatch`会阻塞主线程，`CyclicBarrier`不会阻塞主线程，只会阻塞子线程。
`CountDownLatch`和`CyclicBarrier`都能够实现线程之间的等待，只不过它们侧重点不同。`CountDownLatch`一般用于一个或多个线程，等待其他线程执行完任务后，再执行。`CyclicBarrier`一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行。
`CyclicBarrier` 还可以提供一个 `barrierAction`，合并多线程计算结果。
`CyclicBarrier`是通过`ReentrantLock`的"独占锁"和`Conditon`来实现一组线程的阻塞唤醒的，而`CountDownLatch`则是通过`AQS`的“共享锁”实现。