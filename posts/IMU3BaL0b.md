---
title: '搭建并整合Seata'
date: 2022-07-19 20:40:30
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/IMU3BaL0b.png
isTop: false
---
###  搭建并整合Seata

接下来，我们就正式在项目中整合Seata来实现分布式事务。这里，我们主要整合Seata的AT模式。

#### 搭建Seata基础环境

（1）到https://github.com/seata/seata/releases/tag/v1.4.2链接下载Seata的安装包和源码，这里，下载的是1.4.2版本，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012100579.png)

这里我下载的都是zip压缩文件。

（2）进入Nacos，选择的命名空间，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012119037.png)

点击新建命名空间，并填写Seata相关的信息，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012136201.png)

可以看到，这里我填写的信息如下所示。

- 命名空间ID：seata_namespace_001，如果不填的话Nacos会自动生成命名空间的ID。
- 命名空间名：seata。
- 描述：seata的命名空间。

**「这里，需要记录下命名空间的ID：seata_namespace_001，在后面的配置中会使用到。」**

点击确定后如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012154952.png)

可以看到，这里为Seata在Nacos中创建了命名空间。

（3）解压Seata安装文件，进入解压后的`seata/seata-server-1.4.2/conf`目录，修改`registry.conf`注册文件，修改后的部分文件内容如下所示。

```java
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = "seata_namespace_001"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "seata_namespace_001"
    group = "SEATA_GROUP"
    username = "nacos"
    password = "nacos"
    dataId = "seataServer.properties"
  }
}
```

其中，namespace的值就是在Nacos中配置的Seata的命名空间ID：seata_namespace_001。

**「注意：这里只列出了修改的部分内容，完整的registry.conf文件可以到项目的`doc/nacos/config/chapter25`目录下获取。」**

（4）修改Seata安装文件的`seata/seata-server-1.4.2/conf`目录下的file.conf文件，修改后的部分配置如下所示。

```java
store {
  mode = "db"
  publicKey = ""
  db {
    datasource = "druid"
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata?useSSL=false&useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&serverTimezone=Asia/Shanghai"
    user = "root"
    password = "root"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

**「注意：这里只列出了修改的部分内容，完整的file.conf文件可以到项目的`doc/nacos/config/chapter25`目录下获取。」**

（5）在下载的Seata源码的`seata-1.4.2/script/config-center`目录下找到config.txt文件，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012175639.png)

将其复制到Seata安装包解压的根目录下，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012194228.png)

接下来，修改Seata安装包解压的根目录下的config.txt文件，这里还是只列出修改的部分，如下所示。

```java
service.vgroupMapping.server-order-tx_group=default
service.vgroupMapping.server-product-tx_group=default
store.mode=db
store.publicKey=""
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useSSL=false&useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&serverTimezone=Asia/Shanghai
store.db.user=root
store.db.password=root
store.redis.sentinel.masterName=""
store.redis.sentinel.sentinelHosts=""
store.redis.password=""
```

**「注意：在config.txt中，部分配置的等号“=”后面为空，需要在等号“=“后面添加空字符串""。同样的，小伙伴们可以到项目的`doc/nacos/config/chapter25`目录下获取完整的config.txt文件。」**

（6）在下载的Seata源码的`seata-1.4.2/script/config-center/nacos`目录下找到nacos-config.sh文件，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012213383.png)

将nacos-config.sh文件复制到Seata安装文件解压目录的`seata/seata-server-1.4.2/scripts`目录下，其中scripts目录需要手动创建，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012230067.png)

（7）.sh文件是Linux操作系统上的脚本文件，如果想在Windows操作系统上运行.sh文件，可以在Windows操作系统上安装Git后在运行.sh文件。

接下来，在Git的Bash命令行进入Seata安装文件中nacos-config.sh文件所在的目录，执行如下命令。

```shell
sh nacos-config.sh -h 127.0.0.1 -p 8848 -g SEATA_GROUP -t seata_namespace_001 -u nacos -w nacos
```

其中，命令中的每个参数含义如下所示。

- -h：Nacos所在的IP地址。
- -p：Nacos的端口号。
- -g：分组。
- -t：命名空间的ID，这里我们填写在Nacos中创建的命名空间的ID：seata_namespace_001。如果不填，默认是public命名空间。
- -u：Nacos的用户名。
- -w：Nacos的密码。

执行命令后的结果信息如下所示。

```java
=========================================================================
 Complete initialization parameters,  total-count:89 ,  failure-count:0
=========================================================================
 Init nacos config finished, please start seata-server.
```

可以看到，整个配置执行成功。

（8）打开Nacos的配置管理-配置列表界面，切换到seata命名空间，可以看到有关Seata的配置都注册到Nacos中了，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012247957.png)

（9）在MySQL数据库中创建seata数据库，如下所示。

```sql
create database if not exists seata;
```

接下来，在seata数据库中执行Seata源码包`seata-1.4.2/script/server/db`目录下的mysql.sql脚本文件，mysql.sql脚本的内容如下所示。

```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

