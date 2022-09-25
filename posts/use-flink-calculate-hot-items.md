---
title: 'Flink 零基础实战教程：如何计算实时热门商品'
date: 2021-04-06 15:59:14
tags: [flink]
published: true
hideInList: false
feature: /post-images/use-flink-calculate-hot-items.png
isTop: false
---

在上一篇入门教程中，我们已经能够快速构建一个基础的 Flink 程序了。本文会一步步地带领你实现一个更复杂的 Flink 应用程序：实时热门商品。在开始本文前我们建议你先实践一遍上篇文章，因为本文会沿用上文的my-flink-project项目框架。

通过本文你将学到：

    1. 如何基于 EventTime 处理，如何指定 Watermark
    2. 如何使用 Flink 灵活的 Window API
    3. 何时需要用到 State，以及如何使用
    4. 如何使用 ProcessFunction 实现 TopN 功能
## 实战案例介绍
本案例将实现一个“实时热门商品”的需求，我们可以将“实时热门商品”翻译成程序员更好理解的需求：每隔5分钟输出最近一小时内点击量最多的前 N 个商品。将这个需求进行分解我们大概要做这么几件事情：

    * 抽取出业务时间戳，告诉 Flink 框架基于业务时间做窗口
    * 过滤出点击行为数据
    * 按一小时的窗口大小，每5分钟统计一次，做滑动窗口聚合（Sliding Window）
    * 按每个窗口聚合，输出每个窗口中点击量前N名的商品
## 数据准备
这里我们准备了一份淘宝用户行为数据集（来自阿里云天池公开数据集，特别感谢）。本数据集包含了淘宝上某一天随机一百万用户的所有行为（包括点击、购买、加购、收藏）。数据集的组织形式和MovieLens-20M类似，即数据集的每一行表示一条用户行为，由用户ID、商品ID、商品类目ID、行为类型和时间戳组成，并以逗号分隔。关于数据集中每一列的详细描述如下：

|列名称|	说明|
|--|--|
|用户ID|	整数类型，加密后的用户ID|
|商品ID	|整数类型，加密后的商品ID|
|商品类目ID	|整数类型，加密后的商品所属类目ID|
|行为类型	|字符串，枚举类型，包括(‘pv’, ‘buy’, ‘cart’, ‘fav’)|
|时间戳|	行为发生的时间戳，单位秒|
你可以通过下面的命令下载数据集到项目的 resources 目录下：
```sh
$ cd my-flink-project/src/main/resources
$ curl https://raw.githubusercontent.com/wuchong/my-flink-project/master/src/main/resources/UserBehavior.csv > UserBehavior.csv
```
这里是否使用 curl 命令下载数据并不重要，你也可以使用 wget 命令或者直接访问链接下载数据。关键是，将数据文件保存到项目的` resources` 目录下，方便应用程序访问。

