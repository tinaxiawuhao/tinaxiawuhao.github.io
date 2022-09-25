---
title: 'OOM'
date: 2021-09-13 09:54:41
tags: [java]
published: true
hideInList: false
feature: /post-images/QvbaHSogu.png
isTop: false
---
### Stack overflow

```java
public class StackOverFlowErrorDemo {
    public static void main(String[] args) {
        StackOverFlowError();
    }

    private static void StackOverFlowError() {
        StackOverFlowError();
    }
}
```

```java
Exception in thread "main" java.lang.StackOverflowError
	at com.example.interview.oom.StackOverFlowErrorDemo.StackOverFlowError(StackOverFlowErrorDemo.java:14)
```

递归调用自身方法，不断向栈内压入栈帧，直到撑破栈空间，重复次数不确定

### OutOfMemoryError：Java heap space

```java
/**
 * @author wuhao
 * @desc -Xmx10m -Xms10m -XX:+PrintGCDetails
 * 最大堆内存10M,初始堆内存10M,打印GC详细信息
 * @date 2021-09-10 15:14:50
 */
public class JavaHeapSpaceDemo {
    public static void main(String[] args) {
//        String str= "java";
//        while (true){
//            str+=str+new Random().nextInt(111111)+new Random().nextInt(121312);
//            str.intern();
//        }
        //创建大对象
        byte[] bytes=new byte[20*1024*1024];
    }
}
```

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.example.interview.oom.JavaHeapSpaceDemo.main(JavaHeapSpaceDemo.java:18)
```

常量池，对象空间地址位于堆空间中，循环生成String或者生成超大对象会导致堆空间被占满溢出

### OutOfMemoryError：GC overhead limit exceeded

```java
/**
 * @author wuhao
 * @desc -Xmx20m -Xms10m -XX:+PrintGCDetails
 * 最大堆内存20M,初始堆内存10M,堆外内存(直接内存)5M,打印GC信息
 * GC overhead limit exceeded:超出GC开销限制
 * @date 2021-09-10 15:24:48
 */
public class GCOverHeadDemo {
    public static void main(String[] args) {
        int i=0;
        List<String> list=new ArrayList<>();
        while (true){
            list.add(String.valueOf(++i).intern());
        }
    }
}
```

```java
[Full GC (Ergonomics) [PSYoungGen: 2560K->0K(4608K)] [ParOldGen: 13581K->818K(10752K)] 16141K->818K(15360K), [Metaspace: 3626K->3626K(1056768K)], 0.0078165 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 4608K, used 219K [0x00000000ff980000, 0x0000000100000000, 0x0000000100000000)
  eden space 2560K, 8% used [0x00000000ff980000,0x00000000ff9b6f28,0x00000000ffc00000)
  from space 2048K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x0000000100000000)
  to   space 2048K, 0% used [0x00000000ffc00000,0x00000000ffc00000,0x00000000ffe00000)
 ParOldGen       total 10752K, used 818K [0x00000000fec00000, 0x00000000ff680000, 0x00000000ff980000)
  object space 10752K, 7% used [0x00000000fec00000,0x00000000fecccac0,0x00000000ff680000)
 Metaspace       used 3719K, capacity 4536K, committed 4864K, reserved 1056768K
  class space    used 406K, capacity 428K, committed 512K, reserved 1048576K
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.lang.Integer.toString(Integer.java:401)
	at java.lang.String.valueOf(String.java:3099)
	at com.example.interview.oom.GCOverHeadDemo.main(GCOverHeadDemo.java:17)
```

最大堆内存要大于初始堆内存给GC创造条件，如果两值相等就没有重新分配堆空间操作会直接爆出Java heap space异常

### OutOfMemoryError：Direct buffer memory

```java
import java.nio.ByteBuffer;
/**
 * @author wuhao
 * @desc -Xmx20m -Xms10m -XX:+PrintGCDetails -XX:MaxDirectMemorySize=5m
 * 最大堆内存20M,初始堆内存10M,堆外内存(直接内存)5M,打印GC信息
 * @date 2021-09-10 15:38:57
 */
