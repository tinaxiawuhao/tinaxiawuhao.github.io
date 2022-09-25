---
title: '附录 kafka常见面试问题汇总'
date: 2021-04-22 13:38:23
tags: [kafka]
published: true
hideInList: false
feature: /post-images/aFuh1RRNc.png
isTop: false
---
## 基础题目

### 1、Apache Kafka 是什么?

Apach Kafka 是一款分布式流处理框架，用于实时构建流处理应用。它有一个核心 的功能广为人知，即作为企业级的消息引擎被广泛使用。

你一定要先明确它的流处理框架地位，这样能给面试官留 下一个很专业的印象。

### 2、什么是消费者组?

消费者组是 Kafka 独有的概念，如果面试官问这 个，就说明他对此是有一定了解的。我先给出标准答案：
1、定义：即消费者组是 Kafka 提供的可扩展且具有容错性的消费者机制。
2、原理：在 Kafka 中，消费者组是一个由多个消费者实例 构成的组。多个实例共同订阅若干个主题，实现共同消费。同一个组下的每个实例都配置有 相同的组 ID，被分配不同的订阅分区。当某个实例挂掉的时候，其他实例会自动地承担起 它负责消费的分区。

此时，又有一个小技巧给到你:消费者组的题目，能够帮你在某种程度上掌控下面的面试方向。

#### 2.1消费者组的位移提交机制
broker维护消费者的消费位移信息，老的版本存储在zk上，新版本存储在内部的topic里。本质上，位移信息消费者自己维护也可以，但是如果消费者挂了或者重启，对于某一个分区的消费位移不就丢失了吗？所以，还是需要提交到broker端做持久化的。

#### 2.2消费者组与 Broker 之间的交互

1. Kafka中消费者的消费方式

   consumer采用pull(拉)模式从broker中读取数据。 拉取模式也有不足，如果kafka没有数据，消费者可能会陷入循环中，一直返回空数据。针对这一点，kafka消费者在消费数据时会传入一个时长参数timeout，如果当前没有数据可供消费，consumer会等待一段时间后再返回，这段时长即为timeout

2. Kafka的分区分配策略

   一个消费者组中有多个消费者，一个broker有多个分区，所有必然会涉及到分区分配问题，即确定哪一个分区由哪一个consumer来消费。kafka有两种分区分配策略：RoundRobin和Range
   
    1） RoundRobin
   
   按照消费者组划分，将消费者组作为一个整体，要求整个消费者组内的消费者订阅相同的主题，否则会导致错误的分配的问题。
   
   2） Range （默认）
   
   按照单个主题划分，可能导致消费者消费分区个数不对等的问题；


   **offset的维护**

     由于kafka可能出现故障，故障之后要恢复到上一次消费的位置，往下继续进行消费。因此consumer需要实时记录自己消费到了哪一个offset，从0.9版本开始，consumer默认将offset保存在kafka的内置topic中，该topic为_consumer_offsets（0.9版本前放在zookeeper中） 

### 3、在 Kafka 中，ZooKeeper 的作用是什么?

目前，Kafka 使用 ZooKeeper 存放集群元数据、成员管理、Controller 选举，以及其他一些管理类任务。之后，等 KIP-500 提案完成后，Kafka 将完全不再依赖 于 ZooKeeper。

记住，**一定要突出“目前”**，以彰显你非常了解社区的演进计划。“存放元数据”是指主题 分区的所有数据都保存在 ZooKeeper 中，且以它保存的数据为权威，其他“人”都要与它 保持对齐。“成员管理”是指 Broker 节点的注册、注销以及属性变更，等 等。“Controller 选举”是指选举集群 Controller，而其他管理类任务包括但不限于主题 删除、参数配置等。

不过，抛出 KIP-500 也可能是个双刃剑。碰到非常资深的面试官，他可能会进一步追问你 KIP-500 是怎么做的。一言以蔽之:**KIP-500 思想，是使用社区自研的基于 Raft 的共识算法， 替代 ZooKeeper，实现 Controller 自选举**。

### 4、解释下 Kafka 中位移(offset)的作用

在 Kafka 中，每个 主题分区下的每条消息都被赋予了一个唯一的 ID 数值，用于标识它在分区中的位置。这个 ID 数值，就被称为位移，或者叫偏移量。一旦消息被写入到分区日志，它的位移值将不能 被修改。

答完这些之后，你还可以把整个面试方向转移到你希望的地方。常见方法有以下 3 种:

1. 如果你深谙 Broker 底层日志写入的逻辑，可以强调下消息在日志中的存放格式;
2. 如果你明白位移值一旦被确定不能修改，可以强调下“Log Cleaner 组件都不能影响位 移值”这件事情;
3. 如果你对消费者的概念还算熟悉，可以再详细说说位移值和消费者位移值之间的区别。

### 5、Partition的Leader Replica和Follower Replica的区别

这道题表面上是考核你对 Leader 和 Follower 区别的理解，但很容易引申到 Kafka 的同步 机制上。因此，我建议你主动出击，一次性地把隐含的考点也答出来，也许能够暂时把面试 官“唬住”，并体现你的专业性。

你可以这么回答:**Kafka 副本当前分为领导者副本和追随者副本。只有 Leader 副本才能 对外提供读写服务，响应 Clients 端的请求。Follower 副本只是采用拉(PULL)的方 式，被动地同步 Leader 副本中的数据，并且在 Leader 副本所在的 Broker 宕机后，随时 准备应聘 Leader 副本。**

通常来说，回答到这个程度，其实才只说了 60%，因此，我建议你再回答两个额外的加分 项。

- **强调 Follower 副本也能对外提供读服务**。自 Kafka 2.4 版本开始，社区通过引入新的 Broker 端参数，允许 Follower 副本有限度地提供读服务。
- **强调 Leader 和 Follower 的消息序列在实际场景中不一致**。很多原因都可能造成 Leader 和 Follower 保存的消息序列不一致，比如程序 Bug、网络问题等。这是很严重 的错误，必须要完全规避。你可以补充下，之前确保一致性的主要手段是高水位机制， 但高水位值无法保证 Leader 连续变更场景下的数据一致性，因此，社区引入了 Leader Epoch 机制，来修复高水位值的弊端。关于“Leader Epoch 机制”，国内的资料不是 很多，它的普及度远不如高水位，不妨大胆地把这个概念秀出来，力求惊艳一把。

## 炫技式题目

### 6、LEO、LSO、AR、ISR、HW 都表示什么含义?

- **LEO**:Log End Offset。日志末端位移值或末端偏移量，表示日志下一条待插入消息的 位移值。举个例子，如果日志有 10 条消息，位移值从 0 开始，那么，第 10 条消息的位 移值就是 9。此时，LEO = 10。
- **LSO**:Log Stable Offset。这是 Kafka 事务的概念。如果你没有使用到事务，那么这个 值不存在(其实也不是不存在，只是设置成一个无意义的值)。该值控制了事务型消费 者能够看到的消息范围。它经常与 Log Start Offset，即日志起始位移值相混淆，因为 有些人将后者缩写成 LSO，这是不对的。在 Kafka 中，LSO 就是指代 Log Stable Offset。
- **AR**:Assigned Replicas。AR 是主题被创建后，分区创建时被分配的副本集合，副本个 数由副本因子决定。
- **ISR**:In-Sync Replicas。Kafka 中特别重要的概念，指代的是 AR 中那些与 Leader 保 持同步的副本集合。在 AR 中的副本可能不在 ISR 中，但 Leader 副本天然就包含在 ISR 中。关于 ISR，**还有一个常见的面试题目是如何判断副本是否应该属于 ISR**。目前的判断 依据是:**Follower 副本的 LEO 落后 Leader LEO 的时间，是否超过了 Broker 端参数 replica.lag.time.max.ms 值**。如果超过了，副本就会被从 ISR 中移除。
- **HW**:高水位值(High watermark)。这是控制消费者可读取消息范围的重要字段。一 个普通消费者只能“看到”Leader 副本上介于 Log Start Offset 和 HW(不含)之间的 所有消息。水位以上的消息是对消费者不可见的。关于 HW，问法有很多，我能想到的 最高级的问法，就是让你完整地梳理下 Follower 副本拉取 Leader 副本、执行同步机制 的详细步骤。这就是我们的第 20 道题的题目，一会儿我会给出答案和解析。

### 7，Kafka 的 ISR 机制是什么?

这个机制简单来说，就是会自动给每个 Partition 维护一个 ISR 列表，这个列表里一定会有 Leader，然后还会包含跟 Leader 保持同步的 Follower。

也就是说，只要 Leader 的某个 Follower 一直跟他保持数据同步，那么就会存在于 ISR 列表里。

但是如果 Follower 因为自身发生一些问题，导致不能及时的从 Leader 同步数据过去，那么这个 Follower 就会被认为是“out-of-sync”，被从 ISR 列表里踢出去。