这里，也将mysql.sql文件放在了项目的`doc/nacos/config/chapter25`目录下。

（10）启动Seata服务，进入在命令行进入Seata安装文件的`seata/seata-server-1.4.2/bin`目录，执行如下命令。

```shell
seata-server.bat -p 8091 -h 127.0.0.1 -m db
```

可以看到，在启动Seata的命令行输出了如下信息。

```java
i.s.core.rpc.netty.NettyServerBootstrap  : Server started, listen port: 8091
```

说明Seata已经启动成功。

至此，Seata的基础环境搭建完毕。

#### 项目整合Seata

在我们开发的微服务程序中，订单微服务下单成功后会调用库存微服务扣减商品的库存信息，而用户微服务只提供了查询用户信息的接口。这里，我们在商品微服务和订单微服务中整合Seata。

#### 导入unlog表

我们使用的是Seata的AT模式，需要我们在涉及到使用Seata解决分布式事务问题的每个业务库中创建一个Seata的undo_log数据表，Seata中本身提供了创建数据表的SQL文件，这些SQL文件位于Seata源码包下的`seata-1.4.2/script/client/at/db`目录中，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012267689.png)

这里，我们使用mysql.sql脚本。mysql.sql脚本的内容如下所示。

```sql
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';
```

注意，这里要在shop数据库中执行mysql.sql脚本，同样的，我会将这里的mysql.sql文件放到项目的`doc/nacos/config/chapter25`目录下，并重命名为mysql_client.sql。

#### 商品微服务整合Seata

（1）在商品微服务shop-product的pom.xml文件中引入Seata依赖，如下所示。

```java
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

（2）修改商品微服务shop-product的bootstrap.yml，修改后的文件如下所示。

```java
spring:
  application:
    name: server-product
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: product_group
        shared-configs[0]:
          data_id: server-all.yaml
          group: all_group
          refresh: true
      discovery:
        server-addr: 127.0.0.1:8848
    alibaba:
      seata:
        tx-service-group: ${spring.application.name}-tx_group

  profiles:
    active: dev

seata:
  application-id: ${spring.application.name}
  service:
    vgroup-mapping:
      server-product-tx_group: default

  registry:
    nacos:
      server-addr: ${spring.cloud.nacos.discovery.server-addr}
      username: nacos
      password: nacos
      group: SEATA_GROUP
      namespace: seata_namespace_001
      application: seata-server

  config:
    type: nacos
    nacos:
      server-addr: ${spring.cloud.nacos.discovery.server-addr}
      username: nacos
      password: nacos
      group: SEATA_GROUP
      namespace: seata_namespace_001
```

其中，配置的Nacos的namespace与group与`registry.conf`文件中的一致。

#### 订单微服务整合Seata

（1）在订单微服务shop-product的pom.xml文件中引入Seata依赖，如下所示。

```java
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

（2）修改订单微服务shop-order的bootstrap.yml，修改后的文件如下所示。

```java
spring:
  application:
    name: server-order
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: order_group
        shared-configs[0]:
          data_id: server-all.yaml
          group: all_group
          refresh: true
      discovery:
        server-addr: 127.0.0.1:8848
    alibaba:
      seata:
        tx-service-group: ${spring.application.name}-tx_group

  profiles:
    active: dev

seata:
  application-id: ${spring.application.name}
  service:
    vgroup-mapping:
      server-order-tx_group: default

  registry:
    nacos:
      server-addr: ${spring.cloud.nacos.discovery.server-addr}
      username: nacos
      password: nacos
      group: SEATA_GROUP
      namespace: seata_namespace_001
      application: seata-server

  config:
    type: nacos
    nacos:
      server-addr: ${spring.cloud.nacos.discovery.server-addr}
      username: nacos
      password: nacos
      group: SEATA_GROUP
      namespace: seata_namespace_001
```

（3）修改订单微服务的`io.binghe.shop.order.service.impl.OrderServiceV8Impl`类的saveOrder()方法，在saveOrder()方法上添加Seata的@GlobalTransactional注解，如下所示。

```java
@Override
@GlobalTransactional
public void saveOrder(OrderParams orderParams) {
 //省略具体方法代码
}
```

至此，搭建并整合Seata完毕，就是这么简单。

### 验证Seata事务

#### 重置数据库数据

这里，首先将商品数据表t_product中id为1001的数据的库存信息重置为100，如下所示。

```sql
update t_product set t_pro_stock = 100 where id = 1001;
```

#### 查询数据表数据

（1）打开cmd终端，进入MySQL命令行，并进入shop商城数据库，如下所示。

```shell
C:\Users\binghe>mysql -uroot -p
Enter password: ****
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 5.7.35 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use shop;
Database changed
```

