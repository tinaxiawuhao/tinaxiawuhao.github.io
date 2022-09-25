---
title: 'Nacos核心源码剖析（AP架构）——注册中心'
date: 2022-07-02 20:23:25
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/zMtbXOeeT.png
isTop: false
---
Nacos官方文档：https://nacos.io/zh-cn/docs/quick-start.html

服务端对外暴露的API：https://nacos.io/zh-cn/docs/open-api.html

Nacos的Server端其实就是一个Web服务，对外提供了Http服务接口，所有的客户端与服务端的通讯都通过Http调用完成（短链接）。

> **Nacos注册服务核心类：NacosNamingService**
>
> **Nacos配置中心核心类：NacosConfigService**



------

### 一 微服务中常用的注册中心对比

- Zookeeper（Apache）：典型的CP架构，有Leader节点，在选举Leader的过程中，整个集群对外不可用，为了强一致性，牺牲高可用性！（Client与Server之间为心跳维持的TCP长连接）
- Eureka（Netflix）：AP架构，为了高可用性，牺牲强一致性；服务提供者新节点注册后，消费者需要一定的时间后才能拿到最新服务列表，最长可达60s；
- Nacos（阿里）：参考了Zookeeper+Eureka，同时支持AP/CP架构，集群默认为AP架构，也可以通过配置切换为CP架构（Raft）；服务列表变动后，消费者获取最新列表最然会有一点延迟，但是比Eureka好很多，而且还可以通过udp实时通知，虽然UDP可靠性无法保证！（Client与Server之间为短链接Http调用）

------

### 二 NACOS的服务架构图

