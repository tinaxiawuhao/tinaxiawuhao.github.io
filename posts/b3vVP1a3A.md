---
title: 'netty异步任务调度 ( TaskQueue | ScheduleTaskQueue | SocketChannel 管理 )'
date: 2021-10-08 11:13:12
tags: [netty]
published: true
hideInList: false
feature: /post-images/b3vVP1a3A.png
isTop: false
---
![](https://tinaxiawuhao.github.io/post-images/1633676052562.jfif)
### 一、 任务队列 

任务队列的任务 Task 应用场景 :

1. 自定义任务 : 自己开发的任务 , 然后将该任务提交到任务队列中 ;

2. 自定义定时任务 : 自己开发的任务 , 然后将该任务提交到任务队列中 , 同时可以指定任务的执行时间 ;

3. 其它线程调度任务 : 上面的任务都是在当前的 NioEventLoop ( 反应器 Reactor 线程 ) 中的任务队列中排队执行 , 在其它线程中也可以调度本线程的 Channel 通道与该线程对应的客户端进行数据读写 ;



### 二、 处理器 Handler 同步异步操作

在之前的 Netty 服务器与客户端项目中 , 用户自定义的 Handler 处理器 , 该处理器继承了 ChannelInboundHandlerAdapter 类 , 在重写的 public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception 方法中 , 执行的业务逻辑要注意以下两点 :

`同步操作` : 如果在该业务逻辑中只执行一个短时间的操作 , 那么可以直接执行 ;
`异步操作` : 如果在该业务逻辑中执行访问数据库 , 访问网络 , 读写本地文件 , 执行一系列复杂计算等耗时操作 , 肯定不能在该方法中处理 , 这样会阻塞整个线程 ; 正确的做法是将耗时的操作放入任务队列 TaskQueue , 异步执行 ;

在 ChannelInboundHandlerAdapter 的 channelRead 方法执行时 , 客户端与服务器端的反应器 Reactor 线程 NioEventLoop 是处于阻塞状态的 , 此时服务器端与客户端同时都处于阻塞状态 , 这样肯定不行 , 因为 NioEventLoop 需要为多个客户端服务 , 不能因为与单一客户端交互而产生阻塞 ;





### 三、 异步任务 ( 用户自定义任务 )

1. 用户自定义任务流程 :


      ① 获取通道 : 首先获取 通道 Channel ;
    
      ② 获取线程 : 获取通道对应的 EventLoop 线程 , 就是 NioEventLoop , 该 NioEventLoop 中封装了任务队列 TaskQueue ;
    
      ③ 任务入队 : 向任务队列 TaskQueue 中放入异步任务 Runnable , 调用 NioEventLoop 线程的 execute 方法 , 即可将上述 Runnable 异步任务放入任务队列 TaskQueue ;

2. 多任务执行 : 如果用户连续向任务队列中放入了多个任务 , NioEventLoop 会按照顺序先后执行这些任务 , 注意任务队列中的任务 是先后执行 , 不是同时执行 ;

   顺序执行任务 ( 不是并发 ) : 任务队列任务执行机制是顺序执行的 ; 先执行第一个 , 执行完毕后 , 从任务队列中获取第二个任务 , 执行完毕之后 , 依次从任务队列中取出任务执行 , 前一个任务执行完毕后 , 才从任务队列中取出下一个任务执行 ;

3. 代码示例 : 监听到客户端上传数据后 , channelRead 回调 , 执行 获取通道 -> 获取线程 -> 异步任务调度 流程 ;

   ```java
   /**
    * Handler 处理者, 是 NioEventLoop 线程中处理业务逻辑的类
    *
    * 继承 : 该业务逻辑处理者 ( Handler ) 必须继承 Netty 中的 ChannelInboundHandlerAdapter 类
    * 才可以设置给 NioEventLoop 线程
    *
    * 规范 : 该 Handler 类中需要按照业务逻辑处理规范进行开发
    */
   public class ServerHandr extends ChannelInboundHandlerAdapter {
   
       /**
        * 读取数据 : 在服务器端读取客户端发送的数据
        * @param ctx
        *      通道处理者上下文对象 : 封装了 管道 ( Pipeline ) , 通道 ( Channel ), 客户端地址信息
        *      管道 ( Pipeline ) : 注重业务逻辑处理 , 可以关联很多 Handler
        *      通道 ( Channel ) : 注重数据读写
        * @param msg
        *      客户端上传的数据
        * @throws Exception
        */
       @Override
       public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
           // 1 . 从 ChannelHandlerContext ctx 中获取通道
           Channel channel = ctx.channel();
           // 2 . 获取通道对应的事件循环
           EventLoop eventLoop = channel.eventLoop();
           // 3 . 在 Runnable 中用户自定义耗时操作, 异步执行该操作, 该操作不能阻塞在此处执行
           eventLoop.execute(new Runnable() {
               @Override
               public void run() {
                   //执行耗时操作
               }
           });
       }
   }
   ```

   




### 四、 异步任务 ( 用户自定义定时任务 )

1. 用户自定义定时任务 与 用户自定义任务流程基本类似 , 有以下两个不同之处 :


      *① 调度方法 :*
    
      定时异步任务使用 schedule 方法进行调度 ;
      普通异步任务使用 execute 方法进行调度 ;
    
      *② 任务队列 :*
    
      定时异步任务提交到 ScheduleTaskQueue 任务队列中 ;
      普通异步任务提交到 TaskQueue 任务队列中 ;

2. 用户自定义定时任务流程 :


      ① 获取通道 : 首先获取 通道 Channel ;
    
      ② 获取线程 : 获取通道对应的 EventLoop 线程 , 就是 NioEventLoop , 该 NioEventLoop 中封装了任务队列 TaskQueue ;
    
      ③ 任务入队 : 向任务队列 ScheduleTaskQueue 中放入异步任务 Runnable , 调用 NioEventLoop 线程的 schedule 方法 , 即可将上述 Runnable 异步任务放入任务队列 ScheduleTaskQueue ;

3. 代码示例 : 监听到客户端上传数据后 , channelRead 回调 , 执行 获取通道 -> 获取线程 -> 异步任务调度 流程 ;

   ```java
   /**
    * Handler 处理者, 是 NioEventLoop 线程中处理业务逻辑的类
    *
    * 继承 : 该业务逻辑处理者 ( Handler ) 必须继承 Netty 中的 ChannelInboundHandlerAdapter 类
    * 才可以设置给 NioEventLoop 线程
    *
    * 规范 : 该 Handler 类中需要按照业务逻辑处理规范进行开发
    */
   public class ServerHandr extends ChannelInboundHandlerAdapter {
   
       /**
        * 读取数据 : 在服务器端读取客户端发送的数据
        * @param ctx
        *      通道处理者上下文对象 : 封装了 管道 ( Pipeline ) , 通道 ( Channel ), 客户端地址信息
        *      管道 ( Pipeline ) : 注重业务逻辑处理 , 可以关联很多 Handler
        *      通道 ( Channel ) : 注重数据读写
        * @param msg
        *      客户端上传的数据
        * @throws Exception
        */
       @Override
       public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
           // 1 . 从 ChannelHandlerContext ctx 中获取通道
           Channel channel = ctx.channel();
           // 2 . 获取通道对应的事件循环
           EventLoop eventLoop = channel.eventLoop();
           // 3 . 在 Runnable 中用户自定义耗时操作, 异步执行该操作, 该操作不能阻塞在此处执行
           // schedule(Runnable command, long delay, TimeUnit unit)
           // Runnable command 参数 : 异步任务
           // long delay 参数 : 延迟执行时间
           // TimeUnit unit参数 : 延迟时间单位, 秒, 毫秒, 分钟
           eventLoop.schedule(new Runnable() {
               @Override
               public void run() {
                   //执行耗时操作
               }
           }, 100, TimeUnit.MILLISECONDS);
       }
   }
   ```

   






### 五、 异步任务 ( 其它线程向本线程调度任务 )

1. 通过EventExecutorGroup线程池获取不同线程执行异步耗时任务



2. 代码示例（一）

   ```java
   // 服务器启动对象, 需要为该对象配置各种参数
   ServerBootstrap bootstrap = new ServerBootstrap();
   // 添加线程组异步执行耗时任务
   final EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(16);
   bootstrap.group(bossGroup, workerGroup) // 设置 主从 线程组 , 分别对应 主 Reactor 和 从 Reactor
           .channel(NioServerSocketChannel.class)  // 设置 NIO 网络套接字通道类型
           .option(ChannelOption.SO_BACKLOG, 128)  // 设置线程队列维护的连接个数
           .childOption(ChannelOption.SO_KEEPALIVE, true)  // 设置连接状态行为, 保持连接状态
           .childHandler(  // 为 WorkerGroup 线程池对应的 NioEventLoop 设置对应的事件 处理器 Handler
                   new ChannelInitializer<SocketChannel>() {// 创建通道初始化对象
                       @Override
                       protected void initChannel(SocketChannel ch) throws Exception {
                           // 该方法在服务器与客户端连接建立成功后会回调
                           // 为 管道 Pipeline 设置处理器 Hanedler
                           ch.pipeline().addLast(businessGroup,new NettyServerHandler());
                       }
                   }
           );
   ```

3. 代码示例（二）

   ```java
   public class NettyServerHandler extends SimpleChannelInboundHandler<Object> {
   	final EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(16);
   
       @Override
       protected void channelRead0(ChannelHandlerContext ctx, Object obj) throws Exception {
          businessGroup.submit(new Callable<Object>() {
               @Override
               public Object call() throws Exception {
                   //业务处理
                   return null;
               }
           });
       }
   }
   ```

   

