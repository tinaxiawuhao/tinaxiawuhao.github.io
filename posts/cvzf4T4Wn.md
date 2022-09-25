---
title: 'kafka常用命令'
date: 2021-10-29 15:15:38
tags: [kafka]
published: true
hideInList: false
feature: /post-images/cvzf4T4Wn.png
isTop: false
---

### 管理

```shell
## 创建topic（4个分区，2个副本）
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 2 --partitions 4 --topic test

### kafka版本 >= 2.2
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test

## 分区扩容
### kafka版本 < 2.2
bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic topic1 --partitions 2

### kafka版本 >= 2.2
bin/kafka-topics.sh --bootstrap-server broker_host:port --alter --topic topic1 --partitions 2

## 删除topic
bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic test
```

### 查询

```shell
## 查询集群描述
bin/kafka-topics.sh --describe --zookeeper 127.0.0.1:2181

## 查询集群描述（新）
bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic foo --describe

## topic列表查询
bin/kafka-topics.sh --zookeeper 127.0.0.1:2181 --list

## topic列表查询（支持0.9版本+）
bin/kafka-topics.sh --list --bootstrap-server localhost:9092

## 消费者列表查询（存储在zk中的）
bin/kafka-consumer-groups.sh --zookeeper localhost:2181 --list

## 消费者列表查询（支持0.9版本+）
bin/kafka-consumer-groups.sh --new-consumer --bootstrap-server localhost:9092 --list

## 消费者列表查询（支持0.10版本+）
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

## 显示某个消费组的消费详情（仅支持offset存储在zookeeper上的）
bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --group test

## 显示某个消费组的消费详情（0.9版本 - 0.10.1.0 之前）
bin/kafka-consumer-groups.sh --new-consumer --bootstrap-server localhost:9092 --describe --group test-consumer-group

## 显示某个消费组的消费详情（0.10.1.0版本+）
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group
```

### 发送和消费

```shell
## 生产者
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

## 消费者（已失效）
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test

## 生产者（支持0.9版本+）
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test --producer.config config/producer.properties

## 消费者（支持0.9版本+，已失效）
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --new-consumer --from-beginning --consumer.config config/consumer.properties

## 消费者（最新）
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning --consumer.config config/consumer.properties


## kafka-verifiable-consumer.sh（消费者事件，例如：offset提交等）
bin/kafka-verifiable-consumer.sh --broker-list localhost:9092 --topic test --group-id groupName

## 高级点的用法
bin/kafka-simple-consumer-shell.sh --brist localhost:9092 --topic test --partition 0 --offset 1234  --max-messages 10
```

### 切换leader

```shell
## kafka版本 <= 2.4
bin/kafka-preferred-replica-election.sh --zookeeper zk_host:port/chroot

## kafka新版本
bin/kafka-preferred-replica-election.sh --bootstrap-server broker_host:port
```

### kafka自带压测命令

```shell
bin/kafka-producer-perf-test.sh --topic test --num-records 100 --record-size 1 --throughput 100  --producer-props bootstrap.servers=localhost:9092
```

### kafka持续发送消息

持续发送消息到指定的topic中，且每条发送的消息都会有响应信息：

```shell
kafka-verifiable-producer.sh --broker-list $(hostname -i):9092 --topic test --max-messages 100000
```

### zookeeper-shell.sh

如果kafka集群的zk配置了chroot路径，那么需要加上`/path`。

```shell
bin/zookeeper-shell.sh localhost:2181[/path]
ls /brokers/ids
get /brokers/ids/0
```

### 迁移分区

1. 创建规则json
   ```shell
   cat > increase-replication-factor.json <<EOF
   {"version":1, "partitions":[
   {"topic":"__consumer_offsets","partition":0,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":1,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":2,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":3,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":4,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":5,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":6,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":7,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":8,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":9,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":10,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":11,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":12,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":13,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":14,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":15,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":16,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":17,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":18,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":19,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":20,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":21,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":22,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":23,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":24,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":25,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":26,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":27,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":28,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":29,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":30,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":31,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":32,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":33,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":34,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":35,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":36,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":37,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":38,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":39,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":40,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":41,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":42,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":43,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":44,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":45,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":46,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":47,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":48,"replicas":[0,1]},
   {"topic":"__consumer_offsets","partition":49,"replicas":[0,1]}]
   }
   EOF
   ```
2. 执行
   ```shell
   bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file increase-replication-factor.json --execute
   ```
3. 验证

   ```shell
   bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file increase-replication-factor.json --verify
   ```

### 删除消费者组

查询消费者组列表：

```shell
kafka-consumer-groups.sh --bootstrap-server 172.31.1.245:9092 --list
```

查询消费者组明细：

```shell
kafka-consumer-groups.sh --bootstrap-server {Kafka instance connection address} --describe --group {consumer group name}
```

删除消费者组：

```shell
kafka-consumer-groups.sh --bootstrap-server {Kafka instance connection address} --delete --group {consumer group name}
```