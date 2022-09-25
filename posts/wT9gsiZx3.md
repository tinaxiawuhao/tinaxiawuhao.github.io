---
title: 'ExecutorCompletionService'
date: 2022-03-19 22:16:10
tags: [java]
published: true
hideInList: false
feature: /post-images/wT9gsiZx3.png
isTop: false
---
### Future

Future模式是多线程设计常用的一种设计模式。Future模式可以理解成：我有一个任务，提交给了Future，Future替我完成这个任务。期间我自己可以去做任何想做的事情。一段时间之后，我就便可以从Future那儿取出结果。

#### Future提供了三种功能：

- 判断任务是否完成
- 能够中断任务
- 能够获取任务执行的结果
  向线程池中提交任务的**submit方法不是阻塞方法，而Future.get方法是一个阻塞方法**，当submit提交多个任务时，只有所有任务都完成后，才能使用get按照任务的提交顺序得到返回结果，所以一般需要使用future.isDone先判断任务是否全部执行完成，完成后再使用future.get得到结果。（也可以用get (long timeout, TimeUnit unit)方法可以设置超时时间，防止无限时间的等待）
  三段式的编程：1.启动多线程任务2.处理其他事3.收集多线程任务结果，Future虽然可以实现获取异步执行结果的需求，但是它没有提供通知的机制，要么使用阻塞，在future.get()的地方等待future返回的结果，这时又变成同步操作；要么使用isDone()轮询地判断Future是否完成，这样会耗费CPU的资源。
  **解决方法：CompletionService和CompletableFuture（按照任务完成的先后顺序获取任务的结果）**

### ExecutorService

创建线程池，多线程功能调用

```java
public static void test1() throws Exception {
    ExecutorService executorService = Executors.newCachedThreadPool();
    ArrayList<Future<String>> futureArrayList = new ArrayList<>();
    System.out.println("公司让你通知大家聚餐 你开车去接人");

    Future<String> future10 = executorService.submit(() -> {
        System.out.println("总裁：我在家上大号 我最近拉肚子比较慢 要蹲1个小时才能出来 你等会来接我吧");
        TimeUnit.SECONDS.sleep(10);
        System.out.println("总裁：1小时了 我上完大号了。你来接吧");
        return "总裁上完大号了";
    });
    futureArrayList.add(future10);

    Future<String> future6 = executorService.submit(() -> {
        System.out.println("中层管理：我在家上大号  要蹲10分钟就可以出来 你等会来接我吧");
        TimeUnit.SECONDS.sleep(6);
        System.out.println("中层管理：10分钟 我上完大号了。你来接吧");
        return "中层管理上完大号了";
    });
    futureArrayList.add(future6);

    Future<String> future3 = executorService.submit(() -> {
        System.out.println("研发：我在家上大号 我比较快 要蹲3分钟就可以出来 你等会来接我吧");
        TimeUnit.SECONDS.sleep(3);
        System.out.println("研发：3分钟 我上完大号了。你来接吧");
        return "研发上完大号了";
    });
    futureArrayList.add(future3);

    TimeUnit.SECONDS.sleep(1);
    System.out.println("都通知完了,等着接吧。");
    try {
        for (Future<String> future : futureArrayList) {
            String returnStr = future.get();
            System.out.println(returnStr + "，你去接他");
        }
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        executorService.shutdown();
    }

}
```

**结果**

```java
公司让你通知大家聚餐 你开车去接人
研发：我在家上大号 我比较快 要蹲3分钟就可以出来 你等会来接我吧
总裁：我在家上大号 我最近拉肚子比较慢 要蹲1个小时才能出来 你等会来接我吧
中层管理：我在家上大号  要蹲10分钟就可以出来 你等会来接我吧
都通知完了,等着接吧。
研发：3分钟 我上完大号了。你来接吧
研发上完大号了，你去接他
中层管理：10分钟 我上完大号了。你来接吧
总裁：1小时了 我上完大号了。你来接吧
总裁上完大号了，你去接他
中层管理上完大号了，你去接他
```

### ExecutorCompletionService

```java
public static void test2() throws Exception {
    ExecutorService executorService = Executors.newCachedThreadPool();
    ExecutorCompletionService<String> completionService = new ExecutorCompletionService<>(executorService);
    System.out.println("公司让你通知大家聚餐 你开车去接人");
    completionService.submit(() -> {
        System.out.println("总裁：我在家上大号 我最近拉肚子比较慢 要蹲1个小时才能出来 你等会来接我吧");
        TimeUnit.SECONDS.sleep(10);
        System.out.println("总裁：1小时了 我上完大号了。你来接吧");
        return "总裁上完大号了";
    });
     completionService.submit(() -> {
        System.out.println("中层管理：我在家上大号  要蹲10分钟就可以出来 你等会来接我吧");
        TimeUnit.SECONDS.sleep(6);
        System.out.println("中层管理：10分钟 我上完大号了。你来接吧");
        return "中层管理上完大号了";
    });
    completionService.submit(() -> {
        System.out.println("研发：我在家上大号 我比较快 要蹲3分钟就可以出来 你等会来接我吧");
        TimeUnit.SECONDS.sleep(3);
        System.out.println("研发：3分钟 我上完大号了。你来接吧");
        return "研发上完大号了";
    });
   
    TimeUnit.SECONDS.sleep(1);
    System.out.println("都通知完了,等着接吧。");
    //提交了3个异步任务）
    try {
        for (int i = 0; i < 3; i++) {
            String returnStr = completionService.take().get();
            System.out.println(returnStr + "，你去接他");
        }
        Thread.currentThread().join();
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        executorService.shutdown();
    }

}
```

