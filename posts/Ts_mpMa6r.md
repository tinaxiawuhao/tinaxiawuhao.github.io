---
title: 'netty零拷贝'
date: 2021-10-13 14:11:02
tags: [netty]
published: true
hideInList: false
feature: /post-images/Ts_mpMa6r.png
isTop: false
---
零拷贝的应用程序要求内核（kernel）直接将数据从磁盘文件拷贝到套接字（Socket），而无须通过应用程序。零拷贝不仅提高了应用程序的性能，而且减少了内核和用户模式见上下文切换。 

### 数据传输：传统方法

从文件中读取数据，并将数据传输到网络上的另一个程序的场景：从下图可以看出，拷贝的操作需要4次用户模式和内核模式之间的上下文切换，而且在操作完成前数据被复制了4次。(DMA：直接内存拷贝)

![](https://tinaxiawuhao.github.io/post-images/1634105723930.png)


![](https://tinaxiawuhao.github.io/post-images/1634105729894.png)

 
![](https://tinaxiawuhao.github.io/post-images/1634107211607.png)

 

从磁盘中copy放到一个内存buf中，然后将buf通过socket传输给用户,下面是伪代码实现：

read(file, tmp_buf, len);
write(socket, tmp_buf, len);

从图中可以看出文件经历了4次copy过程：

1.首先，调用read方法，文件从user模式拷贝到了kernel模式；（用户模式->内核模式的上下文切换，在内部发送sys_read() 从文件中读取数据，存储到一个内核地址空间缓存区中）

2.之后CPU控制将kernel模式数据拷贝到user模式下；（内核模式-> 用户模式的上下文切换，read()调用返回，数据被存储到用户地址空间的缓存区中）

3.调用write时候，先将user模式下的内容copy到kernel模式下的socket的buffer中（用户模式->内核模式，数据再次被放置在内核缓存区中，send（）套接字调用）

4.最后将kernel模式下的socket buffer的数据copy到网卡设备中；（send套接字调用返回）

从图中看2，3两次copy是多余的，数据从kernel模式到user模式走了一圈，浪费了2次copy。

 

### 数据传输：mmap 优化

  mmap 通过内存映射，将文件映射到内核缓冲区，同时，用户空间可以共享内核空间的数据。这样，在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数。如下图：

![mmap 流程](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy80MjM2NTUzLWM1ZWEwMGI3OGUxYjkzZmQucG5n?x-oss-process=image/format,png)

   如上图，user buffer 和 kernel buffer 共享 index.html。如果你想把硬盘的 index.html 传输到网络中，再也不用拷贝到用户空间，再从用户空间拷贝到 Socket 缓冲区。

  现在，你只需要从内核缓冲区拷贝到 Socket 缓冲区即可，这将减少一次内存拷贝（从 4 次变成了 3 次），但不减少上下文切换次数。

### 数据传输：零拷贝方法

从传统的场景看，会注意到上图，第2次和第3次拷贝根本就是多余的。应用程序只是起到缓存数据被将传回到套接字的作用而已，别无他用。

应用程序使用zero-copy来请求kernel直接把disk的数据传输到socket中，而不是通过应用程序传输。zero-copy大大提高了应用程序的性能，并且减少了kernel和user模式的上下文切换。

数据可以直接从read buffer 读缓存区传输到套接字缓冲区，也就是省去了将操作系统的read buffer 拷贝到程序的buffer，以及从程序buffer拷贝到socket buffer的步骤，直接将read buffer拷贝到socket buffer。JDK NIO中的的`transferTo()` 方法就能够让您实现这个操作，这个实现依赖于操作系统底层的sendFile（）实现的：

 

```java
public void transferTo(long position, long count, WritableByteChannel target);
```

底层调用sendFile方法：

```java
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

![](https://tinaxiawuhao.github.io/post-images/1634105775492.png)

 

 

![](https://tinaxiawuhao.github.io/post-images/1634105782639.png)

使用了zero-copy技术后，整个过程如下：

1.transferTo()方法使得文件的内容直接copy到了一个read buffer（kernel buffer）中

2.然后数据（kernel buffer）copy到socket buffer中

3.最后将socket buffer中的数据copy到网卡设备（protocol engine）中传输；

这个显然是一个伟大的进步：这里上下文切换从4次减少到2次，同时把数据copy的次数从4次降低到3次；

**但是这是zero-copy么，答案是否定的；**

 

linux 2.1 内核开始引入了sendfile函数，用于将文件通过socket传输。

```java
sendfile(socket, file, len);
```

该函数通过一次调用完成了文件的传输。 该函数通过一次系统调用完成了文件的传输，减少了原来read/write方式的模式切换。此外更是减少了数据的copy，sendfile的详细过程如图：

 

![](https://tinaxiawuhao.github.io/post-images/1634107245108.png)

通过sendfile传送文件只需要一次系统调用，当调用sendfile时：

1.首先通过DMA将数据从磁盘读取到kernel buffer中

2.然后将kernel buffer数据拷贝到socket buffer中

3.最后将socket buffer中的数据copy到网卡设备中（protocol buffer）发送；

sendfile与read/write模式相比，少了一次copy。但是从上述过程中发现从kernel buffer中将数据copy到socket buffer是没有必要的；

Linux2.4 内核对sendfile做了改进，如图：

![](https://tinaxiawuhao.github.io/post-images/1634107253678.png)

 

改进后的处理过程如下：

1. 将文件拷贝到kernel buffer中；(DMA引擎将文件内容copy到内核缓存区)
2. 向socket buffer中追加当前要发生的数据在kernel buffer中的位置和偏移量；
3. 根据socket buffer中的位置和偏移量直接将kernel buffer的数据copy到网卡设备（protocol engine）中；

从图中看到，linux 2.1内核中的 “**数据被copy到socket buffe**r”的动作，在Linux2.4 内核做了优化，取而代之的是只包含关于数据的位置和长度的信息的描述符被追加到了socket buffer 缓冲区中。**DMA引擎直接把数据从内核缓冲区传输到协议引擎**（protocol engine），从而消除了最后一次CPU copy。经过上述过程，数据只经过了2次copy就从磁盘传送出去了。这个才是真正的Zero-Copy(这里的零拷贝是针对kernel来讲的，数据在kernel模式下是Zero-Copy)。

 

正是Linux2.4的内核做了改进，Java中的TransferTo()实现了Zero-Copy,如下图：

![](https://tinaxiawuhao.github.io/post-images/1634105813030.png)

Zero-Copy技术的使用场景有很多，比如Kafka, 又或者是Netty等，可以大大提升程序的性能。

首先我们说零拷贝，是从操作系统的角度来说的。因为内核缓冲区之间，没有数据是重复的（只有 kernel buffer 有一份数据，sendFile 2.1 版本实际上有 2 份数据，算不上零拷贝）。例如我们刚开始的例子，内核缓存区和 Socket 缓冲区的数据就是重复的。

  而零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如更少的上下文切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算。

 mmap 和 sendFile 的区别。

1. mmap 适合小数据量读写，sendFile 适合大文件传输。
2. mmap 需要 4 次上下文切换，3 次数据拷贝；sendFile 需要 3 次上下文切换，最少 2 次数据拷贝。
3. sendFile 可以利用 DMA 方式，减少 CPU 拷贝，mmap 则不能（必须从内核拷贝到 Socket 缓冲区）。

在这个选择上：rocketMQ 在消费消息时，使用了 mmap。kafka 使用了 sendFile。