## 编写程序
在 `src/main/java/myflink` 下创建 `HotItems.java` 文件：
```java
package myflink;

public class HotItems {

  public static void main(String[] args) throws Exception {
    
  }
}
```
与上文一样，我们会一步步往里面填充代码。第一步仍然是创建一个 `StreamExecutionEnvironment`，我们把它添加到 main 函数中。
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 为了打印到控制台的结果不乱序，我们配置全局的并发为1，这里改变并发对结果正确性没有影响
env.setParallelism(1);
```
## 创建模拟数据源
在数据准备章节，我们已经将测试的数据集下载到本地了。由于是一个csv文件，我们将使用 `CsvInputFormat` 创建模拟数据源。

>注：虽然一个流式应用应该是一个一直运行着的程序，需要消费一个无限数据源。但是在本案例教程中，为了省去构建真实数据源的繁琐，我们使用了文件来模拟真实数据源，这并不影响下文要介绍的知识点。这也是一种本地验证 Flink 应用程序正确性的常用方式。

我们先创建一个 `UserBehavior` 的 POJO 类（所有成员变量声明成public便是POJO类），强类型化后能方便后续的处理。
```java
/** 用户行为数据结构 **/
public static class UserBehavior {
  public long userId;         // 用户ID
  public long itemId;         // 商品ID
  public int categoryId;      // 商品类目ID
  public String behavior;     // 用户行为, 包括("pv", "buy", "cart", "fav")
  public long timestamp;      // 行为发生的时间戳，单位秒
}
```
接下来我们就可以创建一个 `PojoCsvInputFormat` 了， 这是一个读取 csv 文件并将每一行转成指定 POJO
类型（在我们案例中是 UserBehavior）的输入器。
```java
// UserBehavior.csv 的本地文件路径
URL fileUrl = HotItems2.class.getClassLoader().getResource("UserBehavior.csv");
Path filePath = Path.fromLocalFile(new File(fileUrl.toURI()));
// 抽取 UserBehavior 的 TypeInformation，是一个 PojoTypeInfo
PojoTypeInfo<UserBehavior> pojoType = (PojoTypeInfo<UserBehavior>) TypeExtractor.createTypeInfo(UserBehavior.class);
// 由于 Java 反射抽取出的字段顺序是不确定的，需要显式指定下文件中字段的顺序
String[] fieldOrder = new String[]{"userId", "itemId", "categoryId", "behavior", "timestamp"};
// 创建 PojoCsvInputFormat
PojoCsvInputFormat<UserBehavior> csvInput = new PojoCsvInputFormat<>(filePath, pojoType, fieldOrder);
```
下一步我们用 `PojoCsvInputFormat` 创建输入源。
```java
DataStream<UserBehavior> dataSource = env.createInput(csvInput, pojoType);
```
这就创建了一个 `UserBehavior` 类型的 `DataStream`。

## EventTime 与 Watermark
当我们说“统计过去一小时内点击量”，这里的“一小时”是指什么呢？ 在 Flink 中它可以是指 `ProcessingTime` ，也可以是 `EventTime`，由用户决定。

>ProcessingTime：事件被处理的时间。也就是由机器的系统时间来决定。
EventTime：事件发生的时间。一般就是数据本身携带的时间。

在本案例中，我们需要统计业务时间上的每小时的点击量，所以要基于 EventTime 来处理。那么如果让 Flink 按照我们想要的业务时间来处理呢？这里主要有两件事情要做。

第一件是告诉 Flink 我们现在按照 `EventTime` 模式进行处理，Flink 默认使用 `ProcessingTime` 处理，所以我们要显式设置下。
```java
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```
第二件事情是指定如何获得业务时间，以及生成 `Watermark`。`Watermark` 是用来追踪业务事件的概念，可以理解成 `EventTime` 世界中的时钟，用来指示当前处理到什么时刻的数据了。由于我们的数据源的数据已经经过整理，没有乱序，即事件的时间戳是单调递增的，所以可以将每条数据的业务时间就当做 `Watermark`。这里我们用 `AscendingTimestampExtractor` 来实现时间戳的抽取和 `Watermark `的生成。

>注：真实业务场景一般都是存在乱序的，所以一般使用 
```java
BoundedOutOfOrdernessTimestampExtractor。

DataStream<UserBehavior> timedData = dataSource
    .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<UserBehavior>() {
      @Override
      public long extractAscendingTimestamp(UserBehavior userBehavior) {
        // 原始数据单位秒，将其转成毫秒
        return userBehavior.timestamp * 1000;
      }
    });
```
这样我们就得到了一个带有时间标记的数据流了，后面就能做一些窗口的操作。

## 过滤出点击事件
在开始窗口操作之前，先回顾下需求“每隔5分钟输出过去一小时内点击量最多的前 N 个商品”。由于原始数据中存在点击、加购、购买、收藏各种行为的数据，但是我们只需要统计点击量，所以先使用 `FilterFunction` 将点击行为数据过滤出来。
```java
DataStream<UserBehavior> pvData = timedData
    .filter(new FilterFunction<UserBehavior>() {
      @Override
      public boolean filter(UserBehavior userBehavior) throws Exception {
        // 过滤出只有点击的数据
        return userBehavior.behavior.equals("pv");
      }
    });
```
## 窗口统计点击量
由于要每隔5分钟统计一次最近一小时每个商品的点击量，所以窗口大小是一小时，每隔5分钟滑动一次。即分别要统计 [09:00, 10:00), [09:05, 10:05), [09:10, 10:10)… 等窗口的商品点击量。是一个常见的滑动窗口需求（Sliding Window）。
```java
DataStream<ItemViewCount> windowedData = pvData
    .keyBy("itemId")
    .timeWindow(Time.minutes(60), Time.minutes(5))
    .aggregate(new CountAgg(), new WindowResultFunction());