**结果**

```java
公司让你通知大家聚餐 你开车去接人
总裁：我在家上大号 我最近拉肚子比较慢 要蹲1个小时才能出来 你等会来接我吧
研发：我在家上大号 我比较快 要蹲3分钟就可以出来 你等会来接我吧
中层管理：我在家上大号  要蹲10分钟就可以出来 你等会来接我吧
都通知完了,等着接吧。
研发：3分钟 我上完大号了。你来接吧
研发上完大号了，你去接他
中层管理：10分钟 我上完大号了。你来接吧
中层管理上完大号了，你去接他
总裁：1小时了 我上完大号了。你来接吧
总裁上完大号了，你去接他
```

### 源码分析
completionService是JUC里的线程池ExecutorCompletionService

#### 初始化线程池

1. 同步初始化一个完成队列completionQueue

   ```java
   public ExecutorCompletionService(Executor executor) {
       if (executor == null)
           throw new NullPointerException();
       this.executor = executor;
       this.aes = (executor instanceof AbstractExecutorService) ?
           (AbstractExecutorService) executor : null;
       this.completionQueue = new LinkedBlockingQueue<Future<V>>();
   }
   ```

   

#### 执行submit方法

1. 线程池执行的时候传入了自己的内部类QueueingFuture

   ```java
   public Future<V> submit(Callable<V> task) {
       if (task == null) throw new NullPointerException();
       RunnableFuture<V> f = newTaskFor(task);
       executor.execute(new QueueingFuture<V>(f, completionQueue));
       return f;
   }
   ```

2. QueueingFuture构造函数，需要关注这里重写了done方法，向阻塞队列里添加task

   ```java
   private static class QueueingFuture<V> extends FutureTask<Void> {
       QueueingFuture(RunnableFuture<V> task,
                      BlockingQueue<Future<V>> completionQueue) {
           super(task, null);
           this.task = task;
           this.completionQueue = completionQueue;
       }
       private final Future<V> task;
       private final BlockingQueue<Future<V>> completionQueue;
       protected void done() { completionQueue.add(task); } //往completionQueue里面存值
   }
   ```

3. 执行FutureTask的run方法

   ```java
   public void run() {
       if (state != NEW ||
           !RUNNER.compareAndSet(this, null, Thread.currentThread()))
           return;
       try {
           Callable<V> c = callable;
           if (c != null && state == NEW) {
               V result;
               boolean ran;
               try {
                   result = c.call();
                   ran = true;
               } catch (Throwable ex) {
                   result = null;
                   ran = false;
                   setException(ex);
               }
               if (ran)
                   set(result); //设置值
           }
       } finally {
           // runner must be non-null until state is settled to
           // prevent concurrent calls to run()
           runner = null;
           // state must be re-read after nulling runner to prevent
           // leaked interrupts
           int s = state;
           if (s >= INTERRUPTING)
               handlePossibleCancellationInterrupt(s);
       }
   }
   ```

4. set方法

   ```java
   protected void set(V v) {
       if (STATE.compareAndSet(this, NEW, COMPLETING)) {
           outcome = v;
           STATE.setRelease(this, NORMAL); // final state
           finishCompletion(); 
       }
   }
   ```

5. 在finishCompletion方法中执行了done方法

   ```java
   private void finishCompletion() {
       // assert state > COMPLETING;
       for (WaitNode q; (q = waiters) != null;) {
           if (WAITERS.weakCompareAndSet(this, q, null)) {
               for (;;) {
                   Thread t = q.thread;
                   if (t != null) {
                       q.thread = null;
                       LockSupport.unpark(t);
                   }
                   WaitNode next = q.next;
                   if (next == null)
                       break;
                   q.next = null; // unlink to help gc
                   q = next;
               }
               break;
           }
       }
   
       done();             //执行done方法-completionQueue.add(task);
   
       callable = null;        // to reduce footprint
   }
   ```

所以每次线程完成任务，都会往completionQueue中新增一条记录。completionQueue中的记录需要通过别的方式取：ExecutorCompletionService.take()。
ExecutorCompletionService的实际用法应该是这样的：CompletionService一边执行任务，一边处理完成的任务结果，这样可以将执行的任务与处理任务隔离开来进行处理，使用submit执行任务，使用take获取已完成的任务，先完成的任务结果先加入队列。获取的结果，跟提交给线程池的任务是无关联的。

```java
Future<String> submit = completionService.submit(() -> {
    return "假装有返回";
});
submit.get();//直接获取返回值
```

如上如果我们在使用过程中由于需要的是当前线程任务的返回值，所以没有调用take方法，而是通过get方法获取到当前线程调用的返回值，会导致task任务在队列中一直增加，从而造成OOM
