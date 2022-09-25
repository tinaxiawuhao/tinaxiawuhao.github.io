---
title: '第五章 Flink 流处理 API'
date: 2021-04-11 09:28:06
tags: [flink]
published: true
hideInList: false
feature: /post-images/lvf921hnA.png
isTop: false
---
## Environment
### getExecutionEnvironment

<p style="text-indent:2em">创建一个执行环境， 表示当前执行程序的上下文。 如果程序是独立调用的， 则
此方法返回本地执行环境； 如果从命令行客户端调用程序以提交到集群， 则此方法
返回此集群的执行环境， 也就是说， getExecutionEnvironment 会根据查询运行的方
式决定返回什么样的运行环境， 是最常用的一种创建执行环境的方式。</p>

```java
// 批处理环境
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment()
// 流式数据处理环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment()
```
如果没有设置并行度， 会以 flink-conf.yaml 中的配置为准， 默认是 1。
![](https://tinaxiawuhao.github.io/post-images/1618450455593.png)
### createLocalEnvironment
返回本地执行环境， 需要在调用时指定默认的并行度。
```java
 LocalStreamEnvironment localEnvironment = StreamExecutionEnvironment.createLocalEnvironment(1);
```
### createRemoteEnvironment

<p style="text-indent:2em">返回集群执行环境， 将 Jar 提交到远程服务器。 需要在调用时指定 JobManager的 IP 和端口号， 并指定要在集群中运行的 Jar 包。</p>

```java
 final ExecutionEnvironment env = ExecutionEnvironment.createRemoteEnvironment(
            cluster.getHostname(),
            cluster.getPort(),
            config
    );
```
scala
```java
val env = ExecutionEnvironment.createRemoteEnvironment("jobmanage-hostname",
6123,"YOURPATH//wordcount.jar")
```

### setParallelism
```java
// 为了打印到控制台的结果不乱序，我们配置全局的并发为1，改变并发对结果正确性没有影响
env.setParallelism(1);
```

## Source
### 从集合读取数据
```java
package myflink;

import lombok.*;

//传感器温度读数的数据类型
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString
public class SensorReading {
    //属性 id,时间戳,温度值
    private String id;
    private Long timestamp;
    private Double temperature;

}

public class SourceReading_Collection {
    public static void main(String[] args) throws Exception {
        //创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行调度
        //env.setParallelism(1);
        //从集合中读取数据
        //属性 id,时间戳,温度值
        DataStream<SensorReading> dataStream = env.fromCollection(Arrays.asList(new SensorReading("sensor_1", 1537718199L, 35.8),
                new SensorReading("sensor_6", 1547718201L, 15.4),
                new SensorReading("sensor_7", 1547718202L, 6.7),
                new SensorReading("sensor_10", 1547718205L, 38.1)));
 
        DataStream<Integer> integerDataStream = env.fromElements(1, 2, 4, 67, 189);
        //打印输出
        dataStream.print("data");
        integerDataStream.print("int");
        //执行 Flink的jobName
        env.execute("SensorReading");
    }
}
```
![](https://tinaxiawuhao.github.io/post-images/1618820709627.png)
### 从文件读取数据
```java
public class SourceReading_File {
    public static void main(String[] args) throws Exception {
        //创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //读取文件
       DataStream<String> dataStream = env.readTextFile(filePath,charsetName);
       //打印输出
        dataStream.print("data");
        //执行 Flink的jobName
        env.execute("SensorReading");
    }
}
```
### 以 kafka 消息队列的数据作为来源
需要引入 kafka 连接器的依赖：
`pom.xml`
```java
<!--
https://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.11
-->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```
```sh
//1. 创建一个Topic名为“test20201217”的主题
kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test20201217
//2. 创建producer(生产者)，生产主题的消息
kafka-console-producer.bat --broker-list localhost:9092 --topic test20201217
//3. 创建consumer(消费者)，消费主题消息
kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test20201217
```
具体代码如下：
```java
/**
 * kafkaSource
 *
 *    從指定的offset出消费kafka
 */
public class StreamingKafkaSource {

    public static void main(String[] args) throws Exception {
         // 创建流处理的执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
 
        // 配置KafKa
        //配置KafKa和Zookeeper的ip和端口
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");
        properties.setProperty("zookeeper.connect", "localhost:2181");
        properties.setProperty("group.id", "consumer-group");
        //将kafka和zookeeper配置信息加载到Flink的执行环境当中StreamExecutionEnvironment
        FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<String>("test20201217", new SimpleStringSchema(),
                properties);
 
        //添加数据源，此处选用数据流的方式，将KafKa中的数据转换成Flink的DataStream类型
        DataStream<String> stream = env.addSource(myConsumer);
 
 
        //打印输出
        stream.print();
        //执行Job，Flink执行环境必须要有job的执行步骤，而以上的整个过程就是一个Job
        env.execute("kafka sink test");
    }
}
```
![](https://tinaxiawuhao.github.io/post-images/1618821668432.png)
### 自定义 Source

<p style="text-indent:2em">除了以上的 source 数据来源， 我们还可以自定义 source。 需要做的， 只是传入一个 SourceFunction 就可以。 具体调用如下：</p>

```java
 env.addSource( new MySensorSource() )
```
我们希望可以随机生成传感器数据， MySensorSource 具体的代码实现如下：
```java
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.datastream.WindowedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;
import org.apache.lucene.analysis.CachingTokenFilter;
 
import java.util.Random;
 
public class MySelfSourceTest01 {
    public static void main(String[] args) {
        Logger.getLogger("org").setLevel(Level.OFF);
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> dataStreamSource = env.addSource(new SourceFunction<String>() {
            @Override
            public void run(SourceContext<String> ctx) throws Exception {
                Random random = new Random();
                // 循环可以不停的读取静态数据
                while (true) {
                    int nextInt = random.nextInt(100);
                    ctx.collect("random : " + nextInt);
                    Thread.sleep(1000);
                }
            }
 
            @Override
            public void cancel() {
 
            }
        });
        WindowedStream<Tuple2<String, Integer>, Tuple, TimeWindow> window = dataStreamSource.map(new MapFunction<String, Tuple2<String, Integer>>() {
            @Override
            public Tuple2<String, Integer> map(String value) throws Exception {
                String[] sps = value.split(":");
                return new Tuple2<>(value, Integer.parseInt(sps[1].trim()));
            }
        }).keyBy(0).timeWindow(Time.seconds(5));
 
        SingleOutputStreamOperator<String> apply = window.apply(new WindowFunction<Tuple2<String, Integer>, String, Tuple, TimeWindow>() {
            @Override
            public void apply(Tuple tuple, TimeWindow window, Iterable<Tuple2<String, Integer>> input, Collector<String> out) throws Exception {
                input.forEach(x -> {
                    System.out.println("apply function -> " + x.f0);
                    out.collect(x.f0);
                });
            }
        });
 
        apply.print();
 
        try {
            env.execute("myself_source_test01");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
 ## Transform
转换算子
### map
![](https://tinaxiawuhao.github.io/post-images/1618450860405.png)
```java
stream.map { x => x * 2 }
// 1.map.把string转化成长度输出
//参数T  R
//T就是传的数据，什么类型都行
//R就是返回的类型
DataStream<Integer> mapStream = stringDataStream.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) throws Exception {
        return value.length();
    }
});

