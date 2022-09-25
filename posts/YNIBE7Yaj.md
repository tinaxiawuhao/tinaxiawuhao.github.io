---
title: 'redis概述二'
date: 2021-05-14 16:43:43
tags: [redis]
published: true
hideInList: false
feature: /post-images/YNIBE7Yaj.png
isTop: false
---
### Redis有哪些数据类型

Redis主要有5种数据类型，包括String，List，Set，Zset，Hash，满足大部分的使用要求

| 数据类型 | 可以存储的值           | 操作                                                         | 应用场景                                                     |
| :------- | :--------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| STRING   | 字符串、整数或者浮点数 | 对整个字符串或者字符串的其中一部分执行操作对整数和浮点数执行自增或者自减操作 | 做简单的键值对缓存                                           |
| LIST     | 列表                   | 从两端压入或者弹出元素对单个或者多个元素进行修剪，只保留一个范围内的元素 | 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的数据 |
| SET      | 无序集合               | 添加、获取、移除单个元素检查一个元素是否存在于集合中计算交集、并集、差集从集合里面随机获取元素 | 交集、并集、差集的操作，比如交集，可以把两个人的粉丝列表整一个交集 |
| HASH     | 包含键值对的无序散列表 | 添加、获取、移除单个键值对获取所有键值对检查某个键是否存在   | 结构化的数据，比如一个对象                                   |
| ZSET     | 有序集合               | 添加、获取、删除元素根据分值范围或者成员来获取元素计算一个键的排名 | 去重但可以排序，如获取排名前几名的用户                       |

### Redis数据详解

#### String

