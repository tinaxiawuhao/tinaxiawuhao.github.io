---
title: 'Clickhouse-1安装'
date: 2021-11-12 09:50:16
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/r2VmiGgor.png
isTop: false
---
# 1.ClickHouse的安装

## 1.1    准备工作

下载RPM包:

[https://repo.yandex.ru/clickhouse/rpm/stable/x86_64/](https://repo.yandex.ru/clickhouse/rpm/stable/x86_64/)

下载完毕如下:

![](https://tianxiawuhao.github.io/post-images/1638237245375.png)

​                               

 

### 1.1.1 确定防火墙处于关闭状态

相关命令:

```shell
sudo systemctl status firewalld

sudo systemctl start firewalld

sudo systemctl stop firewalld

sudo systemctl restart firewalld
```

 

### 1.1.2 CentOS取消打开文件数限制

 

Ø 在 /etc/security/limits.conf文件的末尾加入以下内容

```shell
vim /etc/security/limits.conf

* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```



Ø 在/etc/security/limits.d/20-nproc.conf文件的末尾加入以下内容

```shell
vim /etc/security/limits.d/20-nproc.conf

* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```



Ø 执行同步操作(同步到集群其他机器)

```shell
xsync /etc/security/limits.conf
xsync /etc/security/limits.d/20-nproc.conf
```



### 1.1.3 安装依赖

```shell
sudo yum install -y libtool
```

![](https://tianxiawuhao.github.io/post-images/1638237268510.png)

```shell
sudo yum install -y *unixODBC*
```

![](https://tianxiawuhao.github.io/post-images/1638237282605.png)

 

同样在集群其他机器上执行以上操作

### 1.1.4 CentOS取消SELINUX

SELINUX(美国开源的linux安全增强功能)

Ø 修改/etc/selinux/config中的SELINUX=disabled

```shell
sudo vim /etc/selinux/config

SELINUX=disabled
```

Ø 执行同步操作

```shell
sudo /home/atguigu/bin/xsync /etc/selinux/config
```

Ø 重启三台服务器

 

### 1.1.5 将安装文件同步到其他两个服务器

### 1.1.6 分别在三台机子上安装这4个rpm文件

```shell
sudo rpm -ivh *.rpm(提前把四个*.rpm放在一个目录下)
```

![](https://tianxiawuhao.github.io/post-images/1638237307245.png)

```shell
sudo rpm -qa|grep clickhouse查看安装情况
```

### 1.1.7 修改配置文件

```shell
vim /etc/clickhouse-server/config.xml
```

把 **<listen_host>::</listen_host>** 的注释打开，这样的话才能让ClickHouse被除本机以外的服务器访问

```shell
#分发配置文件

xsync /etc/clickhouse-server/config.xml
```



### 1.1.8 启动Server

```shell
sudo systemctl start clickhouse-server
```

 

### 1.1.9 三台机器上关闭开机自启

```shell
systemctl disable clickhouse-server
```

 

### 1.1.10    使用client连接server

```shell
clickhouse-client -m
```



Clickhouse常用端口号:

![](https://tianxiawuhao.github.io/post-images/1638237372637.png)
![](https://tianxiawuhao.github.io/post-images/1638237379416.png)
![](https://tianxiawuhao.github.io/post-images/1638237384947.png)
![](https://tianxiawuhao.github.io/post-images/1638237390491.png)

# 第2章  副本

副本的目的主要是保障数据的高可用性，即使一台ClickHouse节点宕机，那么也可以从其他服务器获得相同的数据。

## 2.1    副本写入流程

![](https://tianxiawuhao.github.io/post-images/1638237403217.png)

## 2.2    配置步骤

Ø 启动zookeeper集群

Ø 注意:服务器的hostname需要改成对应的ck101、ck102、ck103

![](https://tianxiawuhao.github.io/post-images/1638237417573.png)

Ø 在/etc/clickhouse-server/config.d目录下创建一个名为metrika.xml的配置文件,内容如下：

```xml
<?xml version="1.0"?>
<yandex>
	<zookeeper-servers>
		<node index="1">
			<host>ck101</host>
			<port>2181</port>
		</node>
		<node index="2">
            <host>ck102</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host>ck103</host>
            <port>2181</port>
        </node>
	</zookeeper-servers>
</yandex>
```



Ø 同步到另外两个机器上

```shell
xsync /etc/clickhouse-server/config.d/metrika.xml
```



Ø 在 /etc/clickhouse-server/config.xml中增加

```xml
<zookeeper incl="zookeeper-servers" optional="true" />
<include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
```



![](https://tianxiawuhao.github.io/post-images/1638237432131.png)

Ø 同步到另外两个机器上

```shell
xsync /etc/clickhouse-server/config.xml
```

Ø 分别在另外两个机器上启动ClickHouse服务

注意：因为修改了配置文件，如果以前启动了服务需要重启

```shell
systemctl start clickhouse-server
```



Ø 在另外两个机器上分别建表

**副本只能同步数据，不能同步表结构，所以我们需要在每台机器上自己手动建表**

Ck101

```sql
create table t_order_rep (
  id UInt32,
  sku_id String,
  total_amount Decimal(16,2),
  create_time Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/table/01/t_order_rep','rep_102')
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
```

Ck102

```sql
create table t_order_rep (
  id UInt32,
  sku_id String,
  total_amount Decimal(16,2),
  create_time Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/table/01/t_order_rep','rep_103')
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
```

参数解释

ReplicatedMergeTree 中，

**第一个参数是**分片的zk_path一般按照： /clickhouse/table/{shard}/{table_name} 的格式写，如果只有一个分片就写01即可。

**第二个参数是**副本名称，相同的分片副本名称不能相同。

Ø 在ck101上执行insert语句

```sql
insert into t_order_rep values
(101,'sku_001',1000.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 12:00:00'),
(103,'sku_004',2500.00,'2020-06-01 12:00:00'),
(104,'sku_002',2000.00,'2020-06-01 12:00:00'),
(105,'sku_003',600.00,'2020-06-02 12:00:00');
```



Ø 在**ck102**上执行select，可以查询出结果，说明副本配置正确

 

# 第3章  分片集群

副本虽然能够提高数据的可用性，降低丢失风险，但是每台服务器实际上必须容纳全量数据，对数据的横向扩容没有解决。

要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切分，不同的分片分布到不同的节点上，再通过Distributed表引擎把数据拼接起来一同使用。

Distributed表引擎本身不存储数据，有点类似于MyCat之于MySql，成为一种中间件，通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据。

注意：ClickHouse的集群是表级别的，实际企业中，大部分做了高可用，但是没有用分片，避免降低查询性能以及操作集群的复杂性。

## 3.1    集群写入流程（3分片2副本共6个节点）

![](https://tianxiawuhao.github.io/post-images/1638237448353.png)

## 3.2    集群读取流程（3分片2副本共6个节点） 

![](https://tianxiawuhao.github.io/post-images/1638237458923.png)

## 3.3    配置三节点版本集群及副本

![](https://tianxiawuhao.github.io/post-images/1638237469738.png)

### 3.3.1 集群及副本规划（2个分片，只有第一个分片有副本）

 

| **ck101**                                                    | **ck102**                                                    | **ck103**                                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| <macros>  <shard>01</shard>    <replica>rep_1_1</replica>  </macros> | <macros>  <shard>01</shard>    <replica>rep_1_2</replica>  </macros> | <macros>  <shard>02</shard>    <replica>rep_2_1</replica>  </macros> |

### 3.3.2 配置步骤

#### (1) 在ck101的/etc/clickhouse-server/config.d目录下创建metrika-shard.xml文件

```xml
<?xml version="1.0"?>
<yandex>
    <remote_servers>
        <my_cluster> <!-- 集群名称--> 
            <shard>     <!--集群的第一个分片-->
                <internal_replication>true</internal_replication>
                <replica>  <!--该分片的第一个副本-->
                <host>ck101</host>
                <port>9000</port>
                </replica>
                <replica>  <!--该分片的第二个副本-->
                <host>ck102</host>
                <port>9000</port>
                </replica>
            </shard>

            <shard> <!--集群的第二个分片-->
                <internal_replication>true</internal_replication>
                <replica>  <!--该分片的第一个副本-->
                <host>ck103</host>
                <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>

    <zookeeper-servers>
        <node index="1">
            <host> ck101</host>
            <port>2181</port>
        </node>
        <node index="2">
            <host> ck102</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host> ck103</host>
            <port>2181</port>
        </node>
    </zookeeper-servers>

    <macros>
        <shard>01</shard>  <!--不同机器放的分片数不一样-->
        <replica>rep_1_1</replica> <!--不同机器放的副本数不一样-->
    </macros>
</yandex>
```



#### (2) 将ck101的metrika-shard.xml同步到102和103

```shell
xsync /etc/clickhouse-server/config.d/metrika-shard.xml
```



#### (3) 修改102和103中metrika-shard.xml宏的配置

Ø **102**

```shell
vim /etc/clickhouse-server/config.d/metrika-shard.xml
```

![](https://tianxiawuhao.github.io/post-images/1638237484666.png)

Ø **103**

```shell
vim /etc/clickhouse-server/config.d/metrika-shard.xml
```

![](https://tianxiawuhao.github.io/post-images/1638237492998.png)

#### (4) 在hadoop102上修改/etc/clickhouse-server/config.xml

![](https://tianxiawuhao.github.io/post-images/1638237502544.png)

#### (5) 同步/etc/clickhouse-server/config.xml到103和104

```shell
sudo /home/atguigu/bin/xsync /etc/clickhouse-server/config.xml
```

#### (6) 重启三台服务器上的ClickHouse服务

```shell
systemctl stop clickhouse-server
systemctl start clickhouse-server
systemctl status clickhouse-server
```

#### (7) 在ck101上执行建表语句

Ø 会自动同步到ck102和ck103上

Ø 集群名字要和配置文件中的一致

Ø 分片和副本名称从配置文件的宏定义中获取

```sql
create table st_order_mt on cluster my_cluster (
  id UInt32,
  sku_id String,
  total_amount Decimal(16,2),
  create_time Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/tables/{shard}/st_order_mt','{replica}')
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
```

可以到ck102和ck103上查看表是否创建成功  

#### (8) 在ck101上创建Distribute 分布式表

```sql
create table st_order_mt_all on cluster my_cluster
(
  id UInt32,
  sku_id String,
  total_amount Decimal(16,2),
  create_time Datetime
)engine = Distributed(my_cluster,default, st_order_mt,hiveHash(sku_id));
```



**参数含义**

​    Distributed(集群名称，库名，本地表名，分片键)

分片键必须是整型数字，所以用hiveHash函数转换，也可以rand()

#### (9) 在ck101上插入测试数据

```sql
insert into st_order_mt_all values
(201,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(202,'sku_002',2000.00,'2020-06-01 12:00:00'),
(203,'sku_004',2500.00,'2020-06-01 12:00:00'),
(204,'sku_002',2000.00,'2020-06-01 12:00:00'),
(205,'sku_003',600.00,'2020-06-02 12:00:00');
```

#### (10)   通过查询分布式表和本地表观察输出结果

Ø **分布式表**

```sql
SELECT *  FROM st_order_mt_all;
```

Ø **本地表**

```sql
select * from st_order_mt;
```

Ø 观察数据的分布

| st_order_mt_all      |![](https://tianxiawuhao.github.io/post-images/1638237523851.png) |
| -------------------- | ------------------------------------------------------------ |
| Ck101:  st_order_mt |![](https://tianxiawuhao.github.io/post-images/1638237534695.png)|
| Ck102:  st_order_mt | ![](https://tianxiawuhao.github.io/post-images/1638237544313.png) |
| Ck103:  st_order_mt | ![](https://tianxiawuhao.github.io/post-images/1638237554783.png) |

 

# 第4章  版本信息

Clickhouse: clickhouse-21.9.4.35

Zookeeper: zookeeper-3.4.6
