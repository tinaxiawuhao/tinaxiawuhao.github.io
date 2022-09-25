---
title: '第六章 Flink 的Window 操作'
date: 2021-04-13 10:30:05
tags: [flink]
published: true
hideInList: false
feature: /post-images/5uA_yv-aO.png
isTop: false
---
Window是无限数据流处理的核心，Window将一个无限的stream拆分成有限大小的”buckets”桶，我们可以在这些桶上做计算操作。本文主要聚焦于在Flink中如何进行窗口操作，以及程序员如何从window提供的功能中获得最大的收益。
　　窗口化的Flink程序的一般结构如下，第一个代码段中是分组的流，而第二段是非分组的流。正如我们所见，唯一的区别是分组的stream调用`keyBy(…)`和`window(…)`，而非分组的stream中`window()`换成了`windowAll(…)`，这些也将贯穿都这一页的其他部分中。

## Keyed Windows
```java
stream.keyBy(...)           <-  keyed versus non-keyed windows
       .window(...)         <-  required: "assigner"
      [.trigger(...)]       <-  optional: "trigger" (else default trigger)
      [.evictor(...)]       <-  optional: "evictor" (else no evictor)
      [.allowedLateness()]  <-  optional, else zero
       .reduce/fold/apply() <-  required: "function"
```
## Non-Keyed Windows
```java
stream.windowAll(...)       <-  required: "assigner"
      [.trigger(...)]       <-  optional: "trigger" (else default trigger)
      [.evictor(...)]       <-  optional: "evictor" (else no evictor)
      [.allowedLateness()]  <-  optional, else zero
       .reduce/fold/apply() <-  required: "function"
```
在上面的例子中，方括号[]内的命令是可选的，这表明Flink允许你根据最符合你的要求来定义自己的window逻辑。

## Window 的生命周期
简单地说,当一个属于`window`的元素到达之后这个`window`就创建了,而当当前时间(事件或者处理时间)为`window`的创建时间跟用户指定的延迟时间相加时,窗口将被彻底清除。Flink 确保了只清除基于时间的`window`,其他类型的`window`不清除,例如:全局`window`。例如:对于一个每5分钟创建无覆盖的(即 翻滚窗口)窗口,允许一个1分钟的时延的窗口策略，Flink将会在12:00到12:05这段时间内第一个元素到达时创建窗口,当水印通过12:06时,移除这个窗口。
　　此外,每个 `Window `都有一个`Trigger`和一个附属于 Window 的函数(例如: `WindowFunction`, `ReduceFunction` 及 `FoldFunction`)，函数里包含了应用于`窗口(Window)`内容的计算，而`Trigger(触发器)`则指定了函数在什么条件下可被应用(函数何时被触发),一个触发策略可以是 "当窗口中的元素个数超过4个时" 或者 "当水印达到窗口的边界时"。触发器还可以决定在窗口创建和删除之间的任意时刻清除窗口的内容,本例中的清除仅指清除窗口的内容而不是窗口的元数据,也就是说新的数据还是可以被添加到当前的`window`中。
　　除了上面的提到之外，你还可以指定一个`驱逐者`, `Evictor`将在触发器触发之后或者在函数被应用之前或者之后，清楚窗口中的元素。
　　接下来我们将更深入的去了解上述的部件，我们从上述片段的主要部分开始(如:`Keyed vs Non-Keyed Windows`, `Window Assigner`, 及 `Window Function`),然后是可选部分。

## 分组和非分组Windows (Keyed vs Non-Keyed Windows)

首先，第一件事是指定你的数据流是分组的还是未分组的，这个必须在定义` window` 之前指定好。使用 `keyBy(...)`会将你的无限数据流拆分成逻辑分组的数据流，如果 `keyBy(...)` 函数不被调用的话，你的数据流将不是分组的。
　　在分组数据流中,任何正在传入的事件的属性都可以被当做key,分组数据流将你的window计算通过多任务并发执行，以为每一个逻辑分组流在执行中与其他的逻辑分组流是独立地进行的。
　　在非分组数据流中，你的原始数据流并不会拆分成多个逻辑流并且所有的window逻辑将在一个任务中执行，并发度为1。

