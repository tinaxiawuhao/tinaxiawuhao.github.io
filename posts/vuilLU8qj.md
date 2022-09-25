---
title: 'ClickHouse-8分片集群'
date: 2021-11-19 19:49:19
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/vuilLU8qj.png
isTop: false
---
### ClickHouse-分片集群

副本虽然能够提高数据的可用性，降低丢失风险，但是每台服务器实际上必须容纳全量数据，对数据的横向扩容没有解决。 要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切分，不同的分片分布到不同的节点上，再通过 Distributed 表引擎把数据拼接起来一同使用。

 Distributed 表引擎本身不存储数据，有点类似于 MyCat 之于 MySql，成为一种中间件，通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据。 

注意：ClickHouse 的集群是表级别的，实际企业中，大部分做了高可用，但是没有用分片，避免降低查询性能以及操作集群的复杂性。  

#### 1.集群写入流程（3 分片 2 副本共 6 个节点）

![](https://tianxiawuhao.github.io/post-images/1656505247965.png)

internal_replication:内部副本同步 true：由分片自己同步 false：由distribute表同步，压力大  

#### 2.集群读取流程（3 分片 2 副本共 6 个节点）

![](https://tianxiawuhao.github.io/post-images/1656505265680.png)

 

#### 3 分片 2 副本共 6 个节点集群配置（供参考）

配置的位置还是在之前的/etc/clickhouse-server/config.d/metrika.xml，内容如下 注：也可以不创建外部文件，直接在 config.xml 的<remote_servers>中指定

```xml
<yandex>
    <remote_servers>
        <fz_cluster> <!-- 集群名称-->
            <shard> <!--集群的第一个分片-->
                <internal_replication>true</internal_replication>
                <!--该分片的第一个副本-->
                <replica>
                    <host>node01</host>
                    <port>9000</port>
                </replica>
                <!--该分片的第二个副本-->
                <replica>
                    <host>node02</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard> <!--集群的第二个分片-->
                <internal_replication>true</internal_replication>
                <replica> <!--该分片的第一个副本-->
                    <host>node03</host>
                    <port>9000</port>
                </replica>
                <replica> <!--该分片的第二个副本-->
                    <host>node04</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard> <!--集群的第三个分片-->
                <internal_replication>true</internal_replication>
                <replica> <!--该分片的第一个副本-->
                    <host>node05</host>
                    <port>9000</port>
                </replica>
                <replica> <!--该分片的第二个副本-->
                    <host>node06</host>
                    <port>9000</port>
                </replica>
            </shard>
        </fz_cluster>
    </remote_servers>
</yandex>
```

 

#### 4 .配置三节点版本集群及副本

##### 4.1 集群及副本规划（2 个分片，只有第一个分片有副本）

![](https://tianxiawuhao.github.io/post-images/1656505279732.png)

 

##### 4.2 配置步骤

1）在 Node01 的/etc/clickhouse-server/config.d 目录下创建 metrika-shard.xml 文件 vim /etc/clickhouse-server/config.d/metrika-shard.xml 注：也可以不创建外部文件，直接在 config.xml 的<remote_servers>中指定  

```xml
<?xml version="1.0"?> 
<yandex>     
    <remote_servers>         
        <gmall_cluster> <!-- 集群名称-->             
            <shard> <!--集群的第一个分片-->                 
                <internal_replication>true</internal_replication>                 
                <replica> <!--该分片的第一个副本-->                     
                    <host>Node01</host>                     
                    <port>9000</port>                     
                    <user>default</user>                     
                    <password>1234qwer</password>                 
                </replica>                 
                <replica> <!--该分片的第二个副本-->                     
                    <host>Node02</host>                     
                    <port>9000</port>                     
                    <user>default</user>                     
                    <password>1234qwer</password>                 
                </replica>             
            </shard>             
            <shard> <!--集群的第二个分片-->                 
                <internal_replication>true</internal_replication>                 
                <replica> <!--该分片的第一个副本-->                    
                    <host>Node03</host>                     
                    <port>9000</port>                     
                    <user>default</user>                     
                    <password>1234qwer</password>                 
                </replica>             
            </shard>         
        </gmall_cluster>     
    </remote_servers>          
    <zookeeper-servers>         
        <node index="1">             
            <host>Node01</host>             
            <port>2181</port>         
        </node>         
        <node index="2">             
            <host>Node02</host>             
            <port>2181</port>         
        </node>         
        <node index="3">             
            <host>Node03</host>             
            <port>2181</port>         
        </node>     
    </zookeeper-servers>     
    <macros>         
        <shard>01</shard> <!--不同机器放的分片数不一样-->         
        <replica>rep_1_1</replica> <!--不同机器放的副本数不一样-->     
    </macros> 
</yandex>
```

2）将 Node01 的 metrika-shard.xml 同步到 Node02 和 Node03

```shell
scp /etc/clickhouse-server/config.d/metrika-shard.xml root@Node02:/etc/clickhouse-server/config.d/
scp /etc/clickhouse-server/config.d/metrika-shard.xml root@Node03:/etc/clickhouse-server/config.d/
```

3）修改 Node02 和 Node03 中 metrika-shard.xml 宏的配置 

（1）Node02   

```xml
<macros>
        <shard>01</shard> <!--不同机器放的分片数不一样-->
        <replica>rep_1_2</replica> <!--不同机器放的副本数不一样-->
</macros>1.2.3.4.
```

（2）Node03

```xml
<macros>
        <shard>02</shard> <!--不同机器放的分片数不一样-->
        <replica>rep_2_1</replica> <!--不同机器放的副本数不一样-->
</macros>1.2.3.4.
```

4）在 Node01 上修改/etc/clickhouse-server/config.xml

```shell
vim /etc/clickhouse-server/config.xml 
 
<zookeeper incl="zookeeper-servers" optional="true" />
<include_from>/etc/clickhouse-server/config.d/metrika-shard.xml</include_from>
```

 5）同步/etc/clickhouse-server/config.xml 到 Node02 和 Node03

```shell
scp /etc/clickhouse-server/config.xml root@Node02:/etc/clickhouse-server/
scp /etc/clickhouse-server/config.xml root@Node03:/etc/clickhouse-server/
```

 6）重启三台服务器上的 ClickHouse 服务 sudo clickhouse restart  查看集群

```
superset-BI :) show clusters;
SHOW CLUSTERS
Query id: 391735d2-bf74-43f5-aa86-b6d203c357cd
┌─cluster─────────────────────────────────────────┐
│ gmall_cluster                                   │
│ test_cluster_one_shard_three_replicas_localhost │
│ test_cluster_two_shards                         │
│ test_cluster_two_shards_internal_replication    │
│ test_cluster_two_shards_localhost               │
│ test_shard_localhost                            │
│ test_shard_localhost_secure                     │
│ test_unavailable_shard                          │
└─────────────────────────────────────────────────┘
8 rows in set. Elapsed: 0.002 sec.
```

 7）在 Node01 上执行建表语句 ➢ 会自动同步到 Node02 和 Node03 上 ➢ 集群名字要和配置文件中的一致 ➢ 分片和副本名称从配置文件的宏定义中获取  

```sql
create table st_fz_order_mt_01 on cluster gmall_cluster ( 
    id UInt32, 
    sku_id String, 
    total_amount Decimal(16,2), 
    create_time Datetime 
) engine =ReplicatedMergeTree('/clickhouse/tables/{shard}/st_fz_order_mt_01','{replica}') 
partition by toYYYYMMDD(create_time) 
primary key (id) 
order by (id,sku_id);  
```

```sql
create table st_fz_order_mt_01 on cluster gmall_cluster (  
	id UInt32,        
    sku_id String,         
    total_amount Decimal(16,2),         
    create_time Datetime         
) engine =ReplicatedMergeTree('/clickhouse/tables/{shard}/st_fz_order_mt_01','{replica}')         
partition by toYYYYMMDD(create_time)         
primary key (id)         
order by (id,sku_id); 
```

```sql
CREATE TABLE st_fz_order_mt_01 ON CLUSTER gmall_cluster (  
	`id` UInt32,   
    `sku_id` String,   
    `total_amount` Decimal(16, 2),   
    `create_time` Datetime 
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/st_fz_order_mt_01', '{replica}') 
PARTITION BY toYYYYMMDD(create_time) 
PRIMARY KEY id 
ORDER BY (id, sku_id) 
```

在Node02和Node03上查看表是否创建成功 show tables;  

8）在 Node02 上创建 Distribute 分布式表

```sql
create table st_fz_order_mt_all2 on cluster gmall_cluster (      
    id UInt32,       
    sku_id String,       
    total_amount Decimal(16,2),       
    create_time Datetime       
)engine = Distributed(gmall_cluster,default, st_fz_order_mt_01,hiveHash(sku_id)); 
CREATE TABLE st_fz_order_mt_all2 ON CLUSTER gmall_cluster (   
    `id` UInt32,   
    `sku_id` String,   
    `total_amount` Decimal(16, 2),   
    `create_time` Datetime 
) ENGINE = Distributed(gmall_cluster, default, st_fz_order_mt_01, hiveHash(sku_id)) 
```

参数含义： Distributed（集群名称，库名，本地表名，分片键） 分片键必须是整型数字，所以用 hiveHash 函数转换，也可以 rand()  

9）在 Node01 上插入测试数据

```sql
insert into st_order_mt_all2 values
(201,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(202,'sku_002',2000.00,'2020-06-01 12:00:00'),
(203,'sku_004',2500.00,'2020-06-01 12:00:00'),
(204,'sku_002',2000.00,'2020-06-01 12:00:00'),
(205,'sku_003',600.00,'2020-06-02 12:00:00');1.2.3.4.5.6.
```

10）通过查询分布式表和本地表观察输出结果 （1）分布式表

```sql
select * From st_fz_order_mt_all2;

Query id: d8b676e9-c119-4483-8ca2-f0b5cd150a61
┌──id─┬─sku_id──┬─total_amount─┬─────────create_time─┐
│ 202 │ sku_002 │         2000 │ 2020-06-01 12:00:00 │
│ 203 │ sku_004 │         2500 │ 2020-06-01 12:00:00 │
│ 204 │ sku_002 │         2000 │ 2020-06-01 12:00:00 │
└─────┴─────────┴──────────────┴─────────────────────┘
┌──id─┬─sku_id──┬─total_amount─┬─────────create_time─┐
│ 205 │ sku_003 │          600 │ 2020-06-02 12:00:00 │
└─────┴─────────┴──────────────┴─────────────────────┘
┌──id─┬─sku_id──┬─total_amount─┬─────────create_time─┐
│ 201 │ sku_001 │         1000 │ 2020-06-01 12:00:00 │
└─────┴─────────┴──────────────┴─────────────────────┘
```

 （2）本地表

```sql
Node1:
SELECT *
FROM st_fz_order_mt_01
Query id: ddcb5176-e443-4253-9877-57fec8f57311
┌──id─┬─sku_id──┬─total_amount─┬─────────create_time─┐
│ 202 │ sku_002 │         2000 │ 2020-06-01 12:00:00 │
│ 203 │ sku_004 │         2500 │ 2020-06-01 12:00:00 │
│ 204 │ sku_002 │         2000 │ 2020-06-01 12:00:00 │
└─────┴─────────┴──────────────┴─────────────────────┘
3 rows in set. Elapsed: 0.002 sec. 


Node3:
SELECT *
FROM st_fz_order_mt_01
Query id: 7a336004-7040-4098-948e-1e7c5d983edb
┌──id─┬─sku_id──┬─total_amount─┬─────────create_time─┐
│ 205 │ sku_003 │          600 │ 2020-06-02 12:00:00 │
└─────┴─────────┴──────────────┴─────────────────────┘
┌──id─┬─sku_id──┬─total_amount─┬─────────create_time─┐
│ 201 │ sku_001 │         1000 │ 2020-06-01 12:00:00 │
└─────┴─────────┴──────────────┴─────────────────────┘
2 rows in set. Elapsed: 0.002 sec.
```

可以看到数据分布在Node1和Node3两个节点上。