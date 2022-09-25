---
title: '线程池ThreadPoolExecutor'
date: 2021-09-08 16:05:07
tags: [java]
published: true
hideInList: false
feature: /post-images/y8VkHlUlp.png
isTop: false
---
### Executors目前提供了5种不同的线程池创建配置：

1. newCachedThreadPool（），它是用来处理大量短时间工作任务的线程池，具有几个鲜明特点：它会试图缓存线程并重用，当无缓存线程可用时，就会创建新的工作线程；如果线程闲置时间超过60秒，则被终止并移除缓存；长时间闲置时，这种线程池，不会消耗什么资源。其内部使用SynchronousQueue作为工作队列。

2. newFixedThreadPool（int nThreads），重用指定数目（nThreads）的线程，其背后使用的是无界的工作队列，任何时候最多有nThreads个工作线程是活动的。这意味着，如果任务数量超过了活动线程数目，将在工作队列中等待空闲线程出现；如果工作线程退出，将会有新的工作线程被创建，以补足指定数目nThreads。

3. newSingleThreadExecutor()，它的特点在于工作线程数目限制为1，操作一个无界的工作队列，所以它保证了所有的任务都是被顺序执行，最多会有一个任务处于活动状态，并且不予许使用者改动线程池实例，因此可以避免改变线程数目。

4. newSingleThreadScheduledExecutor()和newScheduledThreadPool(int corePoolSize)，创建的是个ScheduledExecutorService，可以进行定时或周期性的工作调度，区别在于单一工作线程还是多个工作线程。