```
我们使用`.keyBy("itemId")`对商品进行分组，使用`.timeWindow(Time size, Time slide)`对每个商品做滑动窗口（1小时窗口，5分钟滑动一次）。然后我们使用` .aggregate(AggregateFunction af, WindowFunction wf) `做增量的聚合操作，它能使用`AggregateFunction`提前聚合掉数据，减少 `state` 的存储压力。较之`.apply(WindowFunction wf)`会将窗口中的数据都存储下来，最后一起计算要高效地多。`aggregate()`方法的第一个参数用于

这里的`CountAgg`实现了`AggregateFunction`接口，功能是统计窗口中的条数，即遇到一条数据就加一。
```java
/** COUNT 统计的聚合函数实现，每出现一条记录加一 */
public static class CountAgg implements AggregateFunction<UserBehavior, Long, Long> {

  @Override
  public Long createAccumulator() {
    return 0L;
  }

  @Override
  public Long add(UserBehavior userBehavior, Long acc) {
    return acc + 1;
  }

  @Override
  public Long getResult(Long acc) {
    return acc;
  }

  @Override
  public Long merge(Long acc1, Long acc2) {
    return acc1 + acc2;
  }
}
```
`.aggregate(AggregateFunction af, WindowFunction wf) `的第二个参数`WindowFunction`将每个 key每个窗口聚合后的结果带上其他信息进行输出。我们这里实现的`WindowResultFunction`将主键商品ID，窗口，点击量封装成了`ItemViewCount`进行输出。
```java
/** 用于输出窗口的结果 */
public static class WindowResultFunction implements WindowFunction<Long, ItemViewCount, Tuple, TimeWindow> {

  @Override
  public void apply(
      Tuple key,  // 窗口的主键，即 itemId
      TimeWindow window,  // 窗口
      Iterable<Long> aggregateResult, // 聚合函数的结果，即 count 值
      Collector<ItemViewCount> collector  // 输出类型为 ItemViewCount
  ) throws Exception {
    Long itemId = ((Tuple1<Long>) key).f0;
    Long count = aggregateResult.iterator().next();
    collector.collect(ItemViewCount.of(itemId, window.getEnd(), count));
  }
}

/** 商品点击量(窗口操作的输出类型) */
public static class ItemViewCount {
  public long itemId;     // 商品ID
  public long windowEnd;  // 窗口结束时间戳
  public long viewCount;  // 商品的点击量

  public static ItemViewCount of(long itemId, long windowEnd, long viewCount) {
    ItemViewCount result = new ItemViewCount();
    result.itemId = itemId;
    result.windowEnd = windowEnd;
    result.viewCount = viewCount;
    return result;
  }
}
```
现在我们得到了每个商品在每个窗口的点击量的数据流。

## TopN 计算最热门商品
为了统计每个窗口下最热门的商品，我们需要再次按窗口进行分组，这里根据`ItemViewCount`中的`windowEnd`进行`keyBy()`操作。然后使用 `ProcessFunction` 实现一个自定义的 TopN 函数 `TopNHotItems` 来计算点击量排名前3名的商品，并将排名结果格式化成字符串，便于后续输出。
```java
DataStream<String> topItems = windowedData
    .keyBy("windowEnd")
    .process(new TopNHotItems(3));  // 求点击量前3名的商品
```
`ProcessFunction` 是 Flink 提供的一个 `low-level API`，用于实现更高级的功能。它主要提供了定时器 `timer` 的功能（支持`EventTime`或`ProcessingTime`）。本案例中我们将利用 `timer` 来判断何时收齐了某个 `window` 下所有商品的点击量数据。由于 `Watermark` 的进度是全局的，

在 `processElement` 方法中，每当收到一条数据`（ItemViewCount）`，我们就注册一个 `windowEnd+1 `的定时器（Flink 框架会自动忽略同一时间的重复注册）。`windowEnd+1 `的定时器被触发时，意味着收到了`windowEnd+1`的 `Watermark`，即收齐了该`windowEnd`下的所有商品窗口统计值。我们在`onTimer()` 中处理将收集的所有商品及点击量进行排序，选出 `TopN`，并将排名信息格式化成字符串后进行输出。

这里我们还使用了 `ListState<ItemViewCount>` 来存储收到的每条 `ItemViewCount` 消息，保证在发生故障时，状态数据的不丢失和一致性。`ListState` 是 Flink 提供的类似 Java List 接口的 State API，它集成了框架的` checkpoint` 机制，自动做到了 `exactly-once` 的语义保证。
```java
/** 求某个窗口中前 N 名的热门点击商品，key 为窗口时间戳，输出为 TopN 的结果字符串 */
public static class TopNHotItems extends KeyedProcessFunction<Tuple, ItemViewCount, String> {