public class DirectBufferMemoryDemo {
    public static void main(String[] args) {
        System.out.println("配置的DirectMemory"+(sun.misc.VM.maxDirectMemory()/(double)1024/1024)+"mb");
        ByteBuffer bf=ByteBuffer.allocateDirect(6*1024*1024);
    }
}
```

```java
[GC (Allocation Failure) [PSYoungGen: 2048K->504K(2560K)] 2048K->758K(9728K), 0.0008256 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
配置的DirectMemory5.0mb
[GC (System.gc()) [PSYoungGen: 908K->504K(2560K)] 1162K->862K(9728K), 0.0009251 secs] [Times: user=0.06 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 504K->0K(2560K)] [ParOldGen: 358K->801K(7168K)] 862K->801K(9728K), [Metaspace: 3500K->3500K(1056768K)], 0.0046698 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
	at java.nio.Bits.reserveMemory(Bits.java:694)
	at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
	at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
	at com.example.interview.oom.DirectBufferMemoryDemo.main(DirectBufferMemoryDemo.java:13)
Heap
 PSYoungGen      total 2560K, used 1068K [0x00000000ff980000, 0x00000000ffc80000, 0x0000000100000000)
  eden space 2048K, 52% used [0x00000000ff980000,0x00000000ffa8b200,0x00000000ffb80000)
  from space 512K, 0% used [0x00000000ffc00000,0x00000000ffc00000,0x00000000ffc80000)
  to   space 512K, 0% used [0x00000000ffb80000,0x00000000ffb80000,0x00000000ffc00000)
 ParOldGen       total 7168K, used 801K [0x00000000fec00000, 0x00000000ff300000, 0x00000000ff980000)
  object space 7168K, 11% used [0x00000000fec00000,0x00000000fecc86d0,0x00000000ff300000)
 Metaspace       used 4057K, capacity 4568K, committed 4864K, reserved 1056768K
  class space    used 446K, capacity 460K, committed 512K, reserved 1048576K
```

allocateDirect分配的字节缓冲区用中文叫做直接缓冲区（DirectByteBuffer），用allocate分配的ByteBuffer叫做堆字节缓冲区(HeapByteBuffer)..

其实根据类名就可以看出，HeapByteBuffer所创建的字节缓冲区就是在jvm堆中的，即内部所维护的java字节数组。而DirectByteBuffer是直接操作操作系统本地代码创建的内存缓冲数组（c、c++的数组）。

HeapByteBuffer底层其实是java的字节数组，而java字节数组是一个java对象，对象的内存是由jvm的堆进行管理的，那么不可避免的是GC时年轻代的eden、suvivor到老年代的各种复制以及回收。。。当字节数组比较小的时候还好说，如果是大对象，那么对于jvm的GC来说是一个很大的负担。。而使用DirectByteBuffer，则是把字节数组交给操作系统管理（堆外内存）

### OutOfMemoryError：Unable to create to native thread

```java
public class UnableToCreateToNativeThreadDemo {
   
    public static void main(String[] args) {
        for (int i = 0; ; i++) {
            System.out.println("================ i=" + i);
            new Thread(() -> {
                try {
                    Thread.sleep(Integer.MAX_VALUE);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "" + i).start();
        }
    }
}
```

linux一般用户可以最大创建1024个线程，不要在电脑上随意尝试

### meta space

```java
/**
 * @author wuhao
 * @desc -XX:MetaspaceSize=8m -XX:MaxMetaspaceSize=8m -XX:+PrintGCDetails
 * @date 2021-09-13 10:36:15
 */
public class MetaSpaceOOMDemo {
    static class OOMTest{

    }
    public static void main(String[] args) {
        int i=0;
        try {
            while (true){
                i++;
                Enhancer enhancer=new Enhancer();
                enhancer.setSuperclass(OOMTest.class);
                enhancer.setUseCache(false);
                enhancer.setCallback(new MethodInterceptor() {
                    @Override
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        return methodProxy.invokeSuper(o,args);
                    }
                });
                enhancer.create();
            }
        }catch (Throwable e){
            System.out.println("========="+i+"次后发生了异常");
        }

    }
}
```

```java
[Full GC (Last ditch collection) [PSYoungGen: 0K->0K(116224K)] [ParOldGen: 2016K->2016K(225792K)] 2016K->2016K(342016K), [Metaspace: 9911K->9911K(1058816K)], 0.0093402 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 116224K, used 3425K [0x00000000d6400000, 0x00000000de400000, 0x0000000100000000)
  eden space 115712K, 2% used [0x00000000d6400000,0x00000000d67584a0,0x00000000dd500000)
  from space 512K, 0% used [0x00000000de380000,0x00000000de380000,0x00000000de400000)
  to   space 5120K, 0% used [0x00000000dda00000,0x00000000dda00000,0x00000000ddf00000)
 ParOldGen       total 225792K, used 2016K [0x0000000082c00000, 0x0000000090880000, 0x00000000d6400000)
  object space 225792K, 0% used [0x0000000082c00000,0x0000000082df8128,0x0000000090880000)
 Metaspace       used 9942K, capacity 10090K, committed 10240K, reserved 1058816K
  class space    used 884K, capacity 913K, committed 1024K, reserved 1048576K
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
```