5. newWorkStealingPool(int parallelism)，这是一个经常被人忽略的线程池，Java 8 才加入这个创建方法，其内部会构建[ForkJoinPool](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/ForkJoinPool.html)，利用[Work-Stealing](https://en.wikipedia.org/wiki/Work_stealing)算法，并行地处理任务，不保证处理顺序。

```java
public class MyThreadPool {
    public static void main(String[] args) {
//        ExecutorService executorService = Executors.newFixedThreadPool(5);
//        ExecutorService executorService= Executors.newSingleThreadExecutor();
//        ExecutorService executorService= Executors.newCachedThreadPool();
//        ExecutorService executorService= Executors.newScheduledThreadPool(5);

        ExecutorService executorService = new ThreadPoolExecutor(2,
                5,
                2,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(3),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy()
        );
        ThreadPoolInit(executorService);
    }

    private static void ThreadPoolInit(ExecutorService executorService) {

        try {
            for (int i = 1; i <= 10; i++) {
                executorService.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "办理业务");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            executorService.shutdown();
        }
    }
}

```

### Executor框架的基本组成

![](https://tinaxiawuhao.github.io/post-images/1631088364651.png)
而我们创建时，一般使用它的子类：ThreadPoolExecutor.

```java
public ThreadPoolExecutor(int corePoolSize,  
                              int maximumPoolSize,  
                              long keepAliveTime,  
                              TimeUnit unit,  
                              BlockingQueue<Runnable> workQueue,  
                              ThreadFactory threadFactory,  
                              RejectedExecutionHandler handler)
```


这是其中最重要的一个构造方法，这个方法决定了创建出来的线程池的各种属性，下面依靠一张图来更好的理解线程池和这几个参数：

![](https://tinaxiawuhao.github.io/post-images/1631088504377.png)

（1）corePoolSize：线程池中常驻核心线程数

（2）maximumPoolSize：线程池能够容纳同时执行的最大线程数，此值必须大于等于1

（3）keepAliveTime：多余的空闲线程存活时间。当前线程池数量超过corePoolSize时，当空闲时间到达keepAliveTime值时，多余空闲线程会被销毁直到只剩下corePoolSize个线程为止。

（4）unit：keepAliveTime的时间单位

（5）workQueue：任务队列，被提交但尚未执行的任务

（6）threadFactory：表示生成线程池中的工作线程的线程工厂，用于创建线程，一般为默认线程工厂即可

（7）handler：拒绝策略，表示当队列满了并且工作线程大于等于线程池的最大线程数（maximumPoolSize）时如何来拒绝来请求的Runnable的策略

### handler的拒绝策略：

有四种：

1. 第一种AbortPolicy:不执行新任务，直接抛出异常，提示线程池已满
2. 第二种DisCardPolicy:不执行新任务，也不抛出异常
3. 第三种DisCardOldSetPolicy:将消息队列中的第一个任务替换为当前新进来的任务执行
4. 第四种CallerRunsPolicy:直接调用execute来执行当前任务

### 在spring项目中的使用
先创建一个线程池的配置，让Spring Boot加载，用来定义如何创建一个`ThreadPoolTaskExecutor`，要使用`@Configuration`和@`EnableAsync`这两个注解，表示这是个配置类，并且是线程池的配置类

```java
@Configuration
@EnableAsync
public class ExecutorConfig {

    private static final Logger logger = LoggerFactory.getLogger(ExecutorConfig.class);

    @Value("${async.executor.thread.core_pool_size}")
    private int corePoolSize;
    @Value("${async.executor.thread.max_pool_size}")
    private int maxPoolSize;
    @Value("${async.executor.thread.queue_capacity}")
    private int queueCapacity;
    @Value("${async.executor.thread.name.prefix}")
    private String namePrefix;

    @Bean(name = "asyncServiceExecutor")
    public Executor asyncServiceExecutor() {
        logger.info("start asyncServiceExecutor");
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //配置核心线程数
        executor.setCorePoolSize(corePoolSize);
        //配置最大线程数
        executor.setMaxPoolSize(maxPoolSize);
        //配置队列大小
        executor.setQueueCapacity(queueCapacity);
        //配置线程池中的线程的名称前缀
        executor.setThreadNamePrefix(namePrefix);

        // rejection-policy：当pool已经达到max size的时候，如何处理新任务
        // CALLER_RUNS：不在新线程中执行任务，而是有调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        //执行初始化
        executor.initialize();
        return executor;
    }
}
```

`@Value`是我配置在`application.properties`，可以参考配置，自由定义

```xml
# 异步线程配置
# 配置核心线程数
async.executor.thread.core_pool_size = 5
# 配置最大线程数
async.executor.thread.max_pool_size = 5
# 配置队列大小
async.executor.thread.queue_capacity = 99999
# 配置线程池中的线程的名称前缀
async.executor.thread.name.prefix = async-service-
```

创建一个Service接口，是异步线程的接口

```java
public interface AsyncService {

    /** * 执行异步任务 * 可以根据需求，自己加参数拟定，我这里就做个测试演示 */
    void executeAsync();
}
```

实现类

```java
@Service
public class AsyncServiceImpl implements AsyncService {

    private static final Logger logger = LoggerFactory.getLogger(AsyncServiceImpl.class);

    @Override
    @Async("asyncServiceExecutor")
    public void executeAsync() {
        logger.info("start executeAsync");

        System.out.println("异步线程要做的事情");
        System.out.println("可以在这里执行批量插入等耗时的事情");

        logger.info("end executeAsync");
    }
}
```

将Service层的服务异步化，在`executeAsync()`方法上增加注解`@Async("asyncServiceExecutor")`，`asyncServiceExecutor`方法是前面**ExecutorConfig.java** 中的方法名，表明`executeAsync`方法进入的线程池是`asyncServiceExecutor`方法创建的

接下来就是在Controller里或者是哪里通过注解`@Autowired`注入这个Service

```java
@Autowired
private AsyncService asyncService;

@GetMapping("/async")
public void async(){
    asyncService.executeAsync();
}
```

**用postmain或者其他工具来多次测试请求一下**

```java
 2018-07-16 22:15:47.655  INFO 10516 --- [async-service-5] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:15:47.655  INFO 10516 --- [async-service-5] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:15:47.770  INFO 10516 --- [async-service-1] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:15:47.770  INFO 10516 --- [async-service-1] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:15:47.816  INFO 10516 --- [async-service-2] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:15:47.816  INFO 10516 --- [async-service-2] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:15:48.833  INFO 10516 --- [async-service-3] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:15:48.834  INFO 10516 --- [async-service-3] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:15:48.986  INFO 10516 --- [async-service-4] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:15:48.987  INFO 10516 --- [async-service-4] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
```

通过以上日志可以发现，`[async-service-]`是有多个线程的，显然已经在我们配置的线程池中执行了，并且每次请求中，controller的起始和结束日志都是连续打印的，表明每次请求都快速响应了，而耗时的操作都留给线程池中的线程去异步执行；

虽然我们已经用上了线程池，但是还不清楚线程池当时的情况，有多少线程在执行，多少在队列中等待呢？这里我创建了一个ThreadPoolTaskExecutor的子类，在每次提交线程的时候都会将当前线程池的运行状况打印出来

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.util.concurrent.ListenableFuture;

import java.util.concurrent.Callable;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;

/** * @Author: ChenBin * @Date: 2018/7/16/0016 22:19 */
public class VisiableThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {


    private static final Logger logger = LoggerFactory.getLogger(VisiableThreadPoolTaskExecutor.class);

    private void showThreadPoolInfo(String prefix) {
        ThreadPoolExecutor threadPoolExecutor = getThreadPoolExecutor();

        if (null == threadPoolExecutor) {
            return;
        }

        logger.info("{}, {},taskCount [{}], completedTaskCount [{}], activeCount [{}], queueSize [{}]",
                this.getThreadNamePrefix(),
                prefix,
                threadPoolExecutor.getTaskCount(),
                threadPoolExecutor.getCompletedTaskCount(),
                threadPoolExecutor.getActiveCount(),
                threadPoolExecutor.getQueue().size());
    }

    @Override
    public void execute(Runnable task) {
        showThreadPoolInfo("1. do execute");
        super.execute(task);
    }

    @Override
    public void execute(Runnable task, long startTimeout) {
        showThreadPoolInfo("2. do execute");
        super.execute(task, startTimeout);
    }

    @Override
    public Future<?> submit(Runnable task) {
        showThreadPoolInfo("1. do submit");
        return super.submit(task);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        showThreadPoolInfo("2. do submit");
        return super.submit(task);
    }

    @Override
    public ListenableFuture<?> submitListenable(Runnable task) {
        showThreadPoolInfo("1. do submitListenable");
        return super.submitListenable(task);
    }

    @Override
    public <T> ListenableFuture<T> submitListenable(Callable<T> task) {
        showThreadPoolInfo("2. do submitListenable");
        return super.submitListenable(task);
    }
}
```

如上所示，showThreadPoolInfo方法中将任务总数、已完成数、活跃线程数，队列大小都打印出来了，然后Override了父类的execute、submit等方法，在里面调用showThreadPoolInfo方法，这样每次有任务被提交到线程池的时候，都会将当前线程池的基本情况打印到日志中；

修改`ExecutorConfig.java`的`asyncServiceExecutor`方法，将`ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor()`改为`ThreadPoolTaskExecutor executor = new VisiableThreadPoolTaskExecutor()`

```java
@Bean(name = "asyncServiceExecutor")
    public Executor asyncServiceExecutor() {
        logger.info("start asyncServiceExecutor");
        //在这里修改
        ThreadPoolTaskExecutor executor = new VisiableThreadPoolTaskExecutor();
        //配置核心线程数
        executor.setCorePoolSize(corePoolSize);
        //配置最大线程数
        executor.setMaxPoolSize(maxPoolSize);
        //配置队列大小
        executor.setQueueCapacity(queueCapacity);
        //配置线程池中的线程的名称前缀
        executor.setThreadNamePrefix(namePrefix);

        // rejection-policy：当pool已经达到max size的时候，如何处理新任务
        // CALLER_RUNS：不在新线程中执行任务，而是有调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        //执行初始化
        executor.initialize();
        return executor;
    }
```

再次启动该工程测试

```java
2018-07-16 22:23:30.951  INFO 14088 --- [nio-8087-exec-2] u.d.e.e.i.VisiableThreadPoolTaskExecutor : async-service-, 2. do submit,taskCount [0], completedTaskCount [0], activeCount [0], queueSize [0]
2018-07-16 22:23:30.952  INFO 14088 --- [async-service-1] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:23:30.953  INFO 14088 --- [async-service-1] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:23:31.351  INFO 14088 --- [nio-8087-exec-3] u.d.e.e.i.VisiableThreadPoolTaskExecutor : async-service-, 2. do submit,taskCount [1], completedTaskCount [1], activeCount [0], queueSize [0]
2018-07-16 22:23:31.353  INFO 14088 --- [async-service-2] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:23:31.353  INFO 14088 --- [async-service-2] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:23:31.927  INFO 14088 --- [nio-8087-exec-5] u.d.e.e.i.VisiableThreadPoolTaskExecutor : async-service-, 2. do submit,taskCount [2], completedTaskCount [2], activeCount [0], queueSize [0]
2018-07-16 22:23:31.929  INFO 14088 --- [async-service-3] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:23:31.930  INFO 14088 --- [async-service-3] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
2018-07-16 22:23:32.496  INFO 14088 --- [nio-8087-exec-7] u.d.e.e.i.VisiableThreadPoolTaskExecutor : async-service-, 2. do submit,taskCount [3], completedTaskCount [3], activeCount [0], queueSize [0]
2018-07-16 22:23:32.498  INFO 14088 --- [async-service-4] c.u.d.e.executor.impl.AsyncServiceImpl   : start executeAsync
异步线程要做的事情
可以在这里执行批量插入等耗时的事情
2018-07-16 22:23:32.499  INFO 14088 --- [async-service-4] c.u.d.e.executor.impl.AsyncServiceImpl   : end executeAsync
```

注意这一行日志：

```java
2018-07-16 22:23:32.496  INFO 14088 --- [nio-8087-exec-7] u.d.e.e.i.VisiableThreadPoolTaskExecutor : async-service-, 2. do submit,taskCount [3], completedTaskCount [3], activeCount [0], queueSize [0]
```

这说明提交任务到线程池的时候，调用的是submit(Callable task)这个方法，当前已经提交了3个任务，完成了3个，当前有0个线程在处理任务，还剩0个任务在队列中等待，线程池的基本情况一路了然