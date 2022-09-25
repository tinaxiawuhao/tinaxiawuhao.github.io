---
title: 'netty'
date: 2022-06-26 19:54:54
tags: [netty]
published: true
hideInList: false
feature: /post-images/Hbra5M-xo.png
isTop: false
---
### 1. Netty简单介绍
#### 1. 原生NIO存在的问题
为什么有了 NIO ，还会出现 Netty，因为 NIO 有如下问题：
NIO 的类库和 API 繁杂，使用麻烦：需要熟练掌握 Selector、ServerSocketChannel、 SocketChannel、ByteBuffer 等。
需要具备其他的额外技能：要熟悉 Java 多线程编程，因为 NIO 编程涉及到 Reactor 模式，你必须对多线程和网络编程非常熟悉，才能编写出高质量的 NIO 程序。
开发工作量和难度都非常大：例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常流的处理等等。
JDK NIO 的 Bug：例如臭名昭著的 Epoll Bug，它会导致 Selector 空轮询（死循环），最终导致 CPU 100%。直到 JDK 1.7 版本该问题仍旧存在，没有被根本解决

#### 2. Netty官网说明
官网地址 ：https://netty.io/
Netty是一个异步事件驱动的网络应用程序框架，用于快速开发可维护的高性能的服务器和客户端。
Netty 是由 JBOSS 提供的一个 Java 开源框架。Netty 提供异步的、基于事件驱动的网络应用程序框架，用以快速开发高性能、高可靠性的网络 IO 程序。
Netty 可以帮助你快速、简单的开发出一个网络应用，相当于简化和流程化了 NIO 的 开发过程。
Netty 是目前最流行的 NIO 框架，Netty 在互联网领域、大数据分布式计算领域、游戏行业、通信行业等获得了广泛的应用，知名的 Elasticsearch 、Dubbo 框架内部都采 用了 Netty。

