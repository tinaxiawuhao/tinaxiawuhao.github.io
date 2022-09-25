---
title: 'netty_nio_Reactor模式'
date: 2021-09-30 10:14:42
tags: [java,netty]
published: true
hideInList: false
feature: /post-images/wvy4g9Zbn.png
isTop: false
---
Reactor有三种模式：

1. 单reactor单线程工作原理图

![](https://tianxiawuhao.github.io/post-images/1636082432432.jfif)

 

> dispatch与handler在同一个线程中处理. redis就是采用这种模式

 

1. 单reactor多线程工作原理图

![](https://tianxiawuhao.github.io/post-images/1656503549678.png)

 

> （1） reactor对象通过select监控客户端请求事件，收到事件后，通过dispatch进行分发

> （2）如果建立连接请求，则由Acceptor通过accept处理连接请求，然后创建一个Handler对象处理完成连接后的各种事件

> （3）如果不是连接请求，则由reactor分发调用连接对应的Handler来处理

> （4）Handler只负责响应事件，不做具体的业务处理， 通过read读取数据后分发给后面的work线程池中的某个线程。

> （5）work线程池会分配一个独立的线程完成真正的业务 ，并将处理完的业务结果返回给Handler

> （6）Handler收到响应结果后，通过send将结果返回给client

优点：

> （1） 可以充分利用多核cpu的处理能力

> （2） 多线程数据共享和访问比较复杂 ，reactor处理了所有的事件监听和响应，而且是在单线程中运行，在高并发场景容易出现性能瓶颈

 

1. 主从reactor多线程工作原理图

![](https://tianxiawuhao.github.io/post-images/1656503563223.png)

 

>（1）Reactor主线程MainReactor对象通过select监听连接事件，收到连接事件后，通过Acceptor处理连接事件

>（2）当Acceptor处理连接事件后，MainReactor将连接分配给SubReactor, 

>（3）SubReactor将连接加入到连接监听队列进行监听，并创建Handler进行各种事件处理

>（4）当有新的事件发生时， subreactor就会调用对应的handler处理，

>（5）handler通过read读取数据后，分发给后面的worker线程处理

>（6）worker线程池会分配独立的worker线程进行业务处理，并返回结果

>（7）handler收到响应结果后，再通过send将结果返回给client



![](https://tianxiawuhao.github.io/post-images/1632970988216.jpg)
![](https://tianxiawuhao.github.io/post-images/1632970993792.jpg)

实现Reactor模式我们需要实现以下几个类：


**InputSource:** 外部输入类，用来表示需要reactor去处理的原始对象

**Event**: reactor模式的事件类，可以理解为将输入原始对象根据不同状态包装成一个事件类，reactor模式里处理的斗士event事件对象

**EventType**: 枚举类型表示事件的不同类型

**EventHandler**: 处理事件的抽象类，里面包含了不同事件处理器的公共逻辑和公共对象

**AcceptEventHandler\ReadEventhandler等**: 继承自EventHandler的具体事件处理器的实现类，一般根据事件不同的状态来定义不同的处理器

**Dispatcher**: 事件分发器，整个reactor模式解决的主要问题就是在接收到任务后根据分发器快速进行分发给相应的事件处理器，不需要从开始状态就阻塞

**Selector**: 事件轮循选择器，selector主要实现了轮循队列中的事件状态，取出当前能够处理的状态

**Acceptor**:reactor的事件接收类，负责初始化selector和接收缓冲队列

**Server**:负责启动reactor服务并启动相关服务接收请求

**InputSource.java**

```java
import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class InputSource {
    private final Object data;
    private final long id;
}
```

**Event.java**

```java
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class Event {
    private InputSource source;
    private EventType type;
}
```

**EventType**

```java
public enum EventType {
    ACCEPT,
    READ,
    WRITE;
}
```

**EventHandler.java**

```java
@Getter
@Setter
public abstract class EventHandler {

    private InputSource source;
    public abstract void handle(Event event);
}
```

**AcceptEventHandler.java**

```java
public class AcceptEventHandler extends EventHandler {
    private Selector selector;

    public AcceptEventHandler(Selector selector) {
        this.selector = selector;
    }

    @Override
    public void handle(Event event) {
        //处理Accept的event事件
        if (event.getType() == EventType.ACCEPT) {

            //TODO 处理ACCEPT状态的事件

            //将事件状态改为下一个READ状态，并放入selector的缓冲队列中
            Event readEvent = new Event();
            readEvent.setSource(event.getSource());
            readEvent.setType(EventType.READ);

            selector.addEvent(readEvent);
        }
    }
}
```

**Dispatcher.java**

```java
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class Dispatcher {
    //通过ConcurrentHashMap来维护不同事件处理器
    Map<EventType, EventHandler> eventHandlerMap = new ConcurrentHashMap<EventType, EventHandler>();
    //本例只维护一个selector负责事件选择，netty为了保证性能实现了多个selector来保证循环处理性能，不同事件加入不同的selector的事件缓冲队列
    Selector selector;

    Dispatcher(Selector selector) {
        this.selector = selector;
    }

    //在Dispatcher中注册eventHandler
    public void registEventHandler(EventType eventType, EventHandler eventHandler) {
        eventHandlerMap.put(eventType, eventHandler);

    }

    public void removeEventHandler(EventType eventType) {
        eventHandlerMap.remove(eventType);
    }

    public void handleEvents() {
        dispatch();
    }

    //此例只是实现了简单的事件分发给相应的处理器处理，例子中的处理器都是同步，在reactor模式的典型实现NIO中都是在handle异步处理，来保证非阻塞
    private void dispatch() {
        while (true) {
            List<Event> events = selector.select();

            for (Event event : events) {
                EventHandler eventHandler = eventHandlerMap.get(event.getType());
                eventHandler.handle(event);
            }
        }
    }
}
```

**Selector.java**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class Selector {
    //定义一个链表阻塞queue实现缓冲队列，用于保证线程安全
    private final BlockingQueue<Event> eventQueue = new LinkedBlockingQueue<Event>();
    //定义一个object用于synchronize方法块上锁
    private final Object lock = new Object();

    List<Event> select() {
        return select(0);
    }

    //
    List<Event> select(long timeout) {
        if (timeout > 0) {
            if (eventQueue.isEmpty()) {
                synchronized (lock) {
                    if (eventQueue.isEmpty()) {
                        try {
                            lock.wait(timeout);
                        } catch (InterruptedException ignored) {
                        }
                    }
                }

            }
        }
        //TODO 例子中只是简单的将event列表全部返回，可以在此处增加业务逻辑，选出符合条件的event进行返回
        List<Event> events = new ArrayList<Event>();
        eventQueue.drainTo(events);
        return events;
    }

    public void addEvent(Event e) {
        //将event事件加入队列
        boolean success = eventQueue.offer(e);
        if (success) {
            synchronized (lock) {
                //如果有新增事件则对lock对象解锁
                lock.notify();
            }

        }
    }

}
```

**Acceptor.java**

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class Acceptor implements Runnable{
    private final int port; // server socket port
    private final Selector selector;

    // 代表 serversocket，通过LinkedBlockingQueue来模拟外部输入请求队列
    private final BlockingQueue<InputSource> sourceQueue = new LinkedBlockingQueue<InputSource>();

    Acceptor(Selector selector, int port) {
        this.selector = selector;
        this.port = port;
    }

    //外部有输入请求后，需要加入到请求队列中
    public void addNewConnection(InputSource source) {
        sourceQueue.offer(source);
    }

    public int getPort() {
        return this.port;
    }

    public void run() {
        while (true) {

            InputSource source = null;
            try {
                // 相当于 serversocket.accept()，接收输入请求，该例从请求队列中获取输入请求
                source = sourceQueue.take();
            } catch (InterruptedException e) {
                // ignore it;
            }

            //接收到InputSource后将接收到event设置type为ACCEPT，并将source赋值给event
            if (source != null) {
                Event acceptEvent = new Event();
                acceptEvent.setSource(source);
                acceptEvent.setType(EventType.ACCEPT);

                selector.addEvent(acceptEvent);
            }

        }
    }
}
```

**Server.java**

```java
public class Server {
    Selector selector = new Selector();
    Dispatcher eventLooper = new Dispatcher(selector);
    Acceptor acceptor;

    Server(int port) {
        acceptor = new Acceptor(selector, port);
    }

    public void start() {
        eventLooper.registEventHandler(EventType.ACCEPT, new AcceptEventHandler(selector));
        new Thread(acceptor, "Acceptor-" + acceptor.getPort()).start();
        eventLooper.handleEvents();
    }
}
```