```
### flatMap
flatMap 的函数签名： `def flatMap[A,B](as: List[A])(f: A ⇒ List[B]): List[B]`
例如: flatMap(List(1,2,3))(i ⇒ List(i,i))
结果是 List(1,1,2,2,3,3),
而 List("a b", "c d").flatMap(line ⇒ line.split(" "))
结果是 List(a, b, c, d)。
```java
stream.flatMap{
    x => x.split(" ")
}
//2、flatmap,按照逗号分隔
DataStream<String> flatMapStream = stringDataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out) throws Exception {
        String[] fields = value.split(",");
        for(String field:fields)
            out.collect(field);
    }
});

``` 
### Filter
![](https://tinaxiawuhao.github.io/post-images/1618450868563.png)
```java
stream.filter{
    x => x == 1
}
 //3.filter,筛选sensor_1开头的的id对应的数据
DataStream<String> filterStream = stringDataStream.filter(new FilterFunction<String>() {
    @Override
    public boolean filter(String value) throws Exception {
        return value.startsWith("sensor_1");
    }
});
```
### KeyBy

![](https://tinaxiawuhao.github.io/post-images/1618450909734.png)

`DataStream → KeyedStream`： 逻辑地将一个流拆分成不相交的分区， 每个分区包含具有相同 key 的元素， 在内部以 hash 的形式实现的。
```java
 DataStream<Tuple2<String, Integer>> windowCounts = text
                .flatMap(new FlatMapFunction<String, Tuple2<String, Integer>>() {
                    @Override
                    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
                        for (String word : value.split("\\s")) {
                            out.collect(Tuple2.of(word, 1));
                        }
                    }
                })
                //按Tuple2的第一个属性进行分区
                .keyBy(0)

