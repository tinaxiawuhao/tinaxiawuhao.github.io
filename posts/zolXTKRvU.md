---
title: 'Nacos核心源码剖析（CP架构）——注册中心'
date: 2022-07-03 20:30:58
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/zolXTKRvU.png
isTop: false
---
### 一 Nacos CP集群架构的基础知识

####  1 Nacos集群部署后，可以同时支持AP和CP（注意，不是同时支持CAP） 

- AP架构：临时实例
- CP架构：持久化实例

在注册服务时，如果我们让我们的节点注册为：持久化实例，即自动会走CP架构！

```yaml
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: 10.206.73.156:8848
        namespace: haier-iot
        group: dev
        cluster-name: BJ
        ephemeral: true  # 持久化实例走CP架构
```

####  2 Nacos CP架构使用的分布式一致性协议？（简化版的Raft） 

Raft分布式一致性协议 和 Zookeeper使用的ZAB原子广播协议非常相似；

都是一个Leader带领多个Follower，区别在于，Leader选举时的投票机制：

- **ZAB投票时**：**所有候选节点都会发起投票，然后进行选票PK，决定谁胜出；**
- **Raft投票时**：**会让所有节点随机睡眠，先睡醒的节点发起投票，投自己，并将选票发到其它节点等待结果；**

但是，对结果的判断，ZAB 和 Raft 都遵循“半数机制”。

------

### 二 CP架构下，持久化实例的注册逻辑

####  1 注册实例的API接口不变：/nacos/v1/ns/instance 

