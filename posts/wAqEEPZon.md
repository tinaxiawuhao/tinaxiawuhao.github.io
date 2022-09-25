---
title: '第二章 Apache Kafka 整合 Storm'
date: 2021-04-20 13:29:47
tags: [kafka]
published: true
hideInList: false
feature: /post-images/wAqEEPZon.png
isTop: false
---
在本章中，我们将学习如何将Kafka与Apache Storm集成。

## 关于Storm

Storm最初由Nathan Marz和BackType的团队创建。 在短时间内，Apache Storm成为分布式实时处理系统的标准，允许您处理大量数据。 Storm是非常快的，并且一个基准时钟为每个节点每秒处理超过一百万个元组。 Apache Storm持续运行，从配置的源(Spouts)消耗数据，并将数据传递到处理管道(Bolts)。 联合，Spouts和Bolt构成一个拓扑。

## 与Storm集成

Kafka和Storm自然互补，它们强大的合作能够实现快速移动的大数据的实时流分析。 Kafka和Storm集成是为了使开发人员更容易地从Storm拓扑获取和发布数据流。

### 概念流

Spouts是流的源。 例如，一个喷头可以从Kafka Topic读取元组并将它们作为流发送。 Bolt消耗输入流，处理并可能发射新的流。 Bolt可以从运行函数，过滤元组，执行流聚合，流连接，与数据库交谈等等做任何事情。 Storm拓扑中的每个节点并行执行。 拓扑无限运行，直到终止它。 Storm将自动重新分配任何失败的任务。 此外，Storm保证没有数据丢失，即使机器停机和消息被丢弃。

让我们详细了解Kafka-Storm集成API。 有三个主要类集成Kafka与Storm。 他们如下 -

### BrokerHosts - ZkHosts & StaticHosts

BrokerHosts是一个接口，ZkHosts和StaticHosts是它的两个主要实现。 ZkHosts用于通过在ZooKeeper中维护细节来动态跟踪Kafka代理，而StaticHosts用于手动/静态设置Kafka代理及其详细信息。 ZkHosts是访问Kafka代理的简单快捷的方式。

ZkHosts的签名如下 -

```java
public ZkHosts(String brokerZkStr, String brokerZkPath)
public ZkHosts(String brokerZkStr)
```

其中brokerZkStr是ZooKeeper主机，brokerZkPath是ZooKeeper路径以维护Kafka代理详细信息。

### KafkaConfig API

此API用于定义Kafka集群的配置设置。 Kafka Con-fig的签名定义如下

```java
public KafkaConfig(BrokerHosts hosts, string topic)
```

- **主机** - BrokerHosts可以是ZkHosts / StaticHosts。

- **主题** - 主题名称。

### SpoutConfig API

Spoutconfig是KafkaConfig的扩展，支持额外的ZooKeeper信息。

```java
public SpoutConfig(BrokerHosts hosts, string topic, string zkRoot, string id)
```

- **主机** - BrokerHosts可以是BrokerHosts接口的任何实现
- **主题** - 主题名称。
- **zkRoot** - ZooKeeper根路径。
- **id -** spouts存储在Zookeeper中消耗的偏移量的状态。 ID应该唯一标识您的喷嘴。

### SchemeAsMultiScheme

SchemeAsMultiScheme是一个接口，用于指示如何将从Kafka中消耗的ByteBuffer转换为风暴元组。 它源自MultiScheme并接受Scheme类的实现。 有很多Scheme类的实现，一个这样的实现是StringScheme，它将字节解析为一个简单的字符串。 它还控制输出字段的命名。 签名定义如下。

```java
public SchemeAsMultiScheme(Scheme scheme)
```

- **方案** - 从kafka消耗的字节缓冲区。

### KafkaSpout API

KafkaSpout是我们的spout实现，它将与Storm集成。 它从kafka主题获取消息，并将其作为元组发送到Storm生态系统。 KafkaSpout从SpoutConfig获取其配置详细信息。

下面是一个创建一个简单的Kafka喷水嘴的示例代码。

```java
// ZooKeeper connection string
BrokerHosts hosts = new ZkHosts(zkConnString);

//Creating SpoutConfig Object
SpoutConfig spoutConfig = new SpoutConfig(hosts, 
   topicName, "/" + topicName UUID.randomUUID().toString());

//convert the ByteBuffer to String.
spoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());

//Assign SpoutConfig to KafkaSpout.
KafkaSpout kafkaSpout = new KafkaSpout(spoutConfig);
```

## 

## 创建Bolt

Bolt是一个使用元组作为输入，处理元组，并产生新的元组作为输出的组件。 Bolt将实现IRichBolt接口。 在此程序中，使用两个Bolt类WordSplitter-Bolt和WordCounterBolt来执行操作。

IRichBolt接口有以下方法 -

- **准备** - 为Bolt提供要执行的环境。 执行器将运行此方法来初始化喷头。
- **执行** - 处理单个元组的输入。
- **清理** - 当Bolt要关闭时调用。
- **declareOutputFields** - 声明元组的输出模式。

让我们创建SplitBolt.java，它实现逻辑分割一个句子到词和CountBolt.java，它实现逻辑分离独特的单词和计数其出现。

### SplitBolt.java

