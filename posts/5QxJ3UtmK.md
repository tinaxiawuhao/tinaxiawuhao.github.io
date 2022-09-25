---
title: '手写一个生产者消费者模式'
date: 2021-09-08 15:46:13
tags: [java]
published: true
hideInList: false
feature: /post-images/5QxJ3UtmK.png
isTop: false
---
```java
import lombok.SneakyThrows;
import org.springframework.util.StringUtils;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

class MyResource {
    private volatile Boolean FLAG = true;
    private final AtomicInteger atomicInteger = new AtomicInteger();
    BlockingQueue<String> blockingQueue = null;

    public MyResource(BlockingQueue<String> blockingQueue) {
        this.blockingQueue = blockingQueue;
        System.out.println(blockingQueue.getClass().getName());
    }

    @SneakyThrows
    public void myProd() {
        String data = null;
        while (FLAG) {
            data = String.valueOf(atomicInteger.incrementAndGet());
            if (blockingQueue.offer(data, 2L, TimeUnit.SECONDS)) {
                System.out.println(Thread.currentThread().getName() + "插入数据" + data + "成功");
            } else {
                System.out.println(Thread.currentThread().getName() + "插入数据" + data + "失败");
            }
            TimeUnit.SECONDS.sleep(1L);
        }
        System.out.println(Thread.currentThread().getName() + "生产停止");
    }

    @SneakyThrows
    public void myConsumer() {
        String result = null;
        while (FLAG) {
            result = blockingQueue.poll(2L, TimeUnit.SECONDS);
            if (StringUtils.isEmpty(result)) {
                FLAG = false;
                System.out.println(Thread.currentThread().getName() + "超过2秒没有取到值，结束");
                return;
            }
            System.out.println(Thread.currentThread().getName() + "消费数据" + result + "成功");
        }
    }

    public void stop(){
        FLAG=false;
    }
}

public class ProdConsumer_BlockQueue {

    @SneakyThrows
    public static void main(String[] args) {
        MyResource myResource = new MyResource(new ArrayBlockingQueue<>(10));
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "生产线程启动");
            myResource.myProd();
        }, "Prod").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "消费线程启动");
            myResource.myConsumer();
        }, "Consumer").start();

        TimeUnit.SECONDS.sleep(5L);
        System.out.println("5秒时间到，结束");
        myResource.stop();
    }


}
```
![](https://tinaxiawuhao.github.io/post-images/1631087515332.png)