//或者根据对象属性进行分区
 KeyedStream<SensorReading, String> tempkeyedStream = mapDataStream.keyBy(mpdata -> mpdata.getId());
```

### 滚动聚合算子（Rolling Aggregation）
这些算子可以针对 KeyedStream 的每一个支流做聚合。
1. sum()
2. min()
3. max()
4. minBy()
5. maxBy()
### Reduce

`KeyedStream → DataStream`： 一个分组数据流的聚合操作， 合并当前的元素和上次聚合的结果， 产生一个新的值， 返回的流中包含每一次聚合的结果， 而不是只返回最后一次聚合的最终结果。

```java
DataStream<String> stream2 = env.readTextFile("YOUR_PATH\\sensor.txt")
    .map( data => {
        String[] dataArray = data.split(",")
        SensorReading(dataArray(0).trim, dataArray(1).trim.toLong,
        dataArray(2).trim.toDouble)
    })
    .keyBy("id")
    .reduce( (x, y) => SensorReading(x.id, x.timestamp + 1, y.temperature) )
```
### Split 和 Select
Split
![](https://tinaxiawuhao.github.io/post-images/1618450986439.png)
图 Split
`DataStream → SplitStream`： 根据某些特征把一个 DataStream 拆分成两个或者
多个 DataStream。
Select
![](https://tinaxiawuhao.github.io/post-images/1618450993631.png)
图 Select
`SplitStream→ DataStream`： 从一个 SplitStream 中获取一个或者多个
DataStream。
需求： 传感器数据按照温度高低（ 以 30 度为界） ， 拆分成两个流。
```java
public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env=StreamExecutionEnvironment.getExecutionEnvironment();
        DataStream<Long> input=env.generateSequence(0,10);
        SplitStream<Long> splitStream = input.split(new OutputSelector<Long>(){
            @Override
            public Iterable<String> select(Long value) {
                List<String> output = new ArrayList<String>();
                if (value % 2 == 0) {
                    output.add("even");
                }else {
                    output.add("odd");
                }
                return output;
            }
        });
        //splitStream.print();
        DataStream<Long> even = splitStream.select("even");
        DataStream<Long> odd = splitStream.select("odd");
        DataStream<Long> all = splitStream.select("even","odd");
        //even.print();
        odd.print();
        //all.print();
        env.execute();
    }
