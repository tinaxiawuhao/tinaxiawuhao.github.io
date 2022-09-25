---
title: 'HashMap负载因子为什么是0.75，链表转红黑树阈值为什么是8'
date: 2021-05-02 21:04:40
tags: [java]
published: true
hideInList: false
feature: /post-images/mdZO-HjSu.png
isTop: false
---
### 1.HashMap负载因子为什么是0.75
这是时间与空间成本上的折中。
#### 1.1时间成本
假设负载因子是1时，虽然空间利用率高了，但是随之提高的是哈希碰撞的概率。
而Hashmap中哈希碰撞的解决方法采用的拉链法，哈希冲突高了会导致链表越来越长（虽然后面会转换成红黑树），我们知道链表的查询效率是比较低的，所以负载因子太高会导致时间成本上升。
#### 1.2空间成本
那么为了减少哈希冲突，提高查询效率，负载因子是不是越低越好呢？答案显然是否定的。
假设负载因子为0.5时，那么空间利用率只有50%。例如大小为64时，至少有32没有被利用，大小为1024时，就有512没有被利用。扩容后的空间越大，空出的空间也就越大。
所以Java开发人员经过权衡，负载因子不能太大也不能太小，折中选择为0.75。
### 2.链表转红黑树阈值为什么是8
当负载因子是0.75的情况下，哈希碰撞的概率遵循参数约为0.5的泊松分布
这是概率论的范畴，不知道的同学只需要记住当负载因子是0.75的情况下，我们能够计算出哈希碰撞的概率
```java
Because TreeNodes are about twice the size of regular nodes, we
use them only when bins contain enough nodes to warrant use
(see TREEIFY_THRESHOLD). And when they become too small (due to
removal or resizing) they are converted back to plain bins.  In
usages with well-distributed user hashCodes, tree bins are
rarely used.  Ideally, under random hashCodes, the frequency of
nodes in bins follows a Poisson distribution
(http://en.wikipedia.org/wiki/Poisson_distribution) with a
parameter of about 0.5 on average for the default resizing
threshold of 0.75, although with a large variance because of
resizing granularity. Ignoring variance, the expected
occurrences of list size k are (exp(-0.5)*pow(0.5, k)/factorial(k)). 
The first values are:
0:    0.60653066
1:    0.30326533
2:    0.07581633
3:    0.01263606
4:    0.00157952
5:    0.00015795
6:    0.00001316
7:    0.00000094
8:    0.00000006
more: less than 1 in ten million
```

在HashMap源码中，给出了计算出的哈希碰撞的概率。
我们看到碰撞8次(链表长度达到8)的概率为0.00000006,几乎是一个不可能事件。
所以Java开发人员将链表转红黑树阈值默认设为8，是为了避免链表转红黑树这种耗时操作的事件发生。