---
title: 'ClickHouse-7副本'
date: 2021-11-18 19:48:12
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/bps7a3f5p.png
isTop: false
---
### 1 副本写入流程

![](https://tianxiawuhao.github.io/post-images/1656505319411.png)

### 2 配置步骤
1. 启动 zookeeper 集群

2. 在 hadoop102 的/etc/clickhouse-server/config.d 目录下创建一个名为 metrika.xml 的配置文件,内容如下：
   注：也可以不创建外部文件，直接在 config.xml 中指定

   ```xml
   <?xml version="1.0"?>
   <yandex>
   	<zookeeper-servers>
   	<node index="1">
   		<host>hadoop102</host>
   		<port>2181</port>
   	</node>
   	<node index="2">
   		<host>hadoop103</host>
   		<port>2181</port>
   	</node>
   	<node index="3">
   		<host>hadoop104</host>
   		<port>2181</port>
   	</node>
   	</zookeeper-servers>
   </yandex>
   ```

3. 同步到 hadoop103 和 hadoop104 上

   ```shell
   sudo /home/atguigu/bin/xsync /etc/clickhouse-server/config.d/metrika.xml
   ```

4. 在 hadoop102 的/etc/clickhouse-server/config.xml 中增加

   ```shell
   <zookeeper incl="zookeeper-servers" optional="true" />
   <include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
   ```

   

5. 同步到 hadoop103 和 hadoop104 上

   ```shell
   sudo /home/atguigu/bin/xsync /etc/clickhouse-server/config.xml
   ```

   分别在 hadoop102 和 hadoop103 上启动 ClickHouse 服务

   **注意：因为修改了配置文件，如果以前启动了服务需要重启**

   ```shell
   sudo clickhouse restart
   ```

   注意：我们演示副本操作只需要在 hadoop102 和 hadoop103 两台服务器即可，上面的
   操作，我们 hadoop104 可以你不用同步，我们这里为了保证集群中资源的一致性，做了同
   步。

6. 在 hadoop102 和 hadoop103 上分别建表
   副本只能同步数据，不能同步表结构，所以我们需要在每台机器上自己手动建表

   ```shell
   create table t_order_rep2 (
       id UInt32,
       sku_id String,
       total_amount Decimal(16,2),
       create_time
       Datetime
   ) engine = ReplicatedMergeTree('/clickhouse/table/01/t_order_rep','rep_102')
   partition by toYYYYMMDD(create_time)
   primary key (id)
   order by (id,sku_id);
   ```
	> ReplicatedMergeTree(‘/clickhouse/table/01/t_order_rep’,‘rep_102’)中，
	>
	> 第一个参数是分片的 zk_path ， 一般按照：/clickhouse/table/{shard}/{table_name} 的格式写，如果只有一个分片就写 01 即可。
	> 第二个参数是副本名称，相同的分片副本名称不能相同。

7. 在 hadoop102 上执行 insert 语句

   ```shell
   insert into t_order_rep2 values
   (101,'sku_001',1000.00,'2020-06-01 12:00:00'),
   (102,'sku_002',2000.00,'2020-06-01 12:00:00'),
   (103,'sku_004',2500.00,'2020-06-01 12:00:00'),
   (104,'sku_002',2000.00,'2020-06-01 12:00:00'),
   (105,'sku_003',600.00,'2020-06-02 12:00:00');
   ```

8. 在 hadoop103 上执行 select，可以查询出结果，说明副本配置正确