```
### Connect 和 CoMap
![](https://tinaxiawuhao.github.io/post-images/1618451022910.png)
图 Connect 算子
`DataStream,DataStream → ConnectedStreams`： 连接两个保持他们类型的数
据流， 两个数据流被 Connect 之后， 只是被放在了一个同一个流中， 内部依然保持
各自的数据和形式不发生任何变化， 两个流相互独立。

`CoMap`,`CoFlatMap`
![](https://tinaxiawuhao.github.io/post-images/1618451082833.png)
图 CoMap/CoFlatMap

`ConnectedStreams → DataStream`： 作用于 ConnectedStreams 上， 功能与 map
和 flatMap 一样， 对 ConnectedStreams 中的每一个 Stream 分别进行 map 和 flatMap
处理。

```java
DataStream  warning = high.map( sensorData => (sensorData.id,
sensorData.temperature) )
ConnectedStreams connected = warning.connect(low)
DataStream coMap = connected.map(
    warningData => (warningData._1, warningData._2, "warning"),
    lowData => (lowData.id, "healthy")
)
```
 ### Union
 ![](https://tinaxiawuhao.github.io/post-images/1618451109858.png)
图 Union
`DataStream → DataStream`： 对两个或者两个以上的 DataStream 进行 union 操
作， 产生一个包含所有 DataStream 元素的新 DataStream。

```java
//合并以后打印
 DataStream<StartUpLog> unionStream = appStoreStream.union(otherStream)
