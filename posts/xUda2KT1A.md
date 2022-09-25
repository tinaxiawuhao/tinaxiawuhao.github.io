---
title: 'Sentinel源码拓展之——限流的各种实现方式'
date: 2022-07-08 20:56:51
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/xUda2KT1A.png
isTop: false
---
Sentinel源码拓展之——限流的各种实现方式

###  一 常见的限流功能实现有以下三种方式 

滑动时间窗口、令牌桶、漏桶，这三种实现方式，有各自擅长的应用场景，而在 Sentinel 中这三种限流实现都有被用到，只不过使用在不同的限流场景下：

- **滑动时间窗口：**普通QPS限流下的快速失败、Warmup预热，使用场景最多；—— Sentinel的实现，是使用“环形时间窗口”来表示无边无际的时间；
- **令牌桶：**热点参数限流，需要为每一个请求参数，创建一个令牌桶；—— Sentinel的实现，实际不是真的创建很多个桶；
- **漏桶（流量整形）：**普通QPS限流下的排队等待，将请求暂存在一个队列中，按时间间隔拉取并处理；—— Sentinel的实现，实际不是真的放入一个队列中；

![](https://tianxiawuhao.github.io/post-images/1658667500749.png)

####  1 时间窗口（滑动时间窗口） 

  固定时间窗口，由于时间窗口粒度太粗，而时间本身其实是没有边界的，所以无法保证任意单位时间窗口中的QPS都不超限（如下图粉红色区域）；

  ![](https://tianxiawuhao.github.io/post-images/1658667533086.png)

为了解决这种问题，可以将我们设置的时间窗口做N等分，划分为更小粒度的窗口，这就是滑动时间窗口：

- 假设滑动时间跨度Internel为1秒，我们可以做N=2等分，那么最小时间窗口其实就是500ms；
- 在限流统计QPS时，窗口范围就是从( currentTime – interval) 之后的第一个时区开始，到 currentTime 所在时间窗口。

![](https://tianxiawuhao.github.io/post-images/1658667562075.png)

随着N的值越大，限流的精度就会越高。

####  2 令牌桶 

- 以固定的速率生成令牌，存在令牌桶中，如果令牌桶满了以后，多余的令牌将会被丢弃；
- 请求进入后，必须先尝试从令牌桶中获取令牌，获取到令牌的请求才会被处理，没有获取到令牌的请求将会被拒绝，或者等待。

![](https://tianxiawuhao.github.io/post-images/1658667591134.png)

####  3 漏桶 

- 将每一个“请求”视作“水滴”，放入“漏桶”中存储；
- “漏桶”以固定速率向外“漏出”请求进行处理，如果“漏桶”空了则代表没有待处理的请求；
- 如果“漏桶”满了，则多余的“请求”将被快速拒绝；

![](https://tianxiawuhao.github.io/post-images/1658667624330.png)

> 以上都是三种实现方案的理论知识，而Sentinel在实际实现时，是做了一定的优化的！

------

###  二 Sentinel中三种限流方案的实现源码剖析之——滑动时间窗口算法 

**滑动时间窗口 —— QPS快速失败 + WarmUp预热：**

Sentinel中的时间窗口，其实使用的是如下图所示的环形时间窗口：

- 因为时间是没有边界的，
- 而且我们一般也只会关注一个Internel之内的时间，如果一个Internel被分成了N个小窗口，那么我们也只会关注N个小的时间窗口；

所以使用环形时间窗口，既能满足使用需求，又能解决内存空间。

![](https://tianxiawuhao.github.io/post-images/1658667653355.png)

通过上一篇关于 ProcessorSlotChain 插槽链的介绍，我们已经清楚了：

- QPS 的统计工作将会由 StatisticSlot 插槽完成；
- 限流判断的逻辑将会由 FlowSlot 插槽来完成；

而所有的 PrcessorSlot 的处理逻辑，都在他们的 entry() 方法中；

####  1 时间窗口请求量统计：StatisticSlot 

```java
public class StatisticSlot extends AbstractLinkedProcessorSlot<DefaultNode> {
 
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        try {
            // 先去执行后面插槽和请求体的任务
            fireEntry(context, resourceWrapper, node, count, prioritized, args);
 
            // 处理完之后才会回来做统计（如果后面失败了，肯定就不用统计了）
            node.increaseThreadNum();  // 线程数统计（服务隔离）
            node.addPassRequest(count); // QPS请求量统计（服务限流）
             
            ......
        } catch (Exception e){
            ......
        }
    }
}

// com.alibaba.csp.sentinel.node.DefaultNode#addPassRequest
public void addPassRequest(int count) {
    super.addPassRequest(count);  // 为DefaultNode做统计
    this.clusterNode.addPassRequest(count); // 为ClusterNode做统计
}
```

- 我们知道DefaultNode 和 ClusterNode 都是 StatisticNode 的子类，所以这里都会调用到StatisticNode的addPassRequest()方法：

```java
public class StatisticNode implements Node {
 
    // 秒级统计（看变量名就知道是一个环形计数器）
    private transient volatile Metric rollingCounterInSecond = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT,
        IntervalProperty.INTERVAL);
 
    // 分钟级统计（看变量名就知道是一个环形计数器）
    private transient Metric rollingCounterInMinute = new ArrayMetric(60, 60 * 1000, false);
 
    // 统计QPS方法
    public void addPassRequest(int count) {
        rollingCounterInSecond.addPass(count); // 秒级统计
        rollingCounterInMinute.addPass(count); // 分钟级统计
    }    
}

// intervalInMs：滑动窗口的时间间隔，Sentinel默认为 1000ms
// sampleCount：时间窗口的分割数量，Sentinel默认为 2
// 所以最小的时间窗口就为 500ms
public class ArrayMetric implements Metric {
 
    private final LeapArray<MetricBucket> data;
 
    public ArrayMetric(int sampleCount, int intervalInMs) {
        this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
    }
}
```

时间线铺平的效果如下图：

![](https://tianxiawuhao.github.io/post-images/1658667679910.png)

- 获取当前请求所在时间窗口，并增加计数

```java
public class ArrayMetric implements Metric {
 
    private final LeapArray<MetricBucket> data;
     
    public ArrayMetric(int sampleCount, int intervalInMs) {
        this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
    }
 
    @Override
    public void addPass(int count) {
        // 获取当前请求所在时间窗口
        WindowWrap<MetricBucket> wrap = data.currentWindow();
         
        // 计数器 + count
        wrap.value().addPass(count);
    }
}
```

- ArrayMetric 又是如何获取当前所在时间窗口的呢？

首先我们得要弄清楚LeapArray保存了哪些数据？

```java
public abstract class LeapArray<T> {
 
    // 小时间窗口的时间长度，默认为 500ms
    protected int windowLengthInMs;
     
    // 一个Interval时间被划分的数量，秒级统计默认为2，分钟级默认为60
    protected int sampleCount;
     
    // 滑动时间窗口时间间隔，默认为 1000ms
    protected int intervalInMs;
     
    // 每一个Interval时间窗口内的小窗口数组，这里就是2
    protected final AtomicReferenceArray<WindowWrap<T>> array;
}
```

再次上图，LeapArray是一个环形数组：

![](https://tianxiawuhao.github.io/post-images/1658667708685.png)

- 我们直接看data.currentWindow() 方法：

```java
// com.alibaba.csp.sentinel.slots.statistic.base.LeapArray#currentWindow(当前时间戳)
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }
    // 计算当前时间对应的数组角标 = (当前时间/500ms)%16
    int idx = calculateTimeIdx(timeMillis);
    // 计算当前时间所在窗口的开始时间 = 当前时间 - 当前时间 % 500ms
    long windowStart = calculateWindowStart(timeMillis);
 
    /*
    * 先根据角标获取数组中保存的 oldWindow 对象，可能是旧数据，需要判断.
    *
    * (1) oldWindow 不存在, 说明是第一次，创建新 window并存入，然后返回即可
    * (2) oldWindow的 starTime = 本次请求的 windowStar, 说明正是要找的窗口，直接返回.
    * (3) oldWindow的 starTime < 本次请求的 windowStar, 说明是旧数据，需要被覆盖，创建 
    *     新窗口，覆盖旧窗口
    */
    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            // 创建新 window
            WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            // 基于CAS写入数组，避免线程安全问题
            if (array.compareAndSet(idx, null, window)) {
                // 写入成功，返回新的 window
                return window;
            } else {
                // 写入失败，说明有并发更新，等待其它人更新完成即可
                Thread.yield();
            }
        } else if (windowStart == old.windowStart()) {
            return old;
        } else if (windowStart > old.windowStart()) {
            if (updateLock.tryLock()) {
                try {
                    // 获取并发锁，覆盖旧窗口并返回
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                // 获取锁失败，等待其它线程处理就可以了
                Thread.yield();
            }
        } else if (windowStart < old.windowStart()) {
            // 这种情况不应该存在，写这里只是以防万一。
            return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}
```

- 获取到时间窗口后，增加计数就比较简单了，使用“自旋 + CAS”的方式完成计数器安全增加即可。

####  2 滑动窗口QPS计算，并判断限流逻辑：FlowSlot 

```java
public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {
 
    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 先做限流判断
        checkFlow(resourceWrapper, context, node, count, prioritized);
 
        // 判断通过后，才会放行到下一个插槽
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }
}
```

FlowSlot的限流判断最终都由`TrafficShapingController`接口中的`canPass`方法来实现。该接口有三个实现类：

- DefaultController：快速失败，默认的方式，基于滑动时间窗口算法；
- WarmUpController：预热模式，基于滑动时间窗口算法，只不过阈值是动态的；
- RateLimiterController：排队等待模式，基于漏桶算法；

这里我们就以DefaultController.canPass() 方法为例：

```java
com.alibaba.csp.sentinel.slots.block.flow.controller.DefaultController#canPass()
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    // 重点：计算目前为止滑动窗口内已经存在的请求量
    int curCount = avgUsedTokens(node);
    // 判断：已使用请求量 + 需要的请求量（1） 是否大于 窗口的请求阈值
    if (curCount + acquireCount > count) {
        // 大于，说明超出阈值，返回false
        if (prioritized && grade == RuleConstant.FLOW_GRADE_QPS) {
            long currentTime;
            long waitInMs;
            currentTime = TimeUtil.currentTimeMillis();
            waitInMs = node.tryOccupyNext(currentTime, acquireCount, count);
            if (waitInMs < OccupyTimeoutProperty.getOccupyTimeout()) {
                node.addWaitingRequest(currentTime + waitInMs, acquireCount);
                node.addOccupiedPass(acquireCount);
                sleep(waitInMs);
 
                // PriorityWaitException indicates that the request will pass after waiting for {@link @waitInMs}.
                throw new PriorityWaitException(waitInMs);
            }
        }
        return false;
    }
    // 小于等于，说明在阈值范围内，返回true
    return true;
}

```

- 很显然，关键点就在于，如何计算滑动窗口内已经用掉的请求量

```java
private int avgUsedTokens(Node node) {
    if (node == null) {
        return DEFAULT_AVG_USED_TOKENS;
    }
    return grade == RuleConstant.FLOW_GRADE_THREAD ? node.curThreadNum() : (int)(node.passQps());
}

// 我们这里肯定是QPS限流
// com.alibaba.csp.sentinel.node.StatisticNode#passQps
public double passQps() {
    // 请求量 ÷ 滑动窗口时间间隔(默认1秒) ，得到的就是QPS
    return rollingCounterInSecond.pass() / rollingCounterInSecond.getWindowIntervalInSec();
}
```

- 那么rollingCounterInSecond.pass() 又是如何计算当前时间窗口内的请求量的呢？

> 以秒级统计为例，其实环形数组的长度只为2，只需要记录当前小时间窗口和前一个小时间窗口即可，一个都不多记录，节约内存！

 ![](https://tianxiawuhao.github.io/post-images/1658667751679.png)

```java
// com.alibaba.csp.sentinel.slots.statistic.metric.ArrayMetric#pass
public long pass() {
    // 获取当前窗口
    data.currentWindow();
    long pass = 0;
    // 获取 当前时间的 滑动窗口范围内 的所有小窗口
    List<MetricBucket> list = data.values();
    // 遍历
    for (MetricBucket window : list) {
        // 累加求和
        pass += window.pass();
    }
    // 返回
    return pass;
}
|
|
// data.values()如何获取 滑动窗口范围内 的所有小窗口的
// com.alibaba.csp.sentinel.slots.statistic.base.LeapArray#values(long)
public List<T> values(long timeMillis) {
    if (timeMillis < 0) {
        return new ArrayList<T>();
    }
    // 创建空集合，大小等于 LeapArray长度（2）
    int size = array.length();
    List<T> result = new ArrayList<T>(size);
    // 遍历 LeapArray
    for (int i = 0; i < size; i++) {
        // 获取每一个小窗口
        WindowWrap<T> windowWrap = array.get(i);
        // 判断这个小窗口是否在 滑动窗口时间范围内（1秒内）
        if (windowWrap == null || isWindowDeprecated(timeMillis, windowWrap)) {
            // 不在范围内，则跳过
            continue;
        }
        // 在范围内，则添加到集合中
        result.add(windowWrap.value());
    }
    // 返回集合
    return result;
}
|
|
// isWindowDeprecated(timeMillis, windowWrap)又是如何判断窗口是否符合要求呢？
public boolean isWindowDeprecated(long time, WindowWrap<T> windowWrap) {
    // 当前时间 - 窗口开始时间  是否大于 滑动窗口的最大间隔（1秒）
    // 也就是说，我们要统计的是 距离当前时间1秒内的 小窗口的 count之和
    return time - windowWrap.windowStart() > intervalInMs;
}
```

到这里，我们就可以理清了：

- StatisticSlot会帮助我们记录每一次的request请求，统计每个小时间窗口内的请求数；

- - 秒级统计的时间窗口环只有 2 格；
  - 分钟级统计的时间窗口环有 60格；

- FlowSlot在需要的时候，会去除当前时间窗口内包含的所有小窗口，然后累加他们的请求量；

- 最后判断是否溢出限流阈值，允许通过，或直接拒绝！

------

###  三 Sentinel中三种限流方案的实现源码剖析之——令牌桶算法 

> “热点参数”限流策略，不适合使用 StatisticSlot 中常规的 “滑动时间窗口算法”，因为StatisticSlot中统计的维度是Node级别；
>
> 很显然，“热点参数”并不适合使用上面的“环形时间窗口算法”来实现；
>
> 相比下来，“令牌桶算法”最适合用在“热点参数限流”场景下，只需要为每个不同的参数值创建一个令牌桶即可。

- **Controller中的方法资源是不可以进行热点参数限流的：通过Sentinel添加的springmvc拦截器实现，创建Entry时候没有传入params参数；**
- **其它的我们通过@SentinelResource添加的资源才艺进行热点参数限流：通过AOP切面编程实现，创建Entry的时候，也将Params一并传入了；**

```java
public class ParamFlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {
 
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args) throws Throwable {
        if (!ParamFlowRuleManager.hasRules(resourceWrapper.getName())) {
            this.fireEntry(context, resourceWrapper, node, count, prioritized, args);
        } else {
            // 校验参数限流
            this.checkFlow(resourceWrapper, count, args);
            // 放行到下一个插槽
            this.fireEntry(context, resourceWrapper, node, count, prioritized, args);
        }
    }
 
    void checkFlow(ResourceWrapper resourceWrapper, int count, Object... args) throws BlockException {
        // args == null 情况有二：1、确实没有参数； 2、Controller方法无法进行参数限流
        if (args != null) {
            if (ParamFlowRuleManager.hasRules(resourceWrapper.getName())) {
                List<ParamFlowRule> rules = ParamFlowRuleManager.getRulesOfResource(resourceWrapper.getName());
                Iterator var5 = rules.iterator();
     
                ParamFlowRule rule;
                 
                // do-while循环，对每一条rule规则做判断
                do {
                    if (!var5.hasNext()) {
                        return;
                    }
     
                    rule = (ParamFlowRule)var5.next();
                    this.applyRealParamIdx(rule, args.length);
                     
                    // 初始化“令牌桶”—— 加“”号，并非真的令牌桶
                    ParameterMetricStorage.initParamMetricsFor(resourceWrapper, rule);
                } while(ParamFlowChecker.passCheck(resourceWrapper, rule, count, args));
     
                String triggeredParam = "";
                if (args.length > rule.getParamIdx()) {
                    Object value = args[rule.getParamIdx()];
                    triggeredParam = String.valueOf(value);
                }
     
                throw new ParamFlowException(resourceWrapper.getName(), triggeredParam, rule);
            }
        }
    }
}
```

####  1ParameterMetricStorage.initParamMetricsFor() 令牌桶初始化方法 

```java
public final class ParameterMetricStorage {
 
    // 以资源名区分不同资源下的令牌桶容器（因为这还不是真正的令牌桶）
    private static final Map<String, ParameterMetric> metricsMap = new ConcurrentHashMap();
    private static final Object LOCK = new Object();
 
    public static void initParamMetricsFor(ResourceWrapper resourceWrapper, ParamFlowRule rule) {
        if (resourceWrapper != null && resourceWrapper.getName() != null) {
            String resourceName = resourceWrapper.getName();
            ParameterMetric metric;
            if ((metric = (ParameterMetric)metricsMap.get(resourceName)) == null) {
                synchronized(LOCK) {
                    // 如果当前资源还没创建过自己的令牌桶容器，那就创建一个
                    if ((metric = (ParameterMetric)metricsMap.get(resourceName)) == null) {
                        metric = new ParameterMetric();
                        metricsMap.put(resourceWrapper.getName(), metric);
                        RecordLog.info("[ParameterMetricStorage] Creating parameter metric for: " + resourceWrapper.getName(), new Object[0]);
                    }
                }
            }
             
            // 对令牌桶容器进行初始化
            metric.initialize(rule);
        }
    }
}
```

####  2 初始化令牌桶容器，真正的令牌桶即将出现 

Sentinel中的令牌桶，其实是维护2个双层Map：

- 容器中剩余令牌数的Map：<ParamFlowRule, <参数值, 属于这个参数值得桶中的剩余可用令牌数>>：此时第二层Map还是空Map；
- 记录最近一次通过的请求时间戳的Map：<ParamFlowRule, <参数值, 最近一次请求的时间戳>>：此时第二层Map还是空Map；

```java
public class ParameterMetric {
    private static final int THREAD_COUNT_MAX_CAPACITY = 4000;
    private static final int BASE_PARAM_MAX_CAPACITY = 4000;
    private static final int TOTAL_MAX_CAPACITY = 200000;
    private final Object lock = new Object();
     
    // 令牌桶实现之一：双层Map：<ParamFlowRule, <参数值, 上次请求的时间戳>>
    private final Map<ParamFlowRule, CacheMap<Object, AtomicLong>> ruleTimeCounters = new HashMap();
    // 令牌桶之二：桶中剩余的令牌数
    private final Map<ParamFlowRule, CacheMap<Object, AtomicLong>> ruleTokenCounter = new HashMap();
    private final Map<Integer, CacheMap<Object, AtomicInteger>> threadCountMap = new HashMap();
     
    // 初始化令牌桶容器
    public void initialize(ParamFlowRule rule) {
        long size;
        if (!this.ruleTimeCounters.containsKey(rule)) {
            synchronized(this.lock) {
                // 初始化ParamRule对应的“最近请求时间戳Map”
                if (this.ruleTimeCounters.get(rule) == null) {
                    size = Math.min(4000L * rule.getDurationInSec(), 200000L);
                    this.ruleTimeCounters.put(rule, new ConcurrentLinkedHashMapWrapper(size));
                }
            }
        }
     
        if (!this.ruleTokenCounter.containsKey(rule)) {
            synchronized(this.lock) {
                // 初始化ParamRule对应的“桶中可用令牌Map”
                if (this.ruleTokenCounter.get(rule) == null) {
                    size = Math.min(4000L * rule.getDurationInSec(), 200000L);
                    this.ruleTokenCounter.put(rule, new ConcurrentLinkedHashMapWrapper(size));
                }
            }
        }
     
        if (!this.threadCountMap.containsKey(rule.getParamIdx())) {
            synchronized(this.lock) {
                if (this.threadCountMap.get(rule.getParamIdx()) == null) {
                    this.threadCountMap.put(rule.getParamIdx(), new ConcurrentLinkedHashMapWrapper(4000L));
                }
            }
        }
     
    }
}
```

> 以上，**只算是将令牌桶容器初始化好了，但是还没开始正式使用！**

####  3 正式使用上面创建的令牌桶容器，一步步调用至“热点参数限流”判断逻辑核心方法 

```java
ParamFlowChecker.passCheck(resourceWrapper, rule, count, args)

com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowChecker#passLocalCheck

com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowChecker#passSingleValueCheck

com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowChecker#passDefaultLocalCheck
```

####  4ParamFlowChecker.passDefaultLocalCheck() 即为“热点限流逻辑”的核心方法 

![](https://tianxiawuhao.github.io/post-images/1658667803098.jpg)

```java
// com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowChecker#passDefaultLocalCheck  
static boolean passDefaultLocalCheck(ResourceWrapper resourceWrapper, ParamFlowRule rule, int acquireCount, Object value) {
    ParameterMetric metric = getParameterMetric(resourceWrapper);
     
    // 根据rule从上面初始化好的令牌桶容器中去除第二层Map
    // <参数值, 上次请求的时间戳>
    // <参数值, 属于这个参数值得桶中的剩余可用令牌数>
    CacheMap<Object, AtomicLong> tokenCounters = metric == null ? null : metric.getRuleTokenCounter(rule);
    CacheMap<Object, AtomicLong> timeCounters = metric == null ? null : metric.getRuleTimeCounter(rule);
     
    if (tokenCounters != null && timeCounters != null) {
        Set<Object> exclusionItems = rule.getParsedHotItems().keySet();
        long tokenCount = (long)rule.getCount();
        if (exclusionItems.contains(value)) {
            tokenCount = (long)(Integer)rule.getParsedHotItems().get(value);
        }
 
        if (tokenCount == 0L) {
            return false;
        } else {
         
            // 允许的最大请求数，也是桶的最大值，也是我们rule中设置的单机阈值
            // 后者是突发阈值，一般不配置，为0
            long maxCount = tokenCount + (long)rule.getBurstCount(); 
            if ((long)acquireCount > maxCount) {
                return false;
            } else {
                while(true) {
                    long currentTime = TimeUtil.currentTimeMillis();
                     
                    // 从timeMap中获取最近一次请求通过的时间戳
                    AtomicLong lastAddTokenTime = (AtomicLong)timeCounters.putIfAbsent(value, new AtomicLong(currentTime));
                    if (lastAddTokenTime == null) { 
                        // 相同参数第一次，直接放行，并更新tokenMap = maxCount - 1
                        tokenCounters.putIfAbsent(value, new AtomicLong(maxCount - (long)acquireCount));
                        return true;
                    }
 
                    // 距离上一次请求通过的时间的差值
                    long passTime = currentTime - lastAddTokenTime.get();
                    AtomicLong oldQps;
                    long restQps;
                     
                    // 距最近一次时间差 > 一次统计窗口
                    if (passTime > rule.getDurationInSec() * 1000L) {
                        oldQps = (AtomicLong)tokenCounters.putIfAbsent(value, new AtomicLong(maxCount - (long)acquireCount));
                        if (oldQps == null) {
                            lastAddTokenTime.set(currentTime);
                            return true;
                        }
 
                        // tokenMap中剩余的令牌数
                        restQps = oldQps.get();
                         
                        // 在距离上次请求的这段时间内，应该补充生成多少新的令牌
                        long toAddCount = passTime * tokenCount / (rule.getDurationInSec() * 1000L);
                         
                        // 上面2者相加，与maxCount允许的最大令牌数对比，取min值，并减去本次需要的令牌数
                        long newQps = toAddCount + restQps > maxCount ? maxCount - (long)acquireCount : restQps + toAddCount - (long)acquireCount;
                        if (newQps < 0L) {
                            return false;
                        }
 
                        // 通过CAS替换原来的tokenMap，并修改最新通过的请求时间戳
                        if (oldQps.compareAndSet(restQps, newQps)) {
                            lastAddTokenTime.set(currentTime);
                            return true;
                        }
 
                        Thread.yield();
                    } else {  // 距最近一次时间差 < 一次统计窗口
                        oldQps = (AtomicLong)tokenCounters.get(value);
                        if (oldQps != null) {
                            restQps = oldQps.get();
                            // tokenMap中剩余令牌不足，直接拒绝
                            if (restQps - (long)acquireCount < 0L) {
                                return false;
                            }
 
                            // 剩余令牌充足，CAS从tokenMap中减去当前需要的令牌数
                            if (oldQps.compareAndSet(restQps, restQps - (long)acquireCount)) {
                                return true;
                            }
                        }
 
                        Thread.yield();
                    }
                }
            }
        }
    } else {
        return true;
    }
}
```

总结：对于热点参数限流：

- Sentinel 会为每个资源，维护两个双层数组：

- - 容器中剩余令牌数的tokenMap：<ParamFlowRule, <参数值, 属于这个参数值得桶中的剩余可用令牌数>>
  - 记录最近一次通过的请求时间戳的timeMap：<ParamFlowRule, <参数值, 最近一次请求的时间戳>>

- 以参数 x=100，限流为5为例：

- - 第一次到达时，肯定不会超限，放行，同时往tokenMap中put入<100, 4>

- 过一会儿，x = 100 的请求再次到达，则判断，本次请求，距离上一次请求时间，是否超过一个时间统计窗口：

- - 在同一个时间窗口，则在前一次的基础上继续减少token值，tokenMap中值变为 <100, 3>
  - 如果不在一个时间窗口，那么计算距离上次请求的这段时间内，应该新生成的token数 + 现在桶中剩余的token数，与maxToken数做对比，取小值就相当于是此时桶中应该有的令牌数，然后减去自己本次需要的令牌数，之后再更新tokenMap；

------

###  四、Sentinel中三种限流方案的实现源码剖析之——漏桶算法 

漏桶算法的核心思想是：将请求放在漏桶中，漏桶会按照固定时间间隔，向外“漏出”请求，进行处理，这样很明显的好处就是“流量整形”，当瞬间流量过大时，也可以先放在漏桶中，慢慢处理！

漏桶算法的入口也是在FlowSlot中，上面有讲过：

FlowSlot的限流判断最终都由`TrafficShapingController`接口中的`canPass`方法来实现。该接口有三个实现类：

- DefaultController：快速失败，默认的方式，基于滑动时间窗口算法；
- WarmUpController：预热模式，基于滑动时间窗口算法，只不过阈值是动态的；
- RateLimiterController：排队等待模式，基于漏桶算法；

**直接进入RateLimiterController.canPass()方法逻辑：**

```java
// com.alibaba.csp.sentinel.slots.block.flow.controller.RateLimiterController#canPass()
 
// 最新一次的请求执行时间（不一定是已经通过的，而是队列中排在最尾端的请求的预期执行时间）
private final AtomicLong latestPassedTime = new AtomicLong(-1);
 
@Override
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    // Pass when acquire count is less or equal than 0.
    if (acquireCount <= 0) {
        return true;
    }
    // 阈值小于等于 0 ，阻止请求，不太可能
    if (count <= 0) {
        return false;
    }
     
    // 获取当前时间
    long currentTime = TimeUtil.currentTimeMillis();
    // 计算两次请求之间允许的最小时间间隔
    // 正常时候acquireCount=1，那么costTime = 200ms
    long costTime = Math.round(1.0 * (acquireCount) / count * 1000);
 
    // 计算本次请求 允许执行的时间点 = 上一次请求的可能执行时间 + 两次请求的最小间隔
    long expectedTime = costTime + latestPassedTime.get();
    // 如果允许执行的时间点小于当前时间，说明可以立即执行
    if (expectedTime <= currentTime) {
        // 更新上一次的请求的执行时间
        latestPassedTime.set(currentTime);
        // 这种情况说明该执行了，立即执行
        return true;
    } else {
        // 不能立即执行，需要计算 预期等待时长
        // 预期等待时长 = 两次请求的最小间隔 + 最近一次请求的可执行时间 - 当前时间
        long waitTime = costTime + latestPassedTime.get() - TimeUtil.currentTimeMillis();
        // 如果预期等待时间超出阈值，则拒绝请求
        if (waitTime > maxQueueingTimeMs) {
            return false;
        } else {
            // 预期等待时间小于阈值，更新最近一次请求的可执行时间，加上costTime
            long oldTime = latestPassedTime.addAndGet(costTime);
            try {
                // 保险起见，再判断一次预期等待时间，是否超过阈值
                waitTime = oldTime - TimeUtil.currentTimeMillis();
                if (waitTime > maxQueueingTimeMs) {
                    // 如果超过，则把刚才 加 的时间再 减回来
                    latestPassedTime.addAndGet(-costTime);
                    // 拒绝
                    return false;
                }
                // in race condition waitTime may <= 0
                if (waitTime > 0) {
                    // 预期等待时间在阈值范围内，休眠要等待的时间，醒来后继续执行
                    Thread.sleep(waitTime);
                }
                return true;
            } catch (InterruptedException e) {
            }
        }
    }
    return false;
}
```

总结：

Sentinel的漏桶算法，不是真的维护了一个队列，而是通过计算各个请求的预计执行时间；

- 如果预计执行时间 > 最大等待时间，那么久不用等了，直接拒绝；
- 如果预计执行时间 < 最大等待时间，那么就等待吧，自己通过 Thread.sleep(waitTime) 实现等待！

![](https://tianxiawuhao.github.io/post-images/1658667818074.png)