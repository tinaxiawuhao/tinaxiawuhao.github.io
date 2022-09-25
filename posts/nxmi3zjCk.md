---
title: 'Java内存结构JVM'
date: 2021-04-26 10:49:02
tags: [java]
published: true
hideInList: false
feature: /post-images/nxmi3zjCk.png
isTop: false
---
## JVM内存模型

`内存空间(Runtime Data Area)`中可以按照是否线程共享分为两块，线程共享的是`方法区(Method Area)`和`堆(Heap)`，线程独享的是`Java虚拟机栈(Java Stack)`，`本地方法栈(Native Method Stack)`和`PC寄存器(Program Counter Register)`。具体参见下图：

![](https://tianxiawuhao.github.io/post-images/1619319113675.png)

###  虚拟机栈：

虚拟机栈是Java方法执行的内存模型，栈中存放着栈帧，每个栈帧分别对应一个被调用的方法，方法的调用过程对应栈帧在虚拟机中入栈到出栈的过程。

栈是线程私有的，也就是线程之间的栈是隔离的；当程序中某个线程开始执行一个方法时就会相应的创建一个栈帧并且入栈（位于栈顶），在方法结束后，栈帧出栈。

下图表示了一个Java栈的模型以及栈帧的组成：
![](https://tianxiawuhao.github.io/post-images/1619167296313.jpeg)

**栈帧**:是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈的栈元素。

每个栈帧中包括：

1. 局部变量表:用来存储方法中的局部变量（非静态变量、函数形参）。当变量为基本数据类型时，直接存储值，当变量为引用类型时，存储的是指向具体对象的引用。
2. 操作数栈:Java虚拟机的解释执行引擎被称为"基于栈的执行引擎"，其中所指的栈就是指操作数栈。
3. 指向运行时常量池的引用:存储程序执行时可能用到常量的引用。
4. 方法返回地址:存储方法执行完成后的返回地址。

每个线程有一个私有的栈，随着线程的创建而创建。栈里面存放着一种叫做“栈帧”的东西，每个方法会创建一个栈帧，栈帧中存放了局部变量表(基本数据类型和对象引用)、操作数栈、方法出口等信息。栈的大小可以固定也可以动态扩展。当栈调用深度大于`JVM`所允许的范围，会抛出`StackOverflowError`的错误，不过这个深度范围不是一个恒定的值，我们通过下面这段程序可以测试一下这个结果：

```java
// 栈溢出测试源码
ackage com.paddx.test.memory;
 
/**
 * Created by root on 2/28/17.
 */
public class StackErrorMock {
    private static int index = 1;
 
    public void call() {
        index++;
        call();
    }
 
    public static void main(String[] args) {
        StackErrorMock mock = new StackErrorMock();
        try {
            mock.call();
        } catch(Throwable e) {
            System.out.println("Stack deep: " + index);
            e.printStackTrace();
        }
    }
}
```



运行三次，可以看出每次栈的深度都是不一样的，输出结果如下：
![](https://tianxiawuhao.github.io/post-images/1619327479989.png)
![](https://tianxiawuhao.github.io/post-images/1619327486623.png)
![](https://tianxiawuhao.github.io/post-images/1619319135044.png)

查看三张结果图，可以看出每次的Stack deep值都有所不同。究其原因，就需要深入到`JVM`的源码中才能探讨，这里不作赘述。

虚拟机栈除了上述错误外，还有另一种错误，那就是当申请不到空间时，会抛出`OutOfMemoryError`。这里有一个小细节需要注意，catch捕获的是Throwable，而不是Exception，这是因为`StackOverflowError`和`OutOfMemoryError`都不属于Exception的子类。

### 本地方法栈：

这部分主要与虚拟机用到的Native方法相关，一般情况下，Java应用程序员并不需要关系这部分内容。

### PC寄存器：

PC寄存器，也叫程序计数器。`JVM`支持多个线程同时运行，每个线程都有自己的程序计数器。倘若当前执行的是`JVM`方法，则该寄存器中保存当前执行指令的地址；倘若执行的是native方法，则PC寄存器为空。

###  堆 ：

堆内存是`JVM`所有线程共享的部分，在虚拟机启动的时候就已经创建。所有的对象和数组都在堆上进行分配。这部分空间可通过`GC`进行回收。当申请不到空间时，会抛出`OutOfMemoryError`。下面我们简单的模拟一个堆内存溢出的情况：

```java
package com.paddx.test.memory;
 
import java.util.ArrayList;
import java.util.List;
 
/**
 * Created by root on 2/28/17.
 */
public class HeapOomMock {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<byte[]>();
        int i = 0;
        boolean flag = true;
        while(flag) {
            try {
                i++;
                list.add(new byte[1024 * 1024]); // 每次增加1M大小的数组对象
            }catch(Throwable e) {
                e.printStackTrace();
                flag = false;
                System.out.println("Count = " + i); // 记录运行的次数
            }
        }
    }
}
```



首先配置运行时虚拟机的启动参数：
![](https://tianxiawuhao.github.io/post-images/1619319145433.png)

然后运行代码，输出结果如下：

![](https://tianxiawuhao.github.io/post-images/1619319151622.png)

注意，这里我们指定了堆内存的大小为16M，所以这个地方显示的Count=13(这个数字不是固定的)，至于为什么会是13或其他数字，需要根据GC日志来判断。

### 方法区：

   方法区是一块所有线程共享的内存逻辑区域，在`JVM`中只有一个方法区，用来存储一些线程可共享的内容，它是线程安全的，多个线程同时访问方法区中同一个内容时，只能有一个线程装载该数据，其它线程只能等待。

   方法区可存储的内容有：类的全路径名、类的直接超类的权全限定名、类的访问修饰符、类的类型（类或接口）、类的直接接口全限定名的有序列表、常量池（字段，方法信息，静态变量，类型引用（class））等。

关于方法区的内存溢出问题会在下文中详细讨论。

### 方法区和元空间

Java1.7及之前的虚拟机在运行中会把它所管理的内存分为如上图的若干数据区域。

方法区用于存储已被虚拟机加载的类信息、常量、静态变量、动态生成的类等数据。实际上在Java虚拟机的规范中方法区是堆中的一个逻辑部分，但是它却拥有一个叫做非堆(Non-Heap)的别名。

对于方法区的实现，不同虚拟机中策略也不同。以我们常用的HotSpot虚拟机为例，其设计团队使用永久带来实现方法区，并把GC的分代收集扩展至永久带。这样设计的好处就是能够省去专门为方法区编写内存管理的代码。但是在实际的场景中，这样的实现并不是一个好的方式，因为永久带有MAX上限，所以这样做会更容易遇到内存溢出问题。

关于方法区的GC回收，Java虚拟机规范并没有严格的限制。虚拟机在实现中可以自由选择是否实现垃圾回收。主要原因还是方法区内存回收的效果比较难以令人满意，方法区的内存回收主要是针对常量池(1.7已经将常量池逐步移除方法区)以及类型的卸载，但是类型卸载在实际场景中的条件相当苛刻。

另外还需要注意的是在HotSpot虚拟机中永久带和堆虽然相互隔离，但是他们的物理内存是连续的。而且老年代和永久带的垃圾收集器进行了捆绑，因此无论谁满了都会触发永久带和老年的GC。

**因此在Java1.8中，HotSpot虚拟机已经将方法区(永久带)移除，取而代之的就是元空间。**

![](https://tianxiawuhao.github.io/post-images/1660658940224.jpg)

Java1.8的运行时数据区域如图所示。方法区已经不见了踪影，多出来的是叫做元数据区的区域。

元空间在1.8中不在与堆是连续的物理内存，而是改为使用本地内存(Native memory)。元空间使用本地内存也就意味着只要本地内存足够，就不会出现OOM的错误。

## PermGen(永久代)

绝大部分Java程序员应该都见过`“java.lang.OutOfMemoryError: PremGen space”`异常。这里的`“PermGen space”`其实指的就是方法区。不过方法区和`“PermGen space”`又有着本质的区别。前者是`JVM`的规范，而后者则是`JVM`规范的一种实现，并且只有`HotSpot`才有`“PermGen space”`，而对于其他类型的虚拟机，如`JRockit(Oracle)`、`J9(IBM)`并没有`“PermGen space”`。由于方法区主要存储类的相关信息，所以对于动态生成类的情况比较容易出现永久代的内存溢出。最典型的场景就是，在`JSP`页面比较多的情况，容易出现永久代内存溢出。我们现在通过动态生成类来模拟`“PermGen space”`的内存溢出：

```java
package com.paddx.test.memory;
 
import java.io.File;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.ArrayList;
import java.util.List;
 
public class PermGenOomMock {
	public static void main(String[] args) {
		URL url = null;
		List<ClassLoader> classLoaderList = new ArrayList<ClassLoader>();
		try {
			url = new File("/tmp").toURI().toURL();
			URL[] urls = {url};
			while(true) {
				ClassLoader loader = new URLClassLoader(urls);
				classLoaderList.add(loader);
				loader.loadClass("com.paddx.test.memory.Test");
			}
		}catch(Exception e) {
			e.printStackTrace();
		}
	}
}
```

```java
package com.paddx.test.memory;
 
public class Test {}
```

运行结果如下：

![](https://tianxiawuhao.github.io/post-images/1619319159980.png)

本例中使用的`JDK`版本是1.7，指定的`PermGen`区的大小为`8M`。通过每次生成不同`URLClassLoader`对象加载Test类，从而生成不同的类对象，这样就能看到我们熟悉的`“java.lang.OutOfMemoryError: PermGen space”`异常了。这里之所以采用`JDK 1.7`，是因为在`JDK 1.8`中，`HotSpot`已经没有`“PermGen space”`这个区间了，取而代之是一个叫做`Metaspace(元空间)`的东西。下面我们就来看看`Metaspace`与`PermGen space`的区别。

### Metaspace(元空间)

其实，移除永久代的工作从`JDK 1.7`就开始了。`JDK 1.7`中，存储在永久代的部分数据就已经转移到`Java Heap`或者`Native Heap`。但永久代仍存在于`JDK 1.7`中，并没有完全移除，譬如符号引用(Symbols)转移到了native heap；字面量(interned strings)转移到了Java heap；类的静态变量(class statics)转移到了Java heap。我们可以通过一段程序来比较`JDK 1.6`、`JDK 1.7`与`JDK 1.8`的区别，以字符串常量为例：

```java
package com.paddx.test.memory;
 
import java.util.ArrayList;
import java.util.List;
 
public class StringOomMock {
	static String base = "string";
	public static void main(String[] args) {
		List<String> list = new ArrayList<String>();
		for (int i = 0; i < Integer.MAX_VALUE; i++) {
			String str = base + base;
			base = str;
			list.add(str.intern());
		}
	}
}
```

这段程序以2的指数级不断的生成新的字符串，这样可以比较快速的消耗内存。我们通过`JDK 1.6`、`JDK 1.7`和`JDK 1.8`分别运行：

`JDK 1.6`的运行结果：

![](https://tianxiawuhao.github.io/post-images/1619319168920.png)

`JDK 1.7`的运行结果：

![](https://tianxiawuhao.github.io/post-images/1619319175662.png)

`JDK 1.8`的运行结果：

![](https://tianxiawuhao.github.io/post-images/1619319183606.png)

从上述结果可以看出，`JDK 1.6`下，会出现`“PermGen space”`的内存溢出，而在`JDK 1.7`和`JDK 1.8`中，会出现堆内存溢出，并且`JDK 1.8`中参数`PermSize`和`MaxPermSize`已经失效。因此，可以大致验证`JDK 1.7`和`JDK 1.8`中将字符串常量由永久代转移到堆中，并且`JDK 1.8`中已经不存在永久代的结论。现在我们来看一看元空间到底是一个什么东西？

`JDK1.8`对`JVM`架构的改造将类元数据放到本地内存中，另外，将常量池和静态变量放到Java堆里。`HotSpot VM`将会为类的元数据明确分配和释放本地内存。在这种架构下，类元信息就突破了原来`-XX:MaxPermSize`的限制，现在可以使用更多的本地内存。这样就从一定程度上解决了原来在运行时生成大量类造成经常`Full GC`问题，如运行时使用反射、代理等。所以升级以后Java堆空间可能会增加。

元空间的本质和永久代类似，都是对`JVM`规范中方法区的实现。不过元空间与永久代之间的最大区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制，但可以通过以下参数指定元空间的大小：

`-XX:MetaspaceSize`，初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对改值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过`MaxMetaspaceSize`时，适当提高该值。

`-XX:MaxMetaspaceSize`，最大空间，默认是没有限制的。

除了上面的两个指定大小的选项外，还有两个与`GC`相关的属性：

`-XX:MinMetaspaceFreeRatio`，在`GC`之后，最小的`Metaspace`剩余空间容量的百分比，减少为分配空间所导致的垃圾收集。

`-XX:MaxMetaspaceFreeRatio`，在`GC`之后，最大的`Metaspace`剩余空间容量的百分比，减少为释放空间所导致的垃圾收集。

现在我们在`JDK 1.8`重新运行一下上面第二部分`(PermGen(永久代))`的代码，不过这次不再指定`PermSize`和`MaxPermSize`。而是制定`MetaspaceSize`和`MaxMetaspaceSize`的大小。输出结果如下：

![](https://tianxiawuhao.github.io/post-images/1619319190434.png)

从输出结果，我们可以看出，这次不再出现永久代溢出，而是出现元空间的溢出。

**四、总结**

通过上面的分析，大家应该大致了解了`JVM`的内存划分，也清楚了`JDK 1.8`中永久代向元空间的转换。不过大家应该有一个疑问，就是为什么要做这个转换？以下为大家总结几点原因：

1. 字符串在永久代中，容易出现性能问题和内存溢出。
2. 类及方法的信息等比较难确定大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易出现老年代溢出。
3. 永久代会为`GC`带来不必要的复杂度，并且回收效率偏低。
4. Oracle可能会将`HotSpot`与`JRockit`合二为一。