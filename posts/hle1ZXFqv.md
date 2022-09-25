---
title: 'Clickhouse-5数据分区'
date: 2021-11-16 15:16:43
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/hle1ZXFqv.png
isTop: false
---
## MergeTree 数据分区规则

创建按照月份为分区条件的表 **tab_partition**

```csharp
CREATE TABLE tab_partition(`dt` Date, `v` UInt8) 
ENGINE = MergeTree PARTITION BY toYYYYMM(dt) ORDER BY v;
insert into tab_partition(dt,v) values ('2020-02-11',1),('2020-02-13',2);
insert into tab_partition(dt,v) values ('2020-04-11',3),('2020-04-13',4);
insert into tab_partition(dt,v) values ('2020-09-11',5),('2020-09-10',6);
insert into tab_partition(dt,v) values ('2020-10-12',7),('2020-10-09',8);
insert into tab_partition(dt,v) values ('2020-02-14',9),('2020-02-15',10);
insert into tab_partition(dt,v) values ('2020-02-11',23),('2020-02-13',45);
```

MergeTree 存储引擎在写入数据之后生成对应的分区文件为：

![](https://tianxiawuhao.github.io/post-images/1639552667308.webp)

MergeTree 的分区目录是在写入数据的过程中被创建出来，每 insert 一次，就会创建一批次分区目录。也就是说如果仅创建表结构，是不会创建分区目录的，因为木有数据。

MergeTree 数据分区目录命名规则其规则为：**PartitionID_MinBlockNum_MaxBlockNum_Level**
 比如 **202002_4_4_0** 其中 202002 是分区ID ，**4_4** 对应的是
 最小的数据块编号和最大的数据块编号，最后的 _0 表示目前分区合并的层级。
 各部分的含义及命名规则如下：
 PartitionID：该值由 insert 数据时分区键的值来决定。分区键支持使用任何一个或者多个字段组合表达式，针对取值数据类型的不同，分区 ID 的生成逻辑目前有四种规则：

> 不指定分区键：如果建表时未指定分区键，则分区 ID 默认使用 all，所有数据都被写入 all 分区中。
>
> 整型字段：如果分区键取值是整型字段，并且无法转换为 YYYYMMDD 的格式，则会按照该整型字段的字符形式输出，作为分区 ID 取值。
>
> 日期类型：如果分区键属于日期格式，或可以转换为 YYYYMMDD 格式的整型，则按照 YYYYMMDD 格式化后的字符形式输出，作为分区 ID 取值。
>
> 其他类型：如果使用其他类似 Float、String 等类型作为分区键，会通过对其插入数据的 128 位 Hash 值作为分区 ID 的取值。

MinBlockNum 和 MaxBlockNum：BlockNum 是一个整型的自增长型编号，该编号在单张 MergeTree 表中从 1 开始全局累加，当有新的分区目录创建后，该值就加 1，对新的分区目录来讲，MinBlockNum 和 MaxBlockNum 取值相同。例如上面示例数据为 202002_1_1_0  202002_1_5_1，但当分区目录进行合并后，取值规则会发生变化，MinBlockNum 取同一分区所欲目录中最新的 MinBlockNum 值。MaxBlockNum 取同一分区内所有目录中的最大值。
 Level：表示合并的层级。相当于某个分区被合并的次数，它不是以表全局累加，而是以分区为单位，初始创建的分区，初始值为 0，相同分区 ID 发生合并动作时，在相应分区内累计加 1。

## MergeTree 数据分区合并规则

随着数据的写入 MergeTree 存储引擎会很多分区目录。如果分区目录数太多怎么办？因为 Clickhouse 的 MergeTree 存储引擎是基于 LSM 实现的。MergeTree 可以通过分区合并将属于相同分区的多个目录合并为一个新的目录（官方描述在 10 到 15 分钟内会进行合并，也可直接执行 optimize 语句），已经存在的旧目录（也即 system.parts 表中 activie 为 0 的分区）在之后某个时刻通过后台任务删除（默认 8 分钟）。

### 分区合并

我们回顾之前创建的表的分区目录



```bash
# ls 
202002_1_1_0  202004_2_2_0  202009_3_3_0
202002_4_4_0  202002_5_5_0
```

手工触发分区合并



```csharp
qabb-qa-ch00 :) optimize table tab_partition;
OPTIMIZE TABLE tab_partition
Ok.
0 rows in set. Elapsed: 0.003 sec.

qabb-qa-ch00 :) select partition,name,part_type, active from system.parts where  table ='tab_partition';
┌─partition─┬─name─────────┬─part_type─┬─active─┐
│ 202002    │ 202002_1_1_0 │ Wide      │      0 │
│ 202002    │ 202002_1_5_1 │ Wide      │      1 │
│ 202002    │ 202002_4_4_0 │ Wide      │      0 │
│ 202002    │ 202002_5_5_0 │ Wide      │      0 │
│ 202004    │ 202004_2_2_0 │ Wide      │      1 │
│ 202009    │ 202009_3_3_0 │ Wide      │      1 │
└───────────┴──────────────┴───────────┴────────┘

6 rows in set. Elapsed: 0.003 sec.
```

其中 active 为 1 表示经过合并之后的最新分区，为 0 则表示旧分区，查询时会自动过滤 active=0 的分区。
 我们通过分区 202002 最新的分区目录 **202002_1_5_1** 看到合并分区新目录的命名规则如下：

> PartitionID：分区 ID 保持不变
>  MinBlockNum：取同一个分区内所有目录中最小的 MinBlockNum 值
>  MaxBlockNUm：取同一个分区内所有目录中最大的 MaxBlockNum 值
>  Level：取同一个分区内最大 Level 值并加 1

合并之后的目录结构如下：

![](https://tianxiawuhao.github.io/post-images/1639552681661.webp)