![](https://tianxiawuhao.github.io/post-images/1658665954910.png)

这里当然时选择RaftConsistencyServiceImpl的实现：

```java
public void put(String key, Record value) throws NacosException {
    checkIsStopWork();
    try {
        raftCore.signalPublish(key, value);  
    } catch (Exception e) {
        ......
    }
}
```

####  2 整个CP架构下的主节点注册逻辑都在signalPublish()方法中 

```java
public void signalPublish(String key, Record value) throws Exception {
    if (stopWork) {
        throw new IllegalStateException("old raft protocol already stop work");
    }
    // 判断自己是不是Leader
    if (!isLeader()) {
        ObjectNode params = JacksonUtils.createEmptyJsonNode();
        params.put("key", key);
        params.replace("value", JacksonUtils.transferToJsonNode(value));
        Map<String, String> parameters = new HashMap<>(1);
        parameters.put("key", key);
 
        final RaftPeer leader = getLeader();
 
        // 如果本节点不是Leader，就把请求转发给Leader节点处理
        raftProxy.proxyPostLarge(leader.ip, API_PUB, params.toString(), parameters);
        return;
    }
 
    OPERATE_LOCK.lock();
    try {
        ......
 
        // 核心方法，写本地数据，写内存缓存，发布事件——更新内存服务注册表
        onPublish(datum, peers.local());
 
        final String content = json.toString();
 
        // 通过CountDownLatch实现半数ack的统计，如果获得到半数以上的ack，则Countdownlatch逻辑才可以继续向下走！
        final CountDownLatch latch = new CountDownLatch(peers.majorityCount());
        for (final String server : peers.allServersIncludeMyself()) {
            if (isLeader(server)) {  // 首先自己的钥匙先用上
                latch.countDown();
                continue;
            }
            final String url = buildUrl(server, API_ON_PUB);  // /v1/ns/raft/datum/commit
            HttpClient.asyncHttpPostLarge(url, Arrays.asList("key", key), content, new Callback<String>() {
                @Override
                public void onReceive(RestResult<String> result) {
                    latch.countDown();  // Http调用正常后，则当作一次ack
                }
 
                @Override
                public void onError(Throwable throwable) {
                     
                }
 
                @Override
                public void onCancel() {
 
                }
            });
 
        }
    } finally {
        OPERATE_LOCK.unlock();
    }
```

其实Nacos实现的简单Raft协议，逻辑有点不太严谨，就是：即使本次同步不成功，但是主节点的本地磁盘文件 + 内存文件 已经都修改过了，不像Zookeeper的两阶段提交；

后期Nacos的一致性协议会修改为JRaft，这点肯定会解决！

####  3、Leader本节点保存数据的逻辑：onPublish(datum, peers.local()) 

```java
public void onPublish(Datum datum, RaftPeer source) throws Exception {
    ......
 
    // 逻辑能走到这，这个if正常都为true
    if (KeyBuilder.matchPersistentKey(datum.key)) {
        // 核心，将数据写到本地磁盘
        raftStore.write(datum);
    }
 
    // 往内存中保存一些注册表信息
    datums.put(datum.key, datum);
 
    if (isLeader()) {
        local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
    } else {
        if (local.term.get() + PUBLISH_TERM_INCREASE_COUNT > source.term.get()) {
            //set leader term:
            getLeader().term.set(source.term.get());
            local.term.set(getLeader().term.get());
        } else {
            local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
        }
    }
    raftStore.updateTerm(local.term.get());
 
    // 发布一个ValueChangeEvent事件，PersistentNotifier.onEvent(ValueChangeEvent)会去处理这个事件
    NotifyCenter.publishEvent(ValueChangeEvent.builder().key(datum.key).action(DataOperation.CHANGE).build());
    Loggers.RAFT.info("data added/updated, key={}, term={}", datum.key, local.term);
}
```

Leader保存本节点数据，分为三个过程：

- **1. 保存数据到磁盘文件**
- **2. 保存部分信息到缓存datums**
- **3. 保存文件到内存中的服务注册表（双层ConcurrentHashMap）—— 但不是同步保存，而是通过事件发布，实现异步保存**

需要保存到内存中的服务注册表时，会发布一个ValueChangeEvent事件，该事件会被PersisNotifier.onEvent(ValueChangeEvent)捕捉到，并进行处理：

```java
// com.alibaba.nacos.naming.consistency.persistent.PersistentNotifier#onEvent
public void onEvent(ValueChangeEvent event) {
    notify(event.getKey(), event.getAction(), find.apply(event.getKey()));
}

public <T extends Record> void notify(final String key, final DataOperation action, final T value) {
    ......
 
    for (RecordListener listener : listenerMap.get(key)) {
        try {
            if (action == DataOperation.CHANGE) {
                listener.onChange(key, value); //执行updateIps更新内存注册表
                continue;
            }
            if (action == DataOperation.DELETE) {
                listener.onDelete(key);
            }
        } catch (Throwable e) {
            ......
        }
    }
}

public void onChange(String key, Instances value) throws Exception {    
    ......
    
   // 更新注册表（这个方法，在AP架构师，重点介绍过）
    updateIPs(value.getInstanceList(), KeyBuilder.matchEphemeralInstanceListKey(key));
     
    recalculateChecksum();
}
```

至此，整个Leader节点的持久化节点注册逻辑就完成了！

同步数据给其它Follower节点，就是HTTP调用“raft/datum/commit”接口，处理逻辑比主节点简单！

------

### 三 Nacos CP集群选举过程

####  1 集群节点启动时，会执行RaftCore.init()核心方法 

这个Init()核心方法，会启动2个核心定时任务（每500ms执行一次）：

- **选举任务：new MasterElection()**
- **心跳任务：new HeartBeat()**

```java
@Component
public class RaftCore implements Closeable {
    @PostConstruct
    public void init() throws Exception {
        // 启动CP集群节点
        raftStore.loadDatums(notifier, datums);  //从本地磁盘文件加载数据
     
        // 如果Leader周期不存在，则置为0
        setTerm(NumberUtils.toLong(raftStore.loadMeta().getProperty("term"), 0L));
        initialized = true;
     
        // 每500ms做一次选举任务
        masterTask = GlobalExecutor.registerMasterElection(new MasterElection());
        // 每500ms做一次心跳任务
        heartbeatTask = GlobalExecutor.registerHeartbeat(new HeartBeat());
     
        versionJudgement.registerObserver(isAllNewVersion -> {
            stopWork = isAllNewVersion;
            if (stopWork) {
                try {
                    shutdown();
                    raftListener.removeOldRaftMetadata();
                } catch (NacosException e) {
                    throw new NacosRuntimeException(NacosException.SERVER_ERROR, e);
                }
            }
        }, 100);
     
        // 注册PersistentNotifier监听器，用来监听处理 ValueChangeEvent 事件，保存内存中服务注册表时用的
        NotifyCenter.registerSubscriber(notifier);
    }
 
}
```

####  2 选举任务的核心run()方法 

```java
public class MasterElection implements Runnable {
 
    @Override
    public void run() {
        try {
            RaftPeer local = peers.local();
            local.leaderDueMs -= GlobalExecutor.TICK_PERIOD_MS;
 
            if (local.leaderDueMs > 0) {
                return;
            }
 
            // Raft选举前的随机休眠阶段（15s到20s之间的随机值）
            local.resetLeaderDue();
            // 重新心跳时间为5s
            local.resetHeartbeatDue();
 
            // 率先跳出休眠的节点，发起投票
            sendVote();
        } catch (Exception e) {
             
        }
 
    }
 
    private void sendVote() {
 
        RaftPeer local = peers.get(NetUtils.localServer());
         
        // 重置集群节点投票
        peers.reset();
 
        // 选举周期+1
        local.term.incrementAndGet();
        // 默认投给自己
        local.voteFor = local.ip;
        // 把自己的状态改为 “候选者”
        local.state = RaftPeer.State.CANDIDATE;
 
        Map<String, String> params = new HashMap<>(1);
        params.put("vote", JacksonUtils.toJson(local));
        for (final String server : peers.allServersWithoutMySelf()) {
            // 其它节点的API接口为：/raft/vote
            final String url = buildUrl(server, API_VOTE); 
            try {
                // 向其它节点发出选票
                HttpClient.asyncHttpPost(url, null, params, new Callback<String>() {
                    @Override
                    //其它节点给本节点的响应
                    public void onReceive(RestResult<String> result) { 
                        if (!result.ok()) {
                            Loggers.RAFT.error("NACOS-RAFT vote failed: {}, url: {}", result.getCode(), url);
                            return;
                        }
 
                        RaftPeer peer = JacksonUtils.toObj(result.getData(), RaftPeer.class);
 
 
                        // 判断是否达到半数选票，成为Leader
                        peers.decideLeader(peer);
 
                    }
 
                    @Override
                    public void onError(Throwable throwable) {
                         
                    }
 
                    @Override
                    public void onCancel() {
 
                    }
                });
            } catch (Exception e) {
                 
            }
        }
    }
}
```

####  3 其它节点收到投票后的处理逻辑 

- 如果**收到的候选节点的term小于自己本地节点的term，则voteFor自己**；（我更适合做Leader，这一票我投给自己）
- **否则，**重置自己的election timeout，设置**voteFor为收到的候选节点，更新集群周期term为候选节点的term**；（我同意收到的节点做Leader）

给Http的调用方返回response；

------

### 四 Nacos CP集群的心跳任务

####  1 心跳任务由Leader节点发出，有2个作用 

- **确定Follower节点在线；**
- **帮助Follower节点判断数据是否一致；**（因为服务注册或变更时，Leader节点自己修改了，且收到了“过半”以上节点的ack，但是不排除有些节点没有执行成功，所以通过心跳任务，进行纠错容错）

####  2 心跳任务的核心run()方法(只有Leader节点才可以向其它节点发送心跳包)

```java
public class HeartBeat implements Runnable {
 
    @Override
    public void run() {
        try {
            if (stopWork) {
                return;
            }
            if (!peers.isReady()) {
                return;
            }
 
            RaftPeer local = peers.local();
            // 任务每0.5s执行一次，每次减0.5s，总共5s减完后，就可以开始sendBeat()
            local.heartbeatDueMs -= GlobalExecutor.TICK_PERIOD_MS;
            if (local.heartbeatDueMs > 0) {
                return;
            }
             
            // 重置心跳间隔时间为5s
            local.resetHeartbeatDue();
 
            sendBeat();
        } catch (Exception e) {
            Loggers.RAFT.warn("[RAFT] error while sending beat {}", e);
        }
 
    }
 
    private void sendBeat() throws IOException, InterruptedException {
        RaftPeer local = peers.local();
        // 如果当前是单机模式，或者本节点不是Leader节点，则无权发送心跳，直接跳过
        if (EnvUtil.getStandaloneMode() || local.state != RaftPeer.State.LEADER) {
            return;
        }
 
        local.resetLeaderDue();
 
        // build data
        ObjectNode packet = JacksonUtils.createEmptyJsonNode();
        packet.replace("peer", JacksonUtils.transferToJsonNode(local));
 
        ArrayNode array = JacksonUtils.createEmptyArrayNode();
 
        if (switchDomain.isSendBeatOnly()) {
            Loggers.RAFT.info("[SEND-BEAT-ONLY] {}", switchDomain.isSendBeatOnly());
        }
 
        // 封装心跳包，从内存中获取Leader节点注册表缓存中抽取数据的key和timestamp值
        if (!switchDomain.isSendBeatOnly()) {
            for (Datum datum : datums.values()) {
 
                ObjectNode element = JacksonUtils.createEmptyJsonNode();
 
                if (KeyBuilder.matchServiceMetaKey(datum.key)) {
                    element.put("key", KeyBuilder.briefServiceMetaKey(datum.key));
                } else if (KeyBuilder.matchInstanceListKey(datum.key)) {
                    element.put("key", KeyBuilder.briefInstanceListkey(datum.key));
                }
                element.put("timestamp", datum.timestamp.get());
 
                array.add(element);
            }
        }
 
        packet.replace("datums", array);
        // broadcast
        Map<String, String> params = new HashMap<String, String>(1);
        params.put("beat", JacksonUtils.toJson(packet));
 
        String content = JacksonUtils.toJson(params);
 
        // 对心跳包做gzip压缩
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        GZIPOutputStream gzip = new GZIPOutputStream(out);
        gzip.write(content.getBytes(StandardCharsets.UTF_8));
        gzip.close();
 
        byte[] compressedBytes = out.toByteArray();
        String compressedContent = new String(compressedBytes, StandardCharsets.UTF_8);
 
        // 把压缩后的心跳包，发送给除自己外的其他所有节点
        for (final String server : peers.allServersWithoutMySelf()) {
            try {
                // 心跳包的API为：/raft/beat
                final String url = buildUrl(server, API_BEAT);
 
                HttpClient.asyncHttpPostLarge(url, null, compressedBytes, new Callback<String>() {
                    @Override
                    public void onReceive(RestResult<String> result) {
                        peers.update(JacksonUtils.toObj(result.getData(), RaftPeer.class));
                    }
 
                    @Override
                    public void onError(Throwable throwable) {
 
                    }
 
                    @Override
                    public void onCancel() {
 
                    }
                });
            } catch (Exception e) {
 
            }
        }
 
    }
}
```

####  3 Follower节点收到心跳包后的处理逻辑 

- Leader发出的**心跳包中，包含了数据中的所有key和timestamp**，Follower节点**通过遍历对比，可以排查**自己数据是否为最新最全**数据**；
- 如果**数据不是最新或最全的，则**批量从Leader节点**获取不一致的数据的最新值**；（Leader节点新增或修改的数据）
- 同时要**删除掉自己比Leader多出来的数据**；（Leader节点删除掉的数据）

```java
public RaftPeer receivedBeat(JsonNode beat) throws Exception {
     
    ...从心跳包中解析数据...
     
    // 设置Leader为发送心跳包给我的机器，因为只有Leader才可以发送心跳包
    peers.makeLeader(remote);
 
    if (!switchDomain.isSendBeatOnly()) {
        // receivedKeysMap 的作用是判断出本节点 比 Leader节点多出来的数据（见方法的最后）
        Map<String, Integer> receivedKeysMap = new HashMap<>(datums.size());
 
        for (Map.Entry<String, Datum> entry : datums.entrySet()) {
            // 如果这个Map中的数据为0，则代表是本地自己的数据；
            // 接收到的主节点数据时，把对应的值改为1；
            // 那么直到处理最后，这个Map中还有0，说明这条数据在主节点并没有，只有一种可能，这条数据在主节点中被删除掉了！（妙）
            // 最后，可以把这些数据，在本地清除掉
            receivedKeysMap.put(entry.getKey(), 0);
        }
 
        // batch用来收集本节点没有的数据，或者不是最新的数据
        List<String> batch = new ArrayList<>();
 
        int processedCount = 0; // 已处理的数据条数
 
        for (Object object : beatDatums) {
            processedCount = processedCount + 1;
                
            ......
 
            receivedKeysMap.put(datumKey, 1);
 
            try {
                // 包含，且我自己缓存中这条key对应的数据的时间戳>=收到的心跳中的数据，则代表这条数据我有，就可以跳出本轮
                if (datums.containsKey(datumKey) && datums.get(datumKey).timestamp.get() >= timestamp
                        && processedCount < beatDatums.size()) {
                    continue;
                }
 
                // 取反，不满足上面的条件，则说明这条数据我没有，或者不是最新的，则收集到batch中
                // 因为节点变化的时候，虽然Leader节点收到了半数以上的ack，但是毕竟还有可能有些节点没有收到，或者处理不成功，所以这里通过心跳包进行数据同步的容错处理
                if (!(datums.containsKey(datumKey) && datums.get(datumKey).timestamp.get() >= timestamp)) {
                    batch.add(datumKey);
                }
 
                // 当batch的数据量>=50或者数据已经全部处理完，则就可以继续下面向Leader节点发起批量请求数据的逻辑；
                // 反过来，如果batch<50，且数据还没有处理完，那么这里先跳过，不要向Leader节点发起批量获取数据的请求
                if (batch.size() < 50 && processedCount < beatDatums.size()) {
                    continue;
                }
 
                String keys = StringUtils.join(batch, ",");
 
                // 如果batch为空，当然也不用发请求
                if (batch.size() <= 0) {
                    continue;
                }
 
                // update datum entry
                String url = buildUrl(remote.ip, API_GET);
                Map<String, String> queryParam = new HashMap<>(1);
                queryParam.put("keys", URLEncoder.encode(keys, "UTF-8"));
 
                // 从Leader批量获取本节点缺少的或过时的数据
                HttpClient.asyncHttpGet(url, null, queryParam, new Callback<String>() {
                    @Override
                    public void onReceive(RestResult<String> result) {
                     
                        ...获取缺少的或过时的数据成功后...
 
                        for (JsonNode datumJson : datumList) {
                            Datum newDatum = null;
                            OPERATE_LOCK.lock();
                            try {
 
                                ......
 
                                // 和上面Leader节点新增数据时候逻辑相同，写内存注册表
                                raftStore.write(newDatum);
 
                                datums.put(newDatum.key, newDatum);
                                notifier.notify(newDatum.key, DataOperation.CHANGE, newDatum.value);
 
                                ......
 
                            } catch (Throwable e) {
 
                            } finally {
                                OPERATE_LOCK.unlock();
                            }
                        }
                        return;
                    }
 
                    @Override
                    public void onError(Throwable throwable) {
 
                    }
 
                    @Override
                    public void onCancel() {
 
                    }
 
                });
 
                batch.clear();
 
            } catch (Exception e) {
                 
            }
 
        }
 
        // 如果最后receivedKeysMap中还有value为0的数据，说明这些数据在主节点已经被删除了，那我们从节点也主动删除一下
        List<String> deadKeys = new ArrayList<>();
        for (Map.Entry<String, Integer> entry : receivedKeysMap.entrySet()) {
            if (entry.getValue() == 0) {
                deadKeys.add(entry.getKey());
            }
        }
 
        for (String deadKey : deadKeys) {
            try {
                deleteDatum(deadKey);  //删除本节点多出来的数据逻辑
            } catch (Exception e) {
                 
            }
        }
 
    }
 
    return local;
}
```