（2）查看商品数据表，如下所示。

```shell
mysql> select * from t_product;
+------+------------+-------------+-------------+
| id   | t_pro_name | t_pro_price | t_pro_stock |
+------+------------+-------------+-------------+
| 1001 | 华为       |     2399.00 |         100 |
| 1002 | 小米       |     1999.00 |         100 |
| 1003 | iphone     |     4999.00 |         100 |
+------+------------+-------------+-------------+
3 rows in set (0.00 sec)
```

这里，我们以id为1001的商品为例，此时发现商品的库存为100。

（3）查询订单数据表，如下所示。

```shell
mysql> select * from t_order;
Empty set (0.00 sec)
```

可以发现订单数据表为空。

（4）查询订单条目数据表，如下所示。

```shell
mysql> select * from t_order_item;
Empty set (0.00 sec)
```

可以看到，订单条目数据表为空。

#### 验证Seata事务

（1）分别启动Nacos、Sentinel、ZinKin、RocketMQ，Seata，并启动用户微服务，商品微服务，订单微服务和服务网关。打开浏览器访问`http://localhost:10001/server-order/order/submit_order?userId=1001&productId=1001&count=1`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1659012289306.png)

返回的原始数据如下所示。

```java
{"code":500,"codeMsg":"执行失败","data":"/ by zero"}
```

（2）查看各个微服务和网关输出的日志信息，分别如下所示。

- 用户微服务输出的日志如下所示。

```java
获取到的用户信息为：{"address":"北京","id":1001,"password":"c26be8aaf53b15054896983b43eb6a65","phone":"13212345678","username":"binghe"}
```

说明用户微服务无异常信息。

- 商品微服务输出的日志如下所示。

```java
获取到的商品信息为：{"id":1001,"proName":"华为","proPrice":2399.00,"proStock":100}
更新商品库存传递的参数为: 商品id:1001, 购买数量:1 
```

说明商品微服务无异常信息。

值得注意的是，整合Seata后，商品微服务同时输出了如下日志。

```shell
rm handle branch rollback process:xid=192.168.0.111:8091:6638572304823066625,branchId=6638572304823066634,branchType=AT,resourceId=jdbc:mysql://localhost:3306/shop,applicationData=null
Branch Rollbacking: 192.168.0.111:8091:6638572304823066625 6638572304823066634 jdbc:mysql://localhost:3306/shop
xid 192.168.0.111:8091:6638572304823066625 branch 6638572304823066634, undo_log deleted with GlobalFinished
Branch Rollbacked result: PhaseTwo_Rollbacked
```

看上去应该是有事务回滚了。

- 订单微服务输出的日志如下所示。

```java
提交订单时传递的参数:{"count":1,"empty":false,"productId":1001,"userId":1001}
库存扣减成功
服务器抛出了异常：{}
java.lang.ArithmeticException: / by zero
```

说明订单微服务抛出了ArithmeticException异常。

同时，商品微服务会输出如下日志。

```java
Branch Rollbacked result: PhaseTwo_Rollbacked
[192.168.0.111:8091:6638572304823066625] rollback status: Rollbacked
```

看上去应该是有事务回滚了。

- 网关服务输出的日志如下所示。

```java
执行前置过滤器逻辑
执行后置过滤器逻辑
访问接口主机: localhost
访问接口端口: 10001
访问接口URL: /server-order/order/submit_order
访问接口URL参数: userId=1001&productId=1001&count=1
访问接口时长: 1632ms
```

可以看到，网关服务无异常信息。

通过微服务打印出的日志信息，可以看到，有事务回滚了。

#### 查询数据表数据

（1）打开cmd终端，进入MySQL命令行，并进入shop商城数据库，如下所示。

```shell
C:\Users\binghe>mysql -uroot -p
Enter password: ****
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 5.7.35 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use shop;
Database changed
```

（2）查看商品数据表，如下所示。

```shell
mysql> select * from t_product;
+------+------------+-------------+-------------+
| id   | t_pro_name | t_pro_price | t_pro_stock |
+------+------------+-------------+-------------+
| 1001 | 华为       |     2399.00 |         100 |
| 1002 | 小米       |     1999.00 |         100 |
| 1003 | iphone     |     4999.00 |         100 |
+------+------------+-------------+-------------+
3 rows in set (0.00 sec)
```

可以看到，此时商品数据表中，id为1001的商品库存数量仍然为100。

（3）查看订单数据表，如下所示。

```shell
mysql> select * from t_order;
Empty set (0.00 sec)
```

可以看到，订单数据表为空。

（4）查看订单条目数据表，如下所示。

```shell
mysql> select * from t_order_item;
Empty set (0.00 sec)
```

可以看到，订单条目数据表为空。

至此，我们成功在项目中整合了Seata解决了分布式事务的问题