---
title: '指令重排序'
date: 2021-04-30 14:23:15
tags: [java]
published: true
hideInList: false
feature: /post-images/J4bdQ7uDb.png
isTop: false
---

### 数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。数据依赖分下列三种类型：

|名称|代码示例|说明|
|---|---|---|
| 写后读 | a = 1;b = a; | 写一个变量之后，再读这个位置。 |
| 写后写 | a = 1;a = 2; | 写一个变量之后，再写这个变量。 |
| 读后写 | a = b;b = 1; | 读一个变量之后，再写这个变量。 |


上面三种情况，只要重排序两个操作的执行顺序，程序的执行结果将会被改变。

前面提到过，编译器和处理器可能会对操作做重排序。编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序。

注意，这里所说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑。

### as-if-serial 语义

as-if-serial 语义的意思指：不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器，runtime 和处理器都必须遵守 as-if-serial 语义。

为了遵守 as-if-serial 语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作可能被编译器和处理器重排序。为了具体说明，请看下面计算圆面积的代码示例：

```java
double pi  = 3.14;    //A
double r   = 1.0;     //B
double area = pi * r * r; //C  
```



上面三个操作的数据依赖关系如下图所示：

![](https://tinaxiawuhao.github.io/post-images/1619332089100.png)

如上图所示，A 和 C 之间存在数据依赖关系，同时 B 和 C 之间也存在数据依赖关系。因此在最终执行的指令序列中，C 不能被重排序到 A 和 B 的前面（C 排到 A 和 B 的前面，程序的结果将会被改变）。但 A 和 B 之间没有数据依赖关系，编译器和处理器可以重排序 A 和 B 之间的执行顺序。下图是该程序的两种执行顺序：

![](https://tinaxiawuhao.github.io/post-images/1619332096691.webp)

as-if-serial 语义把单线程程序保护了起来，遵守 as-if-serial 语义的编译器，runtime 和处理器共同为编写单线程程序的程序员创建了一个幻觉：单线程程序是按程序的顺序来执行的。as-if-serial 语义使单线程程序员无需担心重排序会干扰他们，也无需担心内存可见性问题。

### 程序顺序规则

根据 happens- before 的程序顺序规则，上面计算圆的面积的示例代码存在三个 happens- before 关系：

1. A happens- before B；
2. B happens- before C；
3. A happens- before C；

这里的第3个 happens- before 关系，是根据 happens- before 的传递性推导出来的。

这里 A happens- before B，但实际执行时 B 却可以排在 A 之前执行（看上面的重排序后的执行顺序）。在第一章提到过，如果 A happens- before B，JMM 并不要求 A 一定要在 B 之前执行。JMM 仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。这里操作 A 的执行结果不需要对操作 B 可见；而且重排序操作 A 和操作 B 后的执行结果，与操作 A 和操作 B 按 happens- before 顺序执行的结果一致。在这种情况下，JMM 会认为这种重排序并不非法（not illegal），JMM 允许这种重排序。

在计算机中，软件技术和硬件技术有一个共同的目标：在不改变程序执行结果的前提下，尽可能的开发并行度。编译器和处理器遵从这一目标，从 happens- before 的定义我们可以看出，JMM 同样遵从这一目标。

### 重排序对多线程的影响

现在让我们来看看，重排序是否会改变多线程程序的执行结果。请看下面的示例代码：

```java
class ReorderExample {
    int a = 0;
    boolean flag = false;

    public void writer() {
        a = 1;                   //1
        flag = true;             //2
    }

    Public void reader() {
        if (flag) {                //3
            int i =  a * a;        //4
            ……
        }
    }
}  
```



flag 变量是个标记，用来标识变量 a 是否已被写入。这里假设有两个线程 A 和 B，A 首先执行writer() 方法，随后 B 线程接着执行 reader() 方法。线程B在执行操作4时，能否看到线程 A 在操作1对共享变量 a 的写入？

答案是：不一定能看到。

由于操作1和操作2没有数据依赖关系，编译器和处理器可以对这两个操作重排序；同样，操作3和操作4没有数据依赖关系，编译器和处理器也可以对这两个操作重排序。让我们先来看看，当操作1和操作2重排序时，可能会产生什么效果？请看下面的程序执行时序图：
![](https://tinaxiawuhao.github.io/post-images/1619332109512.webp)

如上图所示，操作1和操作2做了重排序。程序执行时，线程A首先写标记变量 flag，随后线程 B 读这个变量。由于条件判断为真，线程 B 将读取变量a。此时，变量 a 还根本没有被线程 A 写入，在这里多线程程序的语义被重排序破坏了！

※注：本文统一用红色的虚箭线表示错误的读操作，用绿色的虚箭线表示正确的读操作。

下面再让我们看看，当操作3和操作4重排序时会产生什么效果（借助这个重排序，可以顺便说明控制依赖性）。下面是操作3和操作4重排序后，程序的执行时序图：

![](https://tinaxiawuhao.github.io/post-images/1619332116836.webp)

在程序中，操作3和操作4存在控制依赖关系。当代码中存在控制依赖性时，会影响指令序列执行的并行度。为此，编译器和处理器会采用猜测（Speculation）执行来克服控制相关性对并行度的影响。以处理器的猜测执行为例，执行线程 B 的处理器可以提前读取并计算 a*a，然后把计算结果临时保存到一个名为重排序缓冲（reorder buffer ROB）的硬件缓存中。当接下来操作3的条件判断为真时，就把该计算结果写入变量i中。

从图中我们可以看出，猜测执行实质上对操作3和4做了重排序。重排序在这里破坏了多线程程序的语义！

在单线程程序中，对存在控制依赖的操作重排序，不会改变执行结果（这也是 as-if-serial 语义允许对存在控制依赖的操作做重排序的原因）；但在多线程程序中，对存在控制依赖的操作重排序，可能会改变程序的执行结果。

### 编译器指令重排

下面我们简单看一个编译器重排的例子：

线程 1 线程 2

```bash
1：x2 = a ; 3: x1 = b ;

2: b = 1; 4: a = 2 ;
```



两个线程同时执行，分别有1、2、3、4四段执行代码，其中1、2属于线程1 ， 3、4属于线程2 ，从程序的执行顺序上看，似乎不太可能出现x1 = 1 和x2 = 2 的情况，但实际上这种情况是有可能发现的，因为如果编译器对这段程序代码执行重排优化后，可能出现下列情况

线程 1 线程 2

```bash
2: b = 1; 4: a = 2 ;

1：x2 = a ; 3: x1 = b ;
```

这种执行顺序下就有可能出现`x1 = 1` 和`x2 = 2` 的情况，这也就说明在多线程环境下，由于编译器优化重排的存在，两个线程中使用的变量能否保证一致性是无法确定的。

源代码和Runtime时执行的代码很可能不一样，这是因为编译器、处理器常常会为了追求性能对改变执行顺序。然而改变顺序执行很危险，很有可能使得运行结果和预想的不一样，特别是当重排序共享变量时。

 从源代码到Runtime需要经过三步的重排序：

![](https://tinaxiawuhao.github.io/post-images/1619515556691.png)

## 编译器重排序

 为了提高性能，在不改变单线程的执行结果下，可以改变语句执行顺序。

 比如尽可能的减少寄存器的读写次数，充分利用局部性。像下面这段代码这样，交替的读x、y，会导致寄存器频繁的交替存储x和y，最糟的情况下寄存器要存储3次x和3次y。如果能让x的一系列操作一块做完，y的一块做完，理想情况下寄存器只需要存储1次x和1次y。

```javascript
//优化前
int x = 1;
int y = 2;
int a1 = x * 1;
int b1 = y * 1;
int a2 = x * 2;
int b2 = y * 2;
int a3 = x * 3;
int b3 = y * 3;

//优化后
int x = 1;
int y = 2;
int a1 = x * 1;
int a2 = x * 2;
int a3 = x * 3;
int b1 = y * 1;
int b2 = y * 2;
int b3 = y * 3;
```

## 指令重排序

 指令重排序是处理器层面做的优化。处理器在执行时往往会因为一些限制而等待，如访存的地址不在cache中发生miss，这时就需要到内存甚至外存去取，然而内存和外区的读取速度比CPU执行速度慢得多。

 早期处理器是顺序执行(in-order execution)的，在内存、外存读取数据这段时间，处理器就一直处于等待状态。现在处理器一般都是乱序执行(out-of-order execution)，处理器会在等待数据的时候去**执行其他已准备好的操作**，不会让处理器一直等待。

 满足乱序执行的条件：

1. 该缓存的操作数缓存好
2. 有空闲的执行单元

 对于下面这段汇编代码，操作1如果发生cache miss，则需要等待读取内存外存。看看有没有能优先执行的指令，操作2依赖于操作1，不能被优先执行，操作3不依赖1和2，所以能优先执行操作3。 所以实际执行顺序是3>1>2

```javascript
LDR R1, [R0];//操作1
ADD R2, R1, R1;//操作2
ADD R3, R4, R4;//操作3
```

## 内存系统重排序

 由于处理器有读、写缓存区，*写缓存区没有及时刷新到内存*，造成其他处理器读到的值不是最新的，使得**处理器执行的读写操作与内存上反应出的顺序不一致**。

 如下面这个例子，可能造成处理器A读到的b=0，处理器B读到的a=0。A1写a=1先写到处理器A的写缓存区中，此时内存中a=0。如果这时处理器B从内存中读a，读到的将是0。

 以处理器A来说，处理器A执行的顺序是A1>A2>A3，但是由于写缓存区没有及时刷新到内存，所以实际顺序为A2>A1>A3。

```javascript
初始化：
a = 0;
b = 0;

处理器A执行
a = 1; //A1
read(b); //A2

处理器B执行
b = 2; //B1
read(a); //B2
```

![](https://tinaxiawuhao.github.io/post-images/1619515566926.png)

## 阻止重排序

 不论哪种重排序都可能造成共享变量中线程间不可见，这会改变程序运行结果。所以需要禁止对那些要求可见的共享变量重排序。

- 阻止编译重排序：禁止编译器在某些时候重排序。
- 阻止指令重排序和内存系统重排序：使用内存屏障或Lock前缀指令