所以大家先得明白这个 ISR 是什么，说白了，就是 Kafka 自动维护和监控哪些 Follower 及时的跟上了 Leader 的数据同步。

### 8，Kafka 写入的数据如何保证不丢失?

所以如果要让写入 Kafka 的数据不丢失，你需要保证如下几点：

- 每个 Partition 都至少得有 1 个 Follower 在 ISR 列表里，跟上了 Leader 的数据同步。
- 每次写入数据的时候，都要求至少写入 Partition Leader 成功，同时还有至少一个 ISR 里的 Follower 也写入成功，才算这个写入是成功了。
- 如果不满足上述两个条件，那就一直写入失败，让生产系统不停的尝试重试，直到满足上述两个条件，然后才能认为写入成功。
- 按照上述思路去配置相应的参数，才能保证写入 Kafka 的数据不会丢失。



![](https://tinaxiawuhao.github.io/post-images/1622096799684.png)

如上图所示，假如现在 Leader 没有 Follower 了，或者是刚写入 Leader，Leader 立马就宕机，还没来得及同步给 Follower。

在这种情况下，写入就会失败，然后你就让生产者不停的重试，直到 Kafka 恢复正常满足上述条件，才能继续写入。这样就可以让写入 Kafka 的数据不丢失。

### 9. 高水位和Epoch

在Kafka中，高水位的作用主要有两个

- 定义消息可见性，即用来标识分区下的哪些消息是可以被消费者消费的。
- 帮助Kafka完成副本同步

下面这张图展示了多个与高水位相关的 Kafka 术语。

![](https://tinaxiawuhao.github.io/post-images/1622096512111.png)

假设这是某个分区 Leader 副本的高水位图。首先，请注意图中的“已提交消息”和“未提交消息”。之前在讲到 Kafka 持久性保障的时候，特意对两者进行了区分。现在，再次强调一下。在分区高水位以下的消息被认为是已提交消息，反之就是未提交消息。

消费者只能消费已提交消息，即图中位移小于 8 的所有消息。注意，这里我们不讨论 Kafka 事务，因为事务机制会影响消费者所能看到的消息的范围，它不只是简单依赖高水位来判断。它依靠一个名为 LSO（Log Stable Offset）的位移值来判断事务型消费者的可见性。

另外，**位移值等于高水位的消息也属于未提交消息。也就是说，高水位上的消息是不能被消费者消费的**。

#### Log End Offset（LEO）

Log End Offset（LEO）表示副本写入下一条消息的位移值。注意，数字 15 所在的方框是虚线，这就说明，这个副本当前只有 15 条消息，位移值是从 0 到 14，下一条新消息的位移是 15。显然，介于高水位和 LEO 之间的消息就属于未提交消息。这也从侧面告诉了我们一个重要的事实，那就是：**同一个副本对象，其高水位值不会大于 LEO 值**。

**高水位和 LEO 是副本对象的两个重要属性**。Kafka 所有副本都有对应的高水位和 LEO 值，而不仅仅是 Leader 副本。只不过 Leader 副本比较特殊，Kafka 使用 Leader 副本的高水位来定义所在分区的高水位。换句话说，**分区的高水位就是其 Leader 副本的高水位**。

#### 高水位和LEO更新机制

现在，我们知道了每个副本对象都保存了一组高水位值和 LEO 值，但实际上，在 Leader 副本所在的 Broker 上，还保存了其他 Follower 副本的HW和LEO 值。

![](https://tinaxiawuhao.github.io/post-images/1622096529006.png)

如上图所示，Broker 0 上保存了某分区的 Leader 副本和所有 Follower 副本的 LEO 值，而 Broker 1 上仅仅保存了该分区的某个 Follower 副本。Kafka 把 Broker 0 上保存的这些 Follower 副本又称为远程副本（Remote Replica）。Kafka 副本机制在运行过程中，会更新 Broker 1 上 Follower 副本的高水位和 LEO 值，同时也会更新 Broker 0 上 Leader 副本的高水位和 LEO 以及所有远程副本的 LEO，但它不会更新远程副本的高水位值，也就是我在图中标记为灰色的部分。

保存远程副本的作用主要是帮助 Leader 副本确定其高水位，也就是分区高水位。下图是副本同步机制：
![](https://tinaxiawuhao.github.io/post-images/1622096536181.png)

### 10,Leader副本保持同步

与Leader副本保持同步的判断条件有两个：

1. 该远程Follower副本在ISR中。
2. 该远程Follower副本LEO值落后于Leader副本LEO值的时间，不超过Broker端参数replica.lag.time.max.ms的值（默认值10秒）

#### 副本同步全流程

当生产者发送一条消息时，Leader 和 Follower 副本对应的高水位是怎么被更新的呢？

首先是初始状态。下面这张图中的 remote LEO 就是刚才的远程副本的 LEO 值。在初始状态时，所有值都是 0。

![](https://tinaxiawuhao.github.io/post-images/1622096547680.png)

当生产者给主题分区发送一条消息后，状态变更为：

![](https://tinaxiawuhao.github.io/post-images/1622096554503.png)

此时，Leader 副本成功将消息写入了本地磁盘，故 LEO 值被更新为 1。

Follower 再次尝试从 Leader 拉取消息。和之前不同的是，这次有消息可以拉取了，因此状态进一步变更为：

![](https://tinaxiawuhao.github.io/post-images/1622096561747.png)

这时，Follower 副本也成功地更新 LEO 为 1。此时，Leader 和 Follower 副本的 LEO 都是 1，但各自的高水位依然是 0，还没有被更新。它们需要在下一轮的拉取中被更新，如下图所示：

![](https://tinaxiawuhao.github.io/post-images/1622096573656.png)

在新一轮的拉取请求中，由于位移值是 0 的消息已经拉取成功，因此 Follower 副本这次请求拉取的是位移值 =1 的消息。Leader 副本接收到此请求后，更新远程副本 LEO 为 1，然后更新 Leader 高水位为 1。做完这些之后，它会将当前已更新过的高水位值 1 发送给 Follower 副本。Follower 副本接收到以后，也将自己的高水位值更新成 1。至此，一次完整的消息同步周期就结束了。事实上，Kafka 就是利用这样的机制，实现了 Leader 和 Follower 副本之间的同步。

#### Leader Epoch

从刚才的分析中，我们知道，Follower 副本的高水位更新需要一轮额外的拉取请求才能实现。如果把上面那个例子扩展到多个 Follower 副本，情况可能更糟，也许需要多轮拉取请求。也就是说，Leader 副本高水位更新和 Follower 副本高水位更新在时间上是存在错配的。这种错配是很多“数据丢失”或“数据不一致”问题的根源。基于此，社区在 0.11 版本正式引入了 Leader Epoch 概念，来规避因高水位更新错配导致的各种不一致问题。

所谓 Leader Epoch，我们大致可以认为是 Leader 版本。它由两部分数据组成。

1. Epoch。一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。
2. 起始位移（Start Offset）。Leader 副本在该 Epoch 值上写入的首条消息的位移。

举个例子来说明一下 Leader Epoch。假设现在有两个 Leader Epoch<0, 0> 和 <1, 120>，那么，第一个 Leader Epoch 表示版本号是 0，这个版本的 Leader 从位移 0 开始保存消息，一共保存了 120 条消息。之后，Leader 发生了变更，版本号增加到 1，新版本的起始位移是 120。

Kafka Broker 会在内存中为每个分区都缓存 Leader Epoch 数据，同时它还会定期地将这些信息持久化到一个 checkpoint 文件中。当 Leader 副本写入消息到磁盘时，Broker 会尝试更新这部分缓存。如果该 Leader 是首次写入消息，那么 Broker 会向缓存中增加一个 Leader Epoch 条目，否则就不做更新。这样，每次有 Leader 变更时，新的 Leader 副本会查询这部分缓存，取出对应的 Leader Epoch 的起始位移，以避免数据丢失和不一致的情况。

接下来看一个实际的例子，它展示的是 Leader Epoch 是如何防止数据丢失的。

![](https://tinaxiawuhao.github.io/post-images/1622096585706.png)

引用 Leader Epoch 机制后，Follower 副本 B 重启回来后，需要向 A 发送一个特殊的请求去获取 Leader 的 LEO 值。在这个例子中，该值为 2。当获知到 Leader LEO=2 后，B 发现该 LEO 值不比它自己的 LEO 值小，而且缓存中也没有保存任何起始位移值 > 2 的 Epoch 条目，因此 B 无需执行任何日志截断操作。这是对高水位机制的一个明显改进，即副本是否执行日志截断不再依赖于高水位进行判断。

现在，副本 A 宕机了，B 成为 Leader。同样地，当 A 重启回来后，执行与 B 相同的逻辑判断，发现也不用执行日志截断，至此位移值为 1 的那条消息在两个副本中均得到保留。后面当生产者程序向 B 写入新消息时，副本 B 所在的 Broker 缓存中，会生成新的 Leader Epoch 条目：[Epoch=1, Offset=2]。之后，副本 B 会使用这个条目帮助判断后续是否执行日志截断操作。这样，通过 Leader Epoch 机制，Kafka 完美地规避了这种数据丢失场景。

## 深度思考题

### 11、Kafka 为什么不支持读写分离?

这道题目考察的是你对 Leader/Follower 模型的思考。

Leader/Follower 模型并没有规定 Follower 副本不可以对外提供读服务。很多框架都是允 许这么做的，只是 Kafka 最初为了避免不一致性的问题，而采用了让 Leader 统一提供服 务的方式。

不过，在开始回答这道题时，你可以率先亮出观点:**自 Kafka 2.4 之后，Kafka 提供了有限度的读写分离，也就是说，Follower 副本能够对外提供读服务**。

说完这些之后，你可以再给出之前的版本不支持读写分离的理由。

- **场景不适用**。读写分离适用于那种读负载很大，而写操作相对不频繁的场景，可 Kafka 不属于这样的场景。
- **同步机制**。Kafka 采用 PULL 方式实现 Follower 的同步，因此，Follower 与 Leader 存 在不一致性窗口。如果允许读 Follower 副本，就势必要处理消息滞后(Lagging)的问题。

### 12、如何调优 Kafka?

回答任何调优问题的第一步，就是 **确定优化目标，并且定量给出目标!** 这点特别重要。对于 Kafka 而言，常见的优化目标是吞吐量、延时、持久性和可用性。每一个方向的优化思路都 是不同的，甚至是相反的。

确定了目标之后，还要明确优化的维度。有些调优属于通用的优化思路，比如对操作系统、 JVM 等的优化;有些则是有针对性的，比如要优化 Kafka 的 TPS。我们需要从 3 个方向去考虑

- **Producer 端**:增加 batch.size、linger.ms，启用压缩，关闭重试等。
- **Broker 端**:增加 num.replica.fetchers，提升 Follower 同步 TPS，避免 Broker Full GC 等。
- **Consumer**:增加 fetch.min.bytes 等

### 13、Controller 发生网络分区(Network Partitioning)时，Kafka 会怎 么样?

这道题目能够诱发我们对分布式系统设计、CAP 理论、一致性等多方面的思考。不过，针 对故障定位和分析的这类问题，我建议你首先言明“实用至上”的观点，即不论怎么进行理论分析，永远都要以实际结果为准。一旦发生 Controller 网络分区，那么，第一要务就是 查看集群是否出现“脑裂”，即同时出现两个甚至是多个 Controller 组件。这可以根据 Broker 端监控指标 ActiveControllerCount 来判断。

现在，我们分析下，一旦出现这种情况，Kafka 会怎么样。

由于 Controller 会给 Broker 发送 3 类请求，即LeaderAndIsrRequest、 StopReplicaRequest 和 UpdateMetadataRequest，因此，一旦出现网络分区，这些请求将不能顺利到达 Broker 端。这将影响主题的创建、修改、删除操作的信息同步，表现为 集群仿佛僵住了一样，无法感知到后面的所有操作。因此，网络分区通常都是非常严重的问 题，要赶快修复。

### 14、Java Consumer 为什么采用单线程来获取消息?

在回答之前，如果先把这句话说出来，一定会加分:**Java Consumer 是双线程的设计。一 个线程是用户主线程，负责获取消息;另一个线程是心跳线程，负责向 Kafka 汇报消费者 存活情况。将心跳单独放入专属的线程，能够有效地规避因消息处理速度慢而被视为下线 的“假死”情况。**

单线程获取消息的设计能够避免阻塞式的消息获取方式。单线程轮询方式容易实现异步非阻塞式，这样便于将消费者扩展成支持实时流处理的操作算子。因为很多实时流处理操作算子都不能是阻塞式的。另外一个可能的好处是，可以简化代码的开发。多线程交互的代码是非常容易出错的。

### 15、简述 Follower 副本消息同步的完整流程

首先，Follower 发送 FETCH 请求给 Leader。接着，Leader 会读取底层日志文件中的消 息数据，再更新它内存中的 Follower 副本的 LEO 值，更新为 FETCH 请求中的 fetchOffset 值。最后，尝试更新分区高水位值。Follower 接收到 FETCH 响应之后，会把 消息写入到底层日志，接着更新 LEO 和 HW 值。

Leader 和 Follower 的 HW 值更新时机是不同的，Follower 的 HW 更新永远落后于 Leader 的 HW。这种时间上的错配是造成各种不一致的原因。