![img](https://tinaxiawuhao.github.io/post-images/1620702855667.png)

1. 设置值

   ```java
   //设置值
   set key value [ex seconds] [px milliseconds] [nx|xx]
   
   127.0.0.1:6379> set hello world
   ```

   > set命令有几个选项：
   > ·ex seconds： 为键设置秒级过期时间。
   > ·px milliseconds： 为键设置毫秒级过期时间。
   > ·nx： 键必须不存在， 才可以设置成功， 用于添加。
   > ·xx： 与nx相反， 键必须存在， 才可以设置成功， 用于更新。

   ```java
   // 因为键hello已存在， 所以setnx失败， 返回结果为0
   127.0.0.1:6379> setnx hello redis
   (integer) 0
   //因为键hello已存在， 所以set xx成功， 返回结果为OK：
   127.0.0.1:6379> set hello jedis xx
   OK
   ```

   setnx和setxx在实际使用中有什么应用场景吗？ 以setnx命令为例子， 由于Redis的单线程命令处理机制， 如果有多个客户端同时执行setnx key value，根据setnx的特性只有一个客户端能设置成功， setnx可以作为分布式锁的一种实现方案， Redis官方给出了使用setnx实现分布式锁的方法： http://redis.io/topics/distlock。

2. 获取值

   ```java
   get key
   //下面操作获取键hello的值：
   127.0.0.1:6379> get hello
   "world"
   //如果要获取的键不存在， 则返回nil（空） ：
   127.0.0.1:6379> get not_exist_key
   (nil)
   ```

3. 批量设置值

   ```java
   mset key value [key value ...]
   //下面操作通过mset命令一次性设置4个键值对：
   127.0.0.1:6379> mset a 1 b 2 c 3 d 4
   OK
   ```

4. 批量获取值

   ```java
   mget key [key ...]
   //下面操作批量获取了键a、 b、 c、 d的值：
   127.0.0.1:6379> mget a b c d
   1) "1"
   2) "2"
   3) "3"
   4) "4"
   如果有些键不存在， 那么它的值为nil（空） ， 结果是按照传入键的顺序返回：
   127.0.0.1:6379> mget a b c f
   1) "1"
   2) "2"
   3) "3"
   4) (nil)
   ```

5. 计数

   ```java
   incr key
   //incr命令用于对值做自增操作， 返回结果分为三种情况：
   //·值不是整数， 返回错误。
   //·值是整数， 返回自增后的结果。
   //·键不存在， 按照值为0自增， 返回结果为1。
   //例如对一个不存在的键执行incr操作后， 返回结果是1：
   127.0.0.1:6379> exists key
   (integer) 0
   127.0.0.1:6379> incr key
   (integer) 1
   //再次对键执行incr命令， 返回结果是2：
   127.0.0.1:6379> incr key
   (integer) 2
   //如果值不是整数， 那么会返回错误：
   127.0.0.1:6379> set hello world
   OK
   127.0.0.1:6379> incr hello
   (error) ERR value is not an integer or out of range
   //除了incr命令， Redis提供了decr(自减)、incrby(自增指定数字)、decrby(自减指定数字)、incrbyfloat(自增浮点数)：
   decr key
   incrby key increment
   decrby key decrement
   incrbyfloat key increment
   //很多存储系统和编程语言内部使用CAS机制实现计数功能， 会有一定的CPU开销， 但在Redis中完全不存在这个问题， 因为Redis是单线程架构， 任何命令到了Redis服务端都要顺序执行
   ```

#### String使用场景

![img](https://tinaxiawuhao.github.io/post-images/1620702881825.png)

1. 缓存功能
2. 计数
3. 共享Session
4. 限速

#### List

![img](https://tinaxiawuhao.github.io/post-images/1620702898919.png)
![img](https://tinaxiawuhao.github.io/post-images/1620702909739.png)
![img](https://tinaxiawuhao.github.io/post-images/1620702922813.png)

1. 添加

   ```java
   //从右边插入元素
   rpush key value [value ...]
   //下面代码从右向左插入元素c、 b、 a：
   127.0.0. 1:6379> rpush listkey c b a
   (integer) 3
   //lrange0-1命令可以从左到右获取列表的所有元素：
   127.0.0.1:6379> lrange listkey 0 -1
   1) "c"
   2) "b"
   3) "a"
   ```

   ```java
   //从左边插入元素
   lpush key value [value ...]
   使用方法和rpush相同， 只不过从左侧插入。
   ```

   ```java
   //向某个元素前或者后插入元素
   linsert key before|after pivot value
   //linsert命令会从列表中找到等于pivot的元素， 在其前(before)或者后(after)插入一个新的元素value， 例如下面操作会在列表的元素b前插入java：
   127.0.0.1:6379> linsert listkey before b java
   (integer) 4
   //返回结果为4， 代表当前命令的长度， 当前列表变为：
   127.0.0.1:6379> lrange listkey 0 -1
   1) "c"
   2) "java"
   3) "b"
   4) "a"
   ```

2. 查找

   ```java
   //获取指定范围内的元素列表
   lrange key start end
   //lrange操作会获取列表指定索引范围所有的元素。 索引下标有两个特点： 第一， 索引下标从左到右分别是0到N-1， 但是从右到左分别是-1到-N。
   //第二， lrange中的end选项包含了自身， 这个和很多编程语言不包含end不太相同， 例如想获取列表的第2到第4个元素， 可以执行如下操作：
   127.0.0.1:6379> lrange listkey 1 3
   1) "java"
   2) "b"
   3) "a"
   ```

   ```java
   //获取列表指定索引下标的元素
   lindex key index
   //例如当前列表最后一个元素为a：
   127.0.0.1:6379> lindex listkey -1
   "a"
   ```

   ```java
   // 获取列表长度
   llen key
   //例如， 下面示例当前列表长度为4：
   127.0.0.1:6379> llen listkey
   (integer) 4
   ```

3. 删除

   ```java
   //从列表左侧弹出元素
   lpop key
   //如下操作将列表最左侧的元素c会被弹出， 弹出后列表变为java、 b、a：
   127.0.0.1:6379>t lpop listkey
   "c"
   127.0.0.1:6379> lrange listkey 0 -1
   1) "java"
   2) "b"
   3) "a"
   ```

   ```java
   //从列表右侧弹出
   rpop key
   它的使用方法和lpop是一样的， 只不过从列表右侧弹出
   ```

   ```java
   //删除指定元素
   lrem key count value
   //lrem命令会从列表中找到等于value的元素进行删除， 根据count的不同分为三种情况：
   //·count>0， 从左到右， 删除最多count个元素。
   //·count<0， 从右到左， 删除最多count绝对值个元素。
   //·count=0， 删除所有。
   //例如向列表从左向右插入5个a， 那么当前列表变为“a a a a a java b a”，
   //下面操作将从列表左边开始删除4个为a的元素：
   127.0.0.1:6379> lrem listkey 4 a
   (integer) 4
   127.0.0.1:6379> lrange listkey 0 -1
   1) "a"
   2) "java"
   3) "b"
   4) "a"
   ```

   ```java
   //按照索引范围修剪列表
   ltrim key start end
   //例如， 下面操作会只保留列表listkey第2个到第4个元素：
   127.0.0.1:6379> ltrim listkey 1 3
   OK
   127.0.0.1:6379> lrange listkey 0 -1
   1) "java"
   2) "b"
   3) "a"
   ```

4. 修改

   ```java
   //修改指定索引下标的元素：
   lset key index newValue
   //下面操作会将列表listkey中的第3个元素设置为python：
   127.0.0.1:6379> lset listkey 2 python
   OK
   127.0.0.1:6379> lrange listkey 0 -1
   1) "java"
   2) "b"
   3) "python"
   ```

5. 阻塞操作

   ```java
   //阻塞式弹出如下：
   blpop key [key ...] timeout
   brpop key [key ...] timeout
   //blpop和brpop是lpop和rpop的阻塞版本， 它们除了弹出方向不同， 使用方法基本相同， 所以下面以brpop命令进行说明， brpop命令包含两个参数：
   //·key[key...]： 多个列表的键。
   //·timeout： 阻塞时间（单位： 秒） 。
       
   //列表为空： 如果timeout=3， 那么客户端要等到3秒后返回， 如果timeout=0， 那么客户端一直阻塞等下去：
   127.0.0.1:6379> brpop list:test 3
   (nil)
   (3.10s)
   127.0.0.1:6379> brpop list:test 0
   //...阻塞...
   //如果此期间添加了数据element1， 客户端立即返回：
   127.0.0.1:6379> brpop list:test 3
   1) "list:test"
   2) "element1"
   (2.06s)
       
   //列表不为空： 客户端会立即返回。
   127.0.0.1:6379> brpop list:test 0
   1) "list:test"
   2) "element1"
   
   //在使用brpop时， 有两点需要注意。
   //第一点， 如果是多个键， 那么brpop会从左至右遍历键， 一旦有一个键能弹出元素， 客户端立即返回：
   127.0.0.1:6379> brpop list:1 list:2 list:3 0
   //..阻塞..
   //此时另一个客户端分别向list： 2和list： 3插入元素：
   client-lpush> lpush list:2 element2
   (integer) 1
   client-lpush> lpush list:3 element3
   (integer) 1
   //客户端会立即返回list： 2中的element2， 因为list： 2最先有可以弹出的元素：
   127.0.0.1:6379> brpop list:1 list:2 list:3 0
   1) "list:2"
   2) "element2_1"
   //第二点， 如果多个客户端对同一个键执行brpop， 那么最先执行brpop命令的客户端可以获取到弹出的值。
   //客户端1：
   client-1> brpop list:test 0
   //...阻塞...
   //客户端2：
   client-2> brpop list:test 0
   //...阻塞...
   //客户端3：
   client-3> brpop list:test 0
   //...阻塞...
   //此时另一个客户端lpush一个元素到list： test列表中：
   client-lpush> lpush list:test element
   (integer) 1
   //那么客户端1最会获取到元素， 因为客户端1最先执行brpop， 而客户端2和客户端3继续阻塞：
   client> brpop list:test 0
   1) "list:test"
   2) "element"
   ```

#### List使用场景

> lpush+lpop=Stack（ 栈）
> lpush+rpop=Queue（ 队列）
> lpush+ltrim=Capped Collection（ 有限集合）
> lpush+brpop=Message Queue（ 消息队列）

#### Set

![img](https://tinaxiawuhao.github.io/post-images/1620711808145.png)

- 集合内操作

  1. 添加元素

     ```java
     //sadd key element [element ...]
     //返回结果为添加成功的元素个数， 例如：
     127.0.0.1:6379> exists myset
     (integer) 0
     127.0.0.1:6379> sadd myset a b c
     (integer) 3
     127.0.0.1:6379> sadd myset a b
     (integer) 0
     ```

  2. 删除元素

     ```java
     //srem key element [element ...]
     //返回结果为成功删除元素个数， 例如：
     127.0.0.1:6379> srem myset a b
     (integer) 2
     127.0.0.1:6379> srem myset hello
     (integer) 0
     ```

  3. 计算元素个数

      ```java
      //scard key
      //scard的时间复杂度为O（1） ， 它不会遍历集合所有元素， 而是直接用Redis内部的变量， 例如：
      127.0.0.1:6379> scard myset
      (integer) 1
      ```

  1. 判断元素是否在集合中

     ```java
     //sismember key element
     //如果给定元素element在集合内返回1， 反之返回0， 例如：
     127.0.0.1:6379> sismember myset c
     (integer) 1
     ```

  2. 随机从集合返回指定个数元素

     ```java
     //srandmember key [count]
     //[count]是可选参数， 如果不写默认为1， 例如：
     127.0.0.1:6379> srandmember myset 2
     1) "a"
     2) "c"
     127.0.0.1:6379> srandmember myset
     "d"
     ```

  3. 从集合随机弹出元素

     ```java
     //spop key
     //spop操作可以从集合中随机弹出一个元素， 例如下面代码是一次spop后， 集合元素变为"d b a"：
     123
     127.0.0.1:6379> spop myset
     "c"
     127.0.0.1:6379> smembers myset
     1) "d"
     2) "b"
     3) "a"
     //需要注意的是Redis从3.2版本开始， spop也支持[count]参数。
     //srandmember和spop都是随机从集合选出元素， 两者不同的是spop命令执行后， 元素会从集合中删除， 而srandmember不会。
     ```

  4. 获取所有元素

     ```java
     //smembers key
     //下面代码获取集合myset所有元素， 并且返回结果是无序的：
     127.0.0.1:6379> smembers myset
     1) "d"
     2) "b"
     3) "a"
     //smembers和lrange、 hgetall都属于比较重的命令， 如果元素过多存在阻塞Redis的可能性， 这时候可以使用sscan来完成， 有关sscan命令2.7节会介绍。
     ```

- 集合间操作

  ```java
  //现在有两个集合， 它们分别是user： 1： follow和user： 2： follow：
  127.0.0.1:6379> sadd user:1:follow it music his sports
  (integer) 4
  127.0.0.1:6379> sadd user:2:follow it news ent sports
  (integer) 4
  124
  ```

  1. 求多个集合的交集

     ```java
     //sinter key [key ...]
     //例如下面代码是求user： 1： follow和user： 2： follow两个集合的交集，返回结果是sports、 it：
     127.0.0.1:6379> sinter user:1:follow user:2:follow
     1) "sports"
     2) "it"
     ```

  2. 求多个集合的并集

     ```java
     //suinon key [key ...]
     //例如下面代码是求user： 1： follow和user： 2： follow两个集合的并集，返回结果是sports、 it、 his、 news、 music、 ent：
     127.0.0.1:6379> sunion user:1:follow user:2:follow
     1) "sports"
     2) "it"
     3) "his"
     4) "news"
     5) "music"
     6) "ent"
     ```

  3. 求多个集合的差集

     ```java
     //sdiff key [key ...]
     //例如下面代码是求user： 1： follow和user： 2： follow两个集合的差集，返回结果是music和his：
     127.0.0.1:6379> sdiff user:1:follow user:2:follow
     1) "music"
     2) "his"
     ```

     前面三个命令如图所示
     ![img](https://tinaxiawuhao.github.io/post-images/1620711875701.png)

  4. 将交集、 并集、 差集的结果保存

        ```java
        sinterstore destination key [key ...]
        suionstore destination key [key ...]
        sdiffstore destination key [key ...]
        ```

        集合间的运算在元素较多的情况下会比较耗时， 所以Redis提供了上面三个命令（原命令+store） 将集合间交集、 并集、 差集的结果保存在destination key中， 例如下面操作将user： 1： follow和user： 2： follow两个集合的交集结果保存在user： 1_2： inter中， user： 1_2： inter本身也是集合类
        型：

        ```java
        127.0.0.1:6379> sinterstore user:1_2:inter user:1:follow user:2:follow
        (integer) 2
        127.0.0.1:6379> type user:1_2:inter
        set
        127.0.0.1:6379> smembers user:1_2:inter
        1) "it"
        2) "sports"
        ```

#### Set使用场景

> sadd=Tagging（标签）
> spop/srandmember=Random item（生成随机数， 比如抽奖）
> sadd+sinter=Social Graph（社交需求）

#### Zset

![img](https://tinaxiawuhao.github.io/post-images/1620711899603.png)

- 集合内操作

  1. 添加成员

      ```java
      //zadd key score member [score member ...]
      //下面操作向有序集合user： ranking添加用户tom和他的分数251：
      127.0.0.1:6379> zadd user:ranking 251 tom
      (integer) 1
      //返回结果代表成功添加成员的个数：
      127.0.0.1:6379> zadd user:ranking 1 kris 91 mike 200 frank 220 tim 250 martin
      (integer) 5
      ```

      有关zadd命令有两点需要注意：
      Redis3.2为zadd命令添加了nx、 xx、 ch、 incr四个选项：
      nx： member必须不存在， 才可以设置成功， 用于添加。
      xx： member必须存在， 才可以设置成功， 用于更新。
      ch： 返回此次操作后， 有序集合元素和分数发生变化的个数
      incr： 对score做增加， 相当于后面介绍的zincrby。
      有序集合相比集合提供了排序字段， 但是也产生了代价， zadd的时间复杂度为O（log（n） ） ， sadd的时间复杂度为O（1） 。

  2. 计算成员个数

     ```java
     //zcard key
     //例如下面操作返回有序集合user： ranking的成员数为5， 和集合类型的scard命令一样， zcard的时间复杂度为O（1） 。
     127.0.0.1:6379> zcard user:ranking
     (integer) 5
     ```

  3. 计算某个成员的分数

      ```java
      //zscore key member
      //tom的分数为251， 如果成员不存在则返回nil：
      127.0.0.1:6379> zscore user:ranking tom
      "251"
      127.0.0.1:6379> zscore user:ranking test
      (nil)
      ```

  4. 计算成员的排名

     ```java
     //zrank key member
     //zrevrank key member
     //zrank是从分数从低到高返回排名， zrevrank反之。 例如下面操作中， tom在zrank和zrevrank分别排名第5和第0（排名从0开始计算） 。
     127.0.0.1:6379> zrank user:ranking tom
     (integer) 5
     127.0.0.1:6379> zrevrank user:ranking tom
     (integer) 0
     ```

  5. 删除成员

     ```java
     //zrem key member [member ...]
     //下面操作将成员mike从有序集合user： ranking中删除。
     127.0.0.1:6379> zrem user:ranking mike
     (integer) 1
     //返回结果为成功删除的个数。
     ```

  6. 增加成员的分数

     ```java
     //zincrby key increment member
     //下面操作给tom增加了9分， 分数变为了260分：
     127.0.0.1:6379> zincrby user:ranking 9 tom
     "260"
     ```

  7. 返回指定排名范围的成员

     ```java
     //zrange key start end [withscores]
     //zrevrange key start end [withscores]
     //有序集合是按照分值排名的， zrange是从低到高返回， zrevrange反之。
     //下面代码返回排名最低的是三个成员， 如果加上withscores选项， 同时会返回成员的分数：
     127.0.0.1:6379> zrange user:ranking 0 2 withscores
     1) "kris"
     2) "1"
     3) "frank"
     4) "200"
     5) "tim"
     6) "220"
     127.0.0.1:6379> zrevrange user:ranking 0 2 withscores
     1) "tom"
     2) "260"
     3) "martin"
     4) "250"
     5) "tim"
     6) "220"
     ```

  8. 返回指定分数范围的成员

     ```java
     //zrangebyscore key min max [withscores] [limit offset count]
     //zrevrangebyscore key max min [withscores] [limit offset count]
     //其中zrangebyscore按照分数从低到高返回， zrevrangebyscore反之。 例如下面操作从低到高返回200到221分的成员， withscores选项会同时返回每个成员的分数。 [limit offset count]选项可以限制输出的起始位置和个数：
     127.0.0.1:6379> zrangebyscore user:ranking 200 tinf withscores
     1) "frank"
     2) "200"
     3) "tim"
     4) "220"
     127.0.0.1:6379> zrevrangebyscore user:ranking 221 200 withscores
     1) "tim"
     2) "220"
     3) "frank"
     4) "200"
     //同时min和max还支持开区间（ 小括号） 和闭区间（ 中括号） ， -inf和+inf分别代表无限小和无限大：
     127.0.0.1:6379> zrangebyscore user:ranking (200 +inf withscores
     1) "tim"
     2) "220"
     3) "martin"
     4) "250"
     5) "tom"
     6) "260"
     ```

  9. 返回指定分数范围成员个数

     ```java
     //zcount key min max
     //下面操作返回200到221分的成员的个数：
     127.0.0.1:6379> zcount user:ranking 200 221
     (integer) 2
     ```

  10. 删除指定排名内的升序元素

        ```java
        //zremrangebyrank key start end
        //下面操作删除第start到第end名的成员：
        127.0.0.1:6379> zremrangebyrank user:ranking 0 2
        (integer) 3
        ```

  11. 删除指定分数范围的成员

        ```java
        //zremrangebyscore key min max
        //下面操作将250分以上的成员全部删除， 返回结果为成功删除的个数：
        127.0.0.1:6379> zremrangebyscore user:ranking (250 +inf
        (integer) 2
        ```

- 集合间操作

  将图中的两个有序集合导入到Redis中。
  ![img](https://tinaxiawuhao.github.io/post-images/1620711939016.png)

  ```java
  127.0.0.1:6379> zadd user:ranking:1 1 kris 91 mike 200 frank 220 tim 250 martin
  251 tom
  (integer) 6
  127.0.0.1:6379> zadd user:ranking:2 8 james 77 mike 625 martin 888 tom
  (integer) 4
  ```

  1. 交集

     ```java
     //zinterstore destination numkeys key [key ...] [weights weight [weight ...]][aggregate sum|min|max]
     //这个命令参数较多， 下面分别进行说明：
     //·destination： 交集计算结果保存到这个键。
     //·numkeys： 需要做交集计算键的个数。
     //·key[key...]： 需要做交集计算的键。
     //·weights weight[weight...]： 每个键的权重， 在做交集计算时， 每个键中的每个member会将自己分数乘以这个权重， 每个键的权重默认是1。
     //·aggregate sum|min|max： 计算成员交集后， 分值可以按照sum（ 和） 、min（ 最小值） 、 max（ 最大值） 做汇总， 默认值是sum。
     //下面操作对user： ranking： 1和user： ranking： 2做交集， weights和aggregate使用了默认配置， 可以看到目标键user： ranking： 1_inter_2对分值做了sum操作：
     127.0.0.1:6379> zinterstore user:ranking:1_inter_2 2 user:ranking:1
     user:ranking:2
     (integer) 3
     127.0.0.1:6379> zrange user:ranking:1_inter_2 0 -1 withscores
     1) "mike"
     2) "168"
     3) "martin"
     4) "875"
     5) "tom"
     6) "1139"
     //如果想让user： ranking： 2的权重变为0.5， 并且聚合效果使用max， 可以执行如下操作：
     127.0.0.1:6379> zinterstore user:ranking:1_inter_2 2 user:ranking:1
     user:ranking:2 weights 1 0.5 aggregate max
     (integer) 3
     127.0.0.1:6379> zrange user:ranking:1_inter_2 0 -1 withscores
     1) "mike"
     2) "91"
     3) "martin"
     4) "312.5"
     5) "tom"
     6) "444"
     ```

  2. 并集

     ```java
     //zunionstore destination numkeys key [key ...] [weights weight [weight ...]][aggregate sum|min|max]
     //该命令的所有参数和zinterstore是一致的， 只不过是做并集计算， 例如下面操作是计算user： ranking： 1和user： ranking： 2的并集， weights和
     //aggregate使用了默认配置， 可以看到目标键user： ranking： 1_union_2对分值做了sum操作：
     127.0.0.1:6379> zunionstore user:ranking:1_union_2 2 user:ranking:1
     user:ranking:2
     (integer) 7
     127.0.0.1:6379> zrange user:ranking:1_union_2 0 -1 withscores
     1) "kris"
     2) "1"
     3) "james"
     4) "8"
     5) "mike"
     6) "168"
     7) "frank"
     8) "200"
     9) "tim"
     10) "220"
     11) "martin"
     12) "875"
     13) "tom"
     14) "1139"
     ```

#### Zset使用场景

有序集合比较典型的使用场景就是排行榜系统。 例如视频网站需要对用户上传的视频做排行榜， 榜单的维度可能是多个方面的： 按照时间、 按照播
放数量、 按照获得的赞数。 本节使用赞数这个维度， 记录每天用户上传视频的排行榜。 主要需要实现以下4个功能。

1. 添加用户赞数
   例如用户mike上传了一个视频， 并获得了3个赞， 可以使用有序集合的zadd和zincrby功能：

   ```sh
   zadd user:ranking:2016_03_15 mike 3
   ```

   如果之后再获得一个赞， 可以使用zincrby：

   ```sh
   zincrby user:ranking:2016_03_15 mike 1
   ```

2. 取消用户赞数
   由于各种原因（例如用户注销、 用户作弊） 需要将用户删除， 此时需要将用户从榜单中删除掉， 可以使用zrem。 例如删除成员tom：

   ```sh
   zrem user:ranking:2016_03_15 mike
   ```

3. 展示获取赞数最多的十个用户
   此功能使用zrevrange命令实现：

   ```sh
   zrevrangebyrank user:ranking:2016_03_15 0 9
   ```

4. 展示用户信息以及用户分数
   此功能将用户名作为键后缀， 将用户信息保存在哈希类型中， 至于用户的分数和排名可以使用zscore和zrank两个功能：

   ```sh
   hgetall user:info:tom
   zscore user:ranking:2016_03_15 mike
   zrank user:ranking:2016_03_15 mik  
   ```

#### Hash

![img](https://tinaxiawuhao.github.io/post-images/1620711973275.png)

1. 设置值

   ```java
   //hset key field value
   //下面为user： 1添加一对field-value：
   127.0.0.1:6379> hset user:1 name tom
   (integer) 1
   //如果设置成功会返回1， 反之会返回0。 此外Redis提供了hsetnx命令， 它们的关系就像set和setnx命令一样， 只不过作用域由键变为field。
   ```

2. 获取值

   ```java
   //hget key field
   //例如， 下面操作获取user： 1的name域（属性） 对应的值：
   127.0.0.1:6379> hget user:1 name
   "tom"
   //如果键或field不存在， 会返回nil：
   127.0.0.1:6379> hget user:2 name
   (nil)
   127.0.0.1:6379> hget user:1 age
   (nil)
   ```

3. 删除field

   ```java
   //hdel key field [field ...]
   //hdel会删除一个或多个field， 返回结果为成功删除field的个数， 例如：
   127.0.0.1:6379> hdel user:1 name
   (integer) 1
   127.0.0.1:6379> hdel user:1 age
   (integer) 0
   ```

4. 计算field个数

   ```java
   //hlen key
   //例如user： 1有3个field：
   127.0.0.1:6379> hset user:1 name tom
   (integer) 1
   127.0.0.1:6379> hset user:1 age 23
   (integer) 1
   127.0.0.1:6379> hset user:1 city tianjin
   (integer) 1
   127.0.0.1:6379> hlen user:1
   (integer) 3
   ```

5. 批量设置或获取field-value

    ```java
    //hmget key field [field ...]
    //hmset key field value [field value ...]
    //hmset和hmget分别是批量设置和获取field-value， hmset需要的参数是key和多对field-value， hmget需要的参数是key和多个field。 例如：
    127.0.0.1:6379> hmset user:1 name mike age 12 city tianjin
    OK
    127.0.0.1:6379> hmget user:1 name city
    1) "mike"
    2) "tianjin"
    ```

6. 判断field是否存在

   ```java
   //hexists key field
   //例如， user： 1包含name域， 所以返回结果为1， 不包含时返回0：
   127.0.0.1:6379> hexists user:1 name
   (integer) 1
   ```

7. 获取所有field

   ```java
   //hkeys key
   //hkeys命令应该叫hfields更为恰当， 它返回指定哈希键所有的field， 例如：
   127.0.0.1:6379> hkeys user:1
   1) "name"
   2) "age"
   3) "city"
   ```

8. 获取所有value

   ```java
   //hvals key
   下面操作获取user： 1全部value：
   127.0.0.1:6379> hvals user:1
   1) "mike"
   2) "12"
   3) "tianjin"
   ```

9. 获取所有的field-value

   ```java
   //hgetall key
   //下面操作获取user： 1所有的field-value：
   127.0.0.1:6379> hgetall user:1
   1) "name"
   2) "mike"
   3) "age"
   4) "12"
   5) "city"
   6) "tianjin"
   ```

   在使用hgetall时， 如果哈希元素个数比较多， 会存在阻塞Redis的可能。
   如果开发人员只需要获取部分field， 可以使用hmget， 如果一定要获取全部field-value， 可以使用hscan命令， 该命令会渐进式遍历哈希类型.

10. hincrby hincrbyfloat

    ```java
    //hincrby key field
    //hincrbyfloat key field
    //hincrby和hincrbyfloat， 就像incrby和incrbyfloat命令一样， 但是它们的作用域是filed。
    ```

11. 计算value的字符串长度（需要Redis3.2以上）

    ```java
    //hstrlen key field
    //例如hget user： 1name的value是tom， 那么hstrlen的返回结果是3：
    127.0.0.1:6379> hstrlen user:1 name
    (integer) 3
    ```

#### Hash使用场景

 保存对象信息
![img](https://tinaxiawuhao.github.io/post-images/1620712022434.png)

### Redis 发布订阅

Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息。

Redis 客户端可以订阅任意数量的频道。

下图展示了频道 channel1 ， 以及订阅这个频道的三个客户端 —— client2 、 client5 和 client1 之间的关系：

![](https://tinaxiawuhao.github.io/post-images/1634545852554.png)

当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端：

![](https://tinaxiawuhao.github.io/post-images/1634545878403.png)

------

#### 实例

以下实例演示了发布订阅是如何工作的，需要开启两个 redis-cli 客户端。

在我们实例中我们创建了订阅频道名为 **runoobChat**:

##### 第一个 redis-cli 客户端

> redis 127.0.0.1:6379> SUBSCRIBE runoobChat
>
> Reading messages... (press Ctrl-C to quit)
> 1) "subscribe"
> 2) "redisChat"
> 3) (integer) 1

现在，我们先重新开启个 redis 客户端，然后在同一个频道 runoobChat 发布两次消息，订阅者就能接收到消息。

##### 第二个 redis-cli 客户端

> redis 127.0.0.1:6379> PUBLISH runoobChat "Redis PUBLISH test"
>
> (integer) 1
>
> redis 127.0.0.1:6379> PUBLISH runoobChat "Learn redis by runoob.com"
>
> (integer) 1
>
> \# 订阅者的客户端会显示如下消息
>  1) "message"
> 2) "runoobChat"
> 3) "Redis PUBLISH test"
>  1) "message"
> 2) "runoobChat"
> 3) "Learn redis by runoob.com"

### springboot中实现

**一.配置文件**

```yaml
spring:
  redis:
     host: 192.168.0.200
     port: 6379
     password:
     database: 1
     pool.max-active: 8
     pool.max-wait: -1
     pool.max-idle: 8
     pool.min-idle: 0
     timeout: 5000
```

**二.maven坐标**

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
**三.redis消息监听器容器以及redis监听器注入IOC容器**

```java
package com.example.demo;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.listener.PatternTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;

@Configuration
@EnableCaching
public class RedisConfig{
    /**
	  * Redis消息监听器容器
      * @param connectionFactory
      * @return
    **/
    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        //订阅了一个叫pmp和channel 的通道，多通道
        container.addMessageListener(listenerAdapter(new RedisPmpSub()),new PatternTopic("pmp"));
        container.addMessageListener(listenerAdapter(new RedisChannelSub()),new PatternTopic("channel"));
        //这个container 可以添加多个 messageListener
        return container;
    }
    
    /**
     * 配置消息接收处理类
     * @param redisMsg  自定义消息接收类
     * @return
     */
    @Bean()
    @Scope("prototype")
    MessageListenerAdapter listenerAdapter(RedisMsg redisMsg) {
        //这个地方 是给messageListenerAdapter 传入一个消息接受的处理器，利用反射的方法调用“receiveMessage”
        //也有好几个重载方法，这边默认调用处理器的方法 叫handleMessage 可以自己到源码里面看
        return new MessageListenerAdapter(redisMsg, "receiveMessage");//注意2个通道调用的方法都要为receiveMessage
    }

}
```



**四.普通的消息处理器接口**

```java
package com.example.demo;

import org.springframework.stereotype.Component;


@Component
public interface RedisMsg {

    public void receiveMessage(String message);

}
```

**五.普通的消息处理器POJO**

```java
package com.example.demo;

/**

 * @Auther: Administrator
 * @Date: 2018/7/9 11:01
 * @Description:
   */
    public class RedisChannelSub implements RedisMsg {
   @Override
   public void receiveMessage(String message) {
       //注意通道调用的方法名要和RedisConfig的listenerAdapter的MessageListenerAdapter参数2相同
       System.out.println("这是RedisChannelSub"+"-----"+message);
   }
    }
    package com.example.demo;


public class RedisPmpSub implements RedisMsg{

    /**
     * 接收消息的方法
     * @param message 订阅消息
     */
    public void receiveMessage(String message){
        //注意通道调用的方法名要和RedisConfig的listenerAdapter的MessageListenerAdapter参数2相同
        System.out.println(message);
    }

}
```



**六.消息发布者**

```java
package com.example.demo;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

//定时器
@EnableScheduling
@Component
public class TestSenderController {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    //向redis消息队列index通道发布消息
    @Scheduled(fixedRate = 2000)
    public void sendMessage(){
        stringRedisTemplate.convertAndSend("pmp",String.valueOf(Math.random()));
        stringRedisTemplate.convertAndSend("channel",String.valueOf(Math.random()));
    }

}
```

**七.启动类**

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```
### 3种高级数据结构
#### BitMap

BitMap，即位图，其实也就是 byte 数组，用二进制表示，只有 0 和 1 两个数字。

如图所示：

![](https://tinaxiawuhao.github.io/post-images/1634546661392.webp)

**重要 API**

|               命令                |                             含义                             |
| :-------------------------------: | :----------------------------------------------------------: |
|         getbit key offset         |      对key所存储的字符串值，获取指定偏移量上的位（bit）      |
|      setbit key offset value      | 对key所存储的字符串值，设置或清除指定偏移量上的位（bit） 1. 返回值为该位在setbit之前的值 2. value只能取0或1 3. offset从0开始，即使原位图只能10位，offset可以取1000 |
|     bitcount key [start end]      | 获取位图指定范围中位值为1的个数 如果不指定start与end，则取所有 |
|  bitop op destKey key1 [key2...]  | 做多个BitMap的and（交集）、or（并集）、not（非）、xor（异或）操作并将结果保存在destKey中 |
| bitpos key tartgetBit [start end] | 计算位图指定范围第一个偏移量对应的的值等于targetBit的位置 1. 找不到返回-1 2. start与end没有设置，则取全部 3. targetBit只能取0或者1 |

**演示**

![](https://tinaxiawuhao.github.io/post-images/1634546673051.webp)

**应用场景**

统计每日用户的登录数。每一位标识一个用户ID，当某个用户访问我们的网页或执行了某个操作，就在bitmap中把标识此用户的位设置为1。

这里做了一个 使用 set 和 BitMap 存储的对比。

**场景1：1 亿用户，5千万独立**

| 数据类型 |                  每个 userid 占用空间                  | 需要存储的用户量 |          全部内存量          |
| :------: | :----------------------------------------------------: | :--------------: | :--------------------------: |
|   set    | 32位（假设userid用的是整型，实际很多网站用的是长整型） |    50,000,000    |  32位 * 50,000,000 = 200 MB  |
|  BitMap  |                          1 位                          |   100,000,000    | 1 位 * 100,000,000 = 12.5 MB |

|        | 一天  | 一个月 | 一年 |
| :----: | :---: | :----: | :--: |
|  set   | 200M  |   6G   | 72G  |
| BitMap | 12.5M |  375M  | 4.5G |

**场景2：只有 10 万独立用户**

| 数据类型 |                  每个 userid 占用空间                  | 需要存储的用户量 |          全部内存量          |
| :------: | :----------------------------------------------------: | :--------------: | :--------------------------: |
|   set    | 32位（假设userid用的是整型，实际很多网站用的是长整型） |    1,000,000     |   32位 * 1,000,000 = 4 MB    |
|  BitMap  |                          1 位                          |   100,000,000    | 1 位 * 100,000,000 = 12.5 MB |

通过上面的对比，我们可以看到，如果独立用户数量很多，使用 BitMap 明显更有优势，能节省大量的内存。但如果独立用户数量较少，还是建议使用 set 存储，BitMap 会产生多余的存储开销。

**使用经验**

1. type = string，BitMap 是 sting 类型，最大 512 MB。
2. 注意 setbit 时的偏移量，可能有较大耗时
3. 位图不是绝对好。

#### HyperLogLog

   HyperLogLog 是基于 HyperLogLog 算法的一种数据结构，该算法可以在极小空间完成独立数量统计。

   在本质上还是字符串类型。

![](https://tinaxiawuhao.github.io/post-images/1634546953753.webp)

 **重要 API**

|               命令               |              含义              |
| :------------------------------: | :----------------------------: |
| pfadd key element1 [element2...] |    向HyperLogLog中添加元素     |
|      pfcount key1 [key2...]      |   计算HyperLogLog的独立总数    |
|  pfmerge destKey key1 [key2...]  | 合并多个hyperLogLog到destKey中 |

 **演示**

![](https://tinaxiawuhao.github.io/post-images/1634546964179.webp)

 **内存消耗**

   以百万独立用户为例

|        |     内存消耗      |
| :----: | :---------------: |
|  1 天  |       15 KB       |
| 1 个月 |      450 KB       |
|  1 年  | 15KB * 365 = 5 MB |

   可以看到内存消耗是非常低的，比我们之前学过的 BitMap 还要低得多。

**使用经验**

   > Q：既然 HyperLogLog 那么好，那么是不是以后用这个来存储数据就行了呢？
   >
   > A：这里要考虑两个因素：
   >
   > 1. hyperloglog 的错误率为：0.81%，存储的数据不能百分百准确。
   > 2. hyperloglog 不能取出单条数据。api 中也没有相关操作。

   如果你没有这两个方面的顾虑，那么用 HyperLogLog 来存储大规模数据，还是非常不错的。
#### GEO

- Redis 3.2添加新特性
- 功能：存储经纬度、计算两地距离、范围计算等
- 基于ZSet实现
- 删除操作使用 `zrem key member`

**GEO 相关命令**

**1. geoadd key longitude latitude member [lon lat member...]**

- 含义：增加地理位置信息
  - longitude ：经度
  - latitude     :  纬度
  - member   :  标识信息

**2. geopos key member1 [member2...]**

- 含义：获取地理位置信息

**3. geodist key member1 member2 [unit]**

- 含义：获取两个地理位置的距离
- unit取值范围
  - m（米，默认）
  - km（千米）
  - mi（英里）
  - ft（英尺）

**4. georadius key longitude latitude unit [withcoord] [withdist] [withhash] [COUNT count] [sort] [store key] [storedist key]**

- 含义：以给定的经纬度为中心，返回包含的位置元素当中，与中心距离不超过给定最大距离的所有位置元素。
- unit取值范围
  - m（米）
  - km（千米）
  - mi（英里）
  - ft（英尺）
- withcoord：将位置元素的经度与纬度也一并返回
- withdist：在返回位置元素的同时，将距离也一并返回。距离的单位和用户给定的范围单位保持一致
- withhash：以52位的符号整数形式，返回位置元素经过geohash编码的有序集合分值。用于底层应用或调试，实际作用不大。
- sort取值范围
  - asc：根据中心位置，按照从近到远的方式返回位置元素
  - desc：根据中心位置，按照从远到近的方式返回位置元素
- store key：将返回结果而的地理位置信息保存到指定键
- storedist key：将返回结果距离中心节点的距离保存到指定键

**5. georadiusbymember key member radius unit [withcoord][withdist][withhash][COUNT count][sort][store key][storedist key]**

- 含义：以给定的元素为中心，返回包含的位置元素当中，与中心距离不超过给定最大距离的所有位置元素。
- unit取值范围
  - m（米）
  - km（千米）
  - mi（英里）
  - ft（英尺）
- withcoord：将位置元素的经度与纬度也一并返回
- withdist：在返回位置元素的同时，将距离也一并返回。距离的单位和用户给定的范围单位保持一致
- withhash：以52位的符号整数形式，返回位置元素经过geohash编码的有序集合分值。用于底层应用或调试，实际作用不大。
- sort取值范围
  - asc：根据中心位置，按照从近到远的方式返回位置元素
  - desc：根据中心位置，按照从远到近的方式返回位置元素
- store key：将返回结果而的地理位置信息保存到指定键
- storedist key：将返回结果距离中心节点的距离保存到指定键

**演示**

geo 功能是在 redis-3.2 后引入的

```java
127.0.0.1:6381> geoadd cities:locations 116.28 39.55 beijing
(integer) 1
127.0.0.1:6381> geoadd cities:locations 117.12 39.08 tianjin 114.29 38.02 shijiazhuang 118.01 39.38 tangshan 115.29 38.51 baoding
(integer) 4
127.0.0.1:6381> geopos cities:locations tianjin
1) 1) "117.12000042200088501"
   2) "39.0800000535766543"
127.0.0.1:6381> geodist cities:locations tianjin beijing km
"89.2061"
127.0.0.1:6379> georadiusbymember cities:locations beijing 150 km
1) "beijing"
2) "tianjin"
3) "tangshan"
4) "baoding"。
```
   