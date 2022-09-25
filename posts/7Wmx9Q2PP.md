---
title: 'NIO-FileChannel'
date: 2021-09-17 16:28:44
tags: [java]
published: true
hideInList: false
feature: /post-images/7Wmx9Q2PP.png
isTop: false
---
### NIO介绍
　　在讲解Channel之前，首先了解一下NIO， Java NIO全称java non-blocking IO，是从Java 1.4版本开始引入的一个新的IO API（New IO），可以替代标准的Java IO API，NIO与原来的IO有同样的作用和目的，但是使用的方式完全不同。IO与 NIO区别：

| IO   | NIO  |
| ---- | ---- |
|面向流（Stream Orientend）	|面向缓冲区（Buffer Orientend）|
|阻塞IO（Blocking IO )	|非阻塞IO（Non Blocking IO）|
||选择器（Selector）|

　　NIO支持面向缓冲区的、基于通道的IO操作并以更加高效的方式进行文件的读写操作，其核心API为Channel(通道)，Buffer(缓冲区), Selector(选择器)。Channel负责传输，Buffer负责存储 。

### 缓冲区

```java
public class BioTest {
    @Test
    public void test1() {
        //1.初始化缓冲区数组
        ByteBuffer bf = ByteBuffer.allocate(1024);
        System.out.println("==========allocate============");
        System.out.println(bf.position());
        System.out.println(bf.limit());
        System.out.println(bf.capacity());

        //2.put插入数据
        bf.put("asasas".getBytes());
        System.out.println("==========put============");
        System.out.println(bf.position());
        System.out.println(bf.limit());
        System.out.println(bf.capacity());

        //3.改为读状态
        bf.flip();
        System.out.println("==========flip============");
        System.out.println(bf.position());
        System.out.println(bf.limit());
        System.out.println(bf.capacity());

        //4.获取缓冲区数据
        final byte[] bytes = new byte[bf.limit()];
        bf.get(bytes);
        System.out.println("==========get============");
        System.out.println(new String(bytes,0,bytes.length));
        System.out.println(bf.position());
        System.out.println(bf.limit());
        System.out.println(bf.capacity());

        //5.重置读状态
        bf.rewind();
        System.out.println("==========rewind============");
        System.out.println(bf.position());
        System.out.println(bf.limit());
        System.out.println(bf.capacity());

        //6.清除数据标识
        bf.clear();
        System.out.println("==========clear============");
        System.out.println(bf.position());
        System.out.println(bf.limit());
        System.out.println(bf.capacity());
    }
}
```