unionStream.print("union:::")
```
`Connect` 与 `Union` 区别：
>1. Union 之前两个流的类型必须是一样， Connect 可以不一样， 在之后的 coMap
中再去调整成为一样的。
>2. Connect 只能操作两个流， Union 可以操作多个。
## 支持的数据类型

<p style="text-indent:2em">Flink 流应用程序处理的是以数据对象表示的事件流。 所以在 Flink 内部， 我们
需要能够处理这些对象。 它们需要被序列化和反序列化， 以便通过网络传送它们；
或者从状态后端、 检查点和保存点读取它们。 为了有效地做到这一点， Flink 需要明
确知道应用程序所处理的数据类型。 Flink 使用类型信息的概念来表示数据类型， 并
为每个数据类型生成特定的序列化器、 反序列化器和比较器。
Flink 还具有一个类型提取系统， 该系统分析函数的输入和返回类型， 以自动获
取类型信息， 从而获得序列化器和反序列化器。 但是， 在某些情况下， 例如 lambda
函数或泛型类型， 需要显式地提供类型信息， 才能使应用程序正常工作或提高其性
能。</p>

Flink 支持 Java 和 Scala 中所有常见数据类型。 使用最广泛的类型有以下几种。
### 基础数据类型
Flink 支持所有的 Java 和 Scala 基础数据类型， Int, Double, Long, String, …​
```java
DataStream<Long> numbers = env.fromElements(1L, 2L, 3L, 4L)
numbers.map( n => n + 1 )
```
### Java 和 Scala 元组（Tuples）
```java
DataStream<String, Integer> persons= env.fromElements(
    ("Adam", 17),
    ("Sarah", 23) 
)
persons.filter(p => p._2 > 18)
```
### Scala 样例类（case classes）
```java
case class Person(name: String, age: Int)
val persons: DataStream[Person] = env.fromElements(
Person("Adam", 17),
Person("Sarah", 23) )
persons.filter(p => p.age > 18)
```
### Java 简单对象（POJOs）
```java
public class Person {
    public String name;
    public int age;
    public Person() {}
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
 DataStream<Person> persons = env.fromElements(
new Person("Alex", 42),
new Person("Wendy", 23));
```
### 其它（Arrays, Lists, Maps, Enums, 等等）
Flink 对 Java 和 Scala 中的一些特殊目的的类型也都是支持的， 比如 Java 的
ArrayList， HashMap， Enum 等等。
## 实现 UDF 函数——更细粒度的控制流
### 函数类（Function Classes）
Flink 暴露了所有 udf 函数的接口(实现方式为接口或者抽象类)。 例如
MapFunction, FilterFunction, ProcessFunction 等等。
下面例子实现了 FilterFunction 接口：
```java
public class CustomFilterFunction implements FilterFunction<SensorReading> {
    @Override
    public boolean filter(SensorReading sensorReading) throws Exception {
        return sensorReading.temperature>30.0;
    }
}
public static void main(String[] args) throws Exception {
 // 创建流处理的执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
 
        // 从文件中读取数据
        String inputPath  = "F:\\Projects\\BigData\\Flink\\FlinkTutorial\\src\\main\\resources\\sensor.txt";
        // 获取数据
        DataStreamSource<String> dataStream  = env.readTextFile(inputPath);
 
        // 1、先转换成SensorReading类型（简单转换操作）
        DataStream<SensorReading> stream =  dataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String data) throws Exception {
                String[] arr = data.split(",");
                return new SensorReading(arr[0], arr[1], Double.valueOf(arr[2].toString()));
            }
        });
 
       // 调用自定义CustomFilterFunction类的，实现过滤
       DataStream<SensorReading> dataStreamFilter = stream.filter(new CustomFilterFunction());
 
       dataStreamFilter .print("CustomFilterFunction");
      
       env.execute("Function test");
}
```
还可以将函数实现成匿名类
```java
 DataStream<SensorReading> dataStreamFilter = stream.filter(new FilterFunction<SensorReading>() {
    @Override
    public boolean filter(SensorReading sensorReading) throws Exception {
        return sensorReading.temperature>30.0;
    }
);
```
我们 filter 的字符串"flink"还可以当作参数传进去。
```java
DataStream<String> tweets = ...
DataStream<String> flinkTweets = tweets.filter(new KeywordFilter("flink"))

public class KeywordFilter(String keyWord) implements FilterFunction<String> {
    @Override
    public boolean filter(String value) throws Exception {
        return value.contains(keyWord)
    }
}
```
### 匿名函数（Lambda Functions）
```java
DataStream<String> tweets = ...
DataStream<String> flinkTweets = tweets.filter((value) ->value.contains("flink"))
```
### 富函数（Rich Functions）
“ 富函数” 是 DataStream API 提供的一个函数类的接口， 所有 Flink 函数类都
有其 Rich 版本。 它与常规函数的不同在于， 可以获取运行环境的上下文， 并拥有一
些生命周期方法， 所以可以实现更复杂的功能。
1. RichMapFunction
2. RichFlatMapFunction
3. RichFilterFunction
 …​

Rich Function 有一个生命周期的概念。 典型的生命周期方法有：
1. open()方法是 rich function 的初始化方法， 当一个算子例如 map 或者 filter
被调用之前 open()会被调用。
1. close()方法是生命周期中的最后一个调用的方法， 做一些清理工作。
2. getRuntimeContext()方法提供了函数的 RuntimeContext 的一些信息， 例如函
数执行的并行度， 任务的名字， 以及 state 状态
```java 
    public class MyFlatMap extends RichFlatMapFunction<Tuple2<Integer, Integer>, Tuple2<Integer, Integer>> {
    Integer subTaskIndex = 0
    StreamingRuntimeContext context = (StreamingRuntimeContext) getRuntimeContext()
     public void open(Configuration parameters) throws Exception {
         subTaskIndex = context.getIndexOfThisSubtask
        // 以下可以做一些初始化工作， 例如建立一个和 HDFS 的连接
    }
    @Override
    public void flatMap(Tuple2<Integer, Integer> integerIntegerTuple2, Collector<Tuple2<Integer, Integer>> collector) throws Exception {
            if (in % 2 == subTaskIndex) {
                out.collect((subTaskIndex, in))
            }
        }
    public void close() throws Exception {
        // 以下做一些清理工作， 例如断开和 HDFS 的连接。
    }
}
```
## Sink

<p style="text-indent:2em">Flink 没有类似于 spark 中 foreach 方法， 让用户进行迭代的操作。 虽有对外的
输出操作都要利用 Sink 完成。 最后通过类似如下方式完成整个任务最终输出操作。
stream.addSink(new MySink(xxxx))
官方提供了一部分的框架的 sink。 除此以外， 需要用户自定义实现 sink。</p>

![](https://tinaxiawuhao.github.io/post-images/1618451579569.png)
### Kafka
```java
pom.xml
<!--
https://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.11
-->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```
主函数中添加 sink：
```java
DataStream<String> union = high.union(low).map(item->item.temperature.toString)
union.addSink(new FlinkKafkaProducer<String>("localhost:9092",
"test", new SimpleStringSchema()))
```
### Redis
pom.xml
```java
<!-- https://mvnrepository.com/artifact/org.apache.bahir/flink-connector-redis
-->
<dependency>
    <groupId>org.apache.bahir</groupId>
    <artifactId>flink-connector-redis_2.11</artifactId>
    <version>1.0</version>
</dependency>
```
定义一个 redis 的 mapper 类， 用于定义保存到 redis 时调用的命令：
```java
public class MyRedisMapper extends RedisMapper<SensorReading>{
    public RedisCommandDescription getCommandDescription{
        new RedisCommandDescription(RedisCommand.HSET, "sensor_temperature")
    } 
    public String  getValueFromData(SensorReading t){
        t.temperature.toString
        String getKeyFromData(SensorReading t) {
            return t.id
        }
    }
}
```
在主函数中调用：
```java
dataStream.addSink( new RedisSink<SensorReading>( new
FlinkJedisPoolConfig.Builder().setHost("localhost").setPort(6379).build(), new MyRedisMapper) )
```
### Elasticsearch
pom.xml
```java
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-elasticsearch6_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```
在主函数中调用：
```java
List httpHosts = new ArrayList<HttpHost>()
httpHosts.add(new HttpHost("localhost", 9200))
DataStream esSinkBuilder = new ElasticsearchSink.Builder<SensorReading>( httpHosts,new ElasticsearchSinkFunction<SensorReading> {
    public  Unit process(SensorReading t, RuntimeContext runtimeContext,
    RequestIndexer requestIndexer ) {
        println("saving data: " + t)
        map json = new util.HashMap<String, String>()
        json.put("data", t.toString)
        IndexRequest  indexRequest =
        Requests.indexRequest().index("sensor").`type`("readingData").source(json)
        requestIndexer.add(indexRequest)
        println("saved successfully")
    }
} )
dataStream.addSink( esSinkBuilder.build() )
```
### JDBC 自定义 sink
```java
<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.44</version>
</dependency>
```
添加 MyJdbcSink
```java
public class MyJdbcSink() extends RichSinkFunction<SensorReading>{
    Connection conn;
    PreparedStatement insertStmt;
    PreparedStatement updateStmt;
    // open 主要是创建连接
    public Unit open(Configuration parameters)  = {
        super.open(parameters)
        conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/test",
        "root", "123456")
        insertStmt = conn.prepareStatement("INSERT INTO temperatures (sensor,
        temp) VALUES (?, ?)")
        updateStmt = conn.prepareStatement("UPDATE temperatures SET temp = ? WHERE
        sensor = ?")
    } 
    //调用连接， 执行 sql
    public Unit invoke(SensorReading value , SinkFunction.Context[] context）{
        updateStmt.setDouble(1, value.temperature)
        updateStmt.setString(2, value.id)
        updateStmt.execute()
        if (updateStmt.getUpdateCount == 0) {
            insertStmt.setString(1, value.id)
            insertStmt.setDouble(2, value.temperature)
            insertStmt.execute()
        }
    } 
    public Unit close() {
        insertStmt.close()
        updateStmt.close()
        conn.close()
    }
}
```
在 main 方法中增加， 把明细保存到 mysql 中
```java
dataStream.addSink(new MyJdbcSink())
```