![](https://tianxiawuhao.github.io/post-images/1656503871885.png)

#### 3. Netty的优点
Netty 对 JDK 自带的 NIO 的 API 进行了封装，解决了上述问题。

设计优雅：适用于各种传输类型的统一 API 阻塞和非阻塞 Socket；基于灵活且可扩展的事件模型，可以清晰地分离关注点；高度可定制的线程模型 —— 单线程，一个或多个 线程池.
使用方便：详细记录的 Javadoc，用户指南和示例；没有其他依赖项 (JDK 5 -> Netty 3.x ; 6 -> Netty 4.x 就可以支持了）。
高性能、吞吐量更高：延迟更低；减少资源消耗；最小化不必要的内存复制。
安全：完整的 SSL/TLS 和 StartTLS 支持。
社区活跃、不断更新：社区活跃，版本迭代周期短，发现的 Bug 可以被及时修复， 同时，更多的新功能会被加入

#### 4. Netty版本说明
netty版本分为 netty3.x 和 netty4.x、netty5.x
因为Netty5出现重大bug，已经被官网废弃了，目前推荐使用的是Netty4.x的稳定版本
目前在官网可下载的版本 netty3.x netty4.0.x 和 netty4.1.x
这里使用的是 Netty4.1.x 版本
netty 下载地址： https://bintray.com/netty/downloads/netty/

### 2. 各线程模式
#### 1. 传统阻塞 I/O 服务模型

![](https://tianxiawuhao.github.io/post-images/1656503894532.png)

**模型特点**
采用阻塞IO模式获取输入的数据
每个连接都需要独立的线程完成数据的输入（read），业务处理， 数据返回（send）
**问题分析**
当并发数很大，就会创建大量的线程，占用很大系统资源
连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read 操作，造成线程资源浪费

#### 2. Reactor 模式
针对传统阻塞 I/O 服务模型的 2 个缺点，解决方案：

`基于 I/O 复用模型`：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理

`基于线程池复用线程资源`：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务。

> Reactor 对应的叫法:
> 反应器模式
> 分发者模式(Dispatcher)
> 通知者模式(notifier)

![](https://tianxiawuhao.github.io/post-images/1656503912514.png)

**说明:**
Reactor 模式，通过一个或多个输入同时传递给服务处理器的模式 (基于事件驱动)
服务器端程序处理传入的多个请求, 并将它们同步分派到相应的处理线程， 因此Reactor（反应器）模式也叫 Dispatcher（分发者） 模式
Reactor 模式使用IO复用监听事件, 收到事件后，分发给某个线程(进程), 这点就是网络服务器高并发处理关键

**核心组成：**
Reactor：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处 理程序来对 IO 事件做出反应。 它就像公司的电话接线员，它接听来自客户的电话并 将线路转移到适当的联系人；
Handlers：处理程序执行 I/O 事件要完成的实际事件，类似于客户想要与之交谈的公司中的实际官员。Reactor 通过调度适当的处理程序来响应 I/O 事件，处理程序执行非阻塞操作

**Reactor 模式分类：**

> 单 Reactor 单线程
> 单 Reactor 多线程
> 主从 Reactor 多线程

#### 3. 单Reactor-单线程

![](https://tianxiawuhao.github.io/post-images/1656503926332.png)

**说明：**
Select 是前面 I/O 复用模型介绍的标准网络编程 API，可以实现应用程序通过一个阻塞对象监听多路连接请求
Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发
如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理
如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应
Handler 会完成 Read→业务处理→Send 的完整业务流程 结合实例：服务器端用一个线程通过多路复用搞定所有的 IO 操作（包括连接，读、写 等），编码简单，清晰明了，但是如果客户端连接数量较多，将无法支撑，前面的 NIO 案例就属于这种模型。
**优点：**
模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成
**缺点：**
性能问题，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某 个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈
可靠性问题，线程意外终止，或者进入死循环，会导致整个系统通信模块不 可用，不能接收和处理外部消息，造成节点故障
使用场景：客户端的数量有限，业务处理非常快速，比如 Redis在业务处理的时间复 杂度 O(1) 的情况

#### 4. 单 Reactor 多线程

![](https://tianxiawuhao.github.io/post-images/1656503940131.png)

**说明：**
Reactor 对象通过 select 监控客户端请求事件, 收到事件后，通过 dispatch 进行分发
如果建立连接请求, 则由 Acceptor 通过 accept 处理连接请求, 然后创建一个Handler对象处理完成连接后的各种事件
如果不是连接请求，则由 Reactor 分发调用连接对 应的Handler 来处理
Handler 只负责响应事件，不做具体的业务处理, 通过 read 读取数据后，会分发给后面的 Worker 线程池的某个线程处理业务
Worker 线程池会分配独立线程完成真正的业务， 并将结果返回给 Handler
Handler 收到响应后，通过 send 将结果返回给 Client

**优点**：可以充分的利用多核cpu 的处理能力
**缺点**：多线程数据共享和访问比较复杂,单 Reactor 处理所有的事件的监听和响应，在单线程运行， 在高并发场景容易出现性能瓶颈

#### 5. 主从 Reactor 多线程

![](https://tianxiawuhao.github.io/post-images/1656503952682.png)

**说明**

> 1: Reactor 主线程 MainReactor 对象通过 select 监听连接事件, 收到事件后，通过 Acceptor 处理连接事件(主 Reactor 只处理连接事件)
> 2：当 Acceptor 处理连接事件后，MainReactor 将连接分配给 SubReactor
> 3：SubReactor 将连接加入到连接队列进行监听,并创建 Handler 进行各种事件处理
> 4：当有新事件发生时， SubReactor 就会调用对应的 Handler处理，Handler 通过 read 读取数据，分发给后面的 （Worker 线程池）处理
> 5：（Worker 线程池）分配独立的 （Worker 线程）进行业务处理，并返 回结果
> 6：Handler 收到响应的结果后，再通过 send 将结果返回给 Client
> ps：一个 MainReactor 可以关联多个 SubReactor

![](https://tianxiawuhao.github.io/post-images/1656503963761.png)

**优点**： 父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。
父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据。
**缺点**：编程复杂度较高
结合实例：这种模型在许多项目中广泛使用，包括 Nginx 主从 Reactor 多进程模型， Memcached 主从多线程，Netty 主从多线程模型的支持

#### 6. Reactor 模式小结
3 种模式用生活案例来理解
单 Reactor 单线程，前台接待员和服务员是同一个人，全程为顾客服
单 Reactor 多线程，1 个前台接待员，多个服务员，接待员只负责接待
主从 Reactor 多线程，多个前台接待员，多个服务生
Reactor 模式具有如下的优点：
响应快，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的
可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程 的切换开销
扩展性好，可以方便的通过增加 Reactor 实例个数来充分利用 CPU 资源
复用性好，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性

### 3. Netty模型
#### 1. 简单版

![](https://tianxiawuhao.github.io/post-images/1656503977148.png)

**说明 ：**

> BossGroup 线程维护Selector , 只关注Accecpt
> 当接收到Accept事件，获取到对应的 SocketChannel, 封装成 NIOScoketChannel并注册到 Worker 线程(事件循环), 并进行维护
> 当Worker线程监听到 Selector 中通道发生自己感 兴趣的事件后，就进行处理(就由 Handler 处理)， 注意 Handler 已经加入到通道

#### 2. 进阶版

![](https://tianxiawuhao.github.io/post-images/1656503987315.png)

#### 3. 完整版 非常重要

![](https://tianxiawuhao.github.io/post-images/1656503997684.jpg)

**说明 ：**

> 1：Netty 抽象出两组线程池 BossGroup 专门负责接收客户端的连接, WorkerGroup 专门负责网络的读写
> 2：BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup
> 3：NioEventLoopGroup 相当于一个事件循环组, 这个组中含有多个事件循环 ，每一个事件循环是 NioEventLoop
> 4：NioEventLoop 表示一个不断循环的执行处理任务的线程， 每个 NioEventLoop 都有一个 Selector , 用于监听绑定在其上的 Socket 的网络通讯
> 5:NioEventLoopGroup(BossGroup、WorkerGroup) 可以有多个线程, 即可以含有多个 NioEventLoop
> 6:每个Boss 的 NioEventLoop 循环执行的步骤有3步
> (1):轮询accept 事件
> (2):处理accept 事件 , 与client建立连接 , 生成NioScocketChannel , 并将其注册到 Worker 的 (3):NIOEventLoop 上的 Selector
> 处理任务队列的任务 ， 即 runAllTasks
> 7：每个 Worker 的 NIOEventLoop 循环执行的步骤
> (1):轮询read, write 事件
> (2):处理i/o事件， 即read , write 事件，在对应NioScocketChannel 处理
> (3):处理任务队列的任务 ， 即 runAllTasks
> 每个Worker NIOEventLoop 处理业务时，会使用 Pipeline(管道), Pipeline 中包含了 Channel , 即通过 Pipeline 可以获取到对应通道, 管道中维护了很多的处理器。管道可以使用 Netty 提供的，也可以自定义

#### 4. 理解重制版

![](https://tianxiawuhao.github.io/post-images/1656504035283.png)


服务器端首先创建一个ServerSocketChannel，bossGroup只处理客户端连接请求,workGroup处理读写事件，此二者为线程组，其中每一个NIOEventLoop都是线程组其中一个线程

客户端发送连接请求通过ServerSocketChannel被NIOLoopEventGroup线程组中的一个线程NIOLoopEvent的selector选择器监听，并将事件放入taskqueue队列进行轮询，一旦有accept事件就会封装NIOSocketChannel对象，并通过其去注册到workGroup的线程的selector中让其监听

一旦workGroup的线程的taskqueue轮询到读写事件，在对应的NIOSocketChannel进行处理

### 4. Netty实例分析
#### 1. BossGroup 和 WorkGroup 怎么确定自己有多少个 NIOEventLoop
BossGroup 和 WorkerGroup 含有的子线程数（NioEventLoop）默认为 CPU 核数*2

由源码中的构造方法可知 —— 想要设置线程数只要在参数中输入即可

#### 2. WorkerGroup 是如何分配这些进程的
设置 BossGroup 进程数为 1 ； WorkerGroup 进程数为 4 ； Client 数位 8

在默认情况下，WorkerGroup 分配的逻辑就是按顺序循环分配的

#### 3. BossGroup 和 WorkerGroup 中的 Selector 和 TaskQueue
打断点进行 Debug

![](https://tianxiawuhao.github.io/post-images/1656504060284.png)

![](https://tianxiawuhao.github.io/post-images/1656504081513.png)

**每个子线程都具有自己的 Selector、TaskQueue……**
#### 4. CTX 上下文、Channel、Pipeline 之间关系
修改 `NettyServerHandler` ，并添加端点

![](https://tianxiawuhao.github.io/post-images/1656504102104.png)

先看 CTX 上下文中的信息

![](https://tianxiawuhao.github.io/post-images/1656504116118.png)

Pipeline

![](https://tianxiawuhao.github.io/post-images/1656504129630.png)

Channel

![](https://tianxiawuhao.github.io/post-images/1656504144493.png)

CTX 上下文、Channel、Pipeline 三者关系示意图

![](https://tianxiawuhao.github.io/post-images/1656504158613.png)

#### 5. 设置通道参数
**childOption() 方法**

给每条child channel 连接设置一些TCP底层相关的属性，比如上面，我们设置了两种TCP属性，其中 ChannelOption.SO_KEEPALIVE表示是否开启TCP底层心跳机制，true为开

**option() 方法**

对于server bootstrap而言，这个方法，是给parent channel 连接设置一些TCP底层相关的属性。

TCP连接的参数详细介绍如下。SO_RCVBUF ，SO_SNDBUF

这两个选项就是来设置TCP连接的两个buffer尺寸的。

每个TCP socket在内核中都有一个发送缓冲区和一个接收缓冲区，TCP的全双工的工作模式以及TCP的滑动窗口便是依赖于这两个独立的buffer以及此buffer的填充状态。

**SO_SNDBUF**
  Socket参数，TCP数据发送缓冲区大小。该缓冲区即TCP发送滑动窗口，linux操作系统可使用命令：cat /proc/sys/net/ipv4/tcp_smem 查询其大小。

**TCP_NODELAY**
  TCP参数，立即发送数据，默认值为Ture（Netty默认为True而操作系统默认为False）。该值设置Nagle算法的启用，改算法将小的碎片数据连接成更大的报文来最小化所发送的报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。Netty默认禁用该算法，从而最小化报文传输延时。

这个参数，与是否开启Nagle算法是反着来的，true表示关闭，false表示开启。通俗地说，如果要求高实时性，有数据发送时就马上发送，就关闭，如果需要减少发送次数减少网络交互，就开启。

**SO_KEEPALIVE**
  底层TCP协议的心跳机制。Socket参数，连接保活，默认值为False。启用该功能时，TCP会主动探测空闲连接的有效性。可以将此功能视为TCP的心跳机制，需要注意的是：默认的心跳间隔是7200s即2小时。Netty默认关闭该功能。

**SO_REUSEADDR**
  Socket参数，地址复用，默认值False。有四种情况可以使用：
(1).当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而你希望启动的程序的socket2要占用该地址和端口，比如重启服务且保持先前端口。
(2).有多块网卡或用IP Alias技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同。
(3).单个进程绑定相同的端口到多个socket上，但每个socket绑定的ip地址不同。(4).完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP。

**SO_LINGER**
  Socket参数，关闭Socket的延迟时间，默认值为-1，表示禁用该功能。-1表示socket.close()方法立即返回，但OS底层会将发送缓冲区全部发送到对端。0表示socket.close()方法立即返回，OS放弃发送缓冲区的数据直接向对端发送RST包，对端收到复位错误。非0整数值表示调用socket.close()方法的线程被阻塞直到延迟时间到或发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。

**SO_BACKLOG**
  Socket参数，服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值，Windows为200，其他为128。

```java
 b.option(ChannelOption.SO_BACKLOG, 1024) 
```


表示系统用于临时存放已完成三次握手的请求的队列的最大长度，如果连接建立频繁，服务器处理创建新连接较慢，可以适当调大这个参数.

**SO_BROADCAST**
  Socket参数，设置广播模式。

### 5. TaskQueue 任务队列
任务队列中的 Task 有 3 种典型使用场景

> 用户程序自定义的普通任务
> 用户自定义定时任务
> 非当前 Reactor 线程调用 Channel 的各种方法

### 6. 异步模型
#### 1. 工作示意图

![](https://tianxiawuhao.github.io/post-images/1656504197703.png)

**说明**

1:在使用 Netty 进行编程时，拦截操作和转换出入站数据只需要您提供 callback 或利用 future 即可。这使得链式操作简单、高效, 并有利于编写可重用的、通用的代码。

2:Netty 框架的目标就是让你的业务逻辑从网络基础应用编码中分离出来

#### 2. Future-Listener 机制
当 Future 对象刚刚创建时，处于非完成状态，调用者可以通过返回的 ChannelFuture 来获取操作执行的状态，注册监听函数来执行完成后的操作。
常见有如下操作
• 通过 isDone 方法来判断当前操作是否完成；
• 通过 isSuccess 方法来判断已完成的当前操作是否成功；
• 通过 getCause 方法来获取已完成的当前操作失败的原因；
• 通过 isCancelled 方法来判断已完成的当前操作是否被取消；
• 通过 addListener 方法来注册监听器，当操作已完成(isDone 方法返回完成)，将会通知 指定的监听器；如果 Future 对象已完成，则通知指定的监听器

代码示例

```java
给一个 ChannelFuture 注册监听器，来监控我们关系的事件

channelFuture.addListener(new ChannelFutureListener() {
     @Override
     public void operationComplete(ChannelFuture channelFuture) throws Exception {
          if (channelFuture.isSuccess()){
               System.out.println("监听端口 6668 成功");
          }else {
               System.out.println("监听端口 6668 失败");
          }
      }
});
```


#### 3. 快速入门实例-HTTP服务
Netty 可以做Http服务开发，并且理解Handler实例 和客户端及其请求的关系

**编写代码 —— 服务端代码**

```java
# 编写服务端： HttpServer
public static void main(String[] args) throws Exception {
    //创建BossGroup ,workGroup
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup,workGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new TestServerInitializer());
        ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
        channelFuture.channel().closeFuture().sync();
    } finally {
        bossGroup.shutdownGracefully();
        workGroup.shutdownGracefully();
    }
}
```

**编写 服务初始化器 ：HttpServerInitialize**

```java
public class TestServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        //向管道加入处理器
        ChannelPipeline pipeline = socketChannel.pipeline();
        //加入netty提供的httpServerCodec,netty提供的处理Http的编-解码器
        pipeline.addLast("MyHttpServerCodec",new HttpServerCodec());
        pipeline.addLast("MyTestHttpServerHandler",new TestHttpServerHandler());
    }
}
```

**编写 服务处理器 ：HttpServerHandler**

```java
/**
*
* 1. SimpleChannelInboundHandler 是之前使用的 ChannelInboundHandlerAdapter 的子类
* 2. HttpObject 这个类型表示， 客户端、服务端 相互通信的数据需要被封装成什么类型
*
*/
public class TestHttpServerHandler extends SimpleChannelInboundHandler<HttpObject> {
	/**
    * 读取客户端数据
    * @param ctx
    * @param msg
    * @throws Exception
    */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HttpObject msg) throws Exception {
        //判断msg 是不是HttpRequest 请求
        if(msg instanceof HttpRequest){
        System.out.println("msg 类型 = "+msg.getClass());
        System.out.println("客户端地址"+ctx.channel().remoteAddress());
        // 获取请求的 URI
        HttpRequest httpRequest = (HttpRequest) msg;
        URI uri = new URI(httpRequest.uri());
        // 判断请求路径为 /favicon.ico，就不做处理
        if ("/favicon.ico".equals(uri.getPath())){
        System.out.println("请求了 图标 资源，不做响应");
        	return;
        }
        //回复信息给浏览器【HTTP协议】
        ByteBuf content = Unpooled.copiedBuffer("Hello I am server", CharsetUtil.UTF_8);
        //构造一个http回应，即httpResponse,HttpResponseStatus:状态码
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,content);
        response.headers().set(HttpHeaderNames.CONTENT_TYPE,"text/plain");
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH,content.readableBytes());
        //将构建好的response返回
        ctx.writeAndFlush(response);
    }
}
```

**编写代码 —— 对特定资源的过滤**
      上面的服务端启动后，在页面上不止接收到了文本，还接收到了一个网页的图标 

![](https://tianxiawuhao.github.io/post-images/1656504226962.png)

现在把它过滤掉

```java
// 修改 HttpServerHandler
 // 获取请求的 URI
HttpRequest httpRequest = (HttpRequest) msg;
URI uri = new URI(httpRequest.uri());
// 判断请求路径为 /favicon.ico，就不做处理
if ("/favicon.ico".equals(uri.getPath())){
    System.out.println("请求了 图标 资源，不做响应");
    return;
}
```


### 7. netty核心组件
#### 1. Bootstrap 和 ServerBootstrap
Bootstrap 意思是引导，一个 Netty 应用通常由一个 Bootstrap 开始，主要作用是配置整个 Netty 程序，串联各个组件，Netty 中 Bootstrap类是客户端程序的启动引导类， ServerBootstrap是服务端启动引导类
常见的方法有

> • public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup)，该方法用于服务器端，用来设置两个 EventLoop
> • public B channel(Class<? extends C> channelClass)，该方法用来设置一个服务器端的通道实现
> • public ChannelFuture bind(int inetPort) ，该方法用于服务器端，用来设置占用的端口号
> • public B option(ChannelOption option, T value)，用来给 ServerChannel 添加配置
> • public B group(EventLoopGroup group) ，该方法用于客户端，用来设置一个 EventLoop
> • public ChannelFuture connect(String inetHost, int inetPort) ，该方法用于客户端，用来连接服务器 端
> • public ServerBootstrap childOption(ChannelOption childOption, T value)，用来给接收到的通道添加配置
> • public ServerBootstrap childHandler(ChannelHandler childHandler)，该方法用来设置业务处理类 （自定义的 handler）

```
.childHandler(new TestServerInitializer())//对应workGroup
.handler(null)//对应bossGroup
```


#### 2. Future 和 ChannelFuture
Netty 中所有的 IO 操作都是异步的，不能立刻得知消息是否被正确处理。但是可以过一会等它执行完成或者直接注册一个监听，具体的实现就是通过 Future 和 ChannelFutures，他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件
常见的方法有

> • Channel channel()，返回当前正在进行 IO 操作的通道
> • ChannelFuture sync()，等待异步操作执行完毕

#### 3. Channel
Netty 网络通信的组件，能够用于执行网络 I/O 操作。
通过Channel 可获得当前网络连接的通道的状态
通过Channel 可获得网络连接的配置参数 （例如接收缓冲区大小）
Channel 提供异步的网络 I/O 操作(如建立连接，读写，绑定端口)，异步调用意味着任何 I/O 调用都将立即返回，并且不保证在调用结束时所请求的 I/O 操作已完成
调用立即返回一个 ChannelFuture 实例，通过注册监听器到 ChannelFuture 上，可以 I/O 操作成功、失败或取消时回调通知调用方
支持关联 I/O 操作与对应的处理程序
不同协议、不同的阻塞类型的连接都有不同的 Channel 类型与之对应

常用的 Channel 类型

> • NioSocketChannel，异步的客户端 TCP Socket 连接。
> • NioServerSocketChannel，异步的服务器端 TCP Socket 连接。
> • NioDatagramChannel，异步的 UDP 连接。
> • NioSctpChannel，异步的客户端 Sctp 连接。
> • NioSctpServerChannel，异步的 Sctp 服务器端连接，这些通道涵盖了 UDP 和 TCP 网络 IO 以及文件 IO。

#### 4. Selector
Netty 基于 Selector 对象实现 I/O 多路复用，通过 Selector 一个线程可以监听多个连接的 Channel 事件。
当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以自动不断地查询 (Select) 这些注册的 Channel 是否有已就绪的 I/O 事件（例如可读，可写，网络连接 完成等），这样程序就可以很简单地使用一个线程高效地管理多个 Channel

#### 5. ChannelHandler
我们经常需要自定义一 个 Handler 类去继承 ChannelInboundHandlerA dapter，然后通过重写相应方法实现业务逻辑

常用的方法

```java
public class ChannelInboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelInboundHandler { 
	// 通道注册事件
	public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelRegistered();
    }
	// 通道注销事件
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelUnregistered();
    }
	// 通道就绪事件 
	public void channelActive(ChannelHandlerContext ctx) throws Exception { 
		ctx.fireChannelActive(); 
	}
	// 通道读取数据事件 
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception { 
		ctx.fireChannelRead(msg); 
	}
	// 通道读取数据完毕事件
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelReadComplete();
    }
    // 通道发生异常事件
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.fireExceptionCaught(cause);
    }
}
```


#### 6. Pipeline 和 ChannelPipeline
`ChannelPipeline` 是一个 Handler 的集合，它负责处理和拦截 inbound（入栈） 或者 outbound（出栈） 的事件和操作，相当于一个贯穿 Netty 的链。(也可以这样理解： ChannelPipeline 是 保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站 和出站 事件 / 操作)

ChannelPipeline 实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个的 ChannelHandler 如何相互交互

在 Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应，它们的组成关系如下

![](https://tianxiawuhao.github.io/post-images/1656504246343.png)

**说明 ：**

> • 一个 Channel 包含了一个 ChannelPipeline，而 ChannelPipeline 中又维护了一个由 ChannelHandlerContext 组成的双向链表，并且每个 ChannelHandlerContext 中又关联着一个 ChannelHandler
> • 入站事件和出站事件在一个双向链表中，入站事件会从链表 head 往后传递到最后一个入站的 handler， 出站事件会从链表 tail 往前传递到最前一个出站的 handler，两种类型的 handler 互不干扰

**常用方法**

```
//把一个业务处理类（handler） 添加到链中的第一个位置
ChannelPipeline addFirst(ChannelHandler… handlers)
//把一个业务处理类（handler） 添加到链中的最后一个位置
ChannelPipeline addLast(ChannelHandler… handlers)
```


### 8. Netty 群聊
**要求：**

> 编写一个 Netty 群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）
> 实现多人群聊
> 服务器端：可以监测用户上线，离线，并实现消息转发功能
> 客户端：通过channel 可以无阻塞发送消息给其它所有用户，同时可以接受其它用 户发送的消息(有服务器转发得到)

**1. server端**

```java
public class ChatServer {
    private int port;

    public ChatServer(int port) {
        this.port = port;
    }

    public void run() throws Exception{
        //创建bossGroup,workGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            //创建辅助工具
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            //循环事件组
            serverBootstrap.group(bossGroup,workGroup)//线程组
                    .channel(NioServerSocketChannel.class)//通道类型
                    .option(ChannelOption.SO_BACKLOG,128)
                    .childOption(ChannelOption.SO_KEEPALIVE,true)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            //加入解码器
                            pipeline.addLast("decoder",new StringDecoder());
                            //加入编码器
                            pipeline.addLast("encoder",new StringEncoder());
                            pipeline.addLast(new ChatServerHandler());

                        }
                    });
            System.out.println("server is ok");
            ChannelFuture channelFuture = serverBootstrap.bind(port).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
    public static void main(String[] args) throws Exception {
       new ChatServer(8080).run();
    }
}
```

**2. serverHandler**



```java
public class ChatServerHandler extends SimpleChannelInboundHandler<String> {
    /**
     * 定义一个 Channel 线程组，管理所有的 Channel, 参数 执行器
     *  GlobalEventExecutor => 全局事件执行器
     *  INSTANCE => 表示是单例的
     */
    private  static ChannelGroup channelGroup = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    /**
     * 当连接建立之后，第一个被执行
     * 一连接成功，就把当前的 Channel 加入到 ChannelGroup，并将上线消息推送给其他客户
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        // 将该客户上线的信息，推送给其他在线的 客户端
        // 该方法，会将 ChannelGroup 中所有的 Channel 遍历，并发送消息
        Date date = new Date(System.currentTimeMillis());
        channelGroup.writeAndFlush("[client]" +channel.remoteAddress()+"["+simpleDateFormat.format(date)+"]"+"加入聊天");
        channelGroup.add(channel);

    }

    /**
     * 当断开连接激活，将 XXX 退出群聊消息推送给当前在线的客户
     * 当某个 Channel 执行到这个方法，会自动从 ChannelGroup 中移除
     */
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        Date date = new Date(System.currentTimeMillis());
        channelGroup.writeAndFlush("[client]"+channel.remoteAddress()+"["+simpleDateFormat.format(date)+"]"+"离开聊天");
        //channelGroup.remove(channel); 不需要，handlerRemoved（）直接删除了channel
    }

    /**
     * 提示客户端离线状态
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Date date = new Date(System.currentTimeMillis());
        System.out.println(ctx.channel().remoteAddress()+"["+simpleDateFormat.format(date)+"]"+"下线了");
    }

    /**
     * 提示客户端上线状态
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Date date = new Date(System.currentTimeMillis());
        System.out.println(ctx.channel().remoteAddress()+"["+simpleDateFormat.format(date)+"]"+"上线了");;
    }

    /**
     * 读取消息，转发数据
     * @param ctx
     * @param msg
     * @throws Exception
     */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        Channel channel = ctx.channel();
        Date date = new Date(System.currentTimeMillis());
        //遍历，根据不同对象发送不同数据
        channelGroup.forEach(ch ->{
            if(channel != ch){//不是自己的channel
                ch.writeAndFlush("[客户]"+channel.remoteAddress()+"["+simpleDateFormat.format(date)+"]"+"发送"+msg+"/n");
            }
            else{
                ch.writeAndFlush("[自己]发送了消息"+"["+simpleDateFormat.format(date)+"]"+msg+"/n");
            }
        }

        );
    }

    /**
     * 异常处理
     * @param ctx
     * @param cause
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```


**3. client端**

```java
public class ChatClient {
    private final String HOST;
    private final int PORT;
    public ChatClient(String host, int port) {
        HOST = host;
        PORT = port;
    }

    public void run() throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            //加入解码器
                            pipeline.addLast("decoder",new StringDecoder());
                            //加入编码器
                            pipeline.addLast("encoder",new StringEncoder());
                            pipeline.addLast(new ChatClientHandler());
                        }
                    });

            ChannelFuture channelFuture = bootstrap.connect(HOST, PORT).sync();
            System.out.println("client prepare is ok");
            Channel channel = channelFuture.channel();
            //客户端需要输入信息，定义扫描器
            Scanner scanner = new Scanner(System.in);
            while(scanner.hasNextLine()){
                String s = scanner.nextLine();
                channel.writeAndFlush(s);

            }
            channel.closeFuture().sync();
        } finally {
            eventLoopGroup.shutdownGracefully();
        }

    }
    public static void main(String[] args) throws InterruptedException {
        new ChatClient("localhost",8080).run();
    }
}
```

**4. clientHandler**

```
public class ChatClientHandler extends SimpleChannelInboundHandler<String> {
    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, String s) throws Exception {
        // 直接输出从服务端获得的信息
        System.out.println(s.trim());
    }
}
```


### 9. 心跳机制
**1. server端：**

```java
public class MyServer {
    public static void main(String[] args) throws Exception{
        //创建bossGroup,workGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        try {
            //创建辅助工具
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            //循环事件组
            serverBootstrap.group(bossGroup,workGroup)//线程组
                .channel(NioServerSocketChannel.class)//通道类型
                .handler(new LoggingHandler(LogLevel.INFO))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        ChannelPipeline pipeline = socketChannel.pipeline();
                        /*
                 说明：
                 1. IdleStateHandler 是 Netty 提供的 空闲状态处理器
                 2. 四个参数：
                 readerIdleTime : 表示多久没有 读 事件后，就会发送一个心跳检测包，检测是否还是连接状态
                 writerIdleTime : 表示多久没有 写 事件后，就会发送一个心跳检测包，检测是否还是连接状态
                 allIdleTime : 表示多久时间既没读也没写 后，就会发送一个心跳检测包，检测是否还是连接状态
                 TimeUnit : 时间单位
                 1. 当 Channel 一段时间内没有执行 读 / 写 / 读写 事件后，就会触发一个 IdleStateEvent 空闲状态事件
                 2. 当 IdleStateEvent 触发后，就会传递给 Pipeline 中的下一个 Handler 去处理，通过回调下一个 Handler 的 userEventTriggered 方法，在该方法中处理 IdleStateEvent
                             */
                        pipeline.addLast(new IdleStateHandler(3,5,7, TimeUnit.SECONDS));
                        pipeline.addLast(new MyServerHandler());
                    }
                });
            System.out.println("server is ok");
            ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

}
```

**2. handler**

```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {
    /**
     * @param ctx 上下文
      @param evt 事件
        * @throws Exception
     */
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if(evt instanceof IdleStateEvent){
            //将 evt 向下转型 IdleStateEvent
            IdleStateEvent event =  (IdleStateEvent)evt;
            String eventType = null;
            //IdleStateEvent 枚举
            switch (event.state()){
                case READER_IDLE:
                    eventType = "读空闲";
                    break;
                case WRITER_IDLE:
                    eventType = "写空闲";
                    break;
                case ALL_IDLE:
                    eventType = "读写空闲";
            }
            System.out.println(ctx.channel().remoteAddress()+"---超时事件---"+eventType);
        }
    }
}
```


**说明：**

> 1: IdleStateHandler 是 Netty 提供的 空闲状态处理器
>
> 2: 四个参数：
> readerIdleTime : 表示多久没有 读 事件后，就会发送一个心跳检测包，检测是否还是连接状态
> writerIdleTime : 表示多久没有 写 事件后，就会发送一个心跳检测包，检测是否还是连接状态
> allIdleTime : 表示多久时间既没读也没写 后，就会发送一个心跳检测包，检测是否还是连接状态
> TimeUnit : 时间单位
> 3: 当 Channel 一段时间内没有执行 读 / 写 / 读写 事件后，就会触发一个 IdleStateEvent 空闲状态事件
>
> 4: 当 IdleStateEvent 触发后，就会传递给 Pipeline 中的下一个 Handler 去处理，通过回调下一个 Handler 的 userEventTriggered 方法，在该方法中处理 IdleStateEvent

### 10. 长连接
**要求：**
实现基于webSocket的长连接 的全双工的交互
改变Http协议多次请求的约束，实 现长连接了， 服务器可以发送消息 给浏览器
客户端浏览器和服务器端会相互感 知，比如服务器关闭了，浏览器会 感知，同样浏览器关闭了，服务器 会感知

代码实现

**服务端 ：WebServer**

```java
public class MyServer {
    public static void main(String[] args) throws InterruptedException {
        //创建bossGroup,workGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        try {
            //创建辅助工具
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            //循环事件组
            serverBootstrap.group(bossGroup,workGroup)//线程组
                .channel(NioServerSocketChannel.class)//通道类型
                .handler(new LoggingHandler(LogLevel.INFO))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        ChannelPipeline pipeline = socketChannel.pipeline();
                        //基于http协议使用http的编码和解码器
                        pipeline.addLast(new HttpServerCodec());
                        // 添加块处理器
                        pipeline.addLast(new ChunkedWriteHandler());
                        /*
                               说明：
                               1. 因为 HTTP 数据传输时是分段的，HttpObjectAggregator 可以将多个端聚合
                               2. 这就是为什么浏览器发送大量数据时，就会发出多次 HTTP 请求
                            */
                        pipeline.addLast(new HttpObjectAggregator(8192));
                        /*
                               说明：
                               1. 对于 WebSocket 是以 帧 的形式传递的
                               2. 后面的参数表示 ：请求的 URL
                               3. WebSocketServerProtocolHandler 将 HTTP 协议升级为 WebSocket 协议，即保持长连接
                               4. 切换协议通过一个状态码101
                            */
                        pipeline.addLast(new WebSocketServerProtocolHandler("/hello"));
                        // 自定义的 Handler
                        pipeline.addLast(new MyTextWebSocketFrameHandler());


                    }
                });
            System.out.println("server is ok");
            ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }


}
```


**服务端的处理器 ：MyTextWebSocketFrameHandler**

```java
/**
 * TextWebSocketFrame 类型，表示一个文本帧（flame）
 */
public class MyTextWebSocketFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        System.out.println("服务器收到消息"+msg.text());

        //回复消息
        ctx.channel().writeAndFlush(new TextWebSocketFrame("服务器时间"+ LocalDateTime.now()+msg.text()));
    }

    /**
         * 客户端连接后，触发方法
         * @param ctx
         * @throws Exception
         */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        //id表示唯一的值，Longtext是唯一的shortText 不是唯一
        System.out.println("handlerAdded 被调用"+ctx.channel().id().asLongText());
        System.out.println("handlerAdded 被调用"+ctx.channel().id().asShortText());
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        System.out.println("handlerRemove被调用"+ctx.channel().id().asLongText());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("异常发生"+cause.getMessage());
        //关闭连接
        ctx.close();
    }

}
```


hello.html(浏览器)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <form onsubmit="return false">
        <textarea id="message" name="message" style="height: 300px; width: 300px"></textarea>
        <input type="button" value="发送消息" onclick="send(this.form.message.value)">
        <textarea id="responseText" style="height: 300px; width: 300px"></textarea>
        <input type="button" value="清空内容" onclick="document.getElementById('responseText').value=''">
    </form>
    <script>
        var socket;
        // 判断当前浏览器是否支持 WebSocket
        if (window.WebSocket){
            socket = new WebSocket("ws://localhost:8080/hello");
            // 相当于 channelRead0 方法，ev 收到服务器端回送的消息
            socket.onmessage = function (ev){
                var rt = document.getElementById("responseText");
                rt.value = rt.value + "\n" + ev.data;
            }
            // 相当于连接开启，感知到连接开启
            socket.onopen = function (){
                var rt = document.getElementById("responseText");
                rt.value = rt.value + "\n" + "连接开启……";
            }
            // 感知连接关闭
            socket.onclose = function (){
                var rt = document.getElementById("responseText");
                rt.value = rt.value + "\n" + "连接关闭……";
            }
        }else {
            alert("不支持 WebSocket");
        }

        // 发送消息到服务器
        function send(message){
            // 判断 WebSocket 是否创建好了
            if (!window.socket){
                return ;
            }
            // 判断 WebSocket 是否开启
            if (socket.readyState == WebSocket.OPEN){
                // 通过 Socket 发送消息
                socket.send(message);
            }else {
                alert("连接未开启");
            }
        }
    </script>
</body>
</html>
```

