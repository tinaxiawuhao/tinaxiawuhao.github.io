---
title: '第一章 Kafka的安装与使用'
date: 2021-04-19 10:41:18
tags: [kafka]
published: true
hideInList: false
feature: /post-images/w7MMk_8-a.png
isTop: false
---
### Kafka 基础知识

![](https://tinaxiawuhao.github.io/post-images/1619146086328.png)

对于大数据，我们要考虑的问题有很多，首先海量数据如何收集（如 Flume），然后对于收集到的数据如何存储（典型的分布式文件系统 HDFS、分布式数据库 HBase、NoSQL 数据库 Redis），其次存储的数据不是存起来就没事了，要通过计算从中获取有用的信息，这就涉及到计算模型（典型的离线计算 MapReduce、流式实时计算Storm、Spark），或者要从数据中挖掘信息，还需要相应的机器学习算法。在这些之上，还有一些各种各样的查询分析数据的工具（如 Hive、Pig 等）。除此之外，要构建分布式应用还需要一些工具，比如分布式协调服务 Zookeeper 等等。

这里，我们讲到的是消息系统，Kafka 专为分布式高吞吐量系统而设计，其他消息传递系统相比，Kafka 具有更好的吞吐量，内置分区，复制和固有的容错能力，这使得它非常适合大规模消息处理应用程序。

### 消息系统

![](https://tinaxiawuhao.github.io/post-images/1619146163330.png)

​		点对点消息系统：生产者发送一条消息到queue，一个queue可以有很多消费者，但是一个消息只能被一个消费者接受，当没有消费者可用时，这个消息会被保存直到有 一个可用的消费者，所以Queue实现了一个可靠的负载均衡。

​		发布订阅消息系统：发布者发送到topic的消息，只有订阅了topic的订阅者才会收到消息。topic实现了发布和订阅，当你发布一个消息，所有订阅这个topic的服务都能得到这个消息，所以从1到N个订阅者都能得到这个消息的拷贝。

 

### kafka术语

Apache Kafka 是一个分布式发布 - 订阅消息系统和一个强大的队列，可以处理大量的数据，并使你能够将消息从一个端点传递到另一个端点。 Kafka 适合离线和在线消息消费。 Kafka 消息保留在磁盘上，并在群集内复制以防止数据丢失。 Kafka 构建在 ZooKeeper 同步服务之上。 它与 Apache Storm 和 Spark 非常好地集成，用于实时流式数据分析。

Kafka 是一个分布式消息队列，具有高性能、持久化、多副本备份、横向扩展能力。生产者往队列里写消息，消费者从队列里取消息进行业务逻辑。一般在架构设计中起到解耦、削峰、异步处理的作用。

![](https://tinaxiawuhao.github.io/post-images/1619146174125.png)

​		消息由producer产生，消息按照topic归类，并发送到broker中，broker中保存了一个或多个topic的消息，consumer通过订阅一组topic的消息，通过持续的poll操作从broker获取消息，并进行后续的消息处理。

**Producer** ：消息生产者，就是向broker发指定topic消息的客户端。

**Consumer** ：消息消费者，通过订阅一组topic的消息，从broker读取消息的客户端。

**Broker** ：一个kafka集群包含一个或多个服务器，一台kafka服务器就是一个broker，用于保存producer发送的消息。一个broker可以容纳多个topic。

**Topic** ：每条发送到broker的消息都有一个类别，可以理解为一个队列或者数据库的一张表。

**Partition**：一个topic的消息由多个partition队列存储的，一个partition队列在kafka上称为一个分区。每个partition是一个有序的队列，多个partition间则是无序的。partition中的每条消息都会被分配一个有序的id（offset）。

**Offset**：偏移量。kafka为每条在分区的消息保存一个偏移量offset，这也是消费者在分区的位置。kafka的存储文件都是按照offset.kafka来命名，位于2049位置的即为2048.kafka的文件。比如一个偏移量是5的消费者，表示已经消费了从0-4偏移量的消息，下一个要消费的消息的偏移量是5。

**Consumer Group （CG）**：若干个Consumer组成的集合。这是kafka用来实现一个topic消息的广播（发给所有的consumer）和单播（发给任意一个consumer）的手段。一个topic可以有多个CG。topic的消息会复制（不是真的复制，是概念上的）到所有的CG，但每个CG只会把消息发给该CG中的一个consumer。如果需要实现广播，只要每个consumer有一个独立的CG就可以了。要实现单播只要所有的consumer在同一个CG。用CG还可以将consumer进行自由的分组而不需要多次发送消息到不同的topic。

​		假如一个消费者组有两个消费者，订阅了一个具有4个分区的topic的消息，那么这个消费者组的每一个消费者都会消费两个分区的消息。消费者组的成员是动态维护的，如果新增或者减少了消费者组中的消费者，那么每个消费者消费的分区的消息也会动态变化。比如原来一个消费者组有两个消费者，其中一个消费者因为故障而不能继续消费消息了，那么剩下一个消费者将会消费全部4个分区的消息。

### Apache Kafka基本原理

#### 1、分布式和分区（distributed、partitioned）

我们说 kafka 是一个分布式消息系统，所谓的分布式，实际上我们已经大致了解。消息保存在 Topic 中，而为了能够实现大数据的存储，一个 topic 划分为多个分区，每个分区对应一个文件，可以分别存储到不同的机器上，以实现分布式的集群存储。另外，每个 partition 可以有一定的副本，备份到多台机器上，以提高可用性。

总结起来就是：一个 topic 对应的多个 partition 分散存储到集群中的多个 broker 上，存储方式是一个 partition 对应一个文件，每个 broker 负责存储在自己机器上的 partition 中的消息读写。

#### 2、副本（replicated ）

kafka 还可以配置 partitions 需要备份的个数(replicas),每个 partition 将会被备份到多台机器上,以提高可用性，备份的数量可以通过配置文件指定。

这种冗余备份的方式在分布式系统中是很常见的，那么既然有副本，就涉及到对同一个文件的多个备份如何进行管理和调度。kafka 采取的方案是：每个 partition 选举一个 server 作为“leader”，由 leader 负责所有对该分区的读写，其他 server 作为 follower 只需要简单的与 leader 同步，保持跟进即可。如果原来的 leader 失效，会重新选举由其他的 follower 来成为新的 leader。

至于如何选取 leader，实际上如果我们了解 ZooKeeper，就会发现其实这正是 Zookeeper 所擅长的，Kafka 使用 ZK 在 Broker 中选出一个 Controller，用于 Partition 分配和 Leader 选举。

另外，这里我们可以看到，实际上作为 leader 的 server 承担了该分区所有的读写请求，因此其压力是比较大的，从整体考虑，从多少个 partition 就意味着会有多少个leader，kafka 会将 leader 分散到不同的 broker 上，确保整体的负载均衡。

#### 3、整体数据流程

Kafka 的总体数据流满足下图，该图可以说是概括了整个 kafka 的基本原理。
![](https://tinaxiawuhao.github.io/post-images/1622083894591.jpg)

**（1）数据生产过程（Produce）**

对于生产者要写入的一条记录，可以指定四个参数：分别是 topic、partition、key 和 value，其中 topic 和 value（要写入的数据）是必须要指定的，而 key 和 partition 是可选的。

对于一条记录，先对其进行序列化，然后按照 Topic 和 Partition，放进对应的发送队列中。如果 Partition 没填，那么情况会是这样的：a、Key 有填。按照 Key 进行哈希，相同 Key 去一个 Partition。b、Key 没填。Round-Robin 来选 Partition。

![](https://tinaxiawuhao.github.io/post-images/1622083907859.png)

producer 将会和Topic下所有 partition leader 保持 socket 连接，消息由 producer 直接通过 socket 发送到 broker。其中 partition leader 的位置( host : port )注册在 zookeeper 中，producer 作为 zookeeper client，已经注册了 watch 用来监听 partition leader 的变更事件，因此，可以准确的知道谁是当前的 leader。

producer 端采用异步发送：将多条消息暂且在客户端 buffer 起来，并将他们批量的发送到 broker，小数据 IO 太多，会拖慢整体的网络延迟，批量延迟发送事实上提升了网络效率。

**（2）数据消费过程（Consume）**

对于消费者，不是以单独的形式存在的，每一个消费者属于一个 consumer group，一个 group 包含多个 consumer。特别需要注意的是：订阅 Topic 是以一个消费组来订阅的，发送到 Topic 的消息，只会被订阅此 Topic 的每个 group 中的一个 consumer 消费。

如果所有的 Consumer 都具有相同的 group，那么就像是一个点对点的消息系统；如果每个 consumer 都具有不同的 group，那么消息会广播给所有的消费者。

具体说来，这实际上是根据 partition 来分的，一个 Partition，只能被消费组里的一个消费者消费，但是可以同时被多个消费组消费，消费组里的每个消费者是关联到一个 partition 的，因此有这样的说法：对于一个 topic,同一个 group 中不能有多于 partitions 个数的 consumer 同时消费,否则将意味着某些 consumer 将无法得到消息。

同一个消费组的两个消费者不会同时消费一个 partition。

![](https://tinaxiawuhao.github.io/post-images/1622083921378.png)

在 kafka 中，采用了 pull 方式，即 consumer 在和 broker 建立连接之后，主动去 pull(或者说 fetch )消息，首先 consumer 端可以根据自己的消费能力适时的去 fetch 消息并处理，且可以控制消息消费的进度(offset)。

partition 中的消息只有一个 consumer 在消费，且不存在消息状态的控制，也没有复杂的消息确认机制，可见 kafka broker 端是相当轻量级的。当消息被 consumer 接收之后，需要保存 Offset 记录消费到哪，以前保存在 ZK 中，由于 ZK 的写性能不好，以前的解决方法都是 Consumer 每隔一分钟上报一次，在 0.10 版本后，Kafka 把这个 Offset 的保存，从 ZK 中剥离，保存在一个名叫 consumeroffsets topic 的 Topic 中，由此可见，consumer 客户端也很轻量级。

#### 4、消息传送机制

Kafka 支持 3 种消息投递语义,在业务中，常常都是使用 At least once 的模型。

- At most once：最多一次，消息可能会丢失，但不会重复。
- At least once：最少一次，消息不会丢失，可能会重复。
- Exactly once：只且一次，消息不丢失不重复，只且消费一次。

### kafka安装和使用

在Windows安装运行Kafka：https://blog.csdn.net/weixin_38004638/article/details/91893910

![](https://tinaxiawuhao.github.io/post-images/1619146234260.png)

### kafka运行

![](https://tinaxiawuhao.github.io/post-images/1619146260426.png)

![](https://tinaxiawuhao.github.io/post-images/1619146275373.png)

一次写入，支持多个应用读取，读取信息是相同的

![](https://tinaxiawuhao.github.io/post-images/1619146288083.png)

`kafka-study.pom`

```java
<dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka_2.12</artifactId>
        <version>2.2.1</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-nop</artifactId>
        <version>1.7.24</version>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.0</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```



### Producer生产者 

​		发送消息的方式，只管发送，不管结果：只调用接口发送消息到 Kafka 服务器，但不管成功写入与否。由于 Kafka 是高可用的，因此大部分情况下消息都会写入，但在异常情况下会丢消息

`同步发送`：调用 send() 方法返回一个 Future 对象，我们可以使用它的 get() 方法来判断消息发送成功与否

`异步发送`：调用 send() 时提供一个回调方法，当接收到 broker 结果后回调此方法

```java
public class MyProducer {
    private static KafkaProducer<String, String> producer;
    //初始化
    static {
        Properties properties = new Properties();
        //kafka启动，生产者建立连接broker的地址
        properties.put("bootstrap.servers", "127.0.0.1:9092");
        //kafka序列化方式
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        //自定义分区分配器
        properties.put("partitioner.class", "com.imooc.kafka.CustomPartitioner");
        producer = new KafkaProducer<>(properties);
    }

    /**
     * 创建topic：.\bin\windows\kafka-topics.bat --create --zookeeper localhost:2181
     * --replication-factor 1 --partitions 1 --topic kafka-study
     * 创建消费者：.\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092
     * --topic kafka-study --from-beginning
     */
    //发送消息，发送完后不做处理
    private static void sendMessageForgetResult() {
        ProducerRecord<String, String> record = new ProducerRecord<>("kafka-study", "name", "ForgetResult");
        producer.send(record);
        producer.close();
    }
    //发送同步消息，获取发送的消息
    private static void sendMessageSync() throws Exception {
        ProducerRecord<String, String> record = new ProducerRecord<>("kafka-study", "name", "sync");
        RecordMetadata result = producer.send(record).get();
        System.out.println(result.topic());//kafka-study
        System.out.println(result.partition());//分区为name的hash对应分区
        System.out.println(result.offset());//已发送一条消息，此时偏移量+1
        producer.close();
    }
    /**
     * 创建topic：.\bin\windows\kafka-topics.bat --create --zookeeper localhost:2181
     * --replication-factor 1 --partitions 3 --topic kafka-study-x
     * 创建消费者：.\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092
     * --topic kafka-study-x --from-beginning
     */
      //发送异步消息，获取回调的消息
    private static void sendMessageCallback() {
        ProducerRecord<String, String> record = new ProducerRecord<>("kafka-study-x", "name", "callback");
        producer.send(record, new MyProducerCallback());
        //发送多条消息
        record = new ProducerRecord<>("kafka-study-x", "name-x", "callback");
        producer.send(record, new MyProducerCallback());
        producer.close();
    }
}
```





 ```java
//发送异步消息

//场景：每条消息发送有延迟，多条消息发送，无需同步等待，可以执行其他操作，程序会自动异步调用 
private static class MyProducerCallback implements Callback {
        @Override
        public void onCompletion(RecordMetadata recordMetadata, Exception e) {
            if (e != null) {
                e.printStackTrace();
                return;
            }
            System.out.println("*** MyProducerCallback ***");
            System.out.println(recordMetadata.topic());
            System.out.println(recordMetadata.partition());
            System.out.println(recordMetadata.offset());
        }
    }
    public static void main(String[] args) throws Exception {
        //sendMessageForgetResult();
        //sendMessageSync();
        sendMessageCallback();
    }
}
 ```

自定义分区分配器：决定消息存放在哪个分区.。默认分配器使用轮询存放，轮到已满分区将会写入失败。

```java
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        //获取topic所有分区
        List<PartitionInfo> partitionInfos = cluster.partitionsForTopic(topic);
        int numPartitions = partitionInfos.size();
        //消息必须有key
        if (null == keyBytes || !(key instanceof String)) {
            throw new InvalidRecordException("kafka message must have key");
        }
        //如果只有一个分区，即0号分区
        if (numPartitions == 1) {return 0;}
        //如果key为name，发送至最后一个分区
        if (key.equals("name")) {return numPartitions - 1;}
        return Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1);
    }
    @Override
    public void close() {}
    @Override
    public void configure(Map<String, ?> map) {}
}
```



启动生产者发送消息，通过自定义分区分配器分配，查询到topic信息的offset、partitioner

| topic         | partition | offset |
| ------------- | --------- | ------ |
| kafka-study   | 0         | 1      |
| kafka-study   | 1         | 2      |
| kafka-study-x | 0         | 1      |

### Kafka消费者（组）

```java
public class MyConsumer {
    private static KafkaConsumer<String, String> consumer;
    private static Properties properties;
    //初始化
    static {
        properties = new Properties();
        //建立连接broker的地址
        properties.put("bootstrap.servers", "127.0.0.1:9092");
        //kafka反序列化
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        //指定消费者组
        properties.put("group.id", "KafkaStudy");
    }

    //自动提交位移：由consume自动管理提交
    private static void generalConsumeMessageAutoCommit() {
        //配置
        properties.put("enable.auto.commit", true);
        consumer = new KafkaConsumer<>(properties);
        //指定topic
        consumer.subscribe(Collections.singleton("kafka-study-x"));
        try {
            while (true) {
                boolean flag = true;
                //拉取信息，超时时间100ms
                ConsumerRecords<String, String> records = consumer.poll(100);
                //遍历打印消息
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println(String.format(
                            "topic = %s, partition = %s, key = %s, value = %s",
                            record.topic(), record.partition(), record.key(), record.value()
                    ));
                    //消息发送完成
                    if (record.value().equals("done")) { flag = false; }
                }
                if (!flag) { break; }
            }
        } finally {
            consumer.close();
        }
    }

    //手动同步提交当前位移，根据需求提交，但容易发送阻塞，提交失败会进行重试直到抛出异常
    private static void generalConsumeMessageSyncCommit() {
        properties.put("auto.commit.offset", false);
        consumer = new KafkaConsumer<>(properties);

        consumer.subscribe(Collections.singletonList("kafka-study-x"));
        while (true) {
            boolean flag = true;
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(String.format(
                        "topic = %s, partition = %s, key = %s, value = %s",
                        record.topic(), record.partition(), record.key(), record.value()
                ));
                if (record.value().equals("done")) { flag = false; }
            }
            try {
                //手动同步提交
                consumer.commitSync();
            } catch (CommitFailedException ex) {
                System.out.println("commit failed error: " + ex.getMessage());
            }
            if (!flag) { break; }
        }
    }

    //手动异步提交当前位移，提交速度快，但失败不会记录
    private static void generalConsumeMessageAsyncCommit() {
        properties.put("auto.commit.offset", false);
        consumer = new KafkaConsumer<>(properties);
        consumer.subscribe(Collections.singletonList("kafka-study-x"));
        while (true) {
            boolean flag = true;
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(String.format(
                        "topic = %s, partition = %s, key = %s, value = %s",
                        record.topic(), record.partition(), record.key(), record.value()
                ));
                if (record.value().equals("done")) { flag = false; }
            }
            //手动异步提交
            consumer.commitAsync();
            if (!flag) { break; }
        }
    }

    //手动异步提交当前位移带回调
    private static void generalConsumeMessageAsyncCommitWithCallback() {
        properties.put("auto.commit.offset", false);
        consumer = new KafkaConsumer<>(properties);
        consumer.subscribe(Collections.singletonList("kafka-study-x"));
        while (true) {
            boolean flag = true;
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(String.format(
                        "topic = %s, partition = %s, key = %s, value = %s",
                        record.topic(), record.partition(), record.key(), record.value()
                ));
                if (record.value().equals("done")) { flag = false; }
            }
            //使用java8函数式编程
            consumer.commitAsync((map, e) -> {
                if (e != null) {
                    System.out.println("commit failed for offsets: " + e.getMessage());
                }
            });
            if (!flag) { break; }
        }
    }

    //混合同步与异步提交位移
    @SuppressWarnings("all")
    private static void mixSyncAndAsyncCommit() {
        properties.put("auto.commit.offset", false);
        consumer = new KafkaConsumer<>(properties);
        consumer.subscribe(Collections.singletonList("kafka-study-x"));
        try {
            while (true) {
                //boolean flag = true;
                ConsumerRecords<String, String> records = consumer.poll(100);
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println(String.format(
                            "topic = %s, partition = %s, key = %s, " + "value = %s",
                            record.topic(), record.partition(),
                            record.key(), record.value()
                    ));
                    //if (record.value().equals("done")) { flag = false; }
                }
                //手动异步提交，保证性能
                consumer.commitAsync();
                //if (!flag) { break; }
            }
        } catch (Exception ex) {
            System.out.println("commit async error: " + ex.getMessage());
        } finally {
            try {
                //异步提交失败，再尝试手动同步提交
                consumer.commitSync();
            } finally {
                consumer.close();
            }
        }
    }

    public static void main(String[] args) {
        //自动提交位移
        generalConsumeMessageAutoCommit();
        //手动同步提交当前位移
        //generalConsumeMessageSyncCommit();
        //手动异步提交当前位移
        //generalConsumeMessageAsyncCommit();
        //手动异步提交当前位移带回调
        //generalConsumeMessageAsyncCommitWithCallback()
        //混合同步与异步提交位移
        //mixSyncAndAsyncCommit();
    }
}
```

先启动消费者等待接收消息，再启动生产者发送消息，进行消费消息
| topic         | partition | key    | value    |
| ------------- | --------- | ------ | -------- |
| kafka-study-x | 1         | name-x | callback |
| kafka-study-x | 2         | name-x | callback |
| kafka-study-x | 1         | name-x | callback |
| kafka-study-x | 1         | name-x | callback |
| kafka-study-x | 2         | name-x | callback |