![](https://tinaxiawuhao.github.io/post-images/1631867929854.png)

### 直接缓冲区与非直接缓冲区：

非直接缓冲区：通过 allocate() 方法分配缓冲区，将缓冲区建立在 JVM 的内存中
直接缓冲区：通过 allocateDirect() 方法分配直接缓冲区，将缓冲区建立在物理内存中。可以提高效率

 

1. 字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在机 此缓冲区上执行本机 I/O 操作。也就是说，在每次调用基础操作系统的一个本机 I/O 操作之前（或之后），
2. 虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。
3. 直接字节缓冲区可以通过调用此类的 allocateDirect() 工厂方法 来创建。此方法返回的 缓冲区进行分配和取消分配所需成本通常高于非直接缓冲区 。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的本机 I/O 操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好处时分配它们。
4. 直接字节缓冲区还可以过 通过FileChannel 的 map() 方法 将文件区域直接映射到内存中来创建 。该方法返回MappedByteBuffer 。Java 平台的实现有助于通过 JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区中的某个缓冲区实例指的是不可访问的内存区域，则试图访问该区域不会更改该缓冲区的内容，并且将会在访问期间或稍后的某个时间导致抛出不确定的异常。
5. 字节缓冲区是直接缓冲区还是非直接缓冲区可通过调用其 isDirect() 方法来确定。提供此方法是为了能够在性能关键型代码中执行显式缓冲区管理。

非直接缓冲区：

![](https://tinaxiawuhao.github.io/post-images/1632797370169.png)

直接缓冲区：

![](https://tinaxiawuhao.github.io/post-images/1632797378679.png)

### 通道（Channel ）

　　通道表示打开到 IO 设备(例如：文件、套接字)的连接。若需要使用 NIO 系统，需要获取用于连接 IO 设备的通道以及用于容纳数据的缓冲区。然后操作缓冲区，对数据进行处理。
　　Channel相比IO中的Stream更加高效，可以异步双向传输，但是必须和buffer一起使用。

主要实现类

```java
FileChannel，读写文件中的数据。
SocketChannel，通过TCP读写网络中的数据。
ServerSockectChannel，监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。
DatagramChannel，通过UDP读写网络中的数据。
```


### Channel聚集(gather)写入

聚集写入（ Gathering Writes）是指将多个 Buffer 中的数据“聚集”到 Channel。 特别注意：按照缓冲区的顺序，写入 position 和 limit 之间的数据到 Channel 。 

![](https://tinaxiawuhao.github.io/post-images/1631867939574.png)

### Channel分散(scatter)读取

分散读取（ Scattering Reads）是指从 Channel 中读取的数据“分散” 到多个 Buffer 中。 特别注意：按照缓冲区的顺序，从 Channel 中读取的数据依次将 Buffer 填满。

![](https://tinaxiawuhao.github.io/post-images/1631867947387.png)

### “零拷贝”（FileChannel的transferTo和transferFrom）

> Java NIO中提供的FileChannel拥有transferTo和transferFrom两个方法，可直接把FileChannel中的数据拷贝到另外一个Channel，或者直接把另外一个Channel中的数据拷贝到FileChannel。该接口常被用于高效的网络/文件的数据传输和大文件拷贝。在操作系统支持的情况下，通过该方法传输数据并不需要将源数据从内核态拷贝到用户态，再从用户态拷贝到目标通道的内核态，同时也避免了两次用户态和内核态间的上下文切换，也即使用了“零拷贝”，所以其性能一般高于Java IO中提供的方法。

### 代码案例

```java
/**
 * @author wuhao
 * @desc 本地io:
 * 　　 FileInputStreanm/FileOutputStream
 * 　　RandomAccessFile
 * 网络io:
 * 　　Socket
 * 　　ServerSocket
 * 　　DatagramSocket
 * @date 2021-09-17 15:22:26
 */
public class ChannelTest {

    //利用通道完成文件的复制,非直接缓冲区
    @Test
    @SneakyThrows
    public void test(){
        FileInputStream fis = new FileInputStream("D:\\1.jpg");
        FileOutputStream fos = new FileOutputStream("D:\\2.jpg");
        //获取通道
        FileChannel inChannel = fis.getChannel();
        FileChannel outChannel = fos.getChannel();
        //分配指定大小缓存区
        ByteBuffer buff = ByteBuffer.allocate(1024);// position 0 ,limit 1024
        //将通道的数据存入缓存区
        while(inChannel.read(buff)!=-1){// position 1024 ,limit 1024 ,相当于put
            //切换读模式
            buff.flip();//position 0 ,limit 1024
            //将缓存去的数据写入通道
            outChannel.write(buff);//position 1024 ,limit 1024,相当于get
            //清空缓冲区
            buff.clear();//position 0 ,limit 1024
        }
        outChannel.close();
        inChannel.close();
        fis.close();
        fos.close();
    }

    //利用通道完成文件的复制,直接缓冲区,利用物理内存映射文件
    //会出现文件复制完了，但程序还没结束，原因是JVM资源还在用，当垃圾回收机制回收之后程序就会结束,不稳定
    @Test
    @SneakyThrows
    public void test1(){
        FileChannel inChannel = FileChannel.open(Paths.get("D:\\1.jpg"), StandardOpenOption.READ);
        FileChannel outChannel = FileChannel.open(Paths.get("D:\\4.jpg"), StandardOpenOption.READ,StandardOpenOption.WRITE,StandardOpenOption.CREATE);
        //内存映射文件
        MappedByteBuffer inMapBuff = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, inChannel.size());//==allocateDirect
        MappedByteBuffer outMapBuff = outChannel.map(FileChannel.MapMode.READ_WRITE, 0, inChannel.size());
        byte[] by = new byte[inMapBuff.limit()];
        inMapBuff.get(by);
        outMapBuff.put(by);
        outChannel.close();
        inChannel.close();
    }

    /**
     * 从源信道读取字节到这个通道的文件中。如果源通道的剩余空间小于 count 个字节，则所传输的字节数要小于请求的字节数。这种方法可能比从源通道读取并写入此通道的简单循环更有效率。
     * @param SRC 源通道
     * @param position 调动开始的文件内的位置，必须是非负的
     * @param count 要传输的最大字节数，必须是非负
     * @return 传输文件的大小（单位字节），可能为零，
     * public abstract long transferFrom(ReadableByteChannel src, long position, long count)　throws IOException;
     */
    //复制图片，利用直接缓存区
    @Test
    @SneakyThrows
    public void test2(){
        FileChannel inChannel = FileChannel.open(Paths.get("D:\\1.jpg"), StandardOpenOption.READ);
        FileChannel outChannel = FileChannel.open(Paths.get("D:\\2.jpg"), StandardOpenOption.READ, StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        outChannel.transferFrom(inChannel, 0, inChannel.size());
        inChannel.close();
        outChannel.close();
    }

    /**
     * 将字节从这个通道的文件传输到给定的可写字节通道。
     * @param position 调动开始的文件内的位置，必须是非负的
     * @param count 要传输的最大字节数，必须是非负
     * @param target 目标通道
     * @return 传输文件的大小（单位字节），可能为零，
     * public abstract long transferTo(long position, long count, WritableByteChannel target) throws IOException;
     */
    //复制图片，利用直接缓存区
    @Test
    @SneakyThrows
    public void test3(){
        FileChannel inChannel = FileChannel.open(Paths.get("D:\\1.jpg"), StandardOpenOption.READ);
        FileChannel outChannel = FileChannel.open(Paths.get("D:\\3.jpg"), StandardOpenOption.READ, StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        inChannel.transferTo(0, inChannel.size(), outChannel);
        inChannel.close();
        outChannel.close();
    }

    // 分散读取聚集写入实现文件复制

    @Test
    @SneakyThrows
    public void test4(){
        RandomAccessFile randomAccessFile = null;
        RandomAccessFile randomAccessFile1 = null;
        FileChannel inChannel = null;
        FileChannel outChannel = null;
        try {
            randomAccessFile = new RandomAccessFile(new File("d:\\old.txt"), "rw");
            randomAccessFile1 = new RandomAccessFile(new File("d:\\new.txt"), "rw");
            inChannel = randomAccessFile.getChannel();
            outChannel = randomAccessFile1.getChannel();
            // 分散为三个bytebuffer读取,capcity要设置的足够大，不然如果文件太大，会导致复制的内容不完整
            ByteBuffer byteBuffer1 = ByteBuffer.allocate(1024);
            ByteBuffer byteBuffer2 = ByteBuffer.allocate(1024);
            ByteBuffer byteBuffer3 = ByteBuffer.allocate(10240);
            ByteBuffer[] bbs = new ByteBuffer[]{byteBuffer1,byteBuffer2,byteBuffer3};

            inChannel.read(bbs);// 分散读取

            // 切换为写入模式
            for (int i = 0; i < bbs.length; i++) {
                bbs[i].flip();
            }

            outChannel.write(bbs);

        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```

