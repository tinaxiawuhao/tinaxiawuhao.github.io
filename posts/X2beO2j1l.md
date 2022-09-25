---
title: 'NIO'
date: 2022-06-25 19:53:37
tags: [NIO]
published: true
hideInList: false
feature: /post-images/X2beO2j1l.png
isTop: false
---
### 1. 简介
Netty 是由 JBOSS 提供的一个 Java 开源框架，现为 Github上的独立项目。
Netty 是一个异步的、基于事件驱动（客户端的行为、读写事件）的网络应用框架，用以快速开发高性能、高可靠性的网络 IO 程序。
Netty主要针对在TCP协议下，面向Clients端的高并发应用，或者Peer-to-Peer场景下的大量数据持续传输的应用。
Netty本质是一个NIO框架，适用于服务器通讯相关的多种应用场景
要透彻理解Netty ， 需要先学习 NIO ， 这样我们才能阅读 Netty 的源码。

### 2. I/O 模型基本说明
I/O 模型简单的理解：就是用什么样的通道进行数据的发送和接收，很大程度上决定了程 序通信的性能

#### 2.1. Java共支持3种网络编程模型/IO模式：
> BIO、NIO、AIO

`Java BIO` ： 同步并阻塞(传统阻塞型)，服务器实现模式为一个连接一个线程，即客户端 有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成 不必要的线程开销

