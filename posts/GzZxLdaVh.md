---
title: '第七章 时间语义与 Wartermark'
date: 2021-04-14 15:39:23
tags: [flink]
published: true
hideInList: false
feature: /post-images/GzZxLdaVh.png
isTop: false
---
## Flink 中的时间语义
在 Flink 的流式处理中， 会涉及到时间的不同概念， 如下图所示：
![](https://tinaxiawuhao.github.io/post-images/1618472695574.png)
图 Flink 时间概念
`Event Time`： 是事件创建的时间。 它通常由事件中的时间戳描述， 例如采集的
日志数据中， 每一条日志都会记录自己的生成时间， Flink 通过时间戳分配器访问事
件时间戳。
`Ingestion Time`： 是数据进入 Flink 的时间。
`Processing Time`： 是每一个执行基于时间操作的算子的本地系统时间， 与机器
相关， 默认的时间属性就是 Processing Time。
一个例子——电影《 星球大战》 ：
![](https://tinaxiawuhao.github.io/post-images/1618472731095.png)
例如， 一条日志进入 Flink 的时间为 2020-11-12 10:00:00.123， 到达 Window 的
系统时间为 2020-11-12 10:00:01.234， 日志的内容如下：
```sh
2020-11-02 18:37:15.624 INFO Fail over to rm2
```
对于业务来说， 要统计 1min 内的故障日志个数， 哪个时间是最有意义的？ ——
eventTime， 因为我们要根据日志的生成时间进行统计。
## EventTime 的引入

<p style="text-indent:2em">在 Flink 的流式处理中， 绝大部分的业务都会使用 eventTime， 一般只在eventTime 无法使用时， 才会被迫使用 ProcessingTime 或者 IngestionTime。
如果要使用 EventTime， 那么需要引入 EventTime 的时间属性， 引入方式如下所
示：</p>

```java
// 创建 execution environment
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 告诉系统按照 EventTime 处理
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```
## Watermark
### 基本概念

<p style="text-indent:2em">我们知道， 流处理从事件产生， 到流经 source， 再到 operator， 中间是有一个过程和时间的， 虽然大部分情况下， 流到 operator 的数据都是按照事件产生的时间顺序来的， 但是也不排除由于网络、 分布式等原因， 导致乱序的产生， 所谓乱序， 就
是指 Flink 接收到的事件的先后顺序不是严格按照事件的 Event Time 顺序排列的。</p>

![](https://tinaxiawuhao.github.io/post-images/1618472852122.png)
图 数据的乱序

<p style="text-indent:2em">那么此时出现一个问题， 一旦出现乱序， 如果只根据 eventTime 决定 window 的运行， 我们不能明确数据是否全部到位， 但又不能无限期的等下去， 此时必须要有
个机制来保证一个特定的时间后， 必须触发 window 去进行计算了， 这个特别的机
制， 就是 Watermark。</p>

1. Watermark 是一种衡量 Event Time 进展的机制。
2. Watermark 是用于处理乱序事件的， 而正确的处理乱序事件， 通常用
Watermark 机制结合 window 来实现。
3. 数据流中的 Watermark 用于表示 timestamp 小于 Watermark 的数据， 都已经
到达了， 因此， window 的执行也是由 Watermark 触发的。
4. Watermark 可以理解成一个延迟触发机制， 我们可以设置 Watermark 的延时
时长 t， 每次系统会校验已经到达的数据中最大的 maxEventTime， 然后认定 eventTime
小于 maxEventTime - t 的所有数据都已经到达， 如果有窗口的停止时间等于
maxEventTime – t， 那么这个窗口被触发执行。
有序流的 Watermarker 如下图所示： （ Watermark 设置为 0）
![](https://tinaxiawuhao.github.io/post-images/1618472893459.png)
图 有序数据的 Watermark
乱序流的 Watermarker 如下图所示： （ Watermark 设置为 2）
![](https://tinaxiawuhao.github.io/post-images/1618472899867.png)
图 无序数据的 Watermark

<p style="text-indent:2em">当 Flink 接收到数据时， 会按照一定的规则去生成 Watermark， 这条 Watermark
就等于当前所有到达数据中的 maxEventTime - 延迟时长， 也就是说， Watermark 是
基于数据携带的时间戳生成的， 一旦 Watermark 比当前未触发的窗口的停止时间要
晚， 那么就会触发相应窗口的执行。 由于 event time 是由数据携带的， 因此， 如果
运行过程中无法获取新的数据， 那么没有被触发的窗口将永远都不被触发。</p>

<p style="text-indent:2em">上图中， 我们设置的允许最大延迟到达时间为 2s， 所以时间戳为 7s 的事件对应
的 Watermark 是 5s， 时间戳为 12s 的事件的 Watermark 是 10s， 如果我们的窗口 1
是 1s~5s， 窗口 2 是 6s~10s， 那么时间戳为 7s 的事件到达时的 Watermarker 恰好触
发窗口 1， 时间戳为 12s 的事件到达时的 Watermark 恰好触发窗口 2。Watermark 就是触发前一窗口的“ 关窗时间” ， 一旦触发关门那么以当前时刻为准在窗口范围内的所有所有数据都会收入窗中。
只要没有达到水位那么不管现实中的时间推进了多久都不会触发关窗。</p>

### Watermark 的引入
watermark 的引入很简单， 对于乱序数据， 最常见的引用方式如下：
```java
// 抽取出时间和生成 watermark
dataStream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
    @Override
    public long extractAscendingTimestamp(UserBehavior userBehavior) {
        // 原始数据单位秒，将其转成毫秒
        return userBehavior.timestamp * 1000;
    }
})
```
<p style="text-indent:2em">Event Time 的使用一定要指定数据源中的时间戳。 否则程序无法知道事件的事
件时间是什么(数据源里的数据没有时间戳的话， 就只能使用 Processing Time 了)。
我们看到上面的例子中创建了一个看起来有点复杂的类， 这个类实现的其实就
是分配时间戳的接口。 Flink 暴露了 TimestampAssigner 接口供我们实现， 使我们可
以自定义如何从事件数据中抽取时间戳。</p>

```java
// 创建 execution environment
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 通过连接 socket 获取输入数据，这里连接到本地9000端口，如果9000端口已被占用，请换一个端口
DataStream<String> text = env.socketTextStream("localhost", 9000, "\n");
text.assignTimestampsAndWatermarks(new MyAssigner())
```

MyAssigner 有两种类型
1. AssignerWithPeriodicWatermarks
2. AssignerWithPunctuatedWatermarks
以上两个接口都继承自 TimestampAssigner。
### Assigner with periodic watermarks
周期性的生成 watermark： 系统会周期性的将 watermark 插入到流中(水位线也
是一种特殊的事件!)。 默认周期是 200 毫秒。 可以使用
ExecutionConfig.setAutoWatermarkInterval()方法进行设置。
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
// 每隔 5 秒产生一个 watermark
env.getConfig.setAutoWatermarkInterval(5000)
```

<p style="text-indent:2em">产生 watermark 的逻辑： 每隔 5 秒钟， Flink 会调用
AssignerWithPeriodicWatermarks 的 getCurrentWatermark()方法。 如果方法返回一个
时间戳大于之前水位的时间戳， 新的 watermark 会被插入到流中。 这个检查保证了
水位线是单调递增的。 如果方法返回的时间戳小于等于之前水位的时间戳， 则不会
产生新的 watermark。</p>

例子， 自定义一个周期性的时间戳抽取：
```java
public class PeriodicAssigner extends
AssignerWithPeriodicWatermarks<SensorReading> {
    Long bound= 60 * 1000 // 延时为 1 分钟
    Long maxTs = Long.MinValue // 观察到的最大时间戳
    public Watermark  getCurrentWatermark() {
        new Watermark(maxTs - bound)
    } 
    public void extractTimestamp(SensorReading r,Long previousTS) = {
        maxTs = maxTs.max(r.timestamp)
        r.timestamp
    }
}
``` 
<p style="text-indent:2em">一种简单的特殊情况是， 如果我们事先得知数据流的时间戳是单调递增的， 也就是说没有乱序， 那我们可以使用 assignAscendingTimestamps， 这个方法会直接使
用数据的时间戳生成 watermark。</p>

```java
DataStream<SensorReading> stream:  = ...
DataStream<SensorReading> withTimestampsAndWatermarks = stream
.assignAscendingTimestamps(e => e.timestamp)
>> result: E(1), W(1), E(2), W(2), ...
```
而对于乱序数据流， 如果我们能大致估算出数据流中的事件的最大延迟时间，
就可以使用如下代码：
```java
DataStream<SensorReading> stream:  = ...
DataStream<SensorReading> withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(
    new SensorTimeAssigner()
) 
public class SensorTimeAssigner extends
BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(5)) {
    // 抽取时间戳
    public Long  extractTimestamp(SensorReading r){
        return r.timestamp
    }
}
>> relust: E(10), W(0), E(8), E(7), E(11), W(1), ...
```
### Assigner with punctuated watermarks
<p style="text-indent:2em">间断式地生成 watermark。 和周期性生成的方式不同， 这种方式不是固定时间的，而是可以根据需要对每条数据进行筛选和处理。 直接上代码来举个例子， 我们只给
sensor_1 的传感器的数据流插入 watermark：</p>

```java
public class PunctuatedAssigner extends
    AssignerWithPunctuatedWatermarks<SensorReading> {
    Long bound=60 * 1000
    public  Watermark checkAndGetNextWatermark(SensorReading r , Long extractedTS) {
        if (r.id == "sensor_1") {
            new Watermark(extractedTS - bound)
        } else {
            null
         }
    } 
    public  Long extractTimestamp(SensorReading r, Long previousTS){
        return  r.timestamp
    }
}
```
## EvnetTime 在 window 中的使用
### 滚动窗口（TumblingEventTimeWindows）
```java
public class EventTimeTumblingWindowAllDemo {
    public static void main(String[] args) throws Exception {
        //  2021-03-06 21:00:00,1
        //  2021-03-06 21:00:05,2
        // 结果 ： 2> 1
        //        3> 2
        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(new Configuration());
        // 老版本必须要设置时间标准 （1.20 之前的）
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        DataStreamSource<String> lines = env.socketTextStream("linux01", 8888);
       // flink里面的时间都精确到毫秒
       //  时间提取器 BoundedOutOfOrdernessTimestampExtractor : 允许时间乱序，并且可以指定窗口延时
       // WaterMark （水位线，可以让窗口延迟触发的一种机制）
       // 一个窗口中的一个分区的水位线 = 当前窗口当前分区最大的 EventTime - 延迟时间     Time.seconds(0)
        SingleOutputStreamOperator<String> linesWithWaterMark = lines.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<String>(Time.seconds(0)) {

            private SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss") ;
            @Override
            public long extractTimestamp(String s) {
                String strings = s.split(",")[0];
                long timestamp = 0;
                try {
                    Date date = dateFormat.parse(strings);
                    timestamp = date.getTime();
                } catch (ParseException e) {
                    e.printStackTrace();
                    timestamp = System.currentTimeMillis();
                }
                return timestamp;
            }
        });

        SingleOutputStreamOperator<Integer> nums = linesWithWaterMark.map(new MapFunction<String, Integer>() {
            @Override
            public Integer map(String s) throws Exception {
                int i = Integer.parseInt(s.split(",")[1]);
                return i;
            }
        });
        //  划分窗口
        AllWindowedStream<Integer, TimeWindow> window = nums.windowAll(TumblingProcessingTimeWindows.of(Time.seconds(5)));
        // 对 window 中的数据 进行聚合
        SingleOutputStreamOperator<Integer> sum = window.sum(0);

        sum.print();
        env.execute() ;


    }
}
```

结果是按照 Event Time 的时间窗口计算得出的， 而无关系统的时间（ 包括输入的快慢） 。
### 滑动窗口（SlidingEventTimeWindows）
```java
public class EventTimeSlidingWindowDemo {
    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(new Configuration());
         //  两秒 调一次方法
        env.getConfig().setAutoWatermarkInterval(1000);
        // 老版本必须要设置时间标准 （1.20 之前的）
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        DataStreamSource<String> lines = env.socketTextStream("linux01", 8888);
       // flink里面的时间都精确到毫秒
       //  时间提取器 BoundedOutOfOrdernessTimestampExtractor : 允许时间乱序，并且可以指定窗口延时
       // WaterMark （水位线，可以让窗口延迟触发的一种机制）
       // 一个窗口中的一个分区的水位线 = 当前窗口当前分区最大的 EventTime - 延迟时间     Time.seconds(0)
        SingleOutputStreamOperator<String> linesWithWaterMark = lines.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<String>(Time.seconds(0)) {


            @Override
            public long extractTimestamp(String s) {
                return Long.parseLong(s.split(",")[0]); // EventTime
            }
        });
       //   提取完 EventTime 后生成 WaterMark ，但数据还是原来的老样子
       // 1000,spark,1 --> spark,1
        SingleOutputStreamOperator <Tuple2<String,Integer>> WordAndCount = linesWithWaterMark.map(new MapFunction<String, Tuple2<String,Integer>>() {

            @Override
            public Tuple2<String,Integer> map(String s) throws Exception {
                String[] split = s.split(",");
                return Tuple2.of(split[1],Integer.parseInt(split[2]));
            }
        });

        //  先 keyBy，再划分窗口
        KeyedStream<Tuple2<String, Integer>, String> keyed = WordAndCount.keyBy(t -> t.f0);

        //   划分窗口
        WindowedStream<Tuple2<String, Integer>, String, TimeWindow> window = keyed.window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)));
         //  对窗口里面的数据进行 sum
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = window.sum(1);
        sum.print();
        env.execute() ;
    }
}
```
### 会话窗口（EventTimeSessionWindows）

<p style="text-indent:2em">相邻两次数据的 EventTime 的时间差超过指定的时间间隔就会触发执行。 如果加入 Watermark， 会在符合窗口触发的情况下进行延迟。 到达延迟水位再进行窗口
触发。</p>

```java
public class EventTimeSessionWindowDemo {
    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironmentWithWebUI(new Configuration());
         //  两秒 调一次方法
        env.getConfig().setAutoWatermarkInterval(1000);
        // 老版本必须要设置时间标准 （1.20 之前的）
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        DataStreamSource<String> lines = env.socketTextStream("linux01", 8888);
       // flink里面的时间都精确到毫秒
       //  时间提取器 BoundedOutOfOrdernessTimestampExtractor : 允许时间乱序，并且可以指定窗口延时
       // WaterMark （水位线，可以让窗口延迟触发的一种机制）
       // 一个窗口中的一个分区的水位线 = 当前窗口当前分区最大的 EventTime - 延迟时间     Time.seconds(0)
        SingleOutputStreamOperator<String> linesWithWaterMark = lines.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<String>(Time.seconds(0)) {


            @Override
            public long extractTimestamp(String s) {
                return Long.parseLong(s.split(",")[0]); // EventTime
            }
        });
       //   提取完 EventTime 后生成 WaterMark ，但数据还是原来的老样子
       // 1000,spark,1 --> spark,1
        SingleOutputStreamOperator <Tuple2<String,Integer>> WordAndCount = linesWithWaterMark.map(new MapFunction<String, Tuple2<String,Integer>>() {

            @Override
            public Tuple2<String,Integer> map(String s) throws Exception {
                String[] split = s.split(",");
                return Tuple2.of(split[1],Integer.parseInt(split[2]));
            }
        });

        //  先 keyBy，再划分窗口
        KeyedStream<Tuple2<String, Integer>, String> keyed = WordAndCount.keyBy(t -> t.f0);

        //   划分窗口
        WindowedStream<Tuple2<String, Integer>, String, TimeWindow> window = keyed.window(EventTimeSessionWindows.withGap(Time.seconds(5)));
         //  对窗口里面的数据进行 sum
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = window.sum(1);

        sum.print();
        env.execute() ;


    }
}
```