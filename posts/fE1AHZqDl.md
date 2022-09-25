---
title: 'Java线程同步五种方法'
date: 2021-05-04 17:16:26
tags: [java]
published: true
hideInList: false
feature: /post-images/fE1AHZqDl.png
isTop: false
---
## 线程同步

即当有一个线程在对内存进行操作时，其他线程都不可以对这个内存地址进行操作，直到该线程完成操作， 其他线程才能对该内存地址进行操作，而其他线程又处于等待状态，实现线程同步的方法有很多，下面介绍java线程同步的五种方法。 

### 同步方法 

使用synchronized关键字修饰的方法。由于java的每个对象都有一个内置锁，当用关键字修饰此方法时，内置锁会保护整个方法。在调用该方法前，需要获得内置锁，否则该线程就处于阻塞状态。 

```java
public synchronized void method(){} 
# synchronized关键字也可以修饰静态方法，此时如果调用该静态方法，将会锁住整个类。
public static synchronized void method(){} 
```

### 同步代码块 

即有synchronized关键字修饰的语句块。被关键字修饰的的语句块会自动加上内置锁，从而实现同步。 

```java
synchronized(object){ 
	# 代码
} 
```

**注：同步是一种高开销的操作，因此应该尽量减少同步的内容。通常没有必要去同步整个方法，使用关键字synchronized修饰关键代码即可。**

### 使用特殊域变量修饰符volatile实现线程同步 

（1）volatile关键字是非锁，比synchronized稍弱的同步机制 

（2）保证此变量对所有的线程的可见性，当一个线程修改了这个变量的值，volatile 保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。

（3）volatile不会提供原子操作，并不能保证线程安全，也不能用来修饰final类型的变量。 

（4）禁止指令重排序优化。有volatile修饰的变量，赋值后多执行了一个添加**内存屏障**操作（指令重排序时不能把后面的指令重排序到内存屏障之前的位置） 

```java
public class Test extends Thread  {
    volatile int x = 0; //此处可以将volatile去除 或者 替换为 static，经过对比可看出volatile的作用
    
    private void write() {
        x = 5;
    }
    
    private void read() {
        while (x != 5) {}
        if(x == 5){
            System.out.println("------stoped");
        }
    }
    
    public static void main(String[] args) throws Exception {
        Test example = new Test();
        
        Thread writeThread = new Thread(new Runnable() {
            public void run() {
                example.write();
            }
        });

        Thread readThread = new Thread(new Runnable() {
            public void run() {
                example.read();
            }
        });

        readThread.start();
        TimeUnit.SECONDS.sleep(5); //记住此处一定要暂停5秒，以保证writeThread一定会在readThread中执行
        System.out.println("------");
        writeThread.start();
    }
}
```

注：多线程中的非同步问题主要出现在对域的读写上，如果域自身避免这个问题，那么就不需要修改操作该域的方法。用final域，有锁保护的域可以避免非同步的问题。

### 使用重入锁ReentrantLock实现线程同步

jdk1.5中java.util.concurrent包下ReentrantLock类是可重入、互斥、实现了Lock接口的锁。它与synchronized修饰的方法具有相同的基本行为与语义，并且扩展了其能力。 

ReentrantLock类的常用方法有： 

（1）ReentrantLock()：创建一个ReentrantLock实例。 

（2）lock():获得锁 

（3）unlock()：释放锁

上述代码可以修改为：

```java
public class Test extends Thread  {
    private int x = 0; 
    
    /**
      * 使用可重入锁
      */
    private Lock lock=new ReentrantLock();
    
    private void write() {
     lock.lock();
        try {
            x = 5;
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally {
            lock.unlock();//释放锁
        }
    }
    
    private void read() {
     lock.lock();
        try {
            while (x != 5) {}
            if(x == 5){
                System.out.println("------stoped");
            }
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally {
            lock.unlock();//释放锁
        }
        
    }
    
    public static void main(String[] args) throws Exception {
        Test example = new Test();
        
        Thread writeThread = new Thread(new Runnable() {
            public void run() {
                example.write();
            }
        });

        Thread readThread = new Thread(new Runnable() {
            public void run() {
                example.read();
            }
        });

        readThread.start();
        TimeUnit.SECONDS.sleep(5); //记住此处一定要暂停5秒，以保证writeThread一定会在readThread中执行
        System.out.println("------");
        writeThread.start();
    }
}
```

注： 

1. ReentrantLock()还可以通过public ReentrantLock(boolean fair)构造方法创建公平锁，即，优先运行等待时间最长的线程，这样大幅度降低程序运行效率。 

2. 关于Lock对象和synchronized关键字的选择： 

   （1）最好两个都不用，使用一种java.util.concurrent包提供的机制，能够帮助用户处理所有与锁相关的代码。 

   （2）如果synchronized关键字能够满足用户的需求，就用synchronized，他能简化代码。 

   （3）如果需要使用更高级的功能，就用ReentrantLock类，此时要注意及时释放锁，否则会出现死锁，通常在finally中释放锁。

### 使用ThreadLocal管理局部变量实现线程同步 

ThreadLocal管理变量，则每一个使用该变量的线程都获得一个该变量的副本，副本之间相互独立，这样每一个线程都可以随意修改自己的副本，而不会对其他线程产生影响。 

ThreadLocal类常用的方法： 

1. get()：返回该线程局部变量的当前线程副本中的值。 

2. initialValue()：返回此线程局部变量的当前线程的”初始值“。 

3. remove()：移除此线程局部变量当前线程的值。 

4. set(T value)：将此线程局部变量的当前线程副本中的值设置为指定值value。 

上述代码修改为：

```java
 
public class Test extends Thread  {
    private static ThreadLocal<Integer> count=new ThreadLocal<Integer>(){
            @Override
            protected Integer initialValue() {
                  // TODO Auto-generated method stub
                  return 0;
            }
      };
    
    private void write() {
        count.set(count.get()+5);
    }
    
    private void read() {
        while (count.get() != 5) {}
        if(count.get() == 5){
            System.out.println("------stoped");
        }
    }
    
    public static void main(String[] args) throws Exception {
        Test example = new Test();
        
        Thread writeThread = new Thread(new Runnable() {
            public void run() {
                example.write();
            }
        });

        Thread readThread = new Thread(new Runnable() {
            public void run() {
                example.read();
            }
        });

        readThread.start();
        TimeUnit.SECONDS.sleep(5); //记住此处一定要暂停5秒，以保证writeThread一定会在readThread中执行
        System.out.println("------");
        writeThread.start();
    }
}
```