![](https://tianxiawuhao.github.io/post-images/1656504376989.png)

`Java NIO` ： 同步非阻塞，服务器实现模式为一个线程处理多个请求(连接)，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求就进行处理

![](https://tianxiawuhao.github.io/post-images/1656504386068.png)

`Java AIO(NIO.2)` ： 异步非阻塞，AIO 引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程

#### 2.2. BIO、NIO、AIO适用场景分析
`BIO方式`
适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高， 并发局限于应用中，JDK1.4以前的唯一选择，但程序简单易理解。

`NIO方式`
适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕 系统，服务器间通讯等。编程比较复杂，JDK1.4开始支持。

`AIO方式`
使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分 调用OS参与并发操作，编程比较复杂，JDK7开始支持。

### 3. NIO具体介绍
Java NIO 全称 java non-blocking IO，是指 JDK 提供的新 API。从 JDK1.4 开始，Java 提供了一系列改进的输入/输出 的新特性，被统称为 NIO(即 New IO)，是同步非阻塞的
NIO 相关类都被放在 java.nio 包及子包下，并且对原 java.io 包中的很多类进行改写。
NIO 有三大核心部分：Channel(通道)，Buffer(缓冲区), Selector(选择器)
NIO是 面向缓冲区 ，或者面向块编程的。数据读取到一个 它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就 增加了处理过程中的灵活性，使用它可以提供非阻塞式的高 伸缩性网络
Java NIO的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这 个线程同时可以去做别的事情。
通俗理解：NIO是可以做到用一个线程来处理多个操作的。假设有10000个请求过来, 根据实际情况，可以分配50或者100个线程来处理。不像之前的阻塞IO那样，非得分配10000个。
HTTP2.0使用了多路复用的技术，做到同一个连接并发处理多个请求，而且并发请求 的数量比HTTP1.1大了好几个数量级

#### 3.1. NIO 三大核心
Selector 、 Channel 和 Buffer 的简单关系图

![](https://tianxiawuhao.github.io/post-images/1656504401555.png)

> 关系图的说明:
> 线程是非阻塞，buffer起很大的作用
> 每个 Channel 都会对应一个 Buffer
> Selector 对应一个线程， 一个 Selector 对应多个 Channel(连接)
> 该图反应了有三个 Channel 注册到该 selector
> 程序切换到哪个 Channel 是由事件决定的, Event 就是一个重要的概念
> Selector 会根据不同的事件，在各个 Channel（通道）上切换
> Buffer 就是一个内存块 ， 底层是有一个数组
> 数据的读取写入是通过 Buffer, 这个和BIO , BIO 中要么是输入流，或者是 输出流, 不能双向，但是NIO的 Buffer 是可以读也可以写, 需要 flip 方法切换

#### 3.2. 常用Buffer子类一览
> ByteBuffer，存储字节数据到缓冲区 ===最常用===
ShortBuffer，存储字符串数据到缓冲区
CharBuffer，存储字符数据到缓冲区
IntBuffer，存储整数数据到缓冲区
LongBuffer，存储长整型数据到缓冲区
DoubleBuffer，存储小数到缓冲区
FloatBuffer，存储小数到缓冲区

#### 3.3. buffer常用方法

```java
public abstract class Buffer { 
	//JDK1.4时，引入的api 
	public final int capacity( )// ★ 返回此缓冲区的容量 
	public final int position( )// ★ 返回此缓冲区的位置 
	public final Buffer position (int newPositio)// ★ 设置此缓冲区的位置 
	public final int limit( )// ★ 返回此缓冲区的限制 
	public final Buffer limit (int newLimit)// ★ 设置此缓冲区的限制 
	public final Buffer mark( )//在此缓冲区的位置设置标记 
	public final Buffer reset( )//将此缓冲区的位置重置为以前标记的位置 
	public final Buffer clear( )// ★ 清除此缓冲区, 即将各个标记恢复到初始状态，但是数据并没有真正擦除, 后面操作会覆盖 
	public final Buffer flip( )// ★ 反转此缓冲区 
	public final Buffer rewind( )//重绕此缓冲区 
	public final int remaining( )//返回当前位置与限制之间的元素数 
	public final boolean hasRemaining( )// ★ 告知在当前位置和限制之间是否有元素 
	public abstract boolean isReadOnly( );// ★ 告知此缓冲区是否为只读缓冲区 
    //JDK1.6时引入的api 
    public abstract boolean hasArray();// ★ 告知此缓冲区是否具有可访问的底层实现数组 
    public abstract Object array();// ★ 返回此缓冲区的底层实现数组 
    public abstract int arrayOffset();//返回此缓冲区的底层实现数组中第一个缓冲区元素的偏移量 
    public abstract boolean isDirect();//告知此缓冲区是否为直接缓冲区 
}
```

#### 3.4. 通道(Channel)
*1：基本介绍*
NIO的通道类似于流，但有些区别如下：

> • 通道可以同时进行读写，而流只能读或者只能写
> • 通道可以实现异步读写数据
> • 通道可以从缓冲读数据，也可以写数据到缓冲:

FileChannel主要用来对本地文件进行 IO 操作，常见的方法有

```java
public int read(ByteBuffer dst) ，从通道读取数据并放到缓冲区中
public int write(ByteBuffer src) ，把缓冲区的数据写到通道中
public long transferFrom(ReadableByteChannel src, long position, long count)，从目标通道 中复制数据到当前通道
public long transferTo(long position, long count, WritableByteChannel target)，把数据从当 前通道复制给目标通道
```

首先channel与文件联立，判断是输入输出流看channel与buffer之间的关系

#### 3.5. 关于Buffer 和 Channel的注意事项和细节
1：ByteBuffer 支持类型化的 put 和 get, put 放入的是什么数据类型，get 就应该使用相应的数据类型来取出（取出的顺序也要和存入的顺序一致），否则可能有 BufferUnderflowException 异常。

2： 可以将一个普通Buffer 转成只读Buffer，如果对一个只读类型的 Buffer 进行写操作会报错 ReadOnlyBufferException

```java
ByteBuffer buffer = ByteBuffer.allocate(3);
ByteBuffer byteBuffer = buffer.asReadOnlyBuffer();
System.out.println(buffer);
System.out.println(byteBuffer);
```
3：NIO 还提供了 MappedByteBuffer， 可以让文件直接在内存（堆外的内存）中进行修改， 而如何同步到文件由NIO 来完成.

```java
/* 说明
    1. MappedByteBuffer 可以让文件直接在内存中修改，这样操作系统并不需要拷贝一次
    2. MappedByteBuffer 实际类型是 DirectByteBuffer
*/
public static void main(String[] args) throws Exception {
   RandomAccessFile randomAccessFile = new RandomAccessFile("D:\\file01.txt", "rw");
   // 获取对应的文件通道
   FileChannel channel = randomAccessFile.getChannel();
   // 参数 ：使用 只读/只写/读写 模式 ； 可以修改的起始位置 ； 映射到内存的大小，即可以将文件的多少个字节映射到内存
   // 这里就表示，可以对 file01.txt 文件中 [0,5) 的字节进行 读写操作
   MappedByteBuffer map = channel.map(FileChannel.MapMode.READ_WRITE, 0, 5);
   // 进行修改操作
   map.put(0, (byte) 'A');
   map.put(3, (byte) '3');
   // 关闭通道
   channel.close();
}
```

4：前面我们讲的读写操作，都是通过一个Buffer 完成的，NIO 还支持 通过多个 Buffer (即 Buffer 数组) 完成读写操作，即 Scattering 和 Gathering ，遵循 依次写入，依次读取。

```java
public static void main(String[] args) throws Exception {
    // 使用 ServerSocketChannel 和 InetSocketAddress 网络
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    InetSocketAddress inetSocketAddress = new InetSocketAddress(7000);
    //绑定端口到socket，启动
    serverSocketChannel.socket().bind(inetSocketAddress);

    //创建buffer的2个数组
    ByteBuffer[] byteBuffers = new ByteBuffer[2];
    byteBuffers[0] = ByteBuffer.allocate(5);
    byteBuffers[1] = ByteBuffer.allocate(3);

    //等待客户端连接
    SocketChannel socketChannel = serverSocketChannel.accept();
    int messageLength = 8;
    //循环的读取
    while(true){
        // 表示累计读取的字节数
        int byteRead = 0;
        // 假设从客户端最多接收 8 个字节
        while (byteRead < messageLength){
            // 自动把数据分配到 byteBuffers-0、byteBuffers-1
            long read = socketChannel.read(byteBuffers);
            byteRead += read;
            // 使用流打印，查看当前 Buffer 的 Position 和 Limit
            Arrays.asList(byteBuffers).stream().
                    map(byteBuffer -> "{position: "+byteBuffer.position()+", limit: "+byteBuffer.limit()+"}")
                    .forEach(System.out::println);
        }
        // 将所有的 Buffer 进行反转，为后面的其他操作做准备
        Arrays.asList(byteBuffers).forEach(Buffer ->Buffer.flip());

        // 将数据读出，显示到客户端
        int byteWrite = 0;
        while (byteWrite < messageLength){
            long write = socketChannel.write(byteBuffers);
            byteWrite += write;
        }

        // 将所有的 Buffer 进行清空，为后面的其他操作做准备
        Arrays.asList(byteBuffers).forEach(Buffer->Buffer.clear());

        // 打印处理的字节数
        System.out.println("{byteRead: "+byteRead+", byteWrite: "+byteWrite+"}");

    }
}
```
#### 3.6. Selector选择器
##### 1. 基本介绍
> 1: Java 的 NIO，用非阻塞的 IO 方式。可以用一个线程，处理多个的客户端连接，就会使用到Selector(选择器)
> 2: Selector 能够检测多个注册的通道上是否有事件发生(注意:多个Channel以事件的方式可以注册到同一个Selector)，如果有事件发生，便获取事件然 后针对每个事件进行相应的处理。这样就可以只用一个单线程去管理多个通道，也就是管理多个连接和请求。
> 3: 只有在 连接/通道 真正有读写事件发生时，才会进行读写，就大大地减少 了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程
> 4: 避免了多线程之间的上下文切换导致的开销

##### 2. Selector(选择器)
示意图

![](https://tianxiawuhao.github.io/post-images/1656504430406.png)

说明:

> Netty 的 IO 线程 NioEventLoop 聚合了 Selector(选择器， 也叫多路复用器)，可以同时并发处理成百上千个客 户端连接。
> 当线程从某客户端 Socket 通道进行读写数据时，若没 有数据可用时，该线程可以进行其他任务。
> 线程通常将非阻塞 IO 的空闲时间用于在其他通道上 执行 IO 操作，所以单独的线程可以管理多个输入和 输出通道。
> 由于读写操作都是非阻塞的，这就可以充分提升 IO 线程的运行效率，避免由于频繁 I/O 阻塞导致的线程 挂起。
> 一个 I/O 线程可以并发处理 N 个客户端连接和读写操作，这从根本上解决了传统同步阻塞 I/O 一连接一线 程模型，架构的性能、弹性伸缩能力和可靠性都得到 了极大的提升。



##### 3. Selector类相关方法
Selector 类是一个抽象类, 常用方法和说明如下

```java
public abstract class Selector implements Closeable { 
	public static Selector open();//得到一个选择器对象 
	public int select(long timeout);//监控所有注册的通道，当其 中有 IO 操作可以进行时，将 对应的 SelectionKey 加入到内部集合中并返回，参数用来 设置超时时间 
	public Set<SelectionKey> selectedKeys();//从内部集合中得 到所有的 SelectionKey 
}
```


##### 4. 注意事项
NIO中的 ServerSocketChannel功能类似ServerSocket，SocketChannel功能类 似Socket

selector 相关方法说明

```java
selector.select()//阻塞
selector.select(1000);//阻塞1000毫秒，在1000毫秒后返回
selector.wakeup();//唤醒
selector selector.selectNow();//不阻塞，立马返还
```

#### 3.7. NIO 非阻塞 网络编程原理分析
NIO 非阻塞 网络编程相关的(Selector、SelectionKey、 ServerScoketChannel和SocketChannel) 关系梳理图

![](https://tianxiawuhao.github.io/post-images/1656504451263.png)

说明：

> ServerSocketChannel 需要在selector注册，一旦有register事件（客户端一旦连接）就会建立SocketChannel
客户端连接时需要会通过ServerSocketChannel 在selector注册，一旦有读写事件，反向获取通道channel，把channel数据读出到buffer
事件发生通过SelectorKey来判断

**代码演示：server端**

```java
public static void main(String[] args) throws Exception {
    //服务器端创建ServerSocketChannel ->ServerSocketChannel
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();   
	//得到一个Selector对象
    Selector selector = Selector.open();

    //绑定一个端口6666，在服务器端监听
    serverSocketChannel.socket().bind(new InetSocketAddress(6666));
    //设置为非阻塞
    serverSocketChannel.configureBlocking(false);

    //把serverSocketChannel 注册到 selector 关心的事件：op_ACCEPT
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

    //循环等待客户端连接
    while(true){
        if(selector.select(1000) == 0){//没有事件发生
            System.out.println("服务器等待1s,无连接");
            continue;
        }

        //返回大于0，获取相关的selectionKey集合(获取到关注的事件)
        //selector.selectedKeys() 返回关注的事件集合
        Set<SelectionKey> selectionKeys = selector.selectedKeys();

        //遍历,使用迭代器
        Iterator<SelectionKey> keyIterator = selectionKeys.iterator();

        while(keyIterator.hasNext()){
            //获取到SelectionKey
            SelectionKey key = keyIterator.next();
            //根据key 对应的通道发生的事件做相应的处理
            if(key.isAcceptable()){//如果是OP_ACCEPT,有新的客户端连接
                //该客户端生成一个SocketChannel
                SocketChannel socketChannel = serverSocketChannel.accept();
                System.out.println("客户端连接成功，生成一个socketChannel");
                //将socketChannel设置为非阻塞，这时线程可以做其他事
                socketChannel.configureBlocking(false);
                //将socketChannel 注册到Selector，关注OP_READ，同时给socketChannel关联一个buffer
               socketChannel.register(selector,SelectionKey.OP_READ, ByteBuffer.allocate(1024));

            }
            if(key.isReadable()){//发生OP_READ
                //通过key 反向获取对应的channel
                SocketChannel channel = (SocketChannel) key.channel();
                //获取改channel关联的buffer
                ByteBuffer buffer = (ByteBuffer) key.attachment();
                channel.read(buffer);
                System.out.println("form client "+ new String(buffer.array()));
            }
            //手动从集合中移动当前的selectionKey,防止重复操作
            keyIterator.remove();

        }

    }
}
```
**client端：**

```java
public static void main(String[] args) throws Exception {
    //得到一个网络通道
    SocketChannel socketChannel = SocketChannel.open();
    //设置非阻塞模式
    socketChannel.configureBlocking(false);    
	//提供服务器端的ip和端口
    InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1",6666);

    //连接服务器
    if(!socketChannel.connect(inetSocketAddress)){
        while(!socketChannel.finishConnect()){
            System.out.println("连接需要时间，客户端不会阻塞，可做其他工作");
        }
    }
    //连接成功，发送数据
    String str = "hello,world";
    //获得字节数组到buffer中,且buffer大小与字节长度一致
    ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
    //发送数据，将buffer 数据写入channel
    socketChannel.write(buffer);
    System.in.read();
}
```

#### 3.8. SelectionKey
SelectionKey，表示 Selector 和网络通道的注册关系（OPS）, 共四种：

> int OP_ACCEPT：有新的网络连接可以 accept，值为 16
> int OP_CONNECT：代表连接已经建立，值为 8
> int OP_READ：代表读操作，值为 1
> int OP_WRITE：代表写操作，值为 4

源码中：

> public static final int OP_READ = 1 << 0;
> public static final int OP_WRITE = 1 << 2;
> public static final int OP_CONNECT = 1 << 3;
> public static final int OP_ACCEPT = 1 << 4;


SelectionKey相关方法


```java
public abstract class SelectionKey { 
	public abstract Selector selector();//得到与之关联的 Selector 对象 
	public abstract SelectableChannel channel();//得到与之关 联的通道 
	public final Object attachment();//得到与之关联的共享数 据 
	public abstract SelectionKey interestOps(int ops);//设置或改 变监听事件 
	public final boolean isAcceptable();//是否可以 accept 
	public final boolean isReadable();//是否可以读 
	public final boolean isWritable();//是否可以写 
}
```

#### 3.9. ServerSocketChannel
ServerSocketChannel 在服务器端监听新的客户端 Socket 连接

相关方法如下:

```java
public abstract class ServerSocketChannel extends AbstractSelectableChannel implements NetworkChannel{ 
	public static ServerSocketChannel open()//得到一个 ServerSocketChannel 通道 
	public final ServerSocketChannel bind(SocketAddress local)//设置服务器端端口 号
	public final SelectableChannel configureBlocking(boolean block)//设置阻塞或非 阻塞模式，取值 false 表示采用非阻塞模式 
	public SocketChannel accept()//接受一个连接，返回代表这个连接的通道对象
	public final SelectionKey register(Selector sel, int ops)//注册一个选择器并设置 监听事件 
}
```

#### 3.10. SocketChannel
SocketChannel，网络 IO 通道，具体负责进行读写操作。NIO 把缓冲区的数据写入通 道，或者把通道里的数据读到缓冲区。

相关方法如下

```java
public abstract class SocketChannel extends AbstractSelectableChannel implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, NetworkChannel{ 
	public static SocketChannel open();//得到一个 SocketChannel 通道 
	public final SelectableChannel configureBlocking(boolean block);//设置阻塞或非阻塞 模式，取值 false 表示采用非阻塞模式 
	public boolean connect(SocketAddress remote);//连接服务器 
	public boolean finishConnect();//如果上面的方法连接失败，接下来就要通过该方法 完成连接操作 
	public int write(ByteBuffer src);//往通道里写数据 
	public int read(ByteBuffer dst);//从通道里读数据 
	public final SelectionKey register(Selector sel, int ops, Object att);//注册一个选择器并 设置监听事件，最后一个参数可以设置共享数据 
	public final void close();//关闭通道 
}
```


#### 3.11. NIO 网络编程应用实例-群聊系统
> 要求:
> 编写一个 NIO 群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）
> 实现多人群聊
> 服务器端：可以监测用户上线，离线， 并实现消息转发功能
> 客户端：通过channel 可以无阻塞发送 消息给其它所有用户，同时可以接受 其它用户发送的消息(有服务器转发得到)
> 目的：进一步理解NIO非阻塞网络编程 机制

**代码实现：服务端**


```java
package com.atguigu.groupchat;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;

public class GroupChatServer {
    //定义属性
    private Selector selector;
    private ServerSocketChannel listenChannel;
    private static final int PORT = 6667;
    //构造器
    //初始化工作
    public GroupChatServer(){
        try {
            //得到选择器
            selector = Selector.open();
            //得到ServerSocketChannel
            listenChannel = ServerSocketChannel.open();
            //绑定端口
            listenChannel.socket().bind(new InetSocketAddress(PORT));
            //设置非阻塞
            listenChannel.configureBlocking(false);
            //listenChannel注册到Selector
            listenChannel.register(selector, SelectionKey.OP_ACCEPT);
    	} catch (IOException e) {
            e.printStackTrace();
       	}
	}
    //监听
    public void listen(){
        try {
            //循环处理监听
            while(true){
                int count = selector.select();
                if(count>0){//有事件
                    //遍历得到selectionKey 集合
                    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                    while(iterator.hasNext()){
                        //取得selectionKey
                        SelectionKey key = iterator.next();
                        if(key.isAcceptable()){//连接事件
                            SocketChannel socketChannel = listenChannel.accept();
                            socketChannel.configureBlocking(false);
                            socketChannel.register(selector,SelectionKey.OP_READ);
                            //提示客户上线
                            System.out.println(socketChannel.getRemoteAddress()+"上线了");
                        }
                        else if(key.isReadable()){//read事件
                            readDate(key);
                        }
                        //手动从集合中移动当前的selectionKey,防止重复操作
                        iterator.remove();
                    }

                }
                else{
                    System.out.println("waiting event");
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }finally {

        }
    }
    //读取client数据
    public void readDate(SelectionKey key){
        //定义一个SocketChannel
        SocketChannel socketChannel = null;
        try {
            //取到关联的channel
            socketChannel = (SocketChannel)key.channel();
            //创建缓存
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
            int count = socketChannel.read(byteBuffer);
            if(count>0){
                //buffer的数据转成字符串
                String msg = new String(byteBuffer.array());
                //输出msg
                System.out.println("form client"+msg);
                //向其他客户端转发消息
                sendInfoToOtherClient(msg,socketChannel);
            }
        } catch (IOException e) {
            try {
                System.out.println(socketChannel.getRemoteAddress()+"离线了");
                //取消注册
                key.cancel();
                //关闭通道
                socketChannel.close();
            } catch (IOException ioException) {
                ioException.printStackTrace();
            }
        }

    }
    //转发消息给其他消息
    private void sendInfoToOtherClient(String msg,SocketChannel self) throws IOException{
        //遍历所有注册到selector的SocketChannel
        for(SelectionKey key: selector.keys()){
            Channel targetChannel = key.channel();
            //排除自己
            if(targetChannel != self && targetChannel instanceof SocketChannel){
                //转发
                SocketChannel dest = (SocketChannel)targetChannel;
                //写入buffer
                ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
                //写入通道
                dest.write(buffer);
            }
        }
    }
    public static void main(String[] args) {
        // 创建一个服务器对象
        GroupChatServer server = new GroupChatServer();
        server.listen();
    }
}
```

**代码实现：客户端**



```java
package com.atguigu.groupchat;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Scanner;

public class GroupChatClient {
    // 定义相关属性
    // 服务器的IP
    private final String HOST = "127.0.0.1";
    // 服务器的端口
    private final int PORT = 6667;
    private Selector selector;
    private SocketChannel socketChannel;
    private String username;

    // 构造器
    public GroupChatClient() throws IOException {
        // 完成初始化
        selector = Selector.open();
        // 连接服务器
        socketChannel = SocketChannel.open(new InetSocketAddress(HOST, PORT));
        // 设置 非阻塞
        socketChannel.configureBlocking(false);
        // 将 socketChannel 注册到 Selector
        socketChannel.register(selector, SelectionKey.OP_READ);
        // 得到 username
        username = socketChannel.getLocalAddress().toString().substring(1);
        System.out.println(username + "is OK!");

    }

    // 向服务器发送消息
    public void sendMessage(String message){
        message = username + "说："+ message;
        try {
            // 把 message 写入 buffer
            socketChannel.write(ByteBuffer.wrap(message.getBytes()));
            // 读取从服务器端回复的消息
        }catch (Exception e){
            e.printStackTrace();
        }finally {

        }
    }

    public void readmessage(){
        try {
            int select = selector.select();
            if (select > 0){
                // 有事件发生的通道
                Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                while (iterator.hasNext()){
                    SelectionKey key = iterator.next();
                    if (key.isReadable()){
                        // 得到相关的通道
                        SocketChannel channel = (SocketChannel) key.channel();
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        channel.read(buffer);
                        String msg = new String(buffer.array());
                        System.out.println(msg.trim());
                    }
                }
                iterator.remove();//删除当前的selectionKey
            }else {
            //                System.out.println("没有可用的通道");
     		}
        }catch (Exception e){
            e.printStackTrace();
        }finally {

        }
    }

    public static void main(String[] args) throws IOException {
        // 启动客户端
        GroupChatClient client = new GroupChatClient();

        // 启动一个线程,每个三秒读取从服务器端读取数据
        new Thread(){
            public void run(){
                while (true){
                    client.readmessage();
                    try {
                        Thread.sleep(3000);
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }

        }.start();

        // 发送数据给服务端
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNextLine()){
            String line = scanner.nextLine();
            client.sendMessage(line);
        }
    }
}     
```