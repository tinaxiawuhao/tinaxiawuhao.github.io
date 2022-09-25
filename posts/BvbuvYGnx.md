---
title: 'RocketMQ'
date: 2022-04-27 20:08:50
tags: [MQ]
published: true
hideInList: false
feature: /post-images/BvbuvYGnx.png
isTop: false
---
### 可用性评估

系统可用性(Availability)是信息工业界用来衡量一个信息系统提供持续服务的能力，它表示的是在给定时间区间内系统或者系统某一能力在特定环境中能够正常工作的概率。

简单地说， 可用性是平均故障间隔时间(MTBF)除以平均故障间隔时间(MTBF)和平均故障修复时间(MTTR)之和所得的结果， 即：

![](https://tianxiawuhao.github.io/post-images/1657627908926.png)

通常业界习惯用N个9来表征系统可用性，表示系统可以正常使用时间与总时间(1年)之比，比如：

- 99.9%代表3个9的可用性，意味着全年不可用时间在8.76小时以内，表示该系统在连续运行1年时间里最多可能的业务中断时间是8.76小时；
- 99.99%代表4个9的可用性，意味着全年不可用时间在52.6分钟以内,表示该系统在连续运行1年时间里最多可能的业务中断时间是52.6分钟；
- 99.999%代表5个9的可用性，意味着全年不可用时间必须保证在5.26分钟以内，缺少故障自动恢复机制的系统将很难达到5个9的高可用性。

那么X个9里的X只代表数字35，为什么没有12，也没有大于6的呢？

我们接着往下计算：

```
1个9：(1-90%)*365=36.5天 

*2个9：(1-99%)*365=3.65天 

6个9：(1-99.9999%)*365*24*60*60=31秒
```

可以看到1个9和、2个9分别表示一年时间内业务可能中断的时间是36.5天、3.65天，这种级别的可靠性或许还不配使用“可靠性”这个词；

而6个9则表示一年内业务中断时间最多是31秒，那么这个级别的可靠性并非实现不了，而是要做到从“5个9” 到“6个9”的可靠性提升的话，后者需要付出比前者几倍的成本。

### RocketMQ架构设计

在介绍RocketMQ高可用之前，首先了解一下RocketMQ架构设计

- 技术架构
- 部署架构

#### 技术架构

RocketMQ架构上主要分为四部分，如图所示:

![](https://tianxiawuhao.github.io/post-images/1657627944749.png)

- Producer：消息发布的角色，支持分布式集群方式部署。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。
- Consumer：消息消费的角色，支持分布式集群方式部署。支持以push推，pull拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。
- NameServer：NameServer是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能：Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。
- BrokerServer：Broker主要负责消息的存储、投递和查询以及服务高可用保证，为了实现这些功能，Broker包含了以下几个重要子模块。
  - 1. Remoting Module：整个Broker的实体，负责处理来自clients端的请求。
  - 2. Client Manager：负责管理客户端(Producer/Consumer)和维护Consumer的Topic订阅信息
  - 3. Store Service：提供方便简单的API接口处理消息存储到物理硬盘和查询功能。
  - 4. HA Service：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能。
  - 5. Index Service：根据特定的Message key对投递到Broker的消息进行索引服务，以提供消息的快速查询。

#### 部署架构

RocketMQ的Broker有三种集群部署方式：

- 1.单台Master部署；
- 2.多台Master部署；
- 3.多Master多Slave部署；

基础的rocket高可用，主要采用第3种部署方式

下图是第3种部署方式的简单图：

![](https://tianxiawuhao.github.io/post-images/1657627968609.png)

第3种部署方式网络部署特点

- NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
- Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。 注意：当前RocketMQ版本在部署架构上支持一Master多Slave，但只有BrokerId=1的从服务器才会参与消息的读负载。
- Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic 服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
- Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，消费者在向Master拉取消息时，Master服务器会根据拉取偏移量与最大偏移量的距离（判断是否读老消息，产生读I/O），以及从服务器是否可读等因素建议下一次是从Master还是Slave拉取。

结合部署架构图，描述集群工作流程：

- 启动NameServer，NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，相当于一个路由控制中心。
- Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。
- 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。
- Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发消息。
- Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息。

#### 汇总：RocketMQ 集群部署模式

前面介绍到，RocketMQ的Broker有三种集群部署方式：

- 1.单台Master部署；
- 2.多台Master部署；
- 3.多Master多Slave部署；

第三种模式，根据Master和Slave之节的数据同步方式可以分为：

- 多 master 多 slave 异步复制模式
- 多 master 多 slave 同步复制模式

> 同步方式：同步复制和异步复制（指的一组 master 和 slave 之间数据的同步）

所以，总体来说，RocketMQ 集群部署模式为四种：

- 1.**单 master 模式** 也就是只有一个 master 节点，如果master节点挂掉了，会导致整个服务不可用，线上不宜使用，适合个人学习使用。

- 2.**多 master 模式** 多个 master 节点组成集群，单个 master 节点宕机或者重启对应用没有影响。 优点：所有模式中性能最高 缺点：单个 master 节点宕机期间，未被消费的消息在节点恢复之前不可用，消息的实时性就受到影响。 注意：使用同步刷盘可以保证消息不丢失，同时 Topic 相对应的 queue 应该分布在集群中各个 master 节点，而不是只在某各 master 节点上，否则，该节点宕机会对订阅该 topic 的应用造成影响。

- 3.**多 master 多 slave 异步复制模式** 在多 master 模式的基础上，每个 master 节点都有至少一个对应的 slave。

   master 节点可读可写，但是 slave 只能读不能写，类似于 mysql 的主备模式。 优点： 在 master 宕机时，消费者可以从 slave 读取消息，消息的实时性不会受影响，性能几乎和多 master 一样。 缺点：使用异步复制的同步方式有可能会有消息丢失的问题。

- 4.**多 master 多 slave 同步双写模式** 同多 master 多 slave 异步复制模式类似，区别在于 master 和 slave 之间的数据同步方式。 优点：同步双写的同步模式能保证数据不丢失。 缺点：发送单个消息 RT 会略长，性能相比异步复制低10%左右。 刷盘策略：同步刷盘和异步刷盘（指的是节点自身数据是同步还是异步存储） 注意：要保证数据可靠，需采用同步刷盘和同步双写的方式，但性能会较其他方式低。

### RocketMQ与ZooKeeper的爱恨纠葛

说到高性能消息中间件，第一个想到的肯定是LinkedIn开源的Kafka，虽然最初Kafka是为日志传输而生，但也非常适合互联网公司消息服务的应用场景，他们不要求数据实时的强一致性（事务），更多是希望达到数据的最终一致性。

RocketMQ是MetaQ的3.0版本，而MetaQ最初的设计又参考了Kafka。最初的MetaQ 1.x版本由阿里的原作者庄晓丹开发，后面的MetaQ 2.x版本才进行了开源。

MetaQ 1.x和MetaQ 2.x是依赖ZooKeeper的，但RocketMQ（即MetaQ 3.x）却去掉了ZooKeeper依赖，转而采用自己的NameServer。

ZooKeeper是著名的分布式协作框架，提供了Master选举、分布式锁、数据的发布和订阅等诸多功能。为什么RocketMQ没有选择ZooKeeper，而是自己开发了NameServer，我们来具体看看NameServer在RocketMQ集群中的作用就明了了。

#### RocketMQ的Broker有三种集群部署方式

RocketMQ的Broker有三种集群部署方式：

- 1.单台Master部署；
- 2.多台Master部署；
- 3.多Master多Slave部署；

采用第3种部署方式时，Master和Slave可以采用同步复制和异步复制两种方式。

下图是第3种部署方式的简单图：

![](https://tianxiawuhao.github.io/post-images/1657628006696.png)

当采用多Master方式时，Master与Master之间是不需要知道彼此的，这样的设计直接降低了Broker实现的复杂性。

你可以试想，如果Master与Master之间需要知道彼此的存在，这会需要在Master之中维护一个网络的Master列表，而且必然设计到Master发现和活跃Master数量变更等诸多状态更新问题，所以最简单也最可靠的做法就是Master只做好自己的事情（比如和Slave进行数据同步）即可。

> 这样，在分布式环境中，某台Master宕机或上线，不会对其他Master造成任何影响。

那么怎么才能知道网络中有多少台Master和Slave呢？

你会很自然想到用ZooKeeper，每个活跃的Master或Slave都去约定的ZooKeeper节点下注册一个状态节点，但RocketMQ没有使用ZooKeeper，所以这件事就交给了NameServer来做了（看上图）。

#### NameServer的功能

功能一：NameServer用来保存活跃的broker列表，包括Master和Slave。

功能二：NameServer用来保存所有topic和该topic所有队列的列表。

功能三：NameServer用来保存所有broker的Filter列表。

功能四：NameServer可以理解承担了注册中心的职能

#### NameServer注册中心职能

NameServer是一个非常简单的路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。

- Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
- 路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费

整个Rocketmq集群的工作原理如下图所示：

![](https://tianxiawuhao.github.io/post-images/1657628018847.png)

可以看到，Broker集群、Producer集群、Consumer集群都需要与NameServer集群进行通信：

**Broker集群:**

Broker用于接收生产者发送消息，或者消费者消费消息的请求。一个Broker集群由多组Master/Slave组成，Master可写可读，Slave只可以读，Master将写入的数据同步给Slave。

每个Broker节点，在启动时，都会遍历NameServer列表，与每个NameServer建立长连接，注册自己的信息，之后定时上报。

**Producer集群:**

消息的生产者，通过NameServer集群获得Topic的路由信息，包括Topic下面有哪些Queue，这些Queue分布在哪些Broker上等。Producer只会将消息发送到Master节点上，因此只需要与Master节点建立连接。

**Consumer集群:**

消息的消费者，通过NameServer集群获得Topic的路由信息，连接到对应的Broker上消费消息。注意，由于Master和Slave都可以读取消息，因此Consumer会与Master和Slave都建立连接。

总之：

Name Server 是专为 RocketMQ 设计的轻量级注册中心，具有简单、可集群横吐扩展、无状态，节点之间互不通信等特点。

#### RocketMQ为什么不使用ZooKeeper

来看看RocketMQ为什么不使用ZooKeeper？

ZooKeeper可以提供Master选举功能。比如Kafka用来给每个分区选一个broker作为leader。

但对于RocketMQ来说，topic的数据在每个Master上是对等的，没有哪个Master上有topic上的全部数据，所以这里选举leader没有意义；

RockeqMQ集群中，需要有构件来处理一些通用数据，比如broker列表，broker刷新时间。

虽然ZooKeeper也能存放数据，并有一致性保证。但处理数据之间的一些逻辑关系却比较麻烦，而且数据的逻辑解析操作得交给ZooKeeper客户端来做，如果有多种角色的客户端存在，自己解析多级数据确实是个麻烦事情；

既然RocketMQ集群中没有用到ZooKeeper的一些重量级的功能，只是使用ZooKeeper的数据一致性和发布订阅的话，与其依赖重量级的ZooKeeper，还不如写个轻量级的NameServer，NameServer也可以集群部署，NameServer与NameServer之间无任何信息同步，不需要保障数据一致性， 比zk简单太多。

#### NameServer特性

- NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer，Consumer仍然可以动态感知Broker的路由的信息。
- NameServer实例时间互不通信，这本身也是其设计亮点之一，即允许不同NameServer之间数据不同步(像Zookeeper那样保证各节点数据强一致性会带来额外的性能消耗)

### RocketMQ 的消息类型

RocketMQ 支持普通消息，顺序消息、事务消息，等等多种消息类型：

- 普通消息：没有特殊功能的消息。
- 分区顺序消息：以分区纬度保持顺序进行消费的消息。
- 全局顺序消息：全局顺序消息可以看作是只分一个区，始终在同一个分区上进行消费。
- 定时/延时消息：消息可以延迟一段特定时间进行消费。
- 事务消息：二阶段事务消息，先进行prepare投递消息，此时不能进行消息消费，当二阶段发出commit或者rollback的时候才会进行消息的消费或者回滚。

虽然配置种类比较繁多，但是使用的还是普通消息和分区顺序消息。

本文的主要介绍高可用，主要介绍普通消息，其他消息的高可用策略，也是类似侧。

### RocketMQ高可用

![](https://tianxiawuhao.github.io/post-images/1657628078933.png)

#### NameServer 高可用

由于 NameServer 节点是无状态的，且各个节点直接的数据是一致的，故存在多个 NameServer 节点的情况下，部分 NameServer 不可用也可以保证 MQ 服务正常运行

#### BrokerServer 高可用

RocketMQ是通过 Master 和 Slave 的配合达到 BrokerServer 模块的高可用性的

一个 Master 可以配置多个 Slave，同时也支持配置多个 Master-Slave 组。

**当其中一个 Master 出现问题时：**

- 由于Slave只负责读，当 Master 不可用，它对应的 Slave 仍能保证消息被正常消费
- 由于配置多组 Master-Slave 组，其他的 Master-Slave 组也会保证消息的正常发送和消费

老版本的RocketMQ不支持把Slave自动转成Master，如果机器资源不足， 需要把Slave转成Master，则要手动停止Slave角色的Broker，更改配置文 件，用新的配置文件启动Broker。

新版本的RocketMQ，支持Slave自动转成Master。

#### consumer高可用

Consumer 的高可用是依赖于 Master-Slave 配置的，由于 Master 能够支持读写消息，Slave 支持读消息，当 Master 不可用或繁忙时， Consumer 会被自动切换到从 Slave 读取(自动切换，无需配置)。

故当 Master 的机器故障后，消息仍可从 Slave 中被消费

#### producer高可用

在创建Topic的时候，把Topic的多个Message Queue创建在多个Broker组上（相同Broker名称，不同 brokerId的机器组成一个Broker组）.

这样当一个Broker组的Master不可用后，其他组的Master仍然可用，Producer仍然可以发送消息。

![](https://tianxiawuhao.github.io/post-images/1657628094386.png)

### 实现分布式集群多副本的三种方式

#### M/S模式

即Master/Slaver模式。

该模式在过去使用的最多，RocketMq之前也是使用这样的主从模式来实现的。

主从模式分为同步模式和异步模式，区别是在同步模式下只有主从复制完毕才会返回给客户端；而在异步模式中，主从的复制是异步的，不用等待即可返回。

![](https://tianxiawuhao.github.io/post-images/1657628109179.png)

同步模式

**同步模式特点**

![](https://tianxiawuhao.github.io/post-images/1657628128161.png)

异步模式

**异步模式特点**

#### 基于zookeeper服务

![](https://tianxiawuhao.github.io/post-images/1657628139871.png)

和M/S模式相比zookeeper模式是自动选举的主节点，新版本rocketMq暂时不支持zookeeper。

#### 基于raft

![](https://tianxiawuhao.github.io/post-images/1657628150637.png)

相比zookeeper，raft自身就可以实现选举，raft通过投票的方式实现自身选举leader。去除额外依赖。目前RocketMq 4.5.0已经支持

### 可用性与可靠性

**可用性**

由于消息分布在各个broker上，一旦某个broker宕机，则该broker上的消息读写都会受到影响。所以rocketmq提供了master/slave的结构，salve定时从master同步数据，如果master宕机，则slave提供消费服务，但是不能写入消息，此过程对应用透明，由rocketmq内部解决。

这里有两个关键点：

- 一旦某个broker master宕机，生产者和消费者多久才能发现？受限于rocketmq的网络连接机制，默认情况下，最多需要30秒，但这个时间可由应用设定参数来缩短时间。这个时间段内，发往该broker的消息都是失败的，而且该broker的消息无法消费，因为此时消费者不知道该broker已经挂掉。
- 消费者得到master宕机通知后，转向slave消费，但是slave不能保证master的消息100%都同步过来了，因此会有少量的消息丢失。但是消息最终不会丢的，一旦master恢复，未同步过去的消息会被消费掉。

**可靠性**

- 所有发往broker的消息，有同步刷盘和异步刷盘机制，总的来说，可靠性非常高
- 同步刷盘时，消息写入物理文件才会返回成功，因此非常可靠
- 异步刷盘时，只有机器宕机，才会产生消息丢失，broker挂掉可能会发生，但是机器宕机崩溃是很少发生的，除非突然断电

### Broker消息的零丢失方案

#### 同步刷盘、异步刷盘

RocketMQ的消息是存储到磁盘上的，这样既能保证断电后恢复，又可以让存储的消息量超出内存的限制。RocketMQ为了提高性能，会尽可能地保证磁盘的顺序写。

消息在通过Producer写入RocketMQ的时候，有两种写磁盘方式：

- 异步刷盘方式：

在返回写成功状态时，消息可能只是被写入了内存的PAGECACHE，写操作的返回快，吞吐量大；当内存里的消息量积累到一定程度时，统一触发写磁盘操作，快速写入
优点：性能高
缺点：Master宕机，磁盘损坏的情况下，会丢失少量的消息, 导致MQ的消息状态和生产者/消费者的消息状态不一致

- 同步刷盘方式：

在返回应用写成功状态前，消息已经被写入磁盘。

具体流程是，消息写入内存的PAGECACHE后，立刻通知刷盘线程刷盘，然后等待刷盘完成，刷盘线程执行完成后唤醒等待的线程，给应用返回消息写成功的状态。

优点：可以保持MQ的消息状态和生产者/消费者的消息状态一致
缺点：性能比异步的低
同步刷盘还是异步刷盘，是通过Broker配置文件里的flushDiskType参数设置的，这个参数被设置成SYNC_FLUSH, ASYNC_FLUSH中的一个。

#### 同步复制、异步复制

如果一个broker组有Master和Slave，消息需要从Master复制到Slave上，有同步和异步两种复制方式。

- 同步复制方式：

等Master和Slave均写成功后才反馈给客户端写成功状态
优点：如果Master出故障，Slave上有全部的备份数据，容易恢复，消费者仍可以从Slave消费, 消息不丢失
缺点：增大数据写入延迟，降低系统吞吐量，性能比异步复制模式略低，大约低10%左右，发送单个Master的响应时间会略高

- 异步复制方式：

只要Master写成功即可反馈给客户端写成功状态
优点：系统拥有较低的延迟和较高的吞吐量. Master宕机之后，消费者仍可以从Slave消费，此过程对应用透明，不需要人工干预，性能同多个Master模式几乎一样
缺点：如果Master出了故障，有些数据因为没有被写入Slave，而丢失少量消息。

若一个 Broker 组有一个 Master 和 Slave，消息需要从 Master 复制到 Slave 上，有同步复制和异步复制两种方式

|            | **同步复制**                                                 | **异步复制**                                                 |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **概念**   | 即等 Master 和 Slave 均写成功后才反馈给客户端写成功状态      | 只要 Master 写成功，就反馈客户端写成功状态                   |
| **可靠性** | 可靠性高，若 Master 出现故障，Slave 上有全部的备份数据，容易恢复 | 若 Master 出现故障，可能存在一些数据还没来得及写入 Slave，可能会丢失 |
| **效率**   | 由于是同步复制，会增加数据写入延迟，降低系统吞吐量           | 由于只要写入 Master 即可，故数据写入延迟较低，吞吐量较高     |

同步复制和异步复制是通过Broker配置文件里的brokerRole参数进行设置的，这个参数可以被设置成ASYNC_MASTER、SYNC_MASTER、SLAVE三个值中的一个。

三个值的说明：

- sync_master是同步方式，Master角色Broker中的消息要立刻同步过去。
- async_master是异步方式，Master角色Broker中的消息通过异步处理的方式同步到Slave角色的机器上。
- SLAVE 表明当前是从节点，无需配置 brokerRole

#### 消息零丢失方案

消息零丢失是一把双刃剑，要想用好，还是要视具体的业务场景，在性能和消息零丢失上做平衡。

实际应用中的推荐把Master和Slave设置成ASYNC_FLUSH的异步刷盘方式，主从之间配置成SYNC_MASTER的同步复制方式，这样即使有一台机器出故障，仍然可以保证数据不丢。

- 刷盘方式

Master和Slave都设置成ASYNC_FLUSH的异步刷盘

- 复制方式

Master配置成SYNC_MASTER 同步复制

异步刷盘能够避免频繁触发磁盘写操作，除非服务器宕机，否则不会造成消息丢失。

主从同步复制能够保证消息不丢失，即使 Master 节点异常，也能保证 Slave 节点存储所有消息并被正常消费掉。

### producer高可用

producer具备发送到全部master的能力，如果有多个master，消息会发送到所有的master

![](https://tianxiawuhao.github.io/post-images/1657628176572.png)

另外，在topic的不同的queue之间，producer还具备负载均衡能力。

在实例发送消息时，默认会轮询所有订阅了改 Topic 的 broker 节点上的 message queue，让消息平均落在不同的 queue 上，而由于这些 queue 散落在不同的 broker 节点中，即使某个 broker 节点异常，其他存在订阅了这个 Topic 的 message queue 的 broker 依然能消费消息

### 消息者业务代码出现异常怎么办？

再来看一下消费者的代码中监听器的部分，它说如果消息处理成功，那么就返回消息状态为 CONSUME_SUCCESS，也有可能发放优惠券、积分等操作出现了异常，比如说数据库挂掉了。这个时候应该怎么处理呢？

```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List <MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
        // 对消息的处理，比如发放优惠券、积分等
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

我们可以把代码改一改，捕获异常之后返回消息的状态为 RECONSUME_LATER 表示稍后重试。

```java
// 这次回调接口，接收消息
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List <MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
        try {
            // 对消息的处理，比如发放优惠券、积分等
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        } catch (Exception e) {
            // 万一发生数据库宕机等异常，返回稍后重试消息的状态
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }

    }
});
```

这个时候，消息会进入到 RocketMQ 的重试队列中。

#### 重试队列

比如说消费者所属的消息组名称为**AAAConsumerGroup**
其重试队列名称就叫做**%RETRY%AAAConsumerGroup**
重试队列中的消息过一段时间会再次发送给消费者，如果还是无法正常执行会再次进入重试队列
默认重试16次，还是无法执行，消息就会从重试队列进入到死信队列

#### 死信队列

重试队列中的消息重试16次任然无法执行，将会进入到死信队列
死信队列的名字是 **%DLQ%AAAConsumerGroup**
死信队列中的消息可以后台开一个线程，订阅**%DLQ%AAAConsumerGroup**，并不停重试

### Customer 负载均衡

#### 集群模式

在集群消费模式下，存在多个消费者同时消费消息，同一条消息只会被某一个消费者获取。即消息只需要被投递到订阅了这个 Topic 的消费者Group下的一个实例中即可。

消费者采用主动拉去的方式拉去并消费，在拉取的时候需要明确指定拉取那一条消息队列中的消息。

每当有实例变更，都会触发一次所有消费者实例的负载均衡，这是会按照queue的数量和实例的数量平均分配 queue 给每个消费者实例。

![](https://tianxiawuhao.github.io/post-images/1657628194080.png)

**注意：**

1）在集群模式下，一个 queue 只允许分配给一个消费者实例，这是由于若多个实例同时消费一个 queue 的小，由于拉取操作是由 consumer 主动发生的，可能导致同一个消息在不同的 consumer 实例中被消费。故算法保证了一个 queue 只会被一个 consumer 实例消费，但一个 consumer 实例能够消费多个 queue

2）控制 consumer 数量，应小于 queue 数量。这是由于一个 queue 只允许分配给一个 consumer 实例，若 consumer 实例数量多于 queue，则多出的 consumer 实例无法分配到 queue消费，会浪费系统资源

#### 广播模式

广播模式其实不是负载均衡，由于每个消费者都能够拿到所有消息，故不能达到负载均衡的要求

![](https://tianxiawuhao.github.io/post-images/1657628208176.png)

### 消费者的消息重试

### 顺序消息重试

对于顺序消息，为了保证消息消费的顺序性，当consumer消费失败后，消息队列会自动不断进行消息重试(每次间隔时间为1s)，

这时会导致consumer消费被阻塞的情况，故必须保证应用能够及时监控并处理消费失败的情况，避免阻塞现象的发生

### 无序消息重试

#### 概述

无序消息即普通、定时、延时、事务消息，当consumer消费消息失败时，可以通过设置返回状态实现消息重试

> 注意：无序消息的重试只针对集群消费方式（非广播方式）生效

广播方式不提供失败重试特性，即消费失败后，失败的消息不再重试，而是继续消费新消息

#### 重试次数

消息队列 RocketMQ 默认允许每条消息最多重试 16 次，每次重试的间隔时间如下：

| 第几次重试 | 与上次重试的间隔时间 | 第几次重试 | 与上次重试的间隔时间 |
| :--------- | :------------------- | :--------- | :------------------- |
| 1          | 10 秒                | 9          | 7 分钟               |
| 2          | 30 秒                | 10         | 8 分钟               |
| 3          | 1 分钟               | 11         | 9 分钟               |
| 4          | 2 分钟               | 12         | 10 分钟              |
| 5          | 3 分钟               | 13         | 20 分钟              |
| 6          | 4 分钟               | 14         | 30 分钟              |
| 7          | 5 分钟               | 15         | 1 小时               |
| 8          | 6 分钟               | 16         | 2 小时               |

> 如果消息重试 16 次后仍然失败，消息将不再投递。

如果严格按照上述重试时间间隔计算，某条消息在一直消费失败的前提下，将会在接下来的 4 小时 46 分钟之内进行 16 次重试，超过这个时间范围消息将不再重试投递。

**注意：** 一条消息无论重试多少次，这些重试消息的 Message ID 不会改变。

#### 消息重试相关的处理方式

##### 消费失败后，需要重试的处理方式

集群消费方式（非广播方式）下，消息消费失败后期望消息重试，需要在消息监听器接口的实现中明确进行配置（三种方式任选一种）：

- 方式 1：返回 Action.ReconsumeLater（推荐）
- 方式 2：返回 Null
- 方式 3：抛出异常

示例代码

```java
public class MessageListenerImpl implements MessageListener {

    @Override
    public Action consume(Message message, ConsumeContext context) {
        //消息处理逻辑抛出异常，消息将重试
        doConsumeMessage(message);
   
       //方式 1：返回 Action.ReconsumeLater，消息将重试
        return Action.ReconsumeLater;
     
        //方式 2：返回 null，消息将重试
        return null;
   
        //方式 3：直接抛出异常，消息将重试
        throw new RuntimeException("Consumer Message exception");
    }
}
```

集群消费方式下，消息消费失败后期望消息重试，需要在消息监听器接口的实现中明确进行配置

##### 消费失败后，无需重试的处理方式

集群消费方式下，消息失败后期望消息不重试，需要捕获消费逻辑中可能抛出的异常，最终返回 Action.CommitMessage，此后这条消息将不会再重试。

```java
public class MessageListenerImpl implements MessageListener {

    @Override
    public Action consume(Message message, ConsumeContext context) {
        try {
            doConsumeMessage(message);
        } catch (Throwable e) {
            //捕获消费逻辑中的所有异常，并返回 Action.CommitMessage;
            return Action.CommitMessage;
        }
        //消息处理正常，直接返回 Action.CommitMessage;
        return Action.CommitMessage;
    }
}
```

**3）自定义消息最大重试次数**

消息队列 RocketMQ 允许 Consumer 启动的时候设置最大重试次数，重试时间间隔将按照如下策略：

- 最大重试次数小于等于 16 次，则重试时间间隔同上表描述。
- 最大重试次数大于 16 次，超过 16 次的重试时间间隔均为每次 2 小时。

**设置方式：**

```java
consumer.setMaxReconsumeTimes(20);
```

或者：

```java
Properties properties = new Properties();
//配置对应 Group ID 的最大消息重试次数为 20 次，最大重试次数为字符串类型
properties.put(PropertyKeyConst.MaxReconsumeTimes,"20");
Consumer consumer =ONSFactory.createConsumer(properties);
```

**注意：**

- 消息最大重试次数设置，对相同 Group ID 下的所有 Consumer 实例有效。
- 如果只对相同 Group ID 下两个 Consumer 实例中的其中一个设置了 MaxReconsumeTimes，那么该配置对两个 Consumer 实例均生效。
- 配置采用覆盖的方式生效，即最后启动的 Consumer 实例会覆盖之前的启动实例的配置

##### 获取消息重试次数

消费者收到消息后，可以获取到消息的重试次数

**设置方式：**

```java
public class MessageListenerImpl implements MessageListener {
 
    @Override
    public Action consume(Message message, ConsumeContext context) {
        //获取消息的重试次数
        System.out.println(message.getReconsumeTimes());
        return Action.CommitMessage;
    }
}
```

### 死信队列

#### 死信队列概念

> 在正常情况下无法被消费(超过最大重试次数)的消息称为死信消息(Dead-Letter Message)，存储死信消息的特殊队列就称为死信队列(Dead-Letter Queue)

当一条消息初次消费失败，消息队列 RocketMQ 会自动进行消息重试；

达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列 RocketMQ 不会立刻将消息丢弃，而是将其发送到该**消费者对应的死信队列**中。

代码正常执行返回消息状态为CONSUME_SUCCESS，执行异常返回RECONSUME_LATER
状态为RECONSUME_LATER的消息会进入到重试队列，重试队列的名称为 `%RETRY% + ConsumerGroupName`；
重试16次消息任然没有处理成功，消息就会进入到死信队列`%DLQ% + ConsumerGroupName`;

### 死信特性

**死信消息有以下特点：**

- 不会再被消费者正常消费
- 有效期与正常消息相同，均为 3 天，3 天后会被自动删除。故死信消息应在产生的 3 天内及时处理

**死信队列有以下特点：**

- 一个死信队列对应一个消费者组，而不是对应单个消费者实例
- 一个死信队列包含了对应的 Group ID 所产生的所有死信消息，不论该消息属于哪个 Topic
- 若一个 Group ID 没有产生过死信消息，则 RocketMQ 不会为其创建相应的死信队列

### 查看死信信息和重发

在控制台查看死信队列的主题信息

![](https://tianxiawuhao.github.io/post-images/1657628232656.png)

![](https://tianxiawuhao.github.io/post-images/1657628246206.png)

**重发消息**

![](https://tianxiawuhao.github.io/post-images/1657628261517.png)

### 消息幂等性

#### 消费幂等

消费幂等即无论消费者消费多少次，其结果都是一样的。

RocketMQ 是通过业务上的唯一 Key 来对消息做幂等处理

#### 消费幂等的必要性

在网络环境中，由于网络不稳定等因素，消息队列的消息有可能出现重复，大概有以下几种：

- **发送时消息重复**

  当一条消息已被成功发送到服务端并完成持久化，此时出现了网络闪断或者客户端宕机，导致服务端对客户端应答失败。 如果此时生产者意识到消息发送失败并尝试再次发送消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

- **投递时消息重复**

  消息消费的场景下，消息已投递到消费者并完成业务处理，当客户端给服务端反馈应答的时候网络闪断。 为了保证消息至少被消费一次，消息队列 RocketMQ 的服务端将在网络恢复后再次尝试投递之前已被处理过的消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

- **负载均衡时消息重复**（包括但不限于网络抖动、Broker 重启以及订阅方应用重启）

  当消息队列 RocketMQ 的 Broker 或客户端重启、扩容或缩容时，会触发 Rebalance，此时消费者可能会收到重复消息。

结合三种情况，可以发现消息重发的最后结果都是，消费者接收到了重复消息，那么，我们只需要在消费者端统一进行幂等处理就能够实现消息幂等。

#### 处理方式

##### 消费端实现消息幂等性

RocketMQ 只能够保证消息丢失，但不能保证消息不重复投递，且由于高可用和高性能的考虑，应该在消费端实现消息幂等性。

那么 RocketMQ 是怎样解决消息重复的问题呢？还是“恰好”不解决。

造成消息重复的根本原因是：网络不可达。只要通过网络交换数据，就无法避免这个问题。所以解决这个问题的办法就是绕过这个问题。那么问题就变成了：如果消费端收到两条一样的消息，应该怎样处理？

> - 消费端处理消息的业务逻辑保持幂等性
> - 保证每条消息都有唯一编号且保证消息处理成功与去重表的日志同时出现

- 第1条很好理解，只要保持幂等性，不管来多少条重复消息，最后处理的结果都一样。
- 第2条原理就是利用一张日志表来记录已经处理成功的消息的ID，如果新到的消息ID已经在日志表中，那么就不再处理这条消息。

第1条解决方案，很明显应该在消费端实现，不属于消息系统要实现的功能。第2条可以消息系统实现，也可以业务端实现。正常情况下出现重复消息的概率其实很小，如果由消息系统来实现的话，肯定会对消息系统的吞吐量和高可用有影响，所以最好还是由业务端自己处理消息重复的问题，这也是 RocketMQ 不解决消息重复的问题的原因。

**RocketMQ 不保证消息不重复，如果你的业务需要保证严格的不重复消息，需要你自己在业务端去重。**

在消费端通过业务逻辑实现幂等性操作，最常用的方式就是唯一ID的形式，若已经消费过的消息就不进行处理。例如在秒杀系统中使用订单ID作为关键ID，分布式系统中常用雪花算法生成ID。

> 注：如果需要彻底了解雪花算法，以及里边的位运算逻辑，请参见尼恩的秒杀视频。

在发送消息时，可以对 Message 设置标识唯一标识：

```java
Message message = new Message(); # 设置唯一标识，标识由雪花算法生成message.setKey(idWorker.nextId());
```

订阅方收到消息时，可以获取到这个 Key

```java
consumer.registerMessageListener(new MessageListenerConcurrently()
{
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context)
    {
        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
        for (MessageExt ext : msgs)
        {
            System.out.println(ext.getKeys());
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

### 一、顺序消息

顺序消息（FIFO 消息）是消息队列 RocketMQ 提供的一种严格按照顺序来发布和消费的消息。顺序发布和顺序消费是指对于指定的一个 Topic，生 产者按照一定的先后顺序发布消息；消费者按照既定的先后顺序订阅消息，即先发布的消息一定会先被客户端接收到。

> 顺序消息分为全局顺序消息和分区顺序消息。

#### 1.1、全局顺序消息

RocketMQ 在默认情况下不保证顺序，要保证全局顺序，需要把 Topic 的读写队列数设置为 1，然后生产者和消费者的并发设置也是 1。所以这样的话 高并发，高吞吐量的功能完全用不上。
![](https://tianxiawuhao.github.io/post-images/1657628280551.png)

##### 1.1.1、适用场景

适用于性能要求不高，所有的消息严格按照 FIFO 原则来发布和消费的场景。

##### 1.1.2、示例

要确保全局顺序消息，需要先把 Topic 的读写队列数设置为 1，然后生产者和消费者的并发设置也是 1。
```shell
mqadmin update Topic -t AllOrder -c DefaultCluster -r 1 -w 1 -n 127.0.0.1:9876
```

在证券处理中，以人民币兑换美元为 Topic，在价格相同的情况下，先出价者优先处理，则可以按照 FIFO 的方式发布和消费全局顺序消息。

#### 1.2、部分顺序消息

对于指定的一个 Topic，所有消息根据 Sharding Key 进行区块分区。同一个分区内的消息按照严格的 FIFO 顺序进行发布和消费。Sharding Key 是顺 序消息中用来区分不同分区的关键字段，和普通消息的 Key 是完全不同的概念。

### 二、延时消息

#### 2.1、概念介绍

**延时消息**：Producer 将消息发送到消息队列 RocketMQ 服务端，但并不期望这条消息立马投递，而是延迟一定时间后才投递到 Consumer 进行消费， 该消息即延时消息。

#### 2.2、适用场景

消息生产和消费有时间窗口要求：比如在电商交易中超时未支付关闭订单的场景，在订单创建时会发送一条延时消息。这条消息将会在 30 分钟以 后投递给消费者，消费者收到此消息后需要判断对应的订单是否已完成支付。 如支付未完成，则关闭订单。如已完成支付则忽略。

#### 2.3、使用方式

Apache RocketMQ 目前只支持固定精度的定时消息，因为如果要支持任意的时间精度，在 Broker 层面，必须要做消息排序，如果再涉及到持久化， 那么消息排序要不可避免的产生巨大性能开销。**（阿里云 RocketMQ 提供了任意时刻的定时消息功能，Apache 的 RocketMQ 并没有,阿里并没有开源）**
发送延时消息时需要设定一个延时时间长度，消息将从当前发送时间点开始延迟固定时间之后才开始投递。
延迟消息是根据延迟队列的 level 来的，延迟队列默认是
**msg.setDelayTimeLevel(5)**代表延迟一分钟
**"1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"**
是这 18 个等级（秒（s）、分（m）、小时（h）），level 为 1，表示延迟 1 秒后消费，level 为 5 表示延迟 1 分钟后消费，level 为 18 表示延迟 2 个 小时消费。生产消息跟普通的生产消息类似，只需要在消息上设置延迟队列的 level 即可。消费消息跟普通的消费消息一致。

### 三、死信队列

#### 3.1、概念介绍

死信队列用于处理无法被正常消费的消息。当一条消息初次消费失败，消息队列 **MQ** 会自动进行消息重试；达到最大重试次数后，若消费依然失败， 则表明**Consumer** 在正常情况下无法正确地消费该消息。此时，消息队列**MQ**不会立刻将消息丢弃，而是将这条消息发送到该 **Consumer** 对应的特殊队列中。
消息队列 **MQ **将这种正常情况下无法被消费的消息称为死信消息**（Dead-Letter Message）**，将存储死信消息的特殊队列称为死信队列 **（Dead-Letter Queue)**。

#### 3.2适用场景

##### 3.2.1、死信消息的特性

不会再被消费者正常消费。 有效期与正常消息相同，均为 3 天，3 天后会被自动删除。因此，请在死信消息产生后的 3 天内及时处理。

##### 3.2.2、死信队列的特性

一个死信队列对应一个 Group ID， 而不是对应单个消费者实例。
如果一个 **Group ID** 未产生死信消息，消息队列 **MQ** 不会为其创建相应的死信队列。
一个死信队列包含了对应 Group ID 产生的所有死信消息，不论该消息属于哪个 **Topic**。
消息队列 MQ 控制台提供对死信消息的查询的功能。

一般控制台直接查看死信消息会报错。
![](https://tianxiawuhao.github.io/post-images/1657628298619.png)

进入**RocketMQ**中服务器对应的 **RocketMQ** 中的/bin 目录，执行以下脚本
```shell
sh mqadmin updateTopic -b 192.168.0.128:10911 -n 192.168.0.128:9876 -t %DLQ%group1 -p 6
```

![](https://tianxiawuhao.github.io/post-images/1657628311169.png)
![](https://tianxiawuhao.github.io/post-images/1657628322848.png)

### 四、消费幂等

为了防止消息重复消费导致业务处理异常，消息队列 MQ 的消费者在接收到消息后，有必要根据业务上的唯一 Key 对消息做幂等处理。本文介绍消息幂 等的概念、适用场景以及处理方法。

#### 4.1、什么是消息幂等

当出现消费者对某条消息重复消费的情况时，重复消费的结果与消费一次的结果是相同的，并且多次消费并未对业务系统产生任何负面影响，那么 这整个过程就实现可消息幂等。
例如，在支付场景下，消费者消费扣款消息，对一笔订单执行扣款操作，扣款金额为 100 元。如果因网络不稳定等原因导致扣款消息重复投递，消 费者重复消费了该扣款消息，但最终的业务结果是只扣款一次，扣费 100 元，且用户的扣款记录中对应的订单只有一条扣款流水，不会多次扣除费用。 那么这次扣款操作是符合要求的，整个消费过程实现了消费幂等。

#### 4.2、需要处理的场景

在互联网应用中，尤其在网络不稳定的情况下，消息队列 MQ 的消息有可能会出现重复。如果消息重复会影响您的业务处理，请对消息做幂等处理。 消息重复的场景如下：
**1. 发送时消息重复 **
当一条消息已被成功发送到服务端并完成持久化，此时出现了网络闪断或者客户端宕机，导致服务端对客户端应答失败。 如果此时生产者意识到消 息发送失败并尝试再次发送消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。
**2. 投递时消息重复 **
消息消费的场景下，消息已投递到消费者并完成业务处理，当客户端给服务端反馈应答的时候网络闪断。为了保证消息至少被消费一次，消息队列 MQ 的服务端将在网络恢复后再次尝试投递之前已被处理过的消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。
**3. 负载均衡时消息重复（包括但不限于网络抖动、Broker 重启以及消费者应用重启）**
当消息队列 MQ 的 Broker 或客户端重启、扩容或缩容时，会触发 Rebalance，此时消费者可能会收到重复消息。

#### 4.3、处理方法

因为 Message ID 有可能出现冲突（重复）的情况，所以真正安全的幂等处理，不建议以 Message ID 作为处理依据。最好的方式是以业务唯一标识 作为幂等处理的关键依据，而业务的唯一标识可以通过消息 Key 设置。
以支付场景为例，可以将消息的 Key 设置为订单号，作为幂等处理的依据。具体代码示例如下：

```java
 Message message = new Message();
  message.setKey("ORDERID_100"); 
  SendResult sendResult = producer.send(message); 
```

消费者收到消息时可以根据消息的 Key，即订单号来实现消息幂等：

```java
 consumer.subscribe("ons_test", "*", new MessageListener() {
    
    
  public Action consume(Message message, ConsumeContext context) {
    
     
  String key = message.getKey()
   // 根据业务唯一标识的 Key 做幂等处理
    } });
```