![](https://tianxiawuhao.github.io/post-images/1658665780310.png)

- **服务注册+服务心跳**：首先无论是“服务提供者”还是“服务消费者”都会将自己注册到nacos，并维持心跳。（每5秒发送一次心跳包）
- **服务健康检查**：服务端再启动后，会以Service为单位，开启ClientBeatCheckTask心跳检查任务。（每5秒检查一次，如果某个客户端最后一次心跳超过15秒，标记为不健康，超过30秒踢除）
- **服务发现**：“服务消费者”会根据需要自己的自己所需的目标服务的namespace/group/serviceName/cluster只根据需要查询对应的服务注册表，保存在本地。（定时每10s去服务端更新一次）！
- **服务同步**：服务端集群之间会同步服务注册表，用来保证服务信息的一致性！（注意AP架构的集群中，即使配置了mysql，也不是用来存放注册表）

------

### 三 Nacos的核心注册表结构（双层ConcurrentHashMap）

![](https://tianxiawuhao.github.io/post-images/1658665797704.png)

####  1 Nacos和Eureka的注册表底层都是双层ConcurrentHashMap 

```java
// 本篇只介绍Nacos
public class ServiceManager implements RecordListener<Service> {
    // Nacos服务注册表的实际存储结构（双层Map）
    Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>()
    //Map<nameSpaceId, Map<group::serverName, Service>> ————> 通过nameSpaceId + group::serviceName定位到具体的服务(Service)
         
    //其中的Service服务实例的结构：
    public class Service {
        private Map<String, Cluster> clusterMap = new HashMap<>();
        //Map<clusterName, Cluster> ————> 在具体的serviceInstance内通过clusterName定位到具体的Cluster集群
         
        //而Cluster集群，又是这样的结构
        public class Cluster {
            private Set<Instance> ephemeralInstances = new HashSet<>(); //这就是实际可以对外提供的单个服务（serviceInstanceItem）
        }
    }
}
```

> 总结：Nacos底层数据结构，显示一个双层Map，
> —— 1、服务发现阶段，通过nameSpaceId, group::serviceName找到对应的服务 Service服务
> —— 2、在服务Service内通过clusterName定位到具体的集群Cluster
> —— 3、在Cluster集群里面以HashSet的形式，存放着所有能够提供服务的每个实例Instance（这个Instance中有访问它的详细信息），最后把整个Set列表返回给客户端即可!

####  2 Nacos这么多层的配置，该如何使用？ 

- 最佳实践一（中小型公司）：

```yaml
// namespace：用来区分不通的项目，如haier-iot / haier-code / cold-chain 
// group：用来区分不通项目的 prod / test / dev 等环境
// ————spring.application.name————
// cluster：可以以低于来划分集群：BJ / NJ / SH
 
// 示例：
spring:
  application:
    name: haier-iot-device-manager
  cloud:
    nacos:
      discovery:
        server-addr: 10.206.73.156:8848
        namespace: haier-iot
        group: dev
        cluster-name: BJ  //可以不区分
      config:
        server-addr: 10.206.73.156:8848
        file-extension: yaml
        namespace: haier-iot
        group: dev
        cluster-name: BJ  //可以不区分
```

- 最佳实践二（大型公司）：

```yaml
//与最佳实践一的区别在于，直接使用nacos项目专用，直接使用 namespace 区分环境
// namespace：直接用来区分 prod / test / dev 等环境
// group：使用 DEFAULT_GROUP，因为微服务有可能太多，管理容易混乱；同时这一层可以做扩展，比如多个小服务可能属于另一个大服务下；
// ————spring.application.name————
// cluster：可以以低于来划分集群：BJ / NJ / SH
 
// 示例：
spring:
  application:
    name: haier-iot-device-manager
  cloud:
    nacos:
      discovery:
        server-addr: 10.206.73.156:8848
        namespace: dev 
        group: DEFAULT_GROUP  //默认GROUP可以不指定 
        cluster-name: BJ  //可以不区分
      config:
        server-addr: 10.206.73.156:8848
        namespace: dev
        group: DEFAULT_GROUP  //默认GROUP可以不指定
        file-extension: yaml
```

####  3 为什么Nacos要设计这么复杂的数据结构 

因为Nacos是一个开放的产品，为了适应绝大多数使用者的使用场景，所以扩展性一定要好，这么多层的设计，几乎可以满足任意复杂的业务场景！



------

### 三 Nacos的注册表写入性能保证

####  1 Nacos怎么负责的注册表结构，如何支撑高并发场景？（阻塞队列、异步注册） 

```java
// 使用内存阻塞队列实现异步注册 —— 当接收到provider的注册时，Nacos服务端会将任务封装成Task
public class DistroConsistencyServiceImpl{
    @PostConstruct
    public void init() {
        GlobalExecutor.submitDistroNotifyTask(notifier);
    }
     
    // 而notifier是一个线程，单线程处理服务注册任务，也避免了“并发覆盖”问题！
    public class Notifier implements Runnable {
        private BlockingQueue<Pair<String, DataOperation>> tasks = new ArrayBlockingQueue<>(1024 * 1024);
         
        // run()方法就是在处理放入到tasks队列中的Task任务
        @Override
        public void run() {            
            for (; ; ) {  // 死循环，即使出现异常也不会退出
                try {
                    Pair<String, DataOperation> pair = tasks.take(); //阻塞队列不消耗CPU
                    handle(pair);
                } catch (Throwable e) {
                    Loggers.DISTRO.error("[NACOS-DISTRO] Error while handling notifying task", e);
                }
            }
        }
         
    }
     
     
    // 新的Instance任务被封装成Task任务，放入到Notifier中
    public void put(String key, Record value) throws NacosException {
        onPut(key, value);  // 任务被封装成
        distroProtocol.sync(new DistroKey(key, KeyBuilder.INSTANCE_LIST_KEY_PREFIX), DataOperation.CHANGE,
                globalConfig.getTaskDispatchPeriod() / 2);
    }
     
    public void onPut(String key, Record value) {
        ...边缘逻辑...
        notifier.addTask(key, DataOperation.CHANGE);
    }
 
}
```

####  2 使用阻塞队列实现异步注册，会不会存在不一致问题，还没注册成功就给客户端返回结果？ 

> 是的，肯定会存在这个问题，但是这是一个取舍，高性能的中间件内部都使用了大量的异步操作；
> 想一想，我们的应用程序可能依赖很多第三方服务，如果第三方中间件都用同步的方式去执行自己的内部逻辑，那么应用程序的启动将变得非常地缓慢，最后的效果肯定是难以接受的；
> —— 支持高并发！
> —— 要说不及时，之前的Eureka更严重！
> 其实正常情况下，几乎不会太过阻塞，因为几乎没有多少公司，是一次性增加n多台服务的，都是慢慢添加的，而且即使个别慢了，也是可以接受的，先用其它服务节点即可，站在服务消费者的角度，也就是provider服务起得有点慢而已。

####  3 为了解决高并发下的读写冲突问题，Nacos使用了CopyOnWrite方案 

```java
// 在Notifier.run()方法中：
listener.onChange(datumKey, dataStore.get(datumKey).value);
|
com.alibaba.nacos.naming.core.Service#onChange(){
    updateIPs(value.getInstanceList(), KeyBuilder.matchEphemeralInstanceListKey(key));
}
|
com.alibaba.nacos.naming.core.Service#updateIPs(){
    clusterMap.get(entry.getKey()).updateIps(entryIPs, ephemeral);
}
|
com.alibaba.nacos.naming.core.Cluster#updateIps{
    Set<Instance> toUpdateInstances = ephemeral ? ephemeralInstances : persistentInstances;
 
    // 将旧的临时实例ephemeralInstances列表，复制转化为一个Map进行更新操作
    HashMap<String, Instance> oldIpMap = new HashMap<>(toUpdateInstances.size());
    for (Instance ip : toUpdateInstances) {
        oldIpMap.put(ip.getDatumKey(), ip);
    }
     
    //...对旧Set拷贝后转化为HashMap进行更新操作...
     
    toUpdateInstances = new HashSet<>(ips);
    if (ephemeral) {
        ephemeralInstances = toUpdateInstances;
    } else {
        persistentInstances = toUpdateInstances;
    }
}
```

####  4 同时n多个实例注册或更新，都进行CopyOnWrite，岂不是会存在“更新覆盖”？ 

```java
// 1、首先，根据上面的第1条，Notifier的执行是一个单线程执行任务：
/// Notifier所在类DistroConsistencyServiceImpl是一个单例Service，@PostConstruct决定了init方法只会被调用一次：
//// 而GlobalExecutor 是一个单线程的线程池，所以处理实例注册的最终线程只会有一个
@PostConstruct
public void init() {
    GlobalExecutor.submitDistroNotifyTask(notifier);
}
 
// 2、CopyOnWrite后的集合中的元素不能直接修改，因为集合中的元素是引用！
// —— 当新增时，直接在新集合中新增Instance，然后替换原注册表中的集合即可！
// —— 当删除时，直接将新集合中的对应Instance删除，然后替换原注册表中的集合即可！
// —— 当更新时，新增一个Instance，然后删除原集合中的Instance元素，增加新的Instance元素即可！
// ————原则就是：永远是替换，不直接修改原Instance对象！
```

####  5 随着注册表的不断增大，进行CopyOnWrite时候的成本是不是变得非常大？ 

```java
// 当然不是每次直接Copy整张注册表，那样开销肯定很大
// 每次Copy的粒度是缩小到Service下对应的Cluster中的Set<Instance>集合，这个粒度是很小的！
public class Cluster extends com.alibaba.nacos.api.naming.pojo.Cluster implements Cloneable {
    private Set<Instance> persistentInstances = new HashSet<>();  // AP模式实例列表（服务发现得到的列表就是它）
    private Set<Instance> ephemeralInstances = new HashSet<>();  // CP模式实例列表（服务发现得到的列表就是它）
     
    // 更新节点的操作（Cluster级别）
    public void updateIps(List<Instance> ips, boolean ephemeral) {
        Set<Instance> toUpdateInstances = ephemeral ? ephemeralInstances : persistentInstances;
        HashMap<String, Instance> oldIpMap = new HashMap<>(toUpdateInstances.size());
         
        //...对旧Set拷贝后转化为HashMap进行更新操作...
         
        toUpdateInstances = new HashSet<>(ips);
        if (ephemeral) {
            ephemeralInstances = toUpdateInstances;
        } else {
            persistentInstances = toUpdateInstances;
        }
    }
}
// 为了性能，CopyOnWrite的粒度一定要越小越好！
```

------

### 四 Nacos的心跳机制（定时去调Nacos服务端Http接口）

核心类：NacosNamingService

####  1 Client在向服务端注册服务的同时，开启定时任务向服务端发送心跳请求  

```java
// com.alibaba.nacos.client.naming.NacosNamingService#registerInstance()
// 既是注册服务的核心代码，也是发送心跳的核心代码
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    if (instance.isEphemeral()) {
        BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
        beatReactor.addBeatInfo(groupedServiceName, beatInfo);  // 发送心跳
    }
    serverProxy.registerService(groupedServiceName, groupName, instance); // 注册实例
}
|
public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
    ...
    // 第一次调用 = 触发心跳任务
    executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
    ...
}
|
//BeatTask.run()任务核心代码：
public void run() {
    if (!this.beatInfo.isStopped()) {
        // 计算下一次发送的时间
        long nextTime = this.beatInfo.getPeriod();
 
        try {
            //此处就是去调用“发送心跳API”
            JsonNode result = BeatReactor.this.serverProxy.sendBeat(this.beatInfo, BeatReactor.this.lightBeatEnabled);
            ...对心跳发送结果进行处理...
        } catch (NacosException var11) {
            ...log...
        }
 
        //第二次发送心跳，循环进行，就形成定时发送心跳的效果
        BeatReactor.this.executorService.schedule(BeatReactor.this.new BeatTask(this.beatInfo), nextTime, TimeUnit.MILLISECONDS);
    }
}
```

我们再看看执行“心跳任务”的线程长啥样：

```java
// 定时任务线程
this.executorService = new ScheduledThreadPoolExecutor(threadCount, new ThreadFactory() {
    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r);
        thread.setDaemon(true); // 守护线程（所有用户线程结束后，守护线程会自动结束）
        thread.setName("com.alibaba.nacos.naming.beat.sender");
        return thread;
    }
});
```

> 所以“心跳包”的核心就是：**通过一个定时任务守护线程，定时去调用Nacos服务端的发送心跳包的API接口！**

####  2 “心跳包”的默认间隔时间是多少？（5-15-30） 

```java
// 构建心跳包的信息
BeatInfo beatInfo = this.beatReactor.buildBeatInfo(groupedServiceName, instance);
 
beatInfo.setPeriod(instance.getInstanceHeartBeatInterval());
 
public long getInstanceHeartBeatInterval() {
    return this.getMetaDataByKeyWithDefault("preserved.heart.beat.interval", Constants.DEFAULT_HEART_BEAT_INTERVAL);
}
 
//而Constants.DEFAULT_HEART_BEAT_INTERVAL是个常量
static {
    DEFAULT_HEART_BEAT_TIMEOUT = TimeUnit.SECONDS.toMillis(15L);  //15秒 收不到心跳，则会被标记为“不健康”
    DEFAULT_IP_DELETE_TIMEOUT = TimeUnit.SECONDS.toMillis(30L);   //30秒 收不到心跳，则“剔除”该实例IP
    DEFAULT_HEART_BEAT_INTERVAL = TimeUnit.SECONDS.toMillis(5L);  //默认心跳时间 5秒
```

####  3 Nacos服务端对心跳包的处理逻辑(服务健康检查)？(定时任务，每5秒健康检查)

```java
// 以Service为单位，每个Service在被初始化时，都会创建一个健康检查器HealthCheckReactor
public class Service {
    public void init() {
    // HealthCheckReactor 健康检查器，也是通过schduled线程池去做检查的
    HealthCheckReactor.scheduleCheck(clientBeatCheckTask);
    for (Map.Entry<String, Cluster> entry : clusterMap.entrySet()) {
       entry.getValue().setService(this);
       entry.getValue().init();
    }
   }
}
|
// HealthCheckReactor.scheduleCheck()方法就是开启定时任务线程
|线程池大小为1~核数/2
public static void scheduleCheck(BeatCheckTask task) {
    futureMap.putIfAbsent(task.taskKey(), GlobalExecutor.scheduleNamingHealth(task, 5000, 5000, TimeUnit.MILLISECONDS));
    //延迟5秒后，每5秒执行一次
}
```

健康检查任务的核心 run() 方法逻辑：

```java
public class ClientBeatCheckTask implements BeatCheckTask {
    @Override
    public void run() {
        //拿出Service中所有的实例（后面遍历检查）
        List<Instance> instances = service.allIPs(true);
 
        // 系统当前时间 - 最后一次心跳时间 > 不健康阈值（15秒），标记为不健康
        for (Instance instance : instances) {
            if (System.currentTimeMillis() - instance.getLastBeat() > instance.getInstanceHeartBeatTimeOut()) {
                if (!instance.isMarked()) {
                    if (instance.isHealthy()) {
                        instance.setHealthy(false);
                        Loggers.EVT_LOG
                            .info("{POS} {IP-DISABLED} valid: {}:{}@{}@{}, region: {}, msg: client timeout after {}, last beat: {}",
                                  instance.getIp(), instance.getPort(), instance.getClusterName(),
                                  service.getName(), UtilsAndCommons.LOCALHOST_SITE,
                                  instance.getInstanceHeartBeatTimeOut(), instance.getLastBeat());
                        getPushService().serviceChanged(service);
                    }
                }
            }
        }
 
        if (!getGlobalConfig().isExpireInstance()) {
            return;
        }
 
        // 系统当前时间 - 最后一次心跳时间 > 可剔除阈值（30秒），直接剔除
        for (Instance instance : instances) {
            if (System.currentTimeMillis() - instance.getLastBeat() > instance.getIpDeleteTimeout()) {
                // delete instance
                Loggers.SRV_LOG.info("[AUTO-DELETE-IP] service: {}, ip: {}", service.getName(),
                                     JacksonUtils.toJson(instance));
                deleteIp(instance);
            }
        }
         
    }
}
```

> 所以“服务健康检查”的逻辑就是：**服务端**以Service为单位，**使用**定时任务线程池，每5秒检查一次**Service中所有实例的状态**：
> 
>**最后心跳时间距当前**超过15秒，标记为不健康；
> 
>**最后心跳时间距当前**超过30秒，将此实例踢除！

------

### 五 服务发现

> - 当服务消费者需要查询自己需要的服务列表时，会**优先从本地缓存注册表获取数据，第一次获取为空时，才会从远程Server端获取**；
> - 从远程Server获取服务列表的**粒度为Cluster粒度**，同时还会**将自己的udp端口告诉Server端，便于Server变化时的主动通知**；
> - 从远程Server获取列表的同时，还会**启动定时任务，每隔10秒从Server端同步一次自己的注册表**（只同步自己需要的）；
> - **udp通知的可靠性不能保证**，但是影响不大，因为**有定时任务同步托底**！

####  1 获取服务实例列表核心方法：NacosNamingService#getAllInstances() 

```java
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters, boolean subscribe) {
     
    ServiceInfo serviceInfo;
    if (subscribe) {  // 默认是开启订阅（udp通知），所以走这一分支
        serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),
                StringUtils.join(clusters, ","));
    } else {
        serviceInfo = hostReactor
                .getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName),
                        StringUtils.join(clusters, ","));
    }
    List<Instance> list;
    if (serviceInfo == null || CollectionUtils.isEmpty(list = serviceInfo.getHosts())) {
        return new ArrayList<Instance>();
    }
    return list;
}
|
public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {
    String key = ServiceInfo.getKey(serviceName, clusters);
    if (failoverReactor.isFailoverSwitch()) {
        return failoverReactor.getService(key); // 故障转移功能，从故障转移文件获取服务列表
    }
     
    // 从本地缓存的注册表获取服务列表
    ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);
     
    if (null == serviceObj) { // 第一次启动时候，缓存肯定为空，所以会走这一分支
        serviceObj = new ServiceInfo(serviceName, clusters);
         
        serviceInfoMap.put(serviceObj.getKey(), serviceObj);
         
        updatingMap.put(serviceName, new Object());
        updateServiceNow(serviceName, clusters);  // 核心去远程获取服务列表的方法
        updatingMap.remove(serviceName);
         
    } else if (updatingMap.containsKey(serviceName)) {
         
        if (UPDATE_HOLD_INTERVAL > 0) {
            // hold a moment waiting for update finish
            synchronized (serviceObj) {
                try {
                    serviceObj.wait(UPDATE_HOLD_INTERVAL);
                } catch (InterruptedException e) {
                    NAMING_LOGGER
                            .error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                }
            }
        }
    }
     
    scheduleUpdateIfAbsent(serviceName, clusters);  // 开启定时任务，定时更新本地注册表
     
    return serviceInfoMap.get(serviceObj.getKey());
}
```

从远程获取服务列表没啥看的，我们重点看看定时任务更新本地缓存注册表的逻辑：

```java
public void scheduleUpdateIfAbsent(String serviceName, String clusters) {
    synchronized (futureMap) {
        if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
            return;
        }
         
        // UpdateTask见名知意
        ScheduledFuture<?> future = addTask(new UpdateTask(serviceName, clusters));
        futureMap.put(ServiceInfo.getKey(serviceName, clusters), future);
    }
}
|
// 看看UpdateTask.run()核心方法：
public void run() {
    long delayTime = -1;
     
    try {
         
        ...一系列逻辑，但是最终都会走finally中的逻辑...
        delayTime = serviceObj.getCacheMillis();
         
    } catch (Throwable e) {
        NAMING_LOGGER.warn("[NA] failed to update serviceName: " + serviceName, e);
    } finally {
        if (delayTime > 0) {
            // delayTime默认为10秒
            executor.schedule(this, delayTime, TimeUnit.MILLISECONDS);
        }
    }
     
}
```

> 所以“服务发现”的逻辑就是：在**客户端启动时，会根据需要从Nacos服务端获取自己需要的服务列表（Cluster级别）**，
>
> - 并保存到本地缓存中的注册表中，并开启一个定时任务，**每隔10秒去服务端同步**一下对应的注册表；
> - 之后每次需要时，都是**从本地缓存中的注册表获取服务列表即可**！

####  2 如何尽可能地保证本地注册表的实时性？(开启订阅，开放udp端口) 

从第一条中我们看到一个开启订阅的逻辑，在对应的分支中，从服务端获取服务列表时：

```java
updateServiceNow(serviceName, clusters);
String result = serverProxy.queryList(serviceName, clusters, pushReceiver.getUdpPort(), false);
// pushReceiver.getUdpPort()
// 可以知道，从服务端获取服务列表时，顺便把自己的udp端口也传给了服务端
// 那么当服务端发现对应的服务列表有变动时，就可以通过此Udp端口通知到本Client
```

------

### 六 服务同步

> ​    Nacos集群即使配置了外部mysql数据库，注册表信息也是存储在每个节点的内存中的，而不是存储在mysql中，而当Client向Nacos服务端注册时，只会选择一个Nacos Server节点注册，那么就必须有一套机制能够实现Nacos集群的各个节点都能同步到数据，Nacos自己实现了一套Distro协议，以实现分布式集群各节点之间的数据最终一致性！

####  1 什么时Distro协议？ 

Distro协议时Nacos社区自研的一套AP分布式协议，为了集群的高可用，牺牲强一致性，只追求最终一致性！

- Nacos集群的每个节点时平等的，都可以处理读写请求，同时把数据同步到其他节点；
- 每个节点只负责部分数据（服务健康检查等），定时发送自己负责的数据的校验值到其他节点，以保证数据的一致性；
- 每个节点独立处理请求，不需要经过其他节点同意，及时从本地发起对Client端的相应！

####  2 Nacos集群中的节点，如何知道其它节点的存在？ 

得熟悉Nacos AP集群得部署方式，Nacos集群在部署时，需要在配置 cluster.conf 文件中配置集群得各个节点，这样每台机器就都知道集群中得其它节点得ip:port了;

```java
@Component("serverListManager")
public class ServerListManager extends MemberChangeListener {
 
    @PostConstruct
    public void init() {
        // 集群节点状态同步任务，它会每2秒调用集群其它节点的状态接口，以判断节点是否还在线！
        GlobalExecutor.registerServerStatusReporter(new ServerStatusReporter(), 2000);
        GlobalExecutor.registerServerInfoUpdater(new ServerInfoUpdater());
    }
     
    // 集群节点状态同步任务，每2秒执行一次，
    ServerStatusReporter.run(){
        // 很简单，就是调用其它节点的状态接口，告诉其它机器自己还活着（集群中每两台机器直接都会互相调用）；
        // 如果某个节点在一定时间内，没有收到其它某个节点的状态报告，那就认为这个节点挂了，就会更新自己本地认为的集群存活节点数；
        // 集群存活节点数会直接影响到“服务健康检查”的目标机器核心变量，从而决定每个Service，将会在哪个Server节点被执行健康检查！
        synchronizer.send(server.getAddress(), msg);
    }
}
```

#### 3 “服务注册”任务由哪个节点负责？如何同步数据到其他节点？ 

“服务注册”任务，有Client端发起，根据负载均衡算法挑选一台Server机器进行注册；

被挑选到的Server节点，处理自己的注册任务的同时，通过Distro协议，同步到集群中的其它节点；

```java
// com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl#put
public void put(String key, Record value) throws NacosException {
    // 在本机处理服务注册请求
    onPut(key, value);  
    // 同步给其它机器进行注册
    distroProtocol.sync(new DistroKey(key, KeyBuilder.INSTANCE_LIST_KEY_PREFIX), DataOperation.CHANGE,
            globalConfig.getTaskDispatchPeriod() / 2);
}
```

####  4 如何判断各个Service的健康检查任务，由集群中的哪个节点负责检查？ 

```java
// 我们找到心跳检查任务的run()方法：
ClientBeatCheckTask.run(){
    // 判断是否该由本节点负责该Service的心跳检查任务
    if (!getDistroMapper().responsible(service.getName())) {
        return;
    }
    ...如果是自己负责该Service的心跳检查，才会继续执行心跳检查任务...
}
 
// 判断逻辑
public boolean responsible(String serviceName) {
    final List<String> servers = healthyList;
     
    if (!switchDomain.isDistroEnabled() || EnvUtil.getStandaloneMode()) {
        return true;
    }
     
    if (CollectionUtils.isEmpty(servers)) {
        // means distro config is not ready yet
        return false;
    }
     
    int index = servers.indexOf(EnvUtil.getLocalAddress());
    int lastIndex = servers.lastIndexOf(EnvUtil.getLocalAddress());
    if (lastIndex < 0 || index < 0) {
        return true;  // 自己不在集群列表中，那可能当前就不是集群部署，所以自己得检查
    }
     
    // 对serviceName进行hash后，对当前集群节点数量取余，看看是不是自己
    // 如果不是自己，不用担心，其它机器在被注册时，也会走到这条逻辑，总有一台机器是负责该Service的“健康检查”的
    int target = distroHash(serviceName) % servers.size();
    return target >= index && target <= lastIndex;
}
```

####  5 集群间两个重要的同步任务 

```java
1. ServerListManager下的ServerStatusReporter任务：
—— 上面已经讲过，在集群之间通过定时调用状态接口的方式，同步集群各节点的在线状态！
 
2. ServiceManager下的ServiceReporter任务：
—— 当某个节点执行完健康检查后，如果发现某个Service实例状态改变了，它必须要同步给集群中其它节点，修改各自注册表中的状态（通过调用InstanceController中的API接口实现）
```

####  6 如果有新节点加入集群，如果从其它节点同步数据？ 

```java
// 每个节点启动时，会注入一个DistroProtocol的Bean
@Component
public class DistroProtocol {
    // 在DistroProtocol的构造函数中，会启动DistroTask数据同步任务
    public DistroProtocol(ServerMemberManager memberManager, DistroComponentHolder distroComponentHolder,
        DistroTaskEngineHolder distroTaskEngineHolder, DistroConfig distroConfig) {
        this.memberManager = memberManager;
        this.distroComponentHolder = distroComponentHolder;
        this.distroTaskEngineHolder = distroTaskEngineHolder;
        this.distroConfig = distroConfig;
        startDistroTask();
    }
     
    private void startDistroTask() {
        // 如果时单节点运行，就不用同步啦
        if (EnvUtil.getStandaloneMode()) {
            isInitialized = true;
            return;
        }
        startVerifyTask();
        startLoadTask(); // 开启数据加载任务
    }
     
    private void startLoadTask() {
            DistroCallback loadCallback = new DistroCallback() {
                @Override
                public void onSuccess() {
                    isInitialized = true;
                }
                 
                @Override
                public void onFailed(Throwable throwable) {
                    isInitialized = false;
                }
            };
            GlobalExecutor.submitLoadDataTask(
                    new DistroLoadDataTask(memberManager, distroComponentHolder, distroConfig, loadCallback));
        }
}
 
//DistroLoadDataTask任务的核心run()方法：
DistroLoadDataTask.run(){
    try {
        load(); // 从其它节点加载数据
        if (!checkCompleted()) {
            // 如果不成功，就开个延时任务，过会儿继续尝试去加载
            GlobalExecutor.submitLoadDataTask(this, distroConfig.getLoadDataRetryDelayMillis());
        } else {
            loadCallback.onSuccess();
            Loggers.DISTRO.info("[DISTRO-INIT] load snapshot data success");
        }
    } catch (Exception e) {
        loadCallback.onFailed(e);
        Loggers.DISTRO.error("[DISTRO-INIT] load snapshot data failed. ", e);
    }
}
```

真正的load()从远程加载逻辑：

```java
private void load() throws Exception {
    while (memberManager.allMembersWithoutSelf().isEmpty()) {
        Loggers.DISTRO.info("[DISTRO-INIT] waiting server list init...");
        TimeUnit.SECONDS.sleep(1);
    }
    while (distroComponentHolder.getDataStorageTypes().isEmpty()) {
        Loggers.DISTRO.info("[DISTRO-INIT] waiting distro data storage register...");
        TimeUnit.SECONDS.sleep(1);
    }
    for (String each : distroComponentHolder.getDataStorageTypes()) {
        if (!loadCompletedMap.containsKey(each) || !loadCompletedMap.get(each)) {
            loadCompletedMap.put(each, loadAllDataSnapshotFromRemote(each));
        }
    }
}
 
// for循环尝试从所有远程节点获取注册表全量文件，只要有一个成功，则跳出for循环
private boolean loadAllDataSnapshotFromRemote(String resourceType) {
    DistroTransportAgent transportAgent = distroComponentHolder.findTransportAgent(resourceType);
    DistroDataProcessor dataProcessor = distroComponentHolder.findDataProcessor(resourceType);
    if (null == transportAgent || null == dataProcessor) {
        Loggers.DISTRO.warn("[DISTRO-INIT] Can't find component for type {}, transportAgent: {}, dataProcessor: {}",
                resourceType, transportAgent, dataProcessor);
        return false;
    }
    for (Member each : memberManager.allMembersWithoutSelf()) {
        try {
            // 调取远程节点的获取DatumSnapshot快照数据接口
            DistroData distroData = transportAgent.getDatumSnapshot(each.getAddress());
            // 处理数据，加载到本节点内存的注册表中，完成新节点数据初始化
            boolean result = dataProcessor.processSnapshot(distroData);
 
            if (result) {
                return true;  // 有一个节点成功，则跳出全部for循环，直接返回成功结果
            }
        } catch (Exception e) {
            ......
        }
    }
    return false;
}
```