---
title: 'java基础之一集合'
date: 2021-05-05 14:41:04
tags: [java]
published: true
hideInList: false
feature: /post-images/FRRzAbqMV.png
isTop: false
---
## 基础数据结构说明

​		**数组**：采用一段连续的存储单元来存储数据。对于指定下标的查找，时间复杂度为O(1)；通过给定值进行查找，需要遍历数组，逐一比对给定关键字和数组元素，时间复杂度为O(n)，当然，对于有序数组，则可采用二分查找，插值查找，斐波那契查找等方式，可将查找复杂度提高为O(logn)；对于一般的插入删除操作，涉及到数组元素的移动，其平均复杂度也为O(n)

　　**线性链表**：对于链表的新增，删除等操作（在找到指定操作位置后），仅需处理结点间的引用即可，时间复杂度为O(1)，而查找操作需要遍历链表逐一进行比对，复杂度为O(n)

　　**二叉树**：对一棵相对平衡的有序二叉树，对其进行插入，查找，删除等操作，平均复杂度均为O(logn)。

　　**哈希表**：相比上述几种数据结构，在哈希表中进行添加，删除，查找等操作，性能十分之高，不考虑哈希冲突的情况下，仅需一次定位即可完成，时间复杂度为O(1)，接下来我们就来看看哈希表是如何实现达到惊艳的常数阶O(1)的。

### 哈希表具体说明

　　我们知道，数据结构的物理存储结构只有两种：**顺序存储结构**和**链式存储结构**（像栈，队列，树，图等是从逻辑结构去抽象的，映射到内存中，也是这两种物理组织形式），而在上面我们提到过，在数组中根据下标查找某个元素，一次定位就可以达到，哈希表利用了这种特性，**哈希表的主干就是数组**。

　　比如我们要新增或查找某个元素，我们通过把当前元素的关键字 通过某个函数映射到数组中的某个位置，通过数组下标一次定位就可完成操作。

　　**存储位置 = f(关键字)**

　　其中，这个函数f一般称为**哈希函数**，这个函数的设计好坏会直接影响到哈希表的优劣。举个例子，比如我们要在哈希表中执行插入操作：

![](https://tinaxiawuhao.github.io/post-images/1619592366553.png)

　　查找操作同理，先通过哈希函数计算出实际存储地址，然后从数组中对应地址取出即可。

　　**哈希冲突**

　　然而万事无完美，如果两个不同的元素，通过哈希函数得出的实际存储地址相同怎么办？也就是说，当我们对某个元素进行哈希运算，得到一个存储地址，然后要进行插入的时候，发现已经被其他元素占用了，其实这就是所谓的**哈希冲突**，也叫哈希碰撞。前面我们提到过，哈希函数的设计至关重要，好的哈希函数会尽可能地保证 **计算简单**和**散列地址分布均匀**但是，我们需要清楚的是，数组是一块连续的固定长度的内存空间，再好的哈希函数也不能保证得到的存储地址绝对不发生冲突。那么哈希冲突如何解决呢？哈希冲突的解决方案有多种:`开放定址法（线性探测）`（发生冲突，继续寻找下一块未被占用的存储地址），`再散列函数法`，`链地址法`。

### R-B Tree简介

  R-B Tree，全称是Red-Black Tree，又称为“红黑树”，它一种特殊的二叉查找树。红黑树的每个节点上都有存储位表示节点的颜色，可以是红(Red)或黑(Black)。

**红黑树的特性**:
**（1）每个节点或者是黑色，或者是红色。**
**（2）根节点是黑色。**
**（3）每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]**
**（4）如果一个节点是红色的，则它的子节点必须是黑色的。**
**（5）从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。**

**注意**：
(01) 特性(3)中的叶子节点，是只为空(NIL或null)的节点。
(02) 特性(5)，确保没有一条路径会比其他路径长出俩倍。因而，红黑树是相对是接近平衡的二叉树。

红黑树示意图如下：

