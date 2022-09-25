---
title: '第二章 快速上手'
date: 2021-04-08 14:49:09
tags: [flink]
published: true
hideInList: false
feature: /post-images/pQQ9j5lFD.png
isTop: false
---
## 搭建 maven 工程 FlinkTutorial
### pom 文件
```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>flink-test</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <encoding>UTF-8</encoding>
        <scala.version>2.11</scala.version>
        <flink.version>1.10.0</flink.version>
        <hadoop.version>2.7.7</hadoop.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
            <!--<scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.version}</artifactId>
            <version>${flink.version}</version>
            <!--<scope>provided</scope>-->
        </dependency>
    </dependencies>
</project>
```
### 批处理 wordcount
WordCount 程序是大数据处理框架的入门程序，俗称“单词计数”。用来统计一段文字每个单词的出现次数，该程序主要分为两个部分：一部分是将文字拆分成单词；另一部分是单词进行分组计数并打印输出结果。
```java
public static void main(String[] args) throws Exception {
      // 创建Flink运行的上下文环境
      final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
      // 创建DataSet，这里我们的输入是一行一行的文本
      DataSet<String> text = env.fromElements(
            "Flink Spark Storm",
            "Flink Flink Flink",
            "Spark Spark Spark",
            "Storm Storm Storm"
      );
      // 通过Flink内置的转换函数进行计算
      DataSet<Tuple2<String, Integer>> counts =
            text.flatMap(new LineSplitter())
                  .groupBy(0)
                  .sum(1);
      //结果打印
      counts.printToErr();

   }

   public static final class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
      @Override
      public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
         // 将文本分割
         String[] tokens = value.toLowerCase().split("\\W+");
         for (String token : tokens) {
            if (token.length() > 0) {
               out.collect(new Tuple2<String, Integer>(token, 1));
            }
         }
      }
    }
```
实现的整个过程中分为以下几个步骤。
首先，我们需要创建 Flink 的上下文运行环境：
复制`ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();`
然后，使用 fromElements 函数创建一个 DataSet 对象，该对象中包含了我们的输入，使用 FlatMap、GroupBy、SUM 函数进行转换。
最后，直接在控制台打印输出。
我们可以直接右键运行一下 main 方法，在控制台会出现我们打印的计算结果：
![](https://tinaxiawuhao.github.io/post-images/1618469933995.png)
### 流处理 wordcount
为了模仿一个流式计算环境，我们选择监听一个本地的 Socket 端口，并且使用 Flink 中的滚动窗口，每 5 秒打印一次计算结果。代码如下：
```java
public class StreamingJob {

    public static void main(String[] args) throws Exception {

        // 创建Flink的流式计算环境
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 监听本地9000端口
        DataStream<String> text = env.socketTextStream("127.0.0.1", 9000, "\n");

        // 将接收的数据进行拆分，分组，窗口计算并且进行聚合输出
        DataStream<WordWithCount> windowCounts = text
                .flatMap(new FlatMapFunction<String, WordWithCount>() {
                    @Override
                    public void flatMap(String value, Collector<WordWithCount> out) {
                        for (String word : value.split("\\s")) {
                            out.collect(new WordWithCount(word, 1L));
                        }
                    }
                })
                .keyBy("word")
                .timeWindow(Time.seconds(5), Time.seconds(1))
                .reduce(new ReduceFunction<WordWithCount>() {
                    @Override
                    public WordWithCount reduce(WordWithCount a, WordWithCount b) {
                        return new WordWithCount(a.word, a.count + b.count);
                    }
                });

        // 打印结果
        windowCounts.print().setParallelism(1);

        env.execute("Socket Window WordCount");
    }

    // Data type for words with count
    public static class WordWithCount {

        public String word;
        public long count;

        public WordWithCount() {}

        public WordWithCount(String word, long count) {
            this.word = word;
            this.count = count;
        }

        @Override
        public String toString() {
            return word + " : " + count;
        }
    }
}
```
整个流式计算的过程分为以下几步。

首先创建一个流式计算环境：

复制`StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();`
然后进行监听本地 9000 端口，将接收的数据进行拆分、分组、窗口计算并且进行聚合输出。代码中使用了 Flink 的窗口函数，我们在后面的课程中将详细讲解。
![](https://tinaxiawuhao.github.io/post-images/1618469926125.png)
我们在本地使用 netcat 命令启动一个端口：
```sh
nc -lk 9000
```
然后直接运行我们的 main 方法：
测试——在 linux 系统中用 netcat 命令进行发送测试。
在 nc 中输入：
```sh
$ nc -lk 9000
Flink Flink Flink 
Flink Spark Storm
```
可以在控制台看到：
```sh
Flink : 4
Spark : 1
Storm : 1
```