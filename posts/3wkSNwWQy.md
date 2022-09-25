---
title: 'ClickHouse-6sql操作'
date: 2021-11-17 19:46:57
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/3wkSNwWQy.png
isTop: false
---
### 1 Insert
基本与标准 SQL（MySQL）基本一致

**标准**

```sql
insert into [table_name] values(…),(….) 
```

**从表到表的插入**

```sql
insert into [table_name] select a,b,c from [table_name_2]
```


### 2 Update 和 Delete
ClickHouse 提供了 Delete 和 Update 的能力，这类操作被称为 Mutation 查询，它可以看做 Alter 的一种。
虽然可以实现修改和删除，但是和一般的 OLTP 数据库不一样，Mutation 语句是一种很“重”的操作，而且不支持事务。
“重”的原因主要是每次修改或者删除都会导致放弃目标数据的原有分区，重建新分区。所以尽量做批量的变更，不要进行频繁小数据的操作。

**删除操作**

```sql
alter table t_order_smt delete where sku_id ='sku_001';
```

**修改操作**

```sql
alter table t_order_smt update total_amount=toDecimal32(2000.00,2) where id =102;
```

由于操作比较“重”，所以 Mutation 语句分两步执行，同步执行的部分其实只是进行新增数据新增分区和并把旧分区打上逻辑上的失效标记。直到触发分区合并的时候，才会删除旧数据释放磁盘空间，一般不会开放这样的功能给用户，由管理员完成。

### 3 查询操作
ClickHouse 基本上与标准 SQL 差别不大

> 支持子查询
> 支持 CTE(Common Table Expression 公用表表达式 with 子句)
> 支持各种 JOIN，但是 JOIN 操作无法使用缓存, 尽量避免使用 JOIN，所以即使是两次相同的 JOIN 语句，ClickHouse 也会视为两条新 SQL
> 窗口函数
> 不支持自定义函数
> GROUP BY 操作增加了 with rollup \ with cube \ with total 用来计算小计和总计。

**插入数据**

```sql
alter table t_order_mt delete where 1=1;
insert into t_order_mt values
(101,'sku_001',1000.00,'2020-06-01 12:00:00'),
(101,'sku_002',2000.00,'2020-06-01 12:00:00'),
(103,'sku_004',2500.00,'2020-06-01 12:00:00'),
(104,'sku_002',2000.00,'2020-06-01 12:00:00'),
(105,'sku_003',600.00,'2020-06-02 12:00:00'),
(106,'sku_001',1000.00,'2020-06-04 12:00:00'),
(107,'sku_002',2000.00,'2020-06-04 12:00:00'),
(108,'sku_004',2500.00,'2020-06-04 12:00:00'),
(109,'sku_002',2000.00,'2020-06-04 12:00:00'),
(110,'sku_003',600.00,'2020-06-01 12:00:00');
```

**with rollup**：上卷，从右至左去掉维度进行小计

```sql
select id , sku_id,sum(total_amount) from t_order_mt group by id,sku_id with rollup;
```

**with cube** : 多维分析， 从右至左去掉维度进行小计，再从左至右去掉维度进行小计

```sql
select id , sku_id,sum(total_amount) from t_order_mt group by id,sku_id with cube;
```

**with totals**: 只计算合计

```sql
select id , sku_id,sum(total_amount) from t_order_mt group by id,sku_id with totals;
```

### 4 alter 操作
同 MySQL 的修改字段基本一致

**新增字段**

```sql
alter table tableName add column newcolname String after col1;
```

**修改字段类型**

```sql
alter table tableName modify column newcolname String;
```

**删除字段**

```sql
alter table tableName drop column newcolname;
```

### 5 导出数据
这个用的比较少

```
clickhouse-client --query "select * from t_order_mt where create_time='2020-06-01 12:00:00'" --format CSVWithNames> /opt/module/data/rs1.csv
```

更多支持格式参照：https://clickhouse.com/docs/en/interfaces/formats/