## 窗口分配器(Window Assingers)
指定完你的数据流是分组的还是非分组的之后，接下来你需要定义一个`窗口分配器(window assigner)`，窗口分配器定义了元素如何分配到窗口中，这是通过在分组数据流中调用window(...)或者非分组数据流中调用`windowAll(...)`时你选择的窗口分配器`(WindowAssigner)`来指定的。`WindowAssigner`是负责将每一个到来的元素分配给一个或者多个`窗口(window)`,Flink 提供了一些常用的预定义窗口分配器，即:`滚动窗口`、`滑动窗口`、`会话窗口`和`全局窗口`。你也可以通过继承`WindowAssigner`类来自定义自己的窗口。所有的内置窗口分配器(除了全局窗口 `global window`)都是通过时间来分配元素到窗口中的，这个时间要么是处理的时间，要么是事件发生的时间。请看一下我们的 `event time (https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/event_time.html )`部分来了解更多处理时间和事件时间的区别及`时间戳(timestamp)`和`水印(watermark)`是如何产生的。
　　接下来我们将展示Flink的预定义窗口分配器是如何工作的，以及它们在`DataStream`程序中是如何使用的。接下来我们将展示Flink的预定义窗口分配器是如何工作的，以及它们在`DataStream`程序中是如何使用的。下图中展示了每个分配器是如何工作的，紫色圆圈代表着数据流中的一个元素，这些元素是通过一些key进行分区(在本例中是 user1,user2,user3), X轴显示的是时间进度。