[![img](https://images0.cnblogs.com/i/497634/201403/251730074203156.jpg)](https://images0.cnblogs.com/i/497634/201403/251730074203156.jpg)

 

**性质**
红黑树是每个节点都带有颜色属性的二叉查找树，颜色或红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：
【1】性质1. 节点是红色或黑色。
【2】性质2. 根节点是黑色。
【3】性质3 每个叶节点是黑色的。
【4】性质4 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)
【5】性质5. 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。

**用途和好处**
	红黑树和AVL树一样都对插入时间、删除时间和查找时间提供了最好可能的最坏情况担保。这不只是使它们在时间敏感的应用如即时应用(real time application)中有价值，而且使它们有在提供最坏情况担保的其他数据结构中作为建造板块的价值；例如，在计算几何中使用的很多数据结构都可以基于红黑树。

​	红黑树在函数式编程中也特别有用，在这里它们是最常用的持久数据结构之一，它们用来构造关联数组和集合，在突变之后它们能保持为以前的版本。除了O(log n)的时间之外，红黑树的持久版本对每次插入或删除需要O(log n)的空间。

​	红黑树是 2-3-4树的一种等同。换句话说，对于每个 2-3-4 树，都存在至少一个数据元素是同样次序的红黑树。在 2-3-4 树上的插入和删除操作也等同于在红黑树中颜色翻转和旋转。这使得 2-3-4 树成为理解红黑树背后的逻辑的重要工具，这也是很多介绍算法的教科书在红黑树之前介绍 2-3-4 树的原因，尽管 2-3-4 树在实践中不经常使用。

**红黑树的数据结构**

```java
public class RBTree<T extends Comparable<T>> {

    private RBTNode<T> mRoot;    // 根结点

    private static final boolean RED   = false;
    private static final boolean BLACK = true;

    public class RBTNode<T extends Comparable<T>> {
        boolean color;        // 颜色
        T key;                // 关键字(键值)
        RBTNode<T> left;    // 左孩子
        RBTNode<T> right;    // 右孩子
        RBTNode<T> parent;    // 父结点

        public RBTNode(T key, boolean color, RBTNode<T> parent, RBTNode<T> left, RBTNode<T> right) {
            this.key = key;
            this.color = color;
            this.parent = parent;
            this.left = left;
            this.right = right;
        }

    }

    ...
}

```

## 集合说明

![](https://tinaxiawuhao.github.io/post-images/1619592387947.png)

### ArrayList实现原理要点概括

1. ArrayList是List接口的可变数组非同步实现，并允许包括null在内的所有元素。

2. ArrayList的数据结构如下：

   　　

![](https://tinaxiawuhao.github.io/post-images/1619592476277.png)

   　　说明：底层的数据结构就是数组，数组元素类型为Object类型，即可以存放所有类型数据。我们对ArrayList类的实例的所有的操作底层都是基于数组的。

3. 该集合是可变长度数组，数组扩容时，会将老数组中的元素重新拷贝一份到新的数组中，每次数组容量增长大约是其容量的1.5倍，这种操作的代价很高。

   ```java
   newCapacity = oldCapacity + (oldCapacity >> 1)
   ```

4. 采用了Fail-Fast机制，面对并发的修改时，迭代器很快就会完全失败，而不是冒着在将来某个不确定时间发生任意不确定行为的风险

5. remove方法会让下标到数组末尾的元素向前移动一个单位，并把最后一位的值置空，方便GC

### LinkedList实现原理要点概括

1. LinkedList是List接口的双向链表非同步实现，并允许包括null在内的所有元素。从JDK1.7开始，LinkedList 由双向循环链表改为双向链表

2. LinkedList数据结构如下

![](https://tinaxiawuhao.github.io/post-images/1619592485724.png)

   　　说明：如上图所示，LinkedList底层使用的双向链表结构，有一个头结点和一个尾结点，双向链表意味着我们可以从头开始正向遍历，或者是从尾开始逆向遍历，并且可以针对头部和尾部进行相应的操作。

3. 双向链表节点对应的类Node的实例，Node中包含成员变量：prev，next，item。其中，prev是该节点的上一个节点，next是该节点的下一个节点，item是该节点所包含的值。

   ```java
   public class LinkedList<E>
       extends AbstractSequentialList<E>
       implements List<E>, Deque<E>, Cloneable, java.io.Serializable
   {
       transient int size = 0;
   
       /**
        * Pointer to first node.
        * Invariant: (first == null && last == null) ||
        *            (first.prev == null && first.item != null)
        */
       transient Node<E> first;
   
       /**
        * Pointer to last node.
        * Invariant: (first == null && last == null) ||
        *            (last.next == null && last.item != null)
        */
       transient Node<E> last;
       
        /**
        *  节点结构
        */
       private static class Node<E> {
           E item;
           Node<E> next;
           Node<E> prev;
   
           Node(Node<E> prev, E element, Node<E> next) {
               this.item = element;
               this.next = next;
               this.prev = prev;
           }
       }
   
   }
   
   ```

   说明：LinkedList的属性非常简单，一个头结点、一个尾结点、一个表示链表中实际元素个数的变量。注意，头结点、尾结点都有transient关键字修饰，这也意味着在序列化时该域是不会序列化的。

4. 它的查找是分两半查找，先判断index是在链表的哪一半，然后再去对应区域查找，这样最多只要遍历链表的一半节点即可找到

### HashMap实现原理要点概括

1. HashMap是基于哈希表的Map接口的非同步实现，允许使用null值和null键，但不保证映射的顺序。

2. 底层使用数组实现，数组中每一项是个单向链表，即数组和链表的结合体；当链表长度大于8时，链表转换为红黑树，这样减少链表查询时间。

![](https://tinaxiawuhao.github.io/post-images/1619592496628.png)

3. HashMap在底层将key-value当成一个整体进行处理，这个整体就是一个Node对象。HashMap底层采用一个Node[]数组来保存所有的key-value对，当需要存储一个Node对象时，会根据key的hash算法来决定其在数组中的存储位置，再根据equals方法决定其在该数组位置上的链表中的存储位置；当需要取出一个Node时，也会根据key的hash算法找到其在数组中的存储位置，再根据equals方法从该位置上的链表中取出该Node。

4. HashMap解决hash冲突是采用了`链地址`法，也就是**数组+链表**的方式，

5. HashMap进行数组扩容需要重新计算扩容后每个元素在数组中的位置，很耗性能

6. 采用了Fail-Fast机制，通过一个modCount值记录修改次数，对HashMap内容的修改都将增加这个值。迭代器初始化过程中会将这个值赋给迭代器的expectedModCount，在迭代过程中，判断modCount跟expectedModCount是否相等，如果不相等就表示已经有其他线程修改了Map，马上抛出异常

#### 重写equals方法需同时重写hashCode方法

```java 
public static void main(String []args){
    HashMap<Person,String> map = new HashMap<Person, String>();
    Person person = new Person(1234,"乔峰");
    //put到hashmap中去
    map.put(person,"天龙八部");
    //get取出，从逻辑上讲应该能输出“天龙八部”
    System.out.println("结果:"+map.get(new Person(1234,"乔峰")));
}
```

​		如果我们已经对HashMap的原理有了一定了解，这个结果就不难理解了。尽管我们在进行get和put操作的时候，使用的key从逻辑上讲是等值的（通过equals比较是相等的），但由于没有重写hashCode方法，所以put操作时，key(hashcode1)-->hash-->indexFor-->最终索引位置 ，而通过key取出value的时候 key(hashcode2)-->hash-->indexFor-->最终索引位置，由于hashcode1不等于hashcode2，导致没有定位到一个数组位置而返回逻辑上错误的值null（也有可能碰巧定位到一个数组位置，但是也会判断其entry的hash值是否相等。）

　　所以，在重写equals的方法的时候，必须注意重写hashCode方法，同时还要保证通过equals判断相等的两个对象，调用hashCode方法要返回同样的整数值。而如果equals判断不相等的两个对象，其hashCode可以相同（只不过会发生哈希冲突，应尽量避免）。

### Hashtable实现原理要点概括

![](https://tinaxiawuhao.github.io/post-images/1619592758082.jfif)

1. Hashtable是基于哈希表的Map接口的同步实现，不允许使用null值和null键，Hashtable中的映射不是有序的
2. 底层使用数组实现，数组中每一项是个单链表，即数组和链表的结合体
3. Hashtable在底层将key-value当成一个整体进行处理，这个整体就是一个Entry对象。Hashtable底层采用一个Entry[]数组来保存所有的key-value对，当需要存储一个Entry对象时，会根据key的hash算法来决定其在数组中的存储位置，在根据equals方法决定其在该数组位置上的链表中的存储位置；当需要取出一个Entry时，也会根据key的hash算法找到其在数组中的存储位置，再根据equals方法从该位置上的链表中取出该Entry。
4. synchronized是针对整张Hash表的，即每次锁住整张表让线程独占

### ConcurrentHashMap实现原理要点概括

![](https://tinaxiawuhao.github.io/post-images/1619592765108.jpeg)

1. ConcurrentHashMap允许多个修改操作并发进行，其关键在于使用了锁分离技术。

2. 它使用了多个锁来控制对hash表的不同段进行的修改，每个段其实就是一个小的hashtable，它们有自己的锁。只要多个并发发生在不同的段上，它们就可以并发进行。

3. ConcurrentHashMap在底层将key-value当成一个整体进行处理，这个整体就是一个Entry对象。Hashtable底层采用一个Entry[]数组来保存所有的key-value对，当需要存储一个Entry对象时，会根据key的hash算法来决定其在数组中的存储位置，在根据equals方法决定其在该数组位置上的链表中的存储位置；当需要取出一个Entry时，也会根据key的hash算法找到其在数组中的存储位置，再根据equals方法从该位置上的链表中取出该Entry。

4. 与HashMap不同的是，ConcurrentHashMap使用多个子Hash表，也就是段(Segment)

5. ConcurrentHashMap完全允许多个读操作并发进行，读操作并不需要加锁。如果使用传统的技术，如HashMap中的实现，如果允许可以在hash链的中间添加或删除元素，读操作不加锁将得到不一致的数据。

**ConcurrentHashMap 1.8为什么要使用CAS+Synchronized取代Segment+ReentrantLock**

​	大家应该都知道ConcurrentHashMap在1.8的时候有了很大的改动,当然,我这里要说的改动不是指链表长度大于8就转为红黑树这种常识,我要说的是ConcurrentHashMap在1.8为什么用CAS+Synchronized取代Segment+ReentrantLock了

​	首先,我假设你对CAS,Synchronized,ReentrantLock这些知识很了解,并且知道AQS,自旋锁,偏向锁,轻量级锁,重量级锁这些知识,也知道Synchronized和ReentrantLock在唤醒被挂起线程竞争的时候有什么区别

​	首先我们说下1.8以前的ConcurrentHashMap是怎么保证线程并发的,首先在初始化ConcurrentHashMap的时候,会初始化一个Segment数组,容量为16,而每个Segment呢,都继承了ReentrantLock类,也就是说每个Segment类本身就是一个锁,之后Segment内部又有一个table数组,而每个table数组里的索引数据呢,又对应着一个Node链表.

​	那么这样的好处是什么呢?我先从老版本的添加流程说起吧,由于电脑里没有JDK1.7及以下的版本我没法给你看代码,所以使用文字描述的方式,首先,当我们使用put方法的时候,是对我们的key进行hash拿到一个整型,然后将整型对16取模,拿到对应的Segment,之后调用Segment的put方法,然后上锁,请注意,这里lock()的时候其实是this.lock(),也就是说,每个Segment的锁是分开的

​	其中一个上锁不会影响另一个,此时也就代表了我可以有十六个线程进来,而ReentrantLock上锁的时候如果只有一个线程进来,是不会有线程挂起的操作的,也就是说只需要在AQS里使用CAS改变一个state的值为1,此时就能对代码进行操作,这样一来,我们等于将并发量/16了.

好,说完了老版本的ConcurrentHashMap,我们再说说新版本的,请看下面的图:

![](https://tinaxiawuhao.github.io/post-images/1629875978754.png)

​	请注意Synchronized上锁的对象,请记住,Synchronized是靠对象的对象头和此对象对应的monitor来保证上锁的,也就是对象头里的重量级锁标志指向了monitor,而monitor呢,内部则保存了一个当前线程,也就是抢到了锁的线程.

​	那么这里的这个f是什么呢?它是Node链表里的每一个Node,也就是说,Synchronized是将每一个Node对象作为了一个锁,这样做的好处是什么呢?将锁细化了,也就是说,除非两个线程同时操作一个Node,注意,是一个Node而不是一个Node链表哦,那么才会争抢同一把锁.

​	如果使用ReentrantLock其实也可以将锁细化成这样的,只要让Node类继承ReentrantLock就行了,这样的话调用f.lock()就能做到和Synchronized(f)同样的效果,但为什么不这样做呢?

​	请大家试想一下,锁已经被细化到这种程度了,那么出现并发争抢的可能性还高吗?还有就是,哪怕出现争抢了,只要线程可以在30到50次自旋里拿到锁,那么Synchronized就不会升级为重量级锁,而等待的线程也就不用被挂起,我们也就少了挂起和唤醒这个上下文切换的过程开销.

​	但如果是ReentrantLock呢?它则只有在线程没有抢到锁,然后新建Node节点后再尝试一次而已,不会自旋,而是直接被挂起,这样一来,我们就很容易会多出线程上下文开销的代价.当然,你也可以使用tryLock(),但是这样又出现了一个问题,你怎么知道tryLock的时间呢?在时间范围里还好,假如超过了呢?

​	所以,在锁被细化到如此程度上,使用Synchronized是最好的选择了.这里再补充一句,Synchronized和ReentrantLock他们的开销差距是在释放锁时唤醒线程的数量,Synchronized是唤醒锁池里所有的线程+刚好来访问的线程,而ReentrantLock则是当前线程后进来的第一个线程+刚好来访问的线程.

​	如果是线程并发量不大的情况下,那么Synchronized因为自旋锁,偏向锁,轻量级锁的原因,不用将等待线程挂起,偏向锁甚至不用自旋,所以在这种情况下要比ReentrantLock高效

**1.8弃用分段锁的原因由以下几点：**
    1. 加入多个分段锁浪费内存空间。
    2. 生产环境中， map 在放入时竞争同一个锁的概率非常小，分段锁反而会造成更新等操作的长时间等待。
    3. 为了提高 GC 的效率

### HashSet实现原理要点概括

1. HashSet由哈希表(实际上是一个HashMap实例)支持，不保证set的迭代顺序，并允许使用null元素。
2. 对于HashSet而言，它是基于HashMap实现的，HashSet底层使用HashMap来保存所有元素，因此HashSet 的实现比较简单，相关HashSet的操作，基本上都是直接调用底层HashMap的相关方法来完成，API也是对HashMap的行为进行了封装，可参考HashMap

### LinkedHashMap实现原理要点概括

![](https://tinaxiawuhao.github.io/post-images/1619593405140.png)

1. LinkedHashMap继承于HashMap，底层使用哈希表和双向链表来保存所有元素，并且它是非同步，允许使用null值和null键。
2. 基本操作与父类HashMap相似，通过重写HashMap相关方法，重新定义了数组中保存的元素Entry，来实现自己的链接列表特性。该Entry除了保存当前对象的引用外，还保存了其上一个元素before和下一个元素after的引用，从而构成了双向链接列表。

### LinkedHashSet实现原理要点概括

1. 对于LinkedHashSet而言，它继承与HashSet、又基于LinkedHashMap来实现的。LinkedHashSet底层使用LinkedHashMap来保存所有元素，它继承与HashSet，其所有的方法操作上又与HashSet相同。

## java集合遍历的几种方式总结及比较

### 集合类的通用遍历方式, 用迭代器迭代:

```java
Iterator it = list.iterator();

while(it.hasNext()) {

　　Object obj = it.next();

}
```

### Map遍历方式：

**1、通过获取所有的key按照key来遍历**

```java
//Set<Integer> set = map.keySet(); //得到所有key的集合
for (Integer in : map.keySet()) {
    String str = map.get(in);//得到每个key多对用value的值
}
```

 

**2、通过Map.entrySet使用iterator遍历key和value**

```java
Iterator<Map.Entry<Integer, String>> it = map.entrySet().iterator();
while (it.hasNext()) {
     Map.Entry<Integer, String> entry = it.next();
       System.out.println("key= " + entry.getKey() + " and value= " + entry.getValue());
}
```

 

**3、通过Map.entrySet遍历key和value，推荐，尤其是容量大时**

```java
for (Map.Entry<Integer, String> entry : map.entrySet()) {
    //Map.entry<Integer,String> 映射项（键-值对）  有几个方法：用上面的名字entry
    //entry.getKey() ;entry.getValue(); entry.setValue();
    //map.entrySet()  返回此映射中包含的映射关系的 Set视图。
    System.out.println("key= " + entry.getKey() + " and value= " + entry.getValue());
}
```

 

4、通过Map.values()遍历所有的value，但不能遍历key

```java
for (String v : map.values()) {
    System.out.println("value= " + v);
}
```

### List遍历方式：

**第一种：**

```java
Iterator iterator = list.iterator();
while(iterator.hasNext()){
    int i = (Integer) iterator.next();
    System.out.println(i);
}
```

**第二种：**

```java
for (Object object : list) { 
    System.out.println(object); 
}
```

**第三种：**

```java
for(int i = 0 ;i<list.size();i++) {  
    int j= (Integer) list.get(i);
    System.out.println(j);  
}
```

### 每个遍历方法的实现原理是什么？

1. 传统的for循环遍历，基于计数器的：

​    遍历者自己在集合外部维护一个计数器，然后依次读取每一个位置的元素，当读取到最后一个元素后，停止。主要就是需要按元素的位置来读取元素。

2. 迭代器遍历，Iterator：

​    每一个具体实现的数据集合，一般都需要提供相应的Iterator。相比于传统for循环，Iterator取缔了显式的遍历计数器。所以基于顺序存储集合的Iterator可以直接按位置访问数据。而基于链式存储集合的Iterator，正常的实现，都是需要保存当前遍历的位置。然后根据当前位置来向前或者向后移动指针。

3. foreach循环遍历：

​    根据反编译的字节码可以发现，foreach内部也是采用了Iterator的方式实现，只不过Java编译器帮我们生成了这些代码。

### 各遍历方式对于不同的存储方式，性能如何？

**1、传统的for循环遍历，基于计数器的：**

​    因为是基于元素的位置，按位置读取。所以我们可以知道，对于顺序存储，因为读取特定位置元素的平均时间复杂度是O(1)，所以遍历整个集合的平均时间复杂度为O(n)。而对于链式存储，因为读取特定位置元素的平均时间复杂度是O(n)，所以遍历整个集合的平均时间复杂度为O(n2)（n的平方）。

ArrayList按位置读取的代码：直接按元素位置读取。

```java
transient Object[] elementData;

public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}

E elementData(int index) {
    return (E) elementData[index];
}
```

LinkedList按位置读取的代码：每次都需要从第0个元素开始向后读取。其实它内部也做了小小的优化。


```java
transient int size = 0;
transient Node<E> first;
transient Node<E> last;

public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    if (index < (size >> 1)) {   //查询位置在链表前半部分，从链表头开始查找
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {                     //查询位置在链表后半部分，从链表尾开始查找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

 **2、迭代器遍历，Iterator：**

​    那么对于RandomAccess类型的集合来说，没有太多意义，反而因为一些额外的操作，还会增加额外的运行时间。但是对于Sequential Access的集合来说，就有很重大的意义了，因为Iterator内部维护了当前遍历的位置，所以每次遍历，读取下一个位置并不需要从集合的第一个元素开始查找，只要把指针向后移一位就行了，这样一来，遍历整个集合的时间复杂度就降低为O(n)；

（这里只用LinkedList做例子）LinkedList的迭代器，内部实现，就是维护当前遍历的位置，然后操作指针移动就可以了：

```java
public E next() {
    checkForComodification();
    if (!hasNext())
        throw new NoSuchElementException();

    lastReturned = next;
    next = next.next;
    nextIndex++;
    return lastReturned.item;
}

public E previous() {
    checkForComodification();
    if (!hasPrevious())
        throw new NoSuchElementException();

    lastReturned = next = (next == null) ? last : next.prev;
    nextIndex--;
    return lastReturned.item;
}
```

**3、foreach循环遍历：**

​    分析Java字节码可知，foreach内部实现原理，也是通过Iterator实现的，只不过这个Iterator是Java编译器帮我们生成的，所以我们不需要再手动去编写。但是因为每次都要做类型转换检查，所以花费的时间比Iterator略长。时间复杂度和Iterator一样。

Iterator和foreach字节码如下：

```java
//使用Iterator的字节码：
    Code:
       0: new           #16                 // class java/util/ArrayList
       3: dup
       4: invokespecial #18                 // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: invokeinterface #19,  1           // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
      14: astore_2
      15: goto          25
      18: aload_2
      19: invokeinterface #25,  1           // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
      24: pop
      25: aload_2
      26: invokeinterface #31,  1           // InterfaceMethod java/util/Iterator.hasNext:()Z
      31: ifne          18
      34: return
 
 
//使用foreach的字节码：
    Code:
       0: new           #16                 // class java/util/ArrayList
       3: dup
       4: invokespecial #18                 // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: invokeinterface #19,  1           // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
      14: astore_3
      15: goto          28
      18: aload_3
      19: invokeinterface #25,  1           // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
      24: checkcast     #31                 // class loop/Model
      27: astore_2
      28: aload_3
      29: invokeinterface #33,  1           // InterfaceMethod java/util/Iterator.hasNext:()Z
      34: ifne          18
      37: return
```

### 各遍历方式的适用于什么场合？

1. 传统的for循环遍历，基于计数器的：

   ​    顺序存储：读取性能比较高。适用于遍历顺序存储集合。

   ​    链式存储：时间复杂度太大，不适用于遍历链式存储的集合。

2. 迭代器遍历，Iterator：

   ​    顺序存储：如果不是太在意时间，推荐选择此方式，毕竟代码更加简洁，也防止了Off-By-One的问题。

   ​    链式存储：意义就重大了，平均时间复杂度降为O(n)，还是挺诱人的，所以推荐此种遍历方式。

3. foreach循环遍历：

   ​    foreach只是让代码更加简洁了，但是他有一些缺点，就是遍历过程中不能操作数据集合（删除等），所以有些场合不使用。而且它本身就是基于Iterator实现的，但是由于类型转换的问题，所以会比直接使用Iterator慢一点，但是还好，时间复杂度都是一样的。所以怎么选择，参考上面两种方式，做一个折中的选择。
 

### RandomAccess接口标记

Java数据集合框架中，提供了一个RandomAccess接口，该接口没有方法，只是一个标记。通常被List接口的实现使用，用来标记该List的实现是否支持Random Access。

一个数据集合实现了该接口，就意味着它支持Random Access，按位置读取元素的平均时间复杂度为O(1)。比如ArrayList。

而没有实现该接口的，就表示不支持Random Access。比如LinkedList。

所以看来JDK开发者也是注意到这个问题的，那么推荐的做法就是，如果想要遍历一个List，那么先判断是否支持Random Access，也就是 list instanceof RandomAccess。

比如：

```java
if (list instanceof RandomAccess) {
    //使用传统的for循环遍历。
} else {
    //使用Iterator或者foreach。
}
```

## Array、List、Set互转

### Array、List互转

1. Array 转List

```java
String[] s = new String[]{"A", "B", "C", "D","E"};
List<String> list = Arrays.asList(s);
# 注意这里list里面的元素直接是s里面的元素( list backed by the specified array)，换句话就是说：对s的修改，直接影响list。
s[0] ="AA";
System.out.println("list: " + list);
# 输出结果
list: [AA, B, C, D, E]
```


2. List转Array

```java
String[] dest = list.toArray(new String[0]);//new String[0]是指定返回数组的类型
System.out.println("dest: " + Arrays.toString(dest));
# 输出结果
dest: [AA, B, C, D, E]
# 注意这里的dest里面的元素不是list里面的元素，换句话就是说：对list中关于元素的修改，不会影响dest。
list.set(0, "Z");
System.out.println("modified list: " + list);
System.out.println("dest: " + Arrays.toString(dest));
# 输出结果
modified list: [Z, B, C, D, E]
dest: [AA, B, C, D, E]
# 可以看到list虽然被修改了，但是dest数组没有没修改。
```

### List、Set互转

因为List和Set都实现了Collection接口，且addAll(Collection<? extends E> c);方法，因此可以采用addAll()方法将List和Set互相转换；另外，List和Set也提供了Collection<? extends E> c作为参数的构造函数，因此通常采用构造函数的形式完成互相转化。

```java
//List转Set
Set<String> set = new HashSet<>(list);
System.out.println("set: " + set);
//Set转List
List<String> list_1 = new ArrayList<>(set);
System.out.println("list_1: " + list_1);
# 和toArray()一样，被转换的List(Set)的修改不会对被转化后的Set（List）造成影响。
```

### Array、Set互转

```java
//array转set
s = new String[]{"A", "B", "C", "D","E"};
set = new HashSet<>(Arrays.asList(s));
System.out.println("set: " + set);
//set转array
dest = set.toArray(new String[0]);
System.out.println("dest: " + Arrays.toString(dest));
```

## Java 中初始化 List 集合的 6 种方式!

1. 常规方式

   ```java
    # 后面缺失的泛型类型在 JDK 7 之后就可以不用写具体的类型了，改进后会自动推断类型
    List<String> list = new ArrayList<>();
    list.add("1");
    list.add("2");
    list.add("3");
   ```

2. Arrays 工具类

   ```java
   import static java.util.Arrays.asList;
    List<String> list = asList("1", "2", "3");
    # 注意，上面的 asList 是 Arrays 的静态方法，这里使用了静态导入。这种方式添加的是不可变的 List, 即不能添加、删除等操作，需要警惕。。
    # 如果要可变，那就使用 ArrayList 再包装一下，如下面所示。
    List<String> numbers = new ArrayList<>(Arrays.asList("1", "2", "3"));
    numbers.add("4");
   ```



3. Collections 工具类

   ```java
   List<String> list = Collections.nCopies(3, "list");
    # 这种方式添加的是不可变的、复制某个元素N遍的工具类，以上程序输出：
    # [list, list, list]
    # 老规则，如果要可变，使用 ArrayList 包装一遍。
    List<String> list = new ArrayList<>(Collections.nCopies(3, "list"));
    list.add("list");

    # 还有初始化单个对象的 List 工具类，这种方式也是不可变的，集合内只能有一个元素，这种也用得很少啊。
    List<String> list = Collections.singletonList("list");

    # 还有一个创建空 List 的工具类，没有默认容量，节省空间，但不知道实际工作中没有用。
    List<String> list = Collections.emptyList("list");
   ```

4. 匿名内部类

   ```java
   List<String> list = new ArrayList<>() {{
    add("1");
    add("2");
    add("3");
    }};
   ```

   这种其实有效率和内存泄漏的问题，参考：

   https://blog.csdn.net/xukun5137/article/details/78275201

   https://www.cnblogs.com/wenbronk/p/7000643.html

5. JDK8 Stream

   ```java
   import static java.util.stream.Collectors.toList;
    List<String> list = Stream.of("1", "2", "3").collect(toList());
   ```

6. JDK 9 List.of

   ```java
   List<String> list = List.of("1", "2", "3");
   ```
   这是 JDK 9 里面新增的 List 接口里面的静态方法，同样也是不可变的。