---
title: 'mysql 简单导入 clickhouse'
date: 2021-12-15 15:12:02
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/BHjbgcFm-.png
isTop: false
---
数据迁移需要从 mysql 导入 clickhouse, clickhouse 自身支持的三种方式 。

## create table engin mysql

```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MySQL('host:port', 'database', 'table', 'user', 'password'[, replace_query, 'on_duplicate_clause']);
```

> 官方文档: https://clickhouse.yandex/docs/en/operations/table_engines/mysql/

注意，实际数据存储在远端 mysql 数据库中，可以理解成外表。
可以通过在 mysql 增删数据进行验证。

## insert into select from

```
-- 先建表
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = engine
-- 导入数据
INSERT INTO [db.]table [(c1, c2, c3)] select 列或者* from mysql('host:port', 'db', 'table_name', 'user', 'password')
```

可以自定义列类型，列数，使用 clickhouse 函数对数据进行处理，比如 `select toDate(xx) from mysql("host:port","db","table_name","user_name","password")`

## create table as select from

```
CREATE TABLE [IF NOT EXISTS] [db.]table_name
ENGINE =Log
AS
SELECT *
FROM mysql('host:port', 'db', 'article_clientuser_sum', 'user', 'password')
```

> 网友文章: http://jackpgao.github.io/2018/02/04/ClickHouse-Use-MySQL-Data/

不支持自定义列，参考资料里的博主写的 `ENGIN=MergeTree` 测试失败。
可以理解成 create table 和 insert into select 的组合