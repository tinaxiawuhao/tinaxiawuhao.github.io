---
title: 'mysql概述五'
date: 2021-05-22 13:09:05
tags: [mysql]
published: true
hideInList: false
feature: /post-images/lbX34toJL.png
isTop: false
---
### SQL优化

对于低性能的SQL语句的定位，最重要也是最有效的方法就是使用执行计划，MySQL提供了explain命令来查看语句的执行计划。 我们知道，不管是哪种数据库，或者是哪种数据库引擎，在对一条SQL语句进行执行的过程中都会做很多相关的优化，**对于查询语句，最重要的优化方式就是使用索引**。 而**执行计划，就是显示数据库引擎对于SQL语句的执行的详细情况，其中包含了是否使用索引，使用什么索引，使用的索引的相关信息等**。

![](https://tinaxiawuhao.github.io/post-images/1620973419410.png)

执行计划包含的信息 **id** 有一组数字组成。表示一个查询中各个子查询的执行顺序;

- id相同执行顺序由上至下。
- id不同，id值越大优先级越高，越先被执行。
- id为null时表示一个结果集，不需要使用它查询，常出现在包含union等查询语句中。

**select_type** 每个子查询的查询类型，一些常见的查询类型。

| id   | select_type  | description                               |
| ---- | ------------ | ----------------------------------------- |
| 1    | SIMPLE       | 不包含任何子查询或union等查询             |
| 2    | PRIMARY      | 包含子查询最外层查询就显示为 PRIMARY      |
| 3    | SUBQUERY     | 在select或 where字句中包含的查询          |
| 4    | DERIVED      | from字句中包含的查询                      |
| 5    | UNION        | 出现在union后的查询语句中                 |
| 6    | UNION RESULT | 从UNION中获取结果集，例如上文的第三个例子 |

**type**(非常重要，可以看到有没有走索引) 访问类型

- `ALL` 扫描全表数据
- `index` 遍历索引
- `range` 索引范围查找
- `index_subquery` 在子查询中使用 `ref`
- `unique_subquery` 在子查询中使用 `eq_ref`
- `ref_or_null` 对Null进行索引的优化的 `ref`
- `fulltext` 使用全文索引
- `ref` 使用非唯一索引查找数据
- `eq_ref` 在join查询中使用`PRIMARY KEY`  or  `UNIQUE NOT NULL`索引关联。

**possible_keys** 可能使用的索引，注意不一定会使用。查询涉及到的字段上若存在索引，则该索引将被列出来。当该列为 NULL时就要考虑当前的SQL是否需要优化了。

**key** 显示MySQL在查询中实际使用的索引，若没有使用索引，显示为NULL。

**TIPS**:查询中若使用了覆盖索引(覆盖索引：索引的数据覆盖了需要查询的所有数据)，则该索引仅出现在key列表中

**key_length** 索引长度

**ref** 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

**rows** 返回估算的结果集数目，并不是一个准确的值。

**extra** 的信息非常丰富，常见的有：

1. Using index 使用覆盖索引
2. Using where 使用了用where子句来过滤结果集
3. Using filesort 使用文件排序，使用非索引列进行排序时出现，非常消耗性能，尽量优化。
4. Using temporary 使用了临时表 sql优化的目标可以参考阿里开发手册

【推荐】SQL性能优化的目标：至少要达到 range 级别，要求是ref级别，如果可以是consts最好。  

说明： 

 1） consts 单表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可读取到数据。

 2） ref 指的是使用普通的索引（normal index）。 

 3） range 对索引进行范围检索。  反例：explain表的结果，type=index，索引物理文件全扫描，速度非常慢，这个index级别比较range还低，与全表扫描是小巫见大巫。

### SQL执行顺序

mysql执行sql的顺序从 From 开始，以下是执行的顺序流程

1. FROM  `FROM  table1 left join table2 on` 将table1和table2中的数据产生笛卡尔积，生成Temp1

2. JOIN `JOIN table2`  所以先是确定表，再确定关联条件

3. ON `ON table1.column = table2.columu` 确定表的绑定条件 由Temp1产生中间表Temp2

4. WHERE  对中间表Temp2产生的结果进行过滤  产生中间表Temp3

5. GROUP BY 对中间表Temp3进行分组，产生中间表Temp4

6. HAVING  对分组后的记录进行聚合 产生中间表Temp5

7. SELECT  对中间表Temp5进行列筛选，产生中间表 Temp6

8. DISTINCT 对中间表 Temp6进行去重，产生中间表 Temp7

9. ORDER BY 对Temp7中的数据进行排序，产生中间表Temp8

10. LIMIT 对中间表Temp8进行分页，产生中间表Temp9

### SQL的生命周期

1. 与服务器建立连接，客户端发送一条查询给服务器
2. 服务器先检查查询缓存，如果命中了缓存则立刻返回存储在缓存中的结果，否则执行下一步
3. 服务端进行sql解析，预处理，再由查询优化器生成对应的查询执行计划
4. mysql根据优化器提供的执行计划，调用存储引擎的api来执行查询
5. 通过步骤一的连接，发送结果到客户端
6. 关掉连接，释放资源

![](https://tinaxiawuhao.github.io/post-images/1620973396577.png)

### 优化大表数据查询

1. 优化shema、sql语句+索引；
2. 加缓存，`memcached`, `redis`；
3. 主从复制，读写分离；
4. 垂直拆分，根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统；
5. 水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key, 为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

### 超大分页怎么处理

超大的分页一般从两个方向上来解决.

- 数据库层面,这也是我们主要集中关注的(虽然收效没那么大),类似于`select * from table where age > 20 limit 1000000,10`这种查询其实也是有可以优化的余地的. 这条语句需要`load1000000`数据然后基本上全部丢弃,只取10条当然比较慢. 当时我们可以修改为`select * from table where id in (select id from table where age > 20 limit 1000000,10)`.这样虽然也load了一百万的数据,但是由于索引覆盖,要查询的所有字段都在索引中,所以速度会很快. 同时如果ID连续的好,我们还可以`select * from table where id > 1000000 limit 10`,效率也是不错的,优化的可能性有许多种,但是核心思想都一样,就是减少load的数据.
- 从需求的角度减少这种请求…主要是不做类似的需求(直接跳转到几百万页之后的具体某一页.只允许逐页查看或者按照给定的路线走,这样可预测,可缓存)以及防止ID泄漏且连续被人恶意攻击.

解决超大分页,其实主要是靠缓存,可预测性的提前查到内容,缓存至`redis`等k-V数据库中,直接返回即可.

在阿里巴巴《Java开发手册》中,对超大分页的解决办法是类似于上面提到的第一种.

【推荐】利用延迟关联或者子查询优化超多分页场景。  说明：MySQL并不是跳过offset行，而是取offset+N行，然后返回放弃前offset行，返回N行，那当offset特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行SQL改写。  正例：先快速定位需要获取的id段，然后再关联：  `SELECT a.* FROM 表1 a, (select id from 表1 where 条件 LIMIT 100000,20 ) b where a.id=b.id`

### mysql 分页

LIMIT 子句可以被用于强制 SELECT 语句返回指定的记录数。LIMIT 接受一个或两个数字参数。参数必须是一个整数常量。如果给定两个参数，第一个参数指定第一个返回记录行的偏移量，第二个参数指定返回记录行的最大数目。初始记录行的偏移量是 0(而不是 1)

```mysql
mysql> SELECT * FROM table LIMIT 5,10; // 检索记录行 6-15 
```

为了检索从某一个偏移量到记录集的结束所有的记录行，可以指定第二个参数为 -1：

```mysql
mysql> SELECT * FROM table LIMIT 95,-1; // 检索记录行 96-last. 
```

如果只给定一个参数，它表示返回最大的记录行数目：

```mysql
mysql> SELECT * FROM table LIMIT 5; //检索前 5 个记录行 
```

换句话说，LIMIT n 等价于 LIMIT 0,n。

### 慢查询日志

用于记录执行时间超过某个临界值的SQL日志，用于快速定位慢查询，为我们的优化做参考。

**开启慢查询日志**

配置项：`slow_query_log`

可以使用`show variables like ‘slov_query_log’`查看是否开启，如果状态值为OFF，可以使用`set GLOBAL slow_query_log = on`来开启，它会在`datadir`下产生一个`xxx-slow.log`的文件。

**设置临界时间**

配置项：`long_query_time`

查看：`show VARIABLES like 'long_query_time'`，单位秒

设置：`set long_query_time=0.5`

实操时应该从长时间设置到短的时间，即将最慢的SQL优化掉

查看日志，一旦SQL超过了我们设置的临界时间就会被记录到`xxx-slow.log`中

#### 慢查询处理

在业务系统中，除了使用主键进行的查询，其他的会在测试库上测试其耗时，慢查询的统计主要由运维在做，会定期将业务中的慢查询反馈给我们。

慢查询的优化首先要搞明白慢的原因是什么？ 是查询条件没有命中索引？是load了不需要的数据列？还是数据量太大？

所以优化也是针对这三个方向来的，

- 首先分析语句，看看是否load了额外的数据，可能是查询了多余的行并且抛弃掉了，可能是加载了许多结果中并不需要的列，对语句进行分析以及重写。
- 分析语句的执行计划，然后获得其使用索引的情况，之后修改语句或者修改索引，使得语句可以尽可能的命中索引。
- 如果对语句的优化已经无法进行，可以考虑表中的数据量是否太大，如果是的话可以进行横向或者纵向的分表。

### 主键必要性

主键是数据库确保数据行在整张表唯一性的保障，即使业务上本张表没有主键，也建议添加一个自增长的ID列作为主键。设定了主键之后，在后续的删改查的时候可能更加快速以及确保操作数据范围安全。

#### 主键使用自增ID还是UUID

推荐使用自增ID，不要使用UUID。

因为在InnoDB存储引擎中，主键索引是作为聚簇索引存在的，也就是说，主键索引的B+树叶子节点上存储了主键索引以及全部的数据(按照顺序)，如果主键索引是自增ID，那么只需要不断向后排列即可，如果是UUID，由于到来的ID与原来的大小不确定，会造成非常多的数据插入，数据移动，然后导致产生很多的内存碎片，进而造成插入性能的下降。

总之，在数据量大一些的情况下，用自增主键性能会好一些。

由于主键是聚簇索引，如果没有主键，InnoDB会选择一个唯一键来作为聚簇索引，如果没有唯一键，会生成一个隐式的主键。

**字段为什么要求定义为not null**

null值会占用更多的字节，且会在程序中造成很多与预期不符的情况。

**如果要存储用户的密码散列，应该使用什么字段进行存储？**

密码散列，盐，用户身份证号等固定长度的字符串应该使用char而不是varchar来存储，这样可以节省空间且提高检索效率。

### 优化查询过程中的数据访问

- 访问数据太多导致查询性能下降
- 确定应用程序是否在检索大量超过需要的数据，可能是太多行或列
- 确认MySQL服务器是否在分析大量不必要的数据行
- 避免犯如下SQL语句错误
  - 查询不需要的数据。解决办法：使用limit解决
  - 多表关联返回全部列。解决办法：指定列名
  - 总是返回全部列。解决办法：避免使用SELECT *
  - 重复查询相同的数据。解决办法：可以缓存数据，下次直接读取缓存
  - 是否在扫描额外的记录。解决办法：
    - 使用explain进行分析，如果发现查询需要扫描大量的数据，但只返回少数的行，可以通过如下技巧去优化：
      - 使用索引覆盖扫描，把所有的列都放到索引中，这样存储引擎不需要回表获取对应行就可以返回结果。
      - 改变数据库和表的结构，修改数据表范式
      - 重写SQL语句，让优化器可以以更优的方式执行查询。

### 优化长难的查询语句

- 一个复杂查询还是多个简单查询
- MySQL内部每秒能扫描内存中上百万行数据，相比之下，响应数据给客户端就要慢得多
- 使用尽可能小的查询是好的，但是有时将一个大的查询分解为多个小的查询是很有必要的。
- 切分查询
- 将一个大的查询分为多个小的相同的查询
- 一次性删除1000万的数据要比一次删除1万，暂停一会的方案更加损耗服务器开销。
- 分解关联查询，让缓存的效率更高。
- 执行单个查询可以减少锁的竞争。
- 在应用层做关联更容易对数据库进行拆分。
- 查询效率会有大幅提升。
- 较少冗余记录的查询。

### 优化特定类型的查询语句

- count(*)会忽略所有的列，直接统计所有列数，不要使用count(列名)
- MyISAM中，没有任何where条件的count(*)非常快。
- 当有where条件时，MyISAM的count统计不一定比其它引擎快。
- 可以使用explain查询近似值，用近似值替代count(*)
- 增加汇总表
- 使用缓存

### 优化关联查询

- 确定ON或者USING子句中是否有索引。
- 确保GROUP BY和ORDER BY只有一个表中的列，这样MySQL才有可能使用索引。

### 优化子查询

- 用关联查询替代
- 优化GROUP BY和DISTINCT
  - 这两种查询可以使用索引来优化，是最有效的优化方法
- 关联查询中，使用标识列分组的效率更高
- 如果不需要ORDER BY，进行GROUP BY时加ORDER BY NULL，MySQL不会再进行文件排序。
- WITH ROLLUP超级聚合，可以挪到应用程序处理

### 优化LIMIT分页

- LIMIT偏移量大的时候，查询效率较低
- 可以记录上次查询的最大ID，下次查询时直接根据该ID来查询

### 优化UNION查询

- UNION ALL的效率高于UNION

### 优化WHERE子句

对于此类问题，先说明如何定位低效SQL语句，然后根据SQL语句可能低效的原因做排查，先从索引着手，如果索引没有问题，考虑以上几个方面，数据访问的问题，长难查询句的问题还是一些特定类型优化的问题，逐一回答。

SQL语句优化的一些方法？

- 1.对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

- 2.应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：

  ```mysql
  select id from t where num is null 
  -- 可以在num上设置默认值0，确保表中num列没有null值，然后这样查询： 
  select id from t where num=0
  ```

- 3.应尽量避免在 where 子句中使用!=或<>操作符，否则引擎将放弃使用索引而进行全表扫描。

- 4.应尽量避免在 where 子句中使用or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

  ```mysql
  select id from t where num=10 or num=20 
  -- 可以这样查询： 
  select id from t where num=10 union all select id from t where num=20
  ```

- 5.in 和 not in 也要慎用，否则会导致全表扫描，如：

  ```mysql
  select id from t where num in(1,2,3)  
  -- 对于连续的数值，能用 between 就不要用 in 了： 
  select id from t where num between 1 and 3
  ```

- 6.下面的查询也将导致全表扫描：

  ```mysql
  select id from t where name like ‘%李%’ -- 若要提高效率，可以考虑全文检索。
  ```

  

- 7.如果在 where 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然 而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：

  ```mysql
  select id from t where num=@num 
  -- 可以改为强制查询使用索引： 
  select id from t with(index(索引名)) where num=@num
  ```

- 8.应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：

  ```mysql
  select id from t where num/2=100 
  -- 应改为: 
  select id from t where num=100*2
  ```

- 9.应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

  ```mysql
  select id from t where substring(name,1,3)=’abc’ 
  -- name以abc开头的id应改为: 
  select id from t where name like ‘abc%’
  ```

- 10.不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。