```java
import java.util.Map;

import backtype.storm.tuple.Tuple;
import backtype.storm.tuple.Fields;
import backtype.storm.tuple.Values;

import backtype.storm.task.OutputCollector;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.topology.IRichBolt;
import backtype.storm.task.TopologyContext;

public class SplitBolt implements IRichBolt {
   private OutputCollector collector;
   
   @Override
   public void prepare(Map stormConf, TopologyContext context,
      OutputCollector collector) {
      this.collector = collector;
   }
   
   @Override
   public void execute(Tuple input) {
      String sentence = input.getString(0);
      String[] words = sentence.split(" ");
      
      for(String word: words) {
         word = word.trim();
         
         if(!word.isEmpty()) {
            word = word.toLowerCase();
            collector.emit(new Values(word));
         }
         
      }

      collector.ack(input);
   }
   
   @Override
   public void declareOutputFields(OutputFieldsDeclarer declarer) {
      declarer.declare(new Fields("word"));
   }

   @Override
   public void cleanup() {}
   
   @Override
   public Map<String, Object> getComponentConfiguration() {
      return null;
   }
   
}
```

### CountBolt.java

```java
import java.util.Map;
import java.util.HashMap;

import backtype.storm.tuple.Tuple;
import backtype.storm.task.OutputCollector;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.topology.IRichBolt;
import backtype.storm.task.TopologyContext;

public class CountBolt implements IRichBolt{
   Map<String, Integer> counters;
   private OutputCollector collector;
   
   @Override
   public void prepare(Map stormConf, TopologyContext context,
   OutputCollector collector) {
      this.counters = new HashMap<String, Integer>();
      this.collector = collector;
   }

   @Override
   public void execute(Tuple input) {
      String str = input.getString(0);
      
      if(!counters.containsKey(str)){
         counters.put(str, 1);
      }else {
         Integer c = counters.get(str) +1;
         counters.put(str, c);
      }
   
      collector.ack(input);
   }

   @Override
   public void cleanup() {
      for(Map.Entry<String, Integer> entry:counters.entrySet()){
         System.out.println(entry.getKey()&plus;" : " &plus; entry.getValue());
      }
   }

   @Override
   public void declareOutputFields(OutputFieldsDeclarer declarer) {
   
   }

   @Override
   public Map<String, Object> getComponentConfiguration() {
      return null;
   }
}
```

## 提交拓扑

Storm拓扑基本上是一个Thrift结构。 TopologyBuilder类提供了简单而容易的方法来创建复杂的拓扑。 TopologyBuilder类具有设置spout(setSpout)和设置bolt(setBolt)的方法。 最后，TopologyBuilder有createTopology来创建to-pology。 shuffleGrouping和fieldsGrouping方法有助于为喷头和Bolt设置流分组。

**本地集群** - 为了开发目的，我们可以使用 LocalCluster 对象创建本地集群，然后使用 LocalCluster的 submitTopology 类。

### KafkaStormSample.java

```java
import backtype.storm.Config;
import backtype.storm.LocalCluster;
import backtype.storm.topology.TopologyBuilder;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import backtype.storm.spout.SchemeAsMultiScheme;
import storm.kafka.trident.GlobalPartitionInformation;
import storm.kafka.ZkHosts;
import storm.kafka.Broker;
import storm.kafka.StaticHosts;
import storm.kafka.BrokerHosts;
import storm.kafka.SpoutConfig;
import storm.kafka.KafkaConfig;
import storm.kafka.KafkaSpout;
import storm.kafka.StringScheme;

public class KafkaStormSample {
   public static void main(String[] args) throws Exception{
      Config config = new Config();
      config.setDebug(true);
      config.put(Config.TOPOLOGY_MAX_SPOUT_PENDING, 1);
      String zkConnString = "localhost:2181";
      String topic = "my-first-topic";
      BrokerHosts hosts = new ZkHosts(zkConnString);
      
      SpoutConfig kafkaSpoutConfig = new SpoutConfig (hosts, topic, "/" + topic,    
         UUID.randomUUID().toString());
      kafkaSpoutConfig.bufferSizeBytes = 1024 * 1024 * 4;
      kafkaSpoutConfig.fetchSizeBytes = 1024 * 1024 * 4;
      kafkaSpoutConfig.forceFromStart = true;
      kafkaSpoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());

      TopologyBuilder builder = new TopologyBuilder();
      builder.setSpout("kafka-spout", new KafkaSpout(kafkaSpoutCon-fig));
      builder.setBolt("word-spitter", new SplitBolt()).shuffleGroup-ing("kafka-spout");
      builder.setBolt("word-counter", new CountBolt()).shuffleGroup-ing("word-spitter");
         
      LocalCluster cluster = new LocalCluster();
      cluster.submitTopology("KafkaStormSample", config, builder.create-Topology());

      Thread.sleep(10000);
      
      cluster.shutdown();
   }
}
```

在移动编译之前，Kakfa-Storm集成需要策展人ZooKeeper客户端java库。 策展人版本2.9.1支持Apache Storm 0.9.5版(我们在本教程中使用)。 下载下面指定的jar文件并将其放在java类路径中。

- curator-client-2.9.1.jar
- curator-framework-2.9.1.jar

在包括依赖文件之后，使用以下命令编译程序，

```sh
javac -cp "/path/to/Kafka/apache-storm-0.9.5/lib/*" *.java
```

### 执行

启动Kafka Producer CLI(在上一章节中解释)，创建一个名为 my-first-topic 的新主题，并提供一些样本消息，如下所示 -

```sh
hello
kafka
storm
spark
test message
another test message
```

现在使用以下命令执行应用程序 -

```sh
java -cp “/path/to/Kafka/apache-storm-0.9.5/lib/*":. KafkaStormSample
```

此应用程序的示例输出如下所示 -

```sh
storm : 1
test : 2
spark : 1
another : 1
kafka : 1
hello : 1
message : 2
```

>原文：https://www.w3cschool.cn/apache_kafka/apache_kafka_integration_storm.html