  private final int topSize;

  public TopNHotItems(int topSize) {
    this.topSize = topSize;
  }

  // 用于存储商品与点击数的状态，待收齐同一个窗口的数据后，再触发 TopN 计算
  private ListState<ItemViewCount> itemState;

  @Override
  public void open(Configuration parameters) throws Exception {
    super.open(parameters);
    // 状态的注册
    ListStateDescriptor<ItemViewCount> itemsStateDesc = new ListStateDescriptor<>(
        "itemState-state",
        ItemViewCount.class);
    itemState = getRuntimeContext().getListState(itemsStateDesc);
  }

  @Override
  public void processElement(
      ItemViewCount input,
      Context context,
      Collector<String> collector) throws Exception {

    // 每条数据都保存到状态中
    itemState.add(input);
    // 注册 windowEnd+1 的 EventTime Timer, 当触发时，说明收齐了属于windowEnd窗口的所有商品数据
    context.timerService().registerEventTimeTimer(input.windowEnd + 1);
  }

  @Override
  public void onTimer(
      long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
    // 获取收到的所有商品点击量
    List<ItemViewCount> allItems = new ArrayList<>();
    for (ItemViewCount item : itemState.get()) {
      allItems.add(item);
    }
    // 提前清除状态中的数据，释放空间
    itemState.clear();
    // 按照点击量从大到小排序
    allItems.sort(new Comparator<ItemViewCount>() {
      @Override
      public int compare(ItemViewCount o1, ItemViewCount o2) {
        return (int) (o2.viewCount - o1.viewCount);
      }
    });
    // 将排名信息格式化成 String, 便于打印
    StringBuilder result = new StringBuilder();
    result.append("====================================\n");
    result.append("时间: ").append(new Timestamp(timestamp-1)).append("\n");
    for (int i=0;i<topSize;i++) {
      ItemViewCount currentItem = allItems.get(i);
      // No1:  商品ID=12224  浏览量=2413
      result.append("No").append(i).append(":")
            .append("  商品ID=").append(currentItem.itemId)
            .append("  浏览量=").append(currentItem.viewCount)
            .append("\n");
    }
    result.append("====================================\n\n");

    out.collect(result.toString());
  }
}
```
## 打印输出
最后一步我们将结果打印输出到控制台，并调用env.execute执行任务。
```java
topItems.print();
env.execute("Hot Items Job");
```
运行程序
直接运行 main 函数，就能看到不断输出的每个时间点的热门商品ID。

![](https://tinaxiawuhao.github.io/post-images/1617955791838.png)

## 更换 Kafka 作为数据源
实际生产环境中， 我们的数据流往往是从 Kafka 获取到的。 如果要让代码更贴
近生产实际， 我们只需将 source 更换为 Kafka 即可：
```java
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "consumer-group")
properties.setProperty("key.deserializer",
"org.apache.kafka.common.serialization.StringDeserializer")
properties.setProperty("value.deserializer",
"org.apache.kafka.common.serialization.StringDeserializer")
properties.setProperty("auto.offset.reset", "latest")
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
env.setParallelism(1)
val stream = env
.addSource(new FlinkKafkaConsumer[String]("hotitems", new SimpleStringSchema(),
properties))
```
当然， 根据实际的需要， 我们还可以将 Sink 指定为 Kafka、 ES、 Redis 或其它
存储， 这里就不一一展开实现了

## 总结
本文的完整代码可以通过 GitHub 访问到。本文通过实现一个“实时热门商品”的案例，学习和实践了 Flink 的多个核心概念和 API 用法。包括 `EventTime`、`Watermark `的使用，`State` 的使用，`Window API` 的使用，以及 TopN 的实现。希望本文能加深大家对 Flink 的理解，帮助大家解决实战上遇到的问题。