### 滚动窗口
滚动窗口分配器将每个元素分配的一个指定窗口大小的窗口中，滚动窗口有一个固定的大小，并且不会出现重叠。例如:如果你指定了一个5分钟大小的滚动窗口，当前窗口将被评估并将按下图说明每5分钟创建一个新的窗口。
![](https://tinaxiawuhao.github.io/post-images/1618886583930.png)
下面的代码片段展示了如何使用滚动窗口。

Java 代码
```java
DataStream<T> input = ...;
```
滚动事件时间窗口( tumbling event-time windows )
```java
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>); 
```
滚动处理时间窗口(tumbling processing-time windows)
```java
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);
```
每日偏移8小时的滚动事件时间窗口(daily tumbling event-time windows offset by -8 hours. )
```java
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```
Scala 代码:
```java
val input:DataStream[T] =  
```
滚动事件时间窗口(tumbling event-time windows)
```java
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)
```
滚动处理时间窗口(tumbling processing-time windows)
```java
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)
```
每日偏移8小时的滚动事件时间窗口(daily tumbling event-time windows offset by -8 hours. )
```java
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
```
时间间隔可以通过`Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x)`等其中的一个来指定。
在上面最后的例子中，滚动窗口分配器还接受了一个可选的偏移参数，可以用来改变窗口的排列。例如，没有偏移的话按小时的滚动窗口将按时间纪元来对齐，也就是说你将一个如: 1:00:00.000-1:59:59.999,2:00:00.000-2:59:59.999等，如果你想改变一下，你可以指定一个偏移，如果你指定了一个15分钟的偏移，你将得到1:15:00.000-2:14:59.999,2:15:00.000-3:14:59.999等。时间偏移一个很大的用处是用来调准非0时区的窗口，例如:在中国你需要指定一个8小时的时间偏移。

### 滑动窗口(Sliding Windows)
滑动窗口分配器将元素分配到固定长度的窗口中，与滚动窗口类似，窗口的大小由窗口大小参数来配置，另一个窗口滑动参数控制滑动窗口开始的频率。因此，滑动窗口如果滑动参数小于滚动参数的话，窗口是可以重叠的，在这种情况下元素会被分配到多个窗口中。
　　例如，你有10分钟的窗口和5分钟的滑动，那么每个窗口中5分钟的窗口里包含着上个10分钟产生的数据，如下图所示:
![](https://tinaxiawuhao.github.io/post-images/1618886623719.png)
下面的代码片段中展示了如何使用滑动窗口:

Java 代码:
```java
DataStream<T> input = ...;
```
滑动事件时间窗口
```java
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);
```
滑动处理时间窗口
```java
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);
```
偏移8小时的滑动处理时间窗口(sliding processing-time windows offset by -8 hours)
```java
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```
Scala 代码:
```java
val input: DataStream[T] = ...
```
 滑动事件时间窗口(sliding event-time windows)
```java
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)
```
滑动处理时间窗口(sliding processing-time windows)
```java
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)
```
 偏移8小时的滑动处理时间窗口(sliding processing-time windows offset by -8 hours)
```java
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
```
时间间隔可以通过Time.milliseconds(x),Time.seconds(x),Time.minutes(x)等来指定。
　　正如上述例子所示，滑动窗口分配器也有一个可选的偏移参数来改变窗口的对齐。例如，没有偏移参数，按小时的窗口，有30分钟的滑动，将根据时间纪元来对齐，也就是说你将得到如下的窗口1:00:00.000-1:59:59.999,1:30:00.000-2:29:59.999等。而如果你想改变窗口的对齐，你可以给定一个偏移，如果给定一个15分钟的偏移，你将得到如下的窗口:1:15:00.000-2:14.59.999,　1:45:00.000-2:44:59.999等。时间偏移一个很大的用处是用来调准非0时区的窗口，例如:在中国你需要指定一个8小时的时间偏移。

### 会话窗口(Session Windows)
session窗口分配器通过session活动来对元素进行分组，session窗口跟滚动窗口和滑动窗口相比，不会有重叠和固定的开始时间和结束时间的情况。相反，当它在一个固定的时间周期内不再收到元素，即非活动间隔产生，那个这个窗口就会关闭。一个session窗口通过一个session间隔来配置，这个session间隔定义了非活跃周期的长度。当这个非活跃周期产生，那么当前的session将关闭并且后续的元素将被分配到新的session窗口中去。

![](https://tinaxiawuhao.github.io/post-images/1618886669102.png)

下面的代码片段中展示了如何使用session窗口
Java代码:
```java
DataStream<T> input = ...;
```
 事件时间会话窗口(event-time session windows)
```java
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
```
 处理时间会话窗口(processing-time session windows)
```java
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
```
Scala代码:
```java
val input: DataStream[T] = ...
```
 事件时间会话窗口(event-time session windows)
```java
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)
```
处理时间会话窗口(processing-time session windows)
```java
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)
```
时间间隔可以通过`Time.milliseconds(x),Time.seconds(x),Time.minutes(x)`等来指定。
注意: 因为session看窗口没有一个固定的开始和结束，他们的评估与滑动窗口和滚动窗口不同。在内部，session操作为每一个到达的元素创建一个新的窗口，并合并间隔时间小于指定非活动间隔的窗口。为了进行合并，session窗口的操作需要指定一个`合并触发器(Trigger)`和一个`合并窗口函数(Window Function)`,如:`ReduceFunction`或者`WindowFunction`(FoldFunction不能合并)。

###全局窗口(Global Windows)
全局窗口分配器将所有具有相同key的元素分配到同一个全局窗口中，这个窗口模式仅适用于用户还需自定义触发器的情况。否则，由于全局窗口没有一个自然的结尾，无法执行元素的聚合，将不会有计算被执行。
![](https://tinaxiawuhao.github.io/post-images/1618886845541.png)
下面的代码片段展示了如何使用全局窗口:
Java 代码:
```java
DataStream<T> input = ...;
input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);
```
Scala代码:
```java
val input: DataStream[T] = ...
input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>)
```
## 窗口函数(Window Functions)
定义完窗口分配器后，我们还需要为每一个窗口指定我们需要执行的计算，这是窗口的责任，当系统决定一个窗口已经准备好执行之后，这个窗口函数将被用来处理窗口中的每一个元素(可能是分组的)。请参考:`https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/windows.html#triggers` 来了解当一个窗口准备好之后，Flink是如何决定的。
　　window函数可以是`ReduceFunction`, `FoldFunction` 或者 `WindowFunction` 中的一个。前面两个更高效一些(),因为在每个窗口中增量地对每一个到达的元素执行聚合操作。一个 `WindowFunction` 可以获取一个窗口中的所有元素的一个迭代以及哪个元素属于哪个窗口的额外元信息。
　　有`WindowFunction`的窗口化操作会比其他的操作效率要差一些，因为Flink内部在调用函数之前会将窗口中的所有元素都缓存起来。这个可以通过`WindowFunction`和`ReduceFunction`或者`FoldFunction`结合使用来获取窗口中所有元素的增量聚合和`WindowFunction`接收的额外的窗口元数据，接下来我们将看一看每一种变体的示例。

### ReduceFunction
`ReduceFunction`指定了如何通过两个输入的参数进行合并输出一个同类型的参数的过程，Flink使用`ReduceFunction`来对窗口中的元素进行增量聚合。
　　一个`ReduceFunction` 可以通过如下的方式来定义和使用:
Java 代码:
```java
DataStream<Tuple2<String, Long>> input = ...;
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
      }
    });
```
Scala 代码:
```java
val input: DataStream[(String, Long)] = ...
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce { (v1, v2) => (v1._1, v1._2 + v2._2) }
```
上面的例子是将窗口所有元素中元组的第二个属性进行累加操作。

### FoldFunction
`FoldFunction` 指定了一个输入元素如何与一个输出类型的元素合并的过程，这个`FoldFunction` 会被每一个加入到窗口中的元素和当前的输出值增量地调用，第一个元素是与一个预定义的类型为输出类型的初始值合并。
　　一个`FoldFunction`可以通过如下的方式定义和调用:
Java 代码:
```java
DataStream<Tuple2<String, Long>> input = ...;
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
    });
```
Scala 代码:
```java
 val input: DataStream[(String, Long)] = ...
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("") { (acc, v) => acc + v._2 }
```
上面例子追加所有输入的长整型到一个空的字符串中。
注意 fold()不能应用于回话窗口或者其他可合并的窗口中。

## 窗口函数 —— 一般用法(WindowFunction - The Generic Case)
一个`WindowFunction`将获得一个包含了`window`中的所有`元素迭代(Iterable)`，并且提供所有窗口函数的最大灵活性。这些带来了性能的成本和资源的消耗，因为`window`中的元素无法进行增量迭代，而是缓存起来直到`window`被认为是可以处理时为止。
`WindowFunction`的使用说明如下:
Java 代码:
```java
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {
  /**
   // Evaluates the window and outputs none or several elements.
   // @param key The key for which this window is evaluated.
  // @param window The window that is being evaluated.
   // @param input The elements in the window being evaluated.
   // @param out A collector for emitting elements.
  // @throws Exception The function may throw exceptions to fail the program and trigger recovery.
  */
  void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
```
Scala 代码:
```java
trait WindowFunction[IN, OUT, KEY, W <: Window] extends Function with Serializable {
  /**
    // Evaluates the window and outputs none or several elements.
    //
    // @param key    The key for which this window is evaluated.
    // @param window The window that is being evaluated.
    // @param input  The elements in the window being evaluated.
    // @param out    A collector for emitting elements.
    // @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def apply(key: KEY, window: W, input: Iterable[IN], out: Collector[OUT])
}
```
一个`WindowFunction`可以按如下方式来定义和使用:
Java 代码:
```java
DataStream<Tuple2<String, Long>> input = ...;
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction());
/* ... */
public class MyWindowFunction implements WindowFunction<Tuple<String, Long>, String, String, TimeWindow> {
  void apply(String key, TimeWindow window, Iterable<Tuple<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + window + "count: " + count);
  }
}
```
Scala 代码:
```java
val input: DataStream[(String, Long)] = ...
input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction())
/* ... */
class MyWindowFunction extends WindowFunction[(String, Long), String, String, TimeWindow] {
  def apply(key: String, window: TimeWindow, input: Iterable[(String, Long)], out: Collector[String]): () = {
    var count = 0L
    for (in <- input) {
      count = count + 1
    }
    out.collect(s"Window $window count: $count")
  }
}
```
上面的例子展示了统计一个`window`中元素个数的`WindowFunction`，此外，还将`window`的信息添加到输出中。
注意:使用`WindowFunction`来做简单的聚合操作如计数操作，性能是相当差的。下一章节我们将展示如何将`ReduceFunction`跟`WindowFunction`结合起来，来获取增量聚合和添加到`WindowFunction`中的信息。

### ProcessWindowFunction
在使用`WindowFunction`的地方你也可以用`ProcessWindowFunction`，这跟`WindowFunction`很类似，除了接口允许查询跟多关于`context`的信息，`context`是`window`评估发生的地方。
下面是`ProcessWindowFunction`的接口:
Java 代码:
```java
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {
    /**
     // Evaluates the window and outputs none or several elements.
     //
     // @param key The key for which this window is evaluated.
     // @param context The context in which the window is being evaluated.
     // @param elements The elements in the window being evaluated.
     // @param out A collector for emitting elements.
     //
     // @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;
    /**
     // The context holding window metadata
     */
    public abstract class Context {
        /**
         // @return The window that is being evaluated.
         */
        public abstract W window();
    }
}
```
Scala 代码:
```java
abstract class ProcessWindowFunction[IN, OUT, KEY, W <: Window] extends Function {
  /**
    // Evaluates the window and outputs none or several elements.
    //
    // @param key      The key for which this window is evaluated.
    // @param context  The context in which the window is being evaluated.
    // @param elements The elements in the window being evaluated.
    // @param out      A collector for emitting elements.
    // @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  @throws[Exception]
  def process(
      key: KEY,
      context: Context,
      elements: Iterable[IN],
      out: Collector[OUT])
  /**
    // The context holding window metadata
    */
  abstract class Context {
    /**
      // @return The window that is being evaluated.
      */
    def window: W
  }
}
```
`ProcessWindowFunction`可以通过如下方式调用:
Java 代码:
```java
DataStream<Tuple2<String, Long>> input = ...;
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .process(new MyProcessWindowFunction());`
Scala 代码:
`val input: DataStream[(String, Long)] = ...
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .process(new MyProcessWindowFunction())
```
有增量聚合功能的`WindowFunction (WindowFunction with Incremental Aggregation)`
`WindowFunction`可以跟`ReduceFunction`或者`FoldFunction`结合来增量地对到达`window`中的元素进行聚合，当`window`关闭之后，`WindowFunction`就能提供聚合结果。当获取到`WindowFunction`额外的`window`元信息后就可以进行增量计算窗口了。
标注:你也可以使用`ProcessWindowFunction`替换`WindowFunction`来进行增量窗口聚合。

使用`FoldFunction` 进行`增量窗口聚合(Incremental Window Aggregation with FoldFunction)`
下面的例子展示了一个增量的`FoldFunction`如何跟一个`WindowFunction`结合，来获取窗口的事件数，并同时返回窗口的key和窗口的最后时间。
Java 代码:
```java
DataStream<SensorReading> input = ...;
input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .fold(new Tuple3<String, Long, Integer>("",0L, 0), new MyFoldFunction(), new MyWindowFunction())
// Function definitions
private static class MyFoldFunction
    implements FoldFunction<SensorReading, Tuple3<String, Long, Integer> > {
  public Tuple3<String, Long, Integer> fold(Tuple3<String, Long, Integer> acc, SensorReading s) {
      Integer cur = acc.getField(2);
      acc.setField(2, cur + 1);
      return acc;
  }
}
private static class MyWindowFunction
    implements WindowFunction<Tuple3<String, Long, Integer>, Tuple3<String, Long, Integer>, String, TimeWindow> {
  public void apply(String key,
                    TimeWindow window,
                    Iterable<Tuple3<String, Long, Integer>> counts,
                    Collector<Tuple3<String, Long, Integer>> out) {
    Integer count = counts.iterator().next().getField(2);
    out.collect(new Tuple3<String, Long, Integer>(key, window.getEnd(),count));
  }
}
```
Scala 代码:
```java
val input: DataStream[SensorReading] = ...
 input
 .keyBy(<key selector>)
 .timeWindow(<window assigner>)
 .fold (
    ("", 0L, 0),
    (acc: (String, Long, Int), r: SensorReading) => { ("", 0L, acc._3 + 1) },
    ( key: String,
      window: TimeWindow,
      counts: Iterable[(String, Long, Int)],
      out: Collector[(String, Long, Int)] ) =>
      {
        val count = counts.iterator.next()
        out.collect((key, window.getEnd, count._3))
      }
  )
```
使用`ReduceFunction`进行`增量窗口聚合(Incremental Window Aggregation with ReduceFunction)`
下面例子展示了一个增量额`ReduceFunction`如何跟一个`WindowFunction`结合，来获取窗口中最小的事件和窗口的开始时间。
Java 代码:
```java
DataStream<SensorReading> input = ...;
input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .reduce(new MyReduceFunction(), new MyWindowFunction());
// Function definitions
private static class MyReduceFunction implements ReduceFunction<SensorReading> {
  public SensorReading reduce(SensorReading r1, SensorReading r2) {
      return r1.value() > r2.value() ? r2 : r1;
  }
}
private static class MyWindowFunction
    implements WindowFunction<SensorReading, Tuple2<Long, SensorReading>, String, TimeWindow> {
  public void apply(String key,
                    TimeWindow window,
                    Iterable<SensorReading> minReadings,
                    Collector<Tuple2<Long, SensorReading>> out) {
      SensorReading min = minReadings.iterator().next();
      out.collect(new Tuple2<Long, SensorReading>(window.getStart(), min));
  }
}
```
Scala 代码:
```java
val input: DataStream[SensorReading] = ...
 input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .reduce(
    (r1: SensorReading, r2: SensorReading) => { if (r1.value > r2.value) r2 else r1 },
    ( key: String,
      window: TimeWindow,
      minReadings: Iterable[SensorReading],
      out: Collector[(Long, SensorReading)] ) =>
      {
        val min = minReadings.iterator.next()
        out.collect((window.getStart, min))
      }
  )
  ```
## 触发器(Triggers)
触发器决定了一个窗口何时可以被窗口函数处理，每一个窗口分配器都有一个默认的触发器，如果默认的触发器不能满足你的需要，你可以通过调用`trigger(...)`来指定一个自定义的触发器。触发器的接口有5个方法来允许触发器处理不同的事件:
　　* `onElement()`方法,每个元素被添加到窗口时调用
　　* `onEventTime()`方法,当一个已注册的事件时间计时器启动时调用
　　* `onProcessingTime()`方法,当一个已注册的处理时间计时器启动时调用
　　* `onMerge()`方法，与状态性触发器相关，当使用会话窗口时，两个触发器对应的窗口合并时，合并两个触发器的状态。
　　* 最后一个`clear()`方法执行任何需要清除的相应窗口
上面的方法中有两个需要注意的地方:
1)第一、三通过返回一个TriggerResult来决定如何操作调用他们的事件，这些操作可以是下面操作中的一个；
`CONTINUE`:什么也不做
`FIRE`:触发计算
`PURGE`:清除窗口中的数据
`FIRE_AND_PURGE`:触发计算并清除窗口中的数据
2)这些函数可以被用来为后续的操作注册处理时间定时器或者事件时间计时器

### 触发和清除(Fire and Purge)
一旦一个触发器决定一个窗口已经准备好进行处理，它将触发并返回`FIRE`或者`FIRE_AND_PURGE`。这是窗口操作发送当前窗口结果的信号，给定一个拥有一个`WindowFunction`的窗口那么所有的元素都将发送到`WindowFunction`中(可能之后还会发送到驱逐器(Evitor)中)。有`ReduceFunction`或者`FoldFunction`的`Window`仅仅发送他们的急切聚合结果。
　　当一个触发器触发时，它可以是FIRE或者FIRE_AND_PURGE，如果是FIRE的话，将保持window中的内容，FIRE_AND_PURGE的话，会清除window的内容。默认情况下，预实现的触发器仅仅是FIRE，不会清除window的状态。
注意:清除操作仅清除window的内容，并留下潜在的窗口元信息和完整的触发器状态。

### 窗口分配器默认的触发器(Default Triggers of WindowAssigners)
默认的触发器适用于许多种情况，例如:所有的事件时间分配器都有一个`EventTimeTrigger`作为默认的触发器，这个触发器仅在当水印通过窗口的最后时间时触发。
注意:`GlobalWindow`默认的触发器是`NeverTrigger`，是永远不会触发的，因此，如果你使用的是`GlobalWindow`的话，你需要定义一个自定义触发器。
注意:通过调用`trigger(...)`来指定一个触发器你就重写了`WindowAssigner`的默认触发器。例如:如果你为`TumblingEventTimeWindows`指定了一个`CountTrigger`，你就不会再通过时间来获取触发了，而是通过计数。现在，如果你想通过时间和计数来触发的话，你需要写你自己自定义的触发器。

### 内置的和自定义的触发器(Build-in and Custom Triggers)
Flink有一些内置的触发器:
　　*EventTimeTrigger(前面提到过)触发是根据由水印衡量的事件时间的进度来的
　　*ProcessingTimeTrigger 根据处理时间来触发
　　*CountTrigger 一旦窗口中的元素个数超出了给定的限制就会触发
　　*PurgingTrigger 作为另一个触发器的参数并将它转换成一个清除类型
如果你想实现一个自定义的触发器，你需要查看一下这个抽象类Trigger(https://github.com/apache/flink/blob/master//flink-streaming-java/src/main/java/org/apache/flink/streaming/api/windowing/triggers/Trigger.java ),请注意，这个API还在优化中，后续的Flink版本可能会改变。

## 驱逐器(Evictors)
Flink的窗口模型允许指定一个除了`WindowAssigner`和`Trigger`之外的可选参数`Evitor`，这个可以通过调用`evitor(...)`方法(在这篇文档的开头展示过)来实现。这个驱逐器(evitor)可以在触发器触发之前或者之后，或者窗口函数被应用之前清理窗口中的元素。为了达到这个目的，Evitor接口有两个方法:
```java
/**
 // Optionally evicts elements. Called before windowing function.
 //
 // @param elements The elements currently in the pane.
 // @param size The current number of elements in the pane.
 // @param window The {@link Window}
 // @param evictorContext The context for the Evictor
 ///
void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
/**
 // Optionally evicts elements. Called after windowing function.
 //
 // @param elements The elements currently in the pane.
 // @param size The current number of elements in the pane.
 // @param window The {@link Window}
 // @param evictorContext The context for the Evictor
 */
void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
```
`evitorBefore()`方法包含了在`window function`之前被应用的驱逐逻辑，而`evitorAfter()`方法包含了在`window function`之后被应用的驱逐逻辑。在`window function`应用之前被驱逐的元素将不会再被`window function`处理。
Flink有三个预实现的驱逐器，他们是:
　　`CountEvitor`：在窗口中保持一个用户指定数量的元素，并在窗口的开始处丢弃剩余的其他元素
　　`DeltaEvitor`: 通过一个`DeltaFunction`和一个阈值，计算窗口缓存中最近的一个元素和剩余的所有元素的`delta`值，并清除`delta`值大于或者等于阈值的元素
　　`TimeEvitor`:使用一个`interval`的毫秒数作为参数，对于一个给定的窗口，它会找出元素中的最大时间戳`max_ts`，并清除时间戳小于`max_tx - interval`的元素。
默认情况下:所有预实现的`evitor`都是在`window function`前应用它们的逻辑
注意:指定一个`Evitor`要防止预聚合，因为窗口中的所有元素必须得在计算之前传递到驱逐器中
注意:Flink 并不保证窗口中的元素是有序的，所以驱逐器可能从窗口的开始处清除，元素到达的先后不是那么必要。

### 允许延迟(Allowed Lateness)
当处理事件时间的`window`时，可能会出现元素到达晚了，Flink用来与事件时间联系的水印已经过了元素所属的窗口的最后时间。可以查看事件时间(event time https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/event_time.html )尤其是晚到元素(late elements https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/event_time.html#late-elements )来了解Flink如何处理事件时间的讨论。
　　默认情况下，当水印已经过了窗口的最后时间时晚到的元素会被丢弃。然而，Flink允许为窗口操作指定一个最大允许时延，允许时延指定了元素可以晚到多长时间，默认情况下是0。水印已经过了窗口最后时间后才来的元素，如果还未到窗口最后时间加时延时间，那么元素任然添加到窗口中。如果依赖触发器的使用的话，晚到但是未丢弃的元素可能会导致窗口再次被触发。
　　为了达到这个目的，Flink将保持窗口的状态直到允许时延的发生，一旦发生，Flink将清除`Window`，删除`window`的状态，如`Window` 生命周期章节中所描述的那样。
默认情况下，允许时延为0，也就是说水印之后到达的元素将被丢弃。
你可以按如下方式来指定一个允许时延：
Java 代码:
```java
 DataStream<T> input = ...;
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
```
Scala 代码:
```java
 val input: DataStream[T] = ...
 input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>)
```
注意:当使用`GlobalWindows`分配器时，没有数据会被认为是延迟的，因为`Global Window`的最后时间是`Long.MAX_VALUE`。

###以侧输出来获取延迟数据(Getting Late Data as a Site Output)
使用Flink的侧输出(https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/side_output.html )特性，你可以获得一个已经被丢弃的延迟数据流。
　　首先你需要在窗口化的数据流中调用`sideOutputLateData(OutputTag)`指定你需要获取延迟数据，然后，你就可以在`window` 操作的结果中获取到侧输出流了。
代码如下：
Java 代码：
```java
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};
DataStream<T> input = ...;
DataStream<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);
DataStream<T> lateStream = result.getSideOutput(lateOutputTag);
```
Scala代码：
```java
val lateOutputTag = OutputTag[T]("late-data")
val input: DataStream[T] = ...
val result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>)
val lateStream = result.getSideOutput(lateOutputTag)
```
### 延迟元素考虑(Late elements considerations)
当指定一个允许延迟大于0时，`window`以及`window`中的内容将会继续保持即使水印已经达到了`window`的最后时间。在这种情况下，当一个延迟事件到来而未丢弃时，它可能会触发`window`中的其他触发器。这些触发叫做`late firings`，因为它们是由延迟事件触发的，并相对于`window`中第一个触发即主触发而言。对于`session window`而言，`late firing`还会进一步导致`window`的合并，因为它们桥接了两个之前存在差距，而未合并的`window`。

### 有用状态大小的考虑(Useful state size considerations)
`window` 可以定义一个很长的周期(例如：一天、一周或者一月)，因此积累了相当大的状态。这里有些规则，当估计你的窗口计算的存储要求时，需要记住。
　　1. Flink会在每个窗口中为每个属于它的元素创建一份备份，鉴于此，滚动窗口保存了每个元素的一个备份，与此相反，滑动窗口会为每个元素创建几个备份，如`Window Assigner`章节所述。因此，一个窗口大小为1天，滑动大小为1秒的滑动窗口可能就不是个好的策略了。
　　2. `FoldFunction`和`ReduceFunction`可以制定reduce的存储需求，因为它们预聚合元素并且每个窗口只保存一个值。相反，只有`WindowFunction`需要累积所有的元素。
　　3. 使用`Evitor`需要避免任何预聚合操作，因为窗口中的所有元素都需要在应用于计